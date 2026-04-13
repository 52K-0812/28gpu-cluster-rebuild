# 🛡️Grafana ServiceAccount 분리 및 보안 강화

## 🗂️ 1. 작업 개요

> **작업일:** 2026-04-01
> **목적:** Grafana 전용 ServiceAccount 생성 → 최소 권한 원칙(Least Privilege) 적용
> **대상:** master-01, monitoring 네임스페이스
> **작업 배경:** `default` ServiceAccount 사용으로 인한 권한 격리 부족
> **작업 내용:** Grafana 전용 `grafana-sa` ServiceAccount 생성 및 RBAC 재구성
> **최종 결과:** Grafana만 ConfigMap/Secret 접근 권한 보유 ✅

---

## 2. 문제 인식

### 2-1. 기존 구조의 보안 취약점

**Before (4_01 RBAC 권한 부여 직후):**

```yaml
ServiceAccount: monitoring:default  # 네임스페이스 기본 계정

Role: grafana-sidecar-role
  rules:
    - resources: ["configmaps", "secrets"]
      verbs: ["get", "watch", "list"]

RoleBinding: grafana-sidecar-rolebinding
  subjects:
    - kind: ServiceAccount
      name: default  # ← 문제: 모든 Pod가 공유
      namespace: monitoring
```

**문제점:**

- `monitoring` 네임스페이스의 **모든 Pod**가 `default` ServiceAccount 사용
- Grafana뿐 아니라 **향후 배포되는 모든 Pod**도 ConfigMap/Secret 읽기 권한 획득
- 권한 격리 실패 → 의도치 않은 권한 상승 위험

**시나리오 예시:**

```bash
# 테스트용 Pod 배포 시
kubectl run test-pod --image=busybox -n monitoring -- sleep 3600

# 이 Pod도 자동으로 default SA 사용
# → ConfigMap/Secret 전부 읽기 가능 (의도하지 않음)
```

---

## 3. 해결 방안: ServiceAccount 분리

### 3-1. 설계 원칙

**최소 권한 원칙 (Least Privilege):**

- 각 애플리케이션은 **필요한 권한만** 보유
- ServiceAccount를 애플리케이션별로 분리
- 권한 범위를 최소화하여 보안 강화

**적용 후 구조:**

```yaml
# Grafana 전용 ServiceAccount
ServiceAccount: monitoring:grafana-sa

# Grafana만 사용하는 Role
Role: grafana-sidecar-role
  rules:
    - resources: ["configmaps", "secrets"]
      verbs: ["get", "watch", "list"]

# grafana-sa에만 권한 부여
RoleBinding: grafana-sidecar-rolebinding
  subjects:
    - kind: ServiceAccount
      name: grafana-sa  # ← Grafana 전용
      namespace: monitoring

# 기본 SA는 권한 없음
ServiceAccount: monitoring:default
  → 아무 권한 없음 (다른 Pod용)
```

---

## 4. 작업 과정

### 4-1. 사전 확인

```bash
# 1. 네트워크 전체 점검 (스위치 Port Isolation 확인)
for ip in LB_PUBLIC_IP LB_PUBLIC_IP LB_PUBLIC_IP LB_PUBLIC_IP LB_PUBLIC_IP LB_PUBLIC_IP; do
  result=$(ping -c 2 -W 1 $ip > /dev/null 2>&1 && echo "✅" || echo "❌")
  echo "$ip $result"
done

# 출력: 전체 ✅ (네트워크 문제 없음)

# 2. Grafana Pod 상태 확인
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana
# 출력: 2/2 Running ✅

# 3. 기존 RoleBinding 확인
kubectl get rolebinding grafana-sidecar-rolebinding -n monitoring
# 출력: grafana-sidecar-rolebinding 존재 확인 ✅
```

### 4-2. ServiceAccount 생성

```bash
kubectl create serviceaccount grafana-sa -n monitoring
# 출력: serviceaccount/grafana-sa created
```

### 4-3. RoleBinding 대상 변경

기존 RoleBinding의 `subjects`를 `default` → `grafana-sa`로 수정:

```bash
kubectl patch rolebinding grafana-sidecar-rolebinding -n monitoring --type=json -p='[
  {
    "op": "replace",
    "path": "/subjects/0/name",
    "value": "grafana-sa"
  }
]'
# 출력: rolebinding.rbac.authorization.k8s.io/grafana-sidecar-rolebinding patched
```

### 4-4. Grafana Deployment 수정

Grafana Pod가 `grafana-sa`를 사용하도록 설정:

```bash
kubectl patch deployment monitoring-grafana -n monitoring --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/serviceAccountName",
    "value": "grafana-sa"
  }
]'
# 출력: deployment.apps/monitoring-grafana patched
```

**자동 효과:**

- Deployment 수정 → Pod 재생성 트리거
- 새 Pod는 `grafana-sa` ServiceAccount로 기동

---

## 5. 검증

### 5-1. Pod 재시작 확인

```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana
# 출력:
# NAME                                  READY   STATUS    RESTARTS   AGE
# monitoring-grafana-64789cdf88-b8x4j   2/2     Running   0          55s
```

**확인 포인트:**

- AGE가 최근으로 변경 (Pod 재생성됨)
- STATUS: Running
- READY: 2/2 (메인 + 사이드카 컨테이너 정상)

### 5-2. ServiceAccount 적용 확인

```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].spec.serviceAccountName}'
# 출력: grafana-sa ✅
```

### 5-3. 사이드카 로그 확인

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana \
  -c grafana-sc-dashboard --tail=20
```

**핵심 로그:**

```json
{
  "time": "2026-04-01T05:11:46.198084+00:00",
  "level": "INFO",
  "msg": "Initial sync complete, sidecar is ready."
}
```

**확인 사항:**

- ❌ `403 Forbidden` 에러 없음
- ✅ `Initial sync complete` 메시지 출력
- ✅ ConfigMap에서 대시보드 정상 로드

### 5-4. Grafana 웹 UI 확인

**Tailscale 접속 (`http://TAILSCALE_HOST:8000`):**

- 대시보드 목록 전체 정상 표시 ✅
- Node Exporter Full 대시보드 정상 작동 ✅
- 데이터 수집 및 시각화 정상 ✅

---

## 6. 효과 및 개선 사항

### 6-1. 보안 강화

**Before:**

```
monitoring:default (공유 SA)
  ↓
모든 Pod가 ConfigMap/Secret 읽기 가능
```

**After:**

```
monitoring:grafana-sa (Grafana 전용)
  ↓
Grafana만 ConfigMap/Secret 읽기 가능

monitoring:default (다른 Pod용)
  ↓
권한 없음 (최소 권한 원칙)
```

### 6-2. 권한 추적 용이성

**감사(Audit) 시나리오:**

```bash
# "누가 Secret을 읽었는가?" 조회 시
kubectl get events -n monitoring --field-selector involvedObject.kind=Secret

# Before: system:serviceaccount:monitoring:default (애매함)
# After:  system:serviceaccount:monitoring:grafana-sa (명확함)
```

### 6-3. 재사용 가능한 템플릿

이 작업 패턴은 다른 애플리케이션에도 적용 가능:

```yaml
# 예: Prometheus용 SA 분리
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
---
# RoleBinding에서 prometheus-sa 지정
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
```

---

## 7. 핵심 인사이트

### 7-1. Kubernetes RBAC 베스트 프랙티스

- **default SA 사용 금지**: 프로덕션 환경에서 `default` ServiceAccount 직접 사용 지양
- **애플리케이션별 SA 분리**: 각 워크로드마다 전용 ServiceAccount 생성
- **최소 권한 원칙**: 필요한 리소스와 동작(verb)만 허용

### 7-2. 사이드카 패턴에서의 RBAC

**사이드카가 K8s API 접근 시 필수 확인사항:**

1. ServiceAccount 명시적 지정
2. Role/ClusterRole에 필요한 리소스 정의
3. RoleBinding으로 SA와 Role 연결
4. Pod에서 `serviceAccountName` 필드 설정

### 7-3. 보안 설계는 초기 단계에서

**교훈:**

- 초기 구축 시 `default` SA로 빠르게 작동 확인
- 작동 확인 후 **즉시** 전용 SA 분리 작업 진행
- "나중에 하자"는 결국 기술 부채로 축적됨

---

## 8. 관련 문서

- `4_01_Grafana_대시보드_미표시_및_RBAC_권한_문제.md` — RBAC 권한 부여 초기 작업
- `1_RBAC_Namespace_JupyterHub_구축.md` — RBAC 설계 기본 원칙

---

## 9. 빠른 재현 명령어

다른 애플리케이션에도 동일하게 적용 가능:

```bash
# 1. ServiceAccount 생성
kubectl create serviceaccount <app-name>-sa -n <namespace>

# 2. RoleBinding 수정 (기존에 default 사용 중인 경우)
kubectl patch rolebinding <binding-name> -n <namespace> --type=json -p='[
  {
    "op": "replace",
    "path": "/subjects/0/name",
    "value": "<app-name>-sa"
  }
]'

# 3. Deployment에 SA 지정
kubectl patch deployment <deployment-name> -n <namespace> --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/serviceAccountName",
    "value": "<app-name>-sa"
  }
]'

# 4. 검증
kubectl get pod -n <namespace> -l app=<app-name> \
  -o jsonpath='{.items[0].spec.serviceAccountName}'
```

---

## 10. 향후 작업

### 10-1. 다른 시스템 파드 SA 분리 검토

| 파드           | 현재 SA | 권장 SA         | 우선순위              |
| -------------- | ------- | --------------- | --------------------- |
| Prometheus     | default | prometheus-sa   | 중                    |
| Alertmanager   | default | alertmanager-sa | 낮                    |
| Portainer      | default | portainer-sa    | 낮                    |
| JupyterHub Hub | default | jupyterhub-sa   | 높 (다중 사용자 환경) |

### 10-2. ClusterRole 검토

현재는 `Role` (네임스페이스 단위) 사용 중.
만약 여러 네임스페이스의 리소스 접근 필요 시 `ClusterRole` + `ClusterRoleBinding` 전환 필요.

---

## 11. 마무리

**작업 시간:** 3분
**보안 개선:** 권한 격리 완료 ✅
**운영 영향:** 없음 (Pod 재시작 1분, 서비스 정상)

**포트폴리오 포인트:**

> "Kubernetes RBAC 최소 권한 원칙 적용 — ServiceAccount 분리를 통한 권한 격리 및 보안 강화. 기존 default SA 사용 구조를 애플리케이션별 전용 SA 구조로 개선하여 의도하지 않은 권한 상승 위험 제거."

간단한 작업이지만 **보안 의식과 설계 원칙을 이해하고 있다는 강력한 시그널**이야! 👊

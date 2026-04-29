# 🛡️ [트러블슈팅] Grafana 대시보드 미표시 및 RBAC 권한 문제

> **작업 일자:** 2026-04-01
> **대상:** monitoring 네임스페이스, Grafana (kube-prometheus-stack)
> **환경:** Kubernetes v1.29, NFS PVC, Ubuntu 22.04 LTS
> **증상:** Tailscale 접속 시 Grafana 대시보드 목록 전체 비어있음, 403 Forbidden 에러
> **결과:** 물리 네트워크 연결 복구 + RBAC Role/RoleBinding 적용 → 대시보드 전체 로드 완료

---

## 1. 개요

|항목|내용|
|---|---|
|**최초 증상**|Tailscale로 Grafana 접속 시 대시보드 목록 전체 비어있음|
|**의심 원인**|Pod 재시작 시마다 Grafana 설정 초기화 (휘발성 스토리지)|
|**실제 원인**|① 물리 네트워크 단절 (스위치 포트 문제) ② RBAC 권한 부족|
|**최종 상태**|물리 연결 복구 + RBAC Role/RoleBinding 적용 → 대시보드 전체 로드 완료 ✅|

---

## 2. 문제 현상

### 2-1. 증상

- 외부(집)에서 Tailscale 접속 → Grafana 웹 UI는 열림
- 좌측 대시보드 목록 전체 비어있음
- Pod는 `2/2 Running` 상태로 정상
- 이전(3/31)에 설정한 대시보드 전부 소실

### 2-2. 초기 가설

- **휘발성 스토리지 문제**: Grafana가 `/var/lib/grafana`를 PVC 없이 `emptyDir`로 사용 중
- Pod 재시작 시 SQLite DB 초기화 → 설정 날아감

---

## 3. 원인 분석

### 3-1. 1차 진단 — Grafana Deployment 확인

```bash
kubectl get deploy,sts -n monitoring -l app.kubernetes.io/name=grafana
# 출력: deployment.apps/monitoring-grafana  1/1  Running
```

Deployment 형태 → PVC 자동 생성 안 됨 확인.

### 3-2. 2차 진단 — StorageClass 및 PVC 확인

```bash
kubectl get sc
# 출력: nfs-client (default) - NFS Provisioner 정상 동작 중
```

**설정 휘발성 문제 확인:**

- Grafana Deployment의 `storage` 볼륨이 `emptyDir`로 구성
- Pod 재시작 시 `/var/lib/grafana` 내 SQLite DB 초기화
- 수동으로 설정한 데이터소스, 대시보드 전부 소실

### 3-3. 실제 원인 1 — 물리 네트워크 단절

```bash
kubectl get pod -n monitoring -o wide
# Grafana Pod IP: 192.168.235.132 (Calico Pod IP)

ping 192.168.235.132
# 결과: 100% packet loss ❌
```

**진단 결과:**

- Pod는 Running이지만 실제 네트워크 레벨에서 도달 불가
- 스위치 포트 문제로 물리 연결 끊김 의심

**해결:**

- **master-02 스위치 포트 변경** (랜선 교체)
- 변경 후 즉시 ping 정상 응답 ✅

> **연관 문서:** `3_31_네트워크_장애_및_클러스터_설계_개선.md` — 동일한 스위치 Port Isolation 문제로 추정

### 3-4. 실제 원인 2 — RBAC 권한 부족

물리 네트워크 복구 후에도 대시보드 여전히 미표시. 사이드카 컨테이너 로그 확인:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

**핵심 에러:**

```text
ApiException: (403)
Reason: Forbidden
message: "secrets is forbidden: User \"system:serviceaccount:monitoring:default\" 
cannot list resource \"secrets\" in API group \"\" in the namespace \"monitoring\""
```

**원인:**

- Grafana 사이드카(`grafana-sc-dashboard`)가 ConfigMap/Secret 읽기 권한 없음
- `monitoring` 네임스페이스의 `default` ServiceAccount 사용 중
- RBAC Role/RoleBinding 미설정 상태

---

## 4. 해결 과정

### 4-1. PVC 생성 및 Grafana 연결

**설정 영구 보존을 위한 PVC 생성:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
EOF
```

**Deployment에 PVC 연결 (기존 emptyDir 대체):**

```bash
kubectl patch deployment monitoring-grafana -n monitoring --type=json -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/volumes/1",
    "value": {
      "name": "storage",
      "persistentVolumeClaim": {"claimName": "grafana-pvc"}
    }
  }
]'
```

**효과:**

- 이후 Pod 재시작해도 설정 유지 ✅
- `/var/lib/grafana`가 NFS 스토리지에 영구 저장

### 4-2. 물리 연결 복구

1. master-02 물리 확인 → 스위치 랜선 교체
2. 즉시 ping 정상 응답 확인

### 4-3. RBAC 권한 부여

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: monitoring
  name: grafana-sidecar-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: monitoring
  name: grafana-sidecar-rolebinding
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
roleRef:
  kind: Role
  name: grafana-sidecar-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

**적용 내용:**

- `grafana-sidecar-role`: ConfigMap/Secret에 대한 `get`, `watch`, `list` 권한 부여
- `grafana-sidecar-rolebinding`: `monitoring:default` ServiceAccount에 Role 바인딩

### 4-4. Pod 재시작 및 검증

```bash
# 강제 Pod 재시작
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana --force

# 상태 확인
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana
# 결과: 2/2 Running ✅

# 로그 재확인
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
# 결과: 403 Forbidden 에러 사라짐, ConfigMap 정상 로드 메시지 출력
```

### 4-5. 최종 확인

Grafana 웹 UI 접속 → 대시보드 목록 전체 로드 완료 🎉

**로드된 주요 대시보드:**

- NVIDIA DCGM Exporter Dashboard (GPU 모니터링)
- Node Exporter / Nodes (서버 리소스)
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Networking / Cluster

---

## 5. 결과

| 항목 | 결과 |
|---|---|
| Grafana 대시보드 표시 | ✅ 전체 대시보드 정상 로드 |
| RBAC 권한 오류 | ✅ 403 Forbidden 해소 |
| ConfigMap sidecar 로드 | ✅ 정상 동작 확인 |
| Pod 네트워크 연결 | ✅ 물리 연결 복구 후 정상 |

---

## 6. 핵심 인사이트

### 6-1. 진단 순서의 중요성

```text
Pod 상태 확인 (Running) 
    ↓ (정상이면)
Pod 네트워크 연결 확인 (ping Pod IP)
    ↓ (불통이면)
물리 네트워크 점검 (스위치, 랜선)
    ↓ (정상이면)
애플리케이션 로그 확인 (403, 권한 에러)
    ↓
RBAC 권한 확인 및 적용
```

### 6-2. 레이어별 문제 매핑

|레이어|증상|진단 명령어|해결 방법|
|---|---|---|---|
|**L1 물리**|Pod IP ping 불통|`ping [Pod IP]`|랜선/스위치 포트 확인|
|**L7 애플리케이션**|403 Forbidden 에러|`kubectl logs -c [sidecar]`|RBAC 권한 부여|

### 6-3. Grafana 사이드카 아키텍처 이해

- **메인 컨테이너**: Grafana 서버 (UI 제공)
- **사이드카 컨테이너**: ConfigMap에서 대시보드 자동 로드
- 사이드카는 **Kubernetes API 접근 권한 필수** → RBAC 설정 필수

### 6-4. 재발 방지

1. **신규 네임스페이스 생성 시 표준 RBAC 템플릿 적용**
2. **Helm 차트 설치 시 `serviceAccount.create=true` 확인**
3. **사이드카 패턴 사용 시 RBAC 요구사항 사전 검토**

## 7. 빠른 진단 명령어

```bash
# Pod 상태 확인
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o wide

# Pod IP 연결 테스트
ping [Pod IP from above]

# 사이드카 로그 확인 (403 에러 찾기)
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard | grep -i "forbidden"

# 현재 ServiceAccount 권한 확인
kubectl auth can-i list configmaps --as=system:serviceaccount:monitoring:default -n monitoring
kubectl auth can-i list secrets --as=system:serviceaccount:monitoring:default -n monitoring

# RBAC Role/RoleBinding 존재 확인
kubectl get role,rolebinding -n monitoring | grep grafana

# Grafana Pod 강제 재시작
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana --force
```

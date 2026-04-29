# ⚡ [장애] Grafana chart 의도치 않은 업그레이드 — CrashLoopBackOff 및 NodePort 변경

> **발생 일자:** 2026-04-28
> **작업 맥락:** Phase B monitoring PriorityClass 적용 중 `helm upgrade --reuse-values` 실행
> **감지:** Grafana Pod CrashLoopBackOff 확인
> **복구 완료:** 동일 작업 세션 내 복구
> **심각도:** 중 (서비스 영향 있었으나 데이터 유실 없음)
> **관련 문서:** `journal/5.서비스_노출_및_운영_안정화/4_28_PriorityClass_ResourceQuota_Phase_A_B_B1.md`

---

## 1. 개요

| 항목 | 내용 |
|---|---|
| **발생 원인** | `helm upgrade` 시 `--version` 미지정으로 chart 자동 업그레이드 (82.15.1 → 84.1.2) |
| **영향 범위** | Grafana CrashLoopBackOff, NodePort 30300/30310 변경, Grafana PVC 미마운트 |
| **복구 방법** | ConfigMap patch → helm rollback → kubectl patch (NodePort 복원, PVC 재마운트) |
| **재발 방지** | monitoring Helm upgrade 시 `--version 82.15.1` 고정 의무화 |

---

## 2. 증상

`helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring --reuse-values -f monitoring-priority.yaml` 실행 후:

- Grafana 신규 Pod(`monitoring-grafana-5847fc46f4-s7jbt`) CrashLoopBackOff 반복
- 기존 Grafana Pod 잠시 Running 유지 후 종료
- Grafana NodePort 30300 → 30356 변경 (접속 불가)
- Prometheus NodePort 30310 → 30090 변경 (접속 불가)
- Grafana PVC(`grafana-pvc`) 미마운트 — EmptyDir로 기동

---

## 3. 원인 분석

### 3-1. chart 버전 미고정 — 의도치 않은 업그레이드

`--version` 플래그 미사용으로 Helm이 최신 chart를 자동 선택했다.

```text
이전: kube-prometheus-stack-82.15.1 (revision 11)
이후: kube-prometheus-stack-84.1.2  (revision 12)   ← 의도치 않은 업그레이드
```

chart 버전 변경으로 Grafana, Prometheus Service의 NodePort 기본값이 재렌더링됐다. 운영 중 `kubectl patch svc`로 직접 수정한 NodePort 값은 Helm values에 기록되지 않으므로, chart 재렌더링 시 기본값으로 덮어써진다.

### 3-2. datasource.yaml `isDefault: true` 중복

chart 재렌더링 과정에서 기존 수동 추가 Prometheus datasource와 Helm chart 기본 datasource가 동일 ConfigMap에 공존하게 됐다. 두 항목 모두 `isDefault: true`로 설정되어 Grafana 13.0.1이 기동 자체를 거부했다.

```yaml
# ConfigMap 내 datasource.yaml 중복 상태
datasources:
- name: "Prometheus"        # Helm chart 기본 (isDefault: true)
  url: http://...prometheus.monitoring:9090/
  isDefault: true

- name: Prometheus           # 기존 수동 추가분 (isDefault: true)
  url: http://10.97.237.241:9090   # ClusterIP 직접 참조
  isDefault: true
```

```text
Error: datasource.yaml config is invalid.
Only one datasource per organization can be marked as default
```

### 3-3. Grafana PVC 미마운트

rollback 후에도 Grafana Deployment의 `storage` volume이 `persistentVolumeClaim` 대신 `emptyDir`로 렌더링됐다. PVC 자체(`grafana-pvc`)는 삭제되지 않고 Bound 상태였으나, Deployment spec이 이를 참조하지 않았다.

---

## 4. 복구 과정

### 4-1. Grafana CrashLoopBackOff 해결

```bash
# ConfigMap 백업
kubectl get configmap monitoring-kube-prometheus-grafana-datasource \
  -n monitoring -o yaml \
  > ~/backup/monitoring-grafana-datasource-backup-20260428-033849.yaml

# 중복 datasource 제거 (두 번째 Prometheus 항목 삭제)
kubectl patch configmap monitoring-kube-prometheus-grafana-datasource \
  -n monitoring --type merge -p '{...단일 datasource로 교체...}'

# CrashLoop Pod 수동 삭제 (Deployment가 새 Pod 재생성)
kubectl delete pod monitoring-grafana-5847fc46f4-s7jbt -n monitoring
```

### 4-2. helm rollback으로 chart 버전 복원

```bash
helm rollback monitoring 11 -n monitoring
# kube-prometheus-stack-82.15.1 (revision 13, Rollback to 11)
```

### 4-3. Grafana PVC 재마운트

```bash
# rollback 후에도 PVC 미마운트 상태 확인
kubectl get deploy monitoring-grafana -n monitoring -o jsonpath=\
  '{range .spec.template.spec.volumes[*]}{.name}{" => "}{.persistentVolumeClaim.claimName}{"\n"}{end}'
# storage => (빈 값) — EmptyDir 확인

kubectl patch deployment monitoring-grafana -n monitoring \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/1",
         "value":{"name":"storage","persistentVolumeClaim":{"claimName":"grafana-pvc"}}}]'
```

### 4-4. NodePort 복원

```bash
kubectl patch svc monitoring-grafana -n monitoring \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30300}]'

kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30310}]'
```

### 4-5. monitoring PriorityClass 재적용

rollback으로 Phase B에서 적용한 `system-cluster-critical`이 소실됐다. Helm upgrade 재시도 대신 `kubectl patch`로 개별 적용했다 (chart 재렌더링 없이 priority만 삽입).

```bash
for deploy in monitoring-grafana monitoring-kube-prometheus-operator monitoring-kube-state-metrics; do
  kubectl patch deployment ${deploy} -n monitoring \
    --type=merge -p '{"spec":{"template":{"spec":{"priorityClassName":"system-cluster-critical"}}}}'
done

kubectl patch prometheus monitoring-kube-prometheus-prometheus -n monitoring \
  --type=merge -p '{"spec":{"priorityClassName":"system-cluster-critical"}}'

kubectl patch alertmanager monitoring-kube-prometheus-alertmanager -n monitoring \
  --type=merge -p '{"spec":{"priorityClassName":"system-cluster-critical"}}'
```

---

## 5. 복구 결과

| 항목 | 복구 전 | 복구 후 |
|---|---|---|
| Grafana Pod | CrashLoopBackOff | 3/3 Running |
| Grafana NodePort | 30356 (변경됨) | 30300 (복원) |
| Prometheus NodePort | 30090 (변경됨) | 30310 (복원) |
| Grafana PVC | EmptyDir (미마운트) | grafana-pvc (Bound) |
| Grafana HTTP | 접속 불가 | 302 |
| Prometheus HTTP | 접속 불가 | 302 |
| PriorityClass | 소실 | system-cluster-critical |
| 데이터 유실 | — | 없음 |

---

## 6. 재발 방지

### 6-1. Helm upgrade 시 chart 버전 고정 (즉시 적용)

```bash
# 잘못된 방식 (버전 미지정 → 자동 업그레이드 위험)
helm upgrade monitoring prometheus-community/kube-prometheus-stack ...

# 올바른 방식 (버전 고정)
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --version 82.15.1 ...
```

### 6-2. kubectl patch로 수정한 값은 Helm values에도 명시

NodePort, PVC claimName 등 운영 중 수정한 값은 반드시 values 파일에도 반영해둔다. 그렇지 않으면 다음 chart 재렌더링 시 기본값으로 덮어쓰인다.

```yaml
# monitoring-values.yaml에 명시 필요한 항목 예시
grafana:
  service:
    nodePort: 30300
  persistence:
    enabled: true
    existingClaim: grafana-pvc
prometheus:
  service:
    nodePort: 30310
```

### 6-3. Upgrade 전 values + live resource 백업

```bash
helm get values monitoring -n monitoring --all \
  > ~/backup/monitoring-values-all-$(date +%Y%m%d-%H%M).yaml
kubectl get svc monitoring-grafana -n monitoring -o yaml \
  > ~/backup/monitoring-grafana-svc-live-$(date +%Y%m%d-%H%M).yaml
kubectl get deploy monitoring-grafana -n monitoring -o yaml \
  > ~/backup/monitoring-grafana-deploy-live-$(date +%Y%m%d-%H%M).yaml
```

---

## 7. 핵심 인사이트

**`--reuse-values`는 Helm user-supplied values만 재사용하며, kubectl patch로 직접 수정한 NodePort, PVC claimName 등은 보존 보장이 없다.** chart 버전 변경이 함께 일어나면 이 문제가 더 심각해진다. monitoring처럼 운영 중 직접 수정한 리소스가 많은 컴포넌트는 `--version` 고정과 values 명시적 관리가 필수다.

**Grafana datasource `isDefault: true` 중복은 현재 Helm values의 잠재적 위험이다.** `additionalDataSources`와 `sidecar.datasources` 양쪽에 Prometheus datasource가 정의되어 있어 chart 업그레이드 시 재발 가능하다. 어느 한 쪽으로 단일화할 필요가 있다. 단, 이 작업은 반드시 `--version`으로 chart 버전을 고정한 상태에서 수행해야 한다.

# PriorityClass 도입 — Phase A / B / B-1

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-28
> **작업 목적:** ResourceQuota 적용(Phase E) 전 우선순위 체계 정립. 학습 워크로드 폭주 시 시스템 파드가 자원 경합에서 밀려 evict되는 문제 예방.
> **작업 범위:** Phase A (PriorityClass 정의) → Phase B (시스템 파드 보호) → Phase B-1 (MLflow 보강)
> **대상 서버:** master-01
> **작업 환경:** Kubernetes v1.29.15, Helm
> **최종 결과:** Phase A/B/B-1 전 항목 완료. Phase C (서빙/학습 워크로드 priority 적용) 진입 가능 상태.

---

## 🏗️ 2. 작업 흐름

```text
Phase A: PriorityClass 2종 신규 정의 (serving-critical, training-normal)
    ↓
Phase B: 5개 컴포넌트에 system-cluster-critical 적용
    cert-manager → argo-workflows → monitoring → ingress-nginx → jupyterhub
    (monitoring: chart 의도치 않은 업그레이드 → 복구 후 kubectl patch로 재적용)
    (jupyterhub: JSON schema 불일치 → extraPodSpec 방식으로 수정)
    ↓
Phase B 검수 중 MLflow 누락 발견
    ↓
Phase B-1: MLflow postgres/server에 system-cluster-critical 적용
    ↓
전 항목 검증 완료
```

---

## 📐 3. 배경 및 설계 결정

### 3-1. PriorityClass 도입 필요성

기존 클러스터는 모든 Pod의 `priorityClassName`이 미지정 상태였다. ResourceQuota(Phase E) 적용 후 학습 워크로드 폭주 시 ingress-nginx, Prometheus 등 시스템 핵심 파드가 자원 경합에서 밀려 evict될 수 있다. 또한 namespace quota 적용 후 신규 Pod 생성 시 requests/limits 미선언 문제도 발생한다.

이를 해결하기 위해 ResourceQuota 적용 전 우선순위 체계를 먼저 정립한다.

### 3-2. 4계층 우선순위 체계

| PriorityClass | Value | 용도 |
|---|---|---|
| `system-cluster-critical` | 2000000000 | 시스템 핵심 파드 (K8s 내장, 재사용) |
| `serving-critical` | 1000000 | 서빙 워크로드 (yolov8-serving) |
| `training-normal` | 100 | 학습 워크로드 (Argo workflow steps) |
| (default = 0) | 0 | JupyterHub singleuser 등 일반 워크로드 |

기존 `workflow-controller` PriorityClass(value=1000000)는 정의만 있고 실사용 중이 아니었으므로 그대로 유지했다.

### 3-3. monitoring PriorityClass 적용 방식 — Helm upgrade 대신 kubectl patch

Phase B 도중 monitoring에 `helm upgrade --reuse-values`를 시도했다가 chart 버전이 의도치 않게 업그레이드되면서 Grafana CrashLoopBackOff, NodePort 변경, PVC 미마운트가 발생했다(INC-2026-04-28). rollback 후 monitoring PriorityClass가 소실되어, Helm upgrade 재시도 대신 `kubectl patch`로 개별 적용했다.

| 방식 | 장점 | 단점 | 결정 |
|---|---|---|---|
| Helm upgrade (`--reuse-values`) | Helm 상태와 일치 | `--version` 미지정 시 자동 업그레이드 위험 | ❌ (monitoring) |
| kubectl patch | chart 재렌더링 없이 priority만 삽입 | Helm 상태와 불일치 | ⭐ 채택 (monitoring) |

> ⭐ **재발 방지 결정:** 이후 monitoring Helm upgrade 시 반드시 `--version 82.15.1` 플래그로 chart 버전을 고정한다.

### 3-4. JupyterHub priority 적용 — extraPodSpec escape hatch

chart 4.3.3 JSON schema에서 `hub.podPriorityClassName`, `proxy.chp.priorityClassName`, `scheduling.userScheduler.priorityClassName` 키가 허용되지 않는다. Z2JH chart의 `extraPodSpec` escape hatch를 사용하는 방식으로 수정했다.

```yaml
hub:
  extraPodSpec:
    priorityClassName: system-cluster-critical
proxy:
  chp:
    extraPodSpec:
      priorityClassName: system-cluster-critical
scheduling:
  userScheduler:
    extraPodSpec:
      priorityClassName: system-cluster-critical
```

---

## 🔧 4. 실행 절차

### 4-1. Phase A — PriorityClass 정의

```bash
cat > ~/quota-priority/phase-a/priority-classes.yaml <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: serving-critical
value: 1000000
globalDefault: false
description: "서빙 워크로드용 — yolov8-serving"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: training-normal
value: 100
globalDefault: false
description: "학습 워크로드용 — Argo workflow steps"
EOF

kubectl apply -f ~/quota-priority/phase-a/priority-classes.yaml
```

검증:
```text
serving-critical    1000000    false    생성 완료
training-normal     100        false    생성 완료
```

기존 Pod 영향 없음 (PriorityClass 정의만 추가, 워크로드 미변경).

### 4-2. Phase B — 시스템 파드 5개 컴포넌트 적용

각 컴포넌트에 `helm upgrade --reuse-values -f priority-patch.yaml` 패턴으로 적용했다.

**cert-manager**

```bash
cat > ~/quota-priority/phase-b/cert-manager-priority.yaml <<'EOF'
global:
  priorityClassName: system-cluster-critical
EOF

helm upgrade cert-manager jetstack/cert-manager \
  -n cert-manager --reuse-values \
  -f ~/quota-priority/phase-b/cert-manager-priority.yaml
```

**argo-workflows**

```bash
cat > ~/quota-priority/phase-b/argo-workflows-priority.yaml <<'EOF'
controller:
  priorityClassName: system-cluster-critical
server:
  priorityClassName: system-cluster-critical
EOF

helm upgrade argo-workflows argo/argo-workflows \
  -n argo --reuse-values \
  -f ~/quota-priority/phase-b/argo-workflows-priority.yaml
```

**monitoring** (chart 업그레이드 사고 후 rollback → kubectl patch로 대체)

```bash
# Prometheus, Alertmanager — CRD 기반
kubectl patch prometheus monitoring-kube-prometheus-prometheus -n monitoring \
  --type=merge -p '{"spec":{"priorityClassName":"system-cluster-critical"}}'

kubectl patch alertmanager monitoring-kube-prometheus-alertmanager -n monitoring \
  --type=merge -p '{"spec":{"priorityClassName":"system-cluster-critical"}}'

# Deployment 기반
for deploy in monitoring-grafana monitoring-kube-prometheus-operator monitoring-kube-state-metrics; do
  kubectl patch deployment ${deploy} -n monitoring \
    --type=merge -p '{"spec":{"template":{"spec":{"priorityClassName":"system-cluster-critical"}}}}'
done
```

**ingress-nginx**

```bash
cat > ~/quota-priority/phase-b/ingress-nginx-priority.yaml <<'EOF'
controller:
  priorityClassName: system-cluster-critical
EOF

helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --reuse-values \
  -f ~/quota-priority/phase-b/ingress-nginx-priority.yaml
```

**jupyterhub** (extraPodSpec 방식)

```bash
helm upgrade jupyterhub jupyterhub/jupyterhub \
  --version 4.3.3 -n ai-team --reuse-values \
  -f ~/quota-priority/phase-b/jupyterhub-priority.yaml
# revision 13 완료. hub/proxy/user-scheduler 3개 Pod 재시작. singleuser Pod 영향 없음.
```

### 4-3. Phase B-1 — MLflow system-cluster-critical 적용

MLflow는 Helm 비관리 구조(`kubectl` 직접 배포)임을 확인 후 `kubectl patch`로 적용했다.

```bash
helm list -n mlflow   # 출력 없음 → Helm 비관리 확인
```

```bash
# postgres 먼저 (DB 연결 순서 보장)
kubectl patch deployment mlflow-postgres -n mlflow \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"system-cluster-critical"}}}}'
kubectl rollout status deployment/mlflow-postgres -n mlflow

# server 이후
kubectl patch deployment mlflow-server -n mlflow \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"system-cluster-critical"}}}}'
kubectl rollout status deployment/mlflow-server -n mlflow
```

---

## 🔥 5. 트러블슈팅

### 문제 1 — monitoring chart 의도치 않은 업그레이드 → Grafana CrashLoopBackOff

**증상**

`helm upgrade --reuse-values` 실행 시 chart 버전을 명시하지 않아 `kube-prometheus-stack-82.15.1` → `84.1.2`로 자동 업그레이드됐다.

- Grafana 신규 Pod CrashLoopBackOff 반복
- Grafana NodePort 30300 → 30356 변경 (접속 불가)
- Prometheus NodePort 30310 → 30090 변경 (접속 불가)
- Grafana PVC(`grafana-pvc`) 미마운트 — EmptyDir로 기동

**원인**

- chart 재렌더링으로 Service NodePort 기본값 덮어쓰기 (`kubectl patch`로 수정한 값은 Helm values에 미반영)
- datasource ConfigMap에 `isDefault: true` Prometheus datasource 중복 삽입 → Grafana 13.0.1 시작 거부

**조치**

```bash
# ConfigMap 백업 후 중복 datasource 제거
kubectl patch configmap monitoring-kube-prometheus-grafana-datasource \
  -n monitoring --type merge -p '{...단일 datasource로 교체...}'

# chart 버전 복원
helm rollback monitoring 11 -n monitoring

# Grafana PVC 재마운트
kubectl patch deployment monitoring-grafana -n monitoring \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/1",
         "value":{"name":"storage","persistentVolumeClaim":{"claimName":"grafana-pvc"}}}]'

# NodePort 복원
kubectl patch svc monitoring-grafana -n monitoring \
  --type='json' -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30300}]'

kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
  --type='json' -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30310}]'
```

> ⭐ **재발 방지:** monitoring Helm upgrade 시 반드시 `--version 82.15.1` 고정. `kubectl patch`로 수정한 NodePort, PVC claimName은 반드시 values 파일에도 명시해야 한다.

자세한 사고 경위는 `docs/incidents/4_28_grafana_chart_upgrade.md` 참조.

### 문제 2 — JupyterHub chart 4.3.3 JSON schema 불일치

**증상**

```text
Error: UPGRADE FAILED: values don't meet the specifications of the schema(s)
- at '/hub': additional properties 'podPriorityClassName' not allowed
- at '/proxy/chp': additional properties 'priorityClassName' not allowed
- at '/scheduling/userScheduler': additional properties 'priorityClassName' not allowed
```

**원인**

작업 설계 문서의 priority key들이 chart 4.3.3 JSON schema에서 허용되지 않음.

**조치**

Z2JH chart의 `extraPodSpec` escape hatch를 사용하는 방식으로 수정. (3-4 참조)

---

## ✅ 6. 최종 검증 결과

| 컴포넌트 | PriorityClass | 검증 |
|---|---|---|
| cert-manager (3개 Pod) | system-cluster-critical | 인증서 READY=True |
| argo-workflows (2개 Pod) | system-cluster-critical | UI HTTP 200 |
| monitoring 핵심 5개 | system-cluster-critical | Grafana 302, Prometheus 302 |
| ingress-nginx | system-cluster-critical | JupyterHub HTTPS 302 |
| jupyterhub hub/proxy/user-scheduler | system-cluster-critical | Hub health 200 |
| mlflow-postgres | system-cluster-critical | Running |
| mlflow-server | system-cluster-critical | HTTP 200 |

```text
FastAPI /health: champion_ready=true, champion_version=7
JupyterHub singleuser Pod (jupyter-x-52k-0812, jupyter-yeeeho): 영향 없음
```

**의도적 미적용 항목**

- `prometheus-node-exporter` (DaemonSet) — `system-node-critical`이 더 적합, 별도 작업으로 보류
- `filebrowser` — kubectl 직접 배포, 보류
- JupyterHub singleuser — 사용자 워크로드, 시스템 보호 대상 아님

---

## 📋 7. PriorityClass 현황 요약

```text
system-cluster-critical (2000000000)
  └─ cert-manager (3개)
  └─ argo-workflows (2개)
  └─ monitoring: prometheus, alertmanager, grafana, operator, kube-state-metrics
  └─ ingress-nginx (1개)
  └─ jupyterhub: hub, proxy, user-scheduler
  └─ mlflow: postgres, server

serving-critical (1000000)   ← 정의 완료, Phase C에서 yolov8-serving에 적용 예정

training-normal (100)        ← 정의 완료, Phase C에서 Argo workflow step에 적용 예정

(default = 0)
  └─ JupyterHub singleuser Pod
  └─ yolov8-serving (Phase C 적용 전)
  └─ Argo workflow steps (Phase C 적용 전)
```

---

## 💡 8. 핵심 인사이트

**`helm upgrade` 시 `--version` 미지정은 의도치 않은 chart 업그레이드를 유발한다.** 특히 kube-prometheus-stack처럼 chart 재렌더링 시 Service NodePort, PVC claimName 등을 기본값으로 덮어쓰는 컴포넌트에서 치명적이다. `--reuse-values`는 Helm user-supplied values만 재사용하며, `kubectl patch`로 직접 수정한 값은 보존 보장이 없다.

**JupyterHub chart의 values schema는 버전마다 다르다.** chart 4.3.3에서는 `hub.podPriorityClassName` 등의 키가 허용되지 않는다. `extraPodSpec`을 escape hatch로 활용하면 schema 검증을 통과하면서 원하는 spec을 주입할 수 있다.

**MLflow가 Helm 비관리 구조인 경우 `helm list`로 먼저 확인 후 `kubectl patch`로 직접 적용한다.** 비관리 구조에서 Helm upgrade를 시도하면 release를 찾지 못해 에러가 발생한다.

---

## 🔮 9. 다음 작업

| Phase | 작업 | 비고 |
|---|---|---|
| C | yolov8-serving → `serving-critical` 적용 | 서빙 Pod 1회 재시작 |
| C | Argo WorkflowTemplate → `training-normal` 적용 | 신규 workflow step부터 적용 |
| D | LimitRange 적용 (ai-team) | default request/limit 자동 주입 |
| E | ResourceQuota 적용 (ai-team, GPU 16장 상한) | 상한 초과 시 신규 Pod 차단 |

---

## 📎 10. 관련 문서

- `docs/incidents/4_28_grafana_chart_upgrade.md` — monitoring chart 업그레이드 사고 상세 경위
- `docs/overview/current-architecture.md` — PriorityClass 계층 현황 (Section 12)

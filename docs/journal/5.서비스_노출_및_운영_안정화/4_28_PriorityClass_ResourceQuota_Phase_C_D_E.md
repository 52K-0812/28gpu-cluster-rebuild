# PriorityClass 적용 Phase C + LimitRange Phase D + ResourceQuota Phase E

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-28
> **작업 목적:** ai-team 네임스페이스 자원 거버넌스 체계 완성. 서빙/학습 워크로드에 PriorityClass 연결(C), 컨테이너 기본 request/limit 자동 주입(D), 네임스페이스 단위 GPU 상한 설정(E).
> **작업 범위:** Phase C (워크로드 PriorityClass 적용) → Phase D (LimitRange) → Phase E (ResourceQuota)
> **대상 서버:** master-01
> **작업 환경:** Kubernetes v1.29.15, ai-team 네임스페이스
> **선행 작업:** Phase A/B/B-1 완료 (PriorityClass 4계층 정의 + 시스템 파드 보호)
> **최종 결과:** 3개 Phase 전 항목 완료. ai-team 네임스페이스 자원 거버넌스 체계 확립.

---

## 🏗️ 2. 작업 흐름

```
Phase C: 서빙/학습 워크로드에 PriorityClass 연결
    yolov8-serving Deployment → serving-critical
    yolov8-dag-pipeline WorkflowTemplate → training-normal
    yolov8-visdrone-train WorkflowTemplate → training-normal
    테스트 Workflow 실행 → 신규 Pod에 training-normal 반영 확인
        ↓
Phase D: ai-team LimitRange 적용
    ai-team-default-compute 생성
    smoke test (auto-injection), min/max 위반, 정상값 dry-run 전 통과
        ↓
Phase E: ai-team ResourceQuota 적용
    ai-team-compute-quota 생성 (GPU 16장 상한 포함)
    GPU 17장 초과 차단 확인, LimitRange 기본 주입 정상 유지 확인
        ↓
전 항목 검증 완료 — 기존 운영 Pod 영향 없음
```

---

## 📐 3. 배경 및 설계 결정

### 3-1. Phase C — 워크로드 PriorityClass 연결 필요성

Phase A/B/B-1에서 시스템 파드에 `system-cluster-critical`을 적용했으나, 서빙 워크로드(`yolov8-serving`)와 학습 워크로드(Argo WorkflowTemplate)는 여전히 priority 미설정 상태였다. ResourceQuota(Phase E) 적용 전에 워크로드별 우선순위를 먼저 확정해야 자원 경합 시 스케줄러가 올바른 순서로 처리한다.

### 3-2. Phase C — Argo WorkflowTemplate 필드명 교정

당초 계획에서는 `spec.priorityClassName`으로 patch를 시도했으나 Argo CRD가 해당 필드를 인식하지 못했다.

```
Warning: unknown field "spec.priorityClassName"
workflowtemplate.argoproj.io/yolov8-dag-pipeline patched (no change)
```

`kubectl explain workflowtemplate.spec.podPriorityClassName` 으로 확인한 결과 Argo WorkflowTemplate의 올바른 필드는 `spec.podPriorityClassName`이다. 이 필드는 WorkflowTemplate에서 생성되는 모든 workflow Pod에 PriorityClass를 적용한다.

| 시도한 필드 | 결과 | 채택 |
|---|---|---|
| `spec.priorityClassName` | unknown field — no change | ❌ |
| `spec.podPriorityClassName` | 정상 적용 | ⭐ |

### 3-3. Phase D — LimitRange 설계 근거

ResourceQuota(Phase E)는 네임스페이스 단위 총량만 제한한다. request/limit을 명시하지 않은 Pod가 존재하면 ResourceQuota가 있어도 해당 Pod는 무한 CPU/Memory를 점유할 수 있다. LimitRange를 먼저 적용해 모든 신규 Pod에 기본값을 자동 주입하고, min/max로 선언 범위를 강제한다.

| 항목 | CPU | Memory |
|---|---|---|
| defaultRequest | 500m | 1Gi |
| default (limit) | 4 | 16Gi |
| min | 100m | 256Mi |
| max | 16 | 64Gi |
| maxLimitRequestRatio | 8 | 16 |

### 3-4. Phase E — ResourceQuota 설계 근거

ai-team 네임스페이스에 팀원 학습 워크로드와 서빙 워크로드가 혼재한다. GPU는 클러스터 전체에서 가장 희소한 자원으로, 한 워크로드가 전량을 점유하는 상황을 막기 위해 네임스페이스 단위 상한을 설정한다.

| 항목 | Hard 값 | 설계 근거 |
|---|---|---|
| requests.nvidia.com/gpu | 16 | 전체 27장 중 ai-team 전용 상한. 시스템/서빙 여유분 확보 |
| pods | 80 | 현재 운영 Pod(~12) 대비 여유 확보 |
| requests.cpu | 80 | 노드 합계 대비 여유 있는 상한 |
| requests.memory | 256Gi | 노드 합계 대비 여유 있는 상한 |
| limits.cpu | 256 | LimitRange maxLimitRequestRatio(8) × requests.cpu(80) 이상 |
| limits.memory | 768Gi | LimitRange maxLimitRequestRatio(16) × requests.memory(256/3 기준) |

---

## 🔧 4. 실행 절차

### 4-1. Phase C — yolov8-serving serving-critical 적용

```bash
# 사전 확인
echo "=== PriorityClass 존재 확인 ===" && kubectl get priorityclass serving-critical training-normal
echo "=== yolov8-serving 현재 상태 ===" && kubectl get pods -n ai-team -l app=yolov8-serving \
  -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName'
echo "=== FastAPI health ===" && curl -s http://<MASTER-IP>:30600/health | python3 -m json.tool

# serving-critical 적용
kubectl patch deployment yolov8-serving -n ai-team \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"serving-critical"}}}}'

# rollout 확인
kubectl rollout status deployment/yolov8-serving -n ai-team

# 적용 확인
kubectl get pods -n ai-team -l app=yolov8-serving \
  -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName,STATUS:.status.phase,NODE:.spec.nodeName'
```

검증 결과:
```
yolov8-serving-7dcd96c8bf-5jfb4   serving-critical   Running   2080ti-gpu-03
FastAPI HTTP /health: champion_ready=true, champion_version=7
FastAPI HTTPS /health: champion_ready=true, champion_version=7
```

### 4-2. Phase C — WorkflowTemplate training-normal 적용

```bash
# 백업
kubectl get workflowtemplate yolov8-dag-pipeline -n ai-team -o yaml \
  > ~/quota-priority/phase-c/yolov8-dag-pipeline-before-priority.yaml

kubectl get workflowtemplate yolov8-visdrone-train -n ai-team -o yaml \
  > ~/quota-priority/phase-c/yolov8-visdrone-train-before-priority.yaml

# spec.priorityClassName 시도 → unknown field (no change) — 필드명 교정 필요
kubectl patch workflowtemplate yolov8-dag-pipeline -n ai-team \
  --type=merge \
  -p '{"spec":{"priorityClassName":"training-normal"}}'
# Warning: unknown field "spec.priorityClassName"  → spec.podPriorityClassName으로 전환

# kubectl explain으로 올바른 필드명 확인
kubectl explain workflowtemplate.spec.podPriorityClassName
# FIELD: podPriorityClassName <string>
# DESCRIPTION: PriorityClassName to apply to workflow pods.

# 올바른 필드로 적용
kubectl patch workflowtemplate yolov8-dag-pipeline -n ai-team \
  --type=merge \
  -p '{"spec":{"podPriorityClassName":"training-normal"}}'

kubectl patch workflowtemplate yolov8-visdrone-train -n ai-team \
  --type=merge \
  -p '{"spec":{"podPriorityClassName":"training-normal"}}'

# 적용 확인
kubectl get workflowtemplate -n ai-team \
  -o custom-columns='NAME:.metadata.name,POD_PRIORITY:.spec.podPriorityClassName'
```

확인 결과:
```
NAME                    POD_PRIORITY
yolov8-dag-pipeline     training-normal
yolov8-visdrone-train   training-normal
```

### 4-3. Phase C — 테스트 Workflow 실행하여 신규 Pod 검증

```bash
# argo CLI 미설치 → kubectl create -f 방식으로 대체
kubectl create -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: test-priority-
  namespace: ai-team
spec:
  workflowTemplateRef:
    name: yolov8-dag-pipeline
  arguments:
    parameters:
    - name: epochs
      value: "1"
    - name: batch-size
      value: "16"
    - name: model-version
      value: "test-priority"
EOF
# workflow.argoproj.io/test-priority-k56tp created

# 생성된 Pod priority 확인
kubectl get pods -n ai-team \
  -l workflows.argoproj.io/workflow=test-priority-k56tp \
  -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName,STATUS:.status.phase'
```

### 4-4. Phase D — LimitRange 적용

```bash
mkdir -p ~/quota-priority/phase-d

# manifest 작성
cat > ~/quota-priority/phase-d/limitrange-ai-team.yaml <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: ai-team-default-compute
  namespace: ai-team
  labels:
    phase: phase-d
    component: resource-governance
    managed-by: kubectl
  annotations:
    description: "Default CPU/Memory requests and limits for ai-team namespace before ResourceQuota phase."
    phase: "Phase D - LimitRange"
    next-phase: "Phase E - ResourceQuota"
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: "500m"
        memory: "1Gi"
      default:
        cpu: "4"
        memory: "16Gi"
      min:
        cpu: "100m"
        memory: "256Mi"
      max:
        cpu: "16"
        memory: "64Gi"
      maxLimitRequestRatio:
        cpu: "8"
        memory: "16"
EOF

# server dry-run 검증
kubectl apply --dry-run=server -f ~/quota-priority/phase-d/limitrange-ai-team.yaml
# limitrange/ai-team-default-compute created (server dry run)

# 실제 적용
kubectl apply -f ~/quota-priority/phase-d/limitrange-ai-team.yaml
# limitrange/ai-team-default-compute created

# 적용 확인
kubectl describe limitrange ai-team-default-compute -n ai-team
```

확인 결과:
```
Type       Resource  Min    Max   Default Request  Default Limit  Max Limit/Request Ratio
Container  cpu       100m   16    500m             4              8
Container  memory    256Mi  64Gi  1Gi              16Gi           16
```

smoke test (request/limit 미선언 Pod의 자동 주입 확인):
```bash
kubectl run lr-smoke-default -n ai-team --image=busybox:1.36 --restart=Never --command -- sleep 60
kubectl get pod lr-smoke-default -n ai-team \
  -o jsonpath='{.spec.containers[0].resources}{"\n"}'
# {"limits":{"cpu":"4","memory":"16Gi"},"requests":{"cpu":"500m","memory":"1Gi"}}
kubectl delete pod lr-smoke-default -n ai-team --ignore-not-found=true
```

min/max 위반 dry-run 확인:
- cpu request=50m → `minimum cpu usage per Container is 100m` Forbidden ✅
- cpu limit=32, memory limit=128Gi → `maximum cpu usage per Container is 16` Forbidden ✅
- 정상 명시값(cpu req=1, limit=4 / mem req=2Gi, limit=8Gi) → server dry-run created ✅

### 4-5. Phase E — ResourceQuota 적용

```bash
mkdir -p ~/quota-priority/phase-e/manifests ~/quota-priority/phase-e/result

# manifest 작성
cat > manifests/ai-team-resourcequota.yaml <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ai-team-compute-quota
  namespace: ai-team
  labels:
    component: resource-governance
    managed-by: kubectl
    phase: phase-e
  annotations:
    phase: "Phase E - ResourceQuota"
    description: "Namespace-level CPU/Memory/GPU quota for ai-team after LimitRange phase."
spec:
  hard:
    requests.cpu: "80"
    requests.memory: "256Gi"
    limits.cpu: "256"
    limits.memory: "768Gi"
    requests.nvidia.com/gpu: "16"
    pods: "80"
EOF

# server dry-run 검증
kubectl apply --dry-run=server -f manifests/ai-team-resourcequota.yaml
# resourcequota/ai-team-compute-quota created (server dry run)

# 실제 적용
kubectl apply -f manifests/ai-team-resourcequota.yaml
# resourcequota/ai-team-compute-quota created

# 적용 직후 상태
kubectl describe resourcequota ai-team-compute-quota -n ai-team
```

적용 직후 상태:
```
Resource                  Used   Hard
--------                  ----   ----
limits.cpu                4      256
limits.memory             16Gi   768Gi
pods                      12     80
requests.cpu              1      80
requests.memory           4Gi    256Gi
requests.nvidia.com/gpu   2      16
```

GPU quota 검증:
```bash
# GPU 17장 초과 요청 → 차단 확인
kubectl apply --dry-run=server -f gpu-exceed-17.yaml
# Error: exceeded quota: ai-team-compute-quota, requested: requests.nvidia.com/gpu=17, used: 2, limited: 16

# GPU 1장 요청 → 허용 확인
kubectl apply --dry-run=server -f gpu-allowed-1.yaml
# pod/rq-gpu-allowed-test created (server dry run)
```

---

## 🔥 5. 트러블슈팅

### 문제 — Argo WorkflowTemplate `spec.priorityClassName` 미인식

**증상**

```
Warning: unknown field "spec.priorityClassName"
workflowtemplate.argoproj.io/yolov8-dag-pipeline patched (no change)
```

**원인**

Argo WorkflowTemplate CRD(v1alpha1)는 `spec.priorityClassName` 필드를 정의하지 않는다. Kubernetes 네이티브 Deployment의 `spec.template.spec.priorityClassName`과 혼동한 것.

**조치**

`kubectl explain workflowtemplate.spec.podPriorityClassName`으로 올바른 필드 확인 후 `spec.podPriorityClassName`으로 전환.

> ⭐ **핵심:** Argo WorkflowTemplate에서 workflow Pod에 PriorityClass를 적용하는 필드는 `spec.podPriorityClassName`이다. K8s 네이티브 Deployment의 `spec.template.spec.priorityClassName`과 다르다.

---

## ✅ 6. 최종 검증 결과

| 항목 | 결과 |
|---|---|
| yolov8-serving priorityClassName | serving-critical ✅ |
| yolov8-dag-pipeline podPriorityClassName | training-normal ✅ |
| yolov8-visdrone-train podPriorityClassName | training-normal ✅ |
| 테스트 Workflow Pod priority | training-normal 반영 ✅ |
| FastAPI /health (HTTP + HTTPS) | champion_ready=true, champion_version=7 ✅ |
| LimitRange ai-team-default-compute | 적용 완료 ✅ |
| LimitRange 기본 주입 (smoke test) | cpu=500m/4, memory=1Gi/16Gi 자동 주입 ✅ |
| LimitRange min 위반 차단 | Forbidden 정상 ✅ |
| LimitRange max 위반 차단 | Forbidden 정상 ✅ |
| ResourceQuota ai-team-compute-quota | 적용 완료 ✅ |
| GPU 17장 초과 요청 차단 | exceeded quota 정상 ✅ |
| GPU 1장 요청 허용 | server dry-run created ✅ |
| LimitRange + ResourceQuota 동시 동작 | LimitRanger 주입 annotation 확인 ✅ |
| 기존 운영 Pod 영향 | 없음 (hub/proxy/yolov8-serving/jupyter 모두 Running 유지) ✅ |

---

## 📋 7. LimitRange / ResourceQuota 현황 요약

### LimitRange (ai-team-default-compute)

```
Type       Resource  Min    Max   Default Request  Default Limit  Max Limit/Request Ratio
Container  cpu       100m   16    500m             4              8
Container  memory    256Mi  64Gi  1Gi              16Gi           16
```

### ResourceQuota (ai-team-compute-quota) — 2026-04-28 기준

```
Resource                  Used   Hard
--------                  ----   ----
requests.cpu              1      80
requests.memory           4Gi    256Gi
limits.cpu                4      256
limits.memory             16Gi   768Gi
requests.nvidia.com/gpu   2      16
pods                      12     80
```

---

## 💡 8. 핵심 인사이트

**Argo WorkflowTemplate priority 필드는 `spec.podPriorityClassName`이다.** `kubectl explain`으로 먼저 필드 존재 여부를 확인한 뒤 patch하는 것이 원칙이다. `unknown field` 경고와 `(no change)` 메시지가 동시에 나오면 필드명이 잘못된 것이다.

**LimitRange는 admission 시점에만 작용하며 기존 Pod에 소급 적용되지 않는다.** 적용 후 기존 Pod 재시작 없이 신규 Pod부터만 기본값이 주입된다. smoke test로 신규 Pod의 실제 주입값을 반드시 확인해야 한다.

**ResourceQuota와 LimitRange는 독립적으로 동작하지만 함께 있어야 완전하다.** LimitRange 없이 ResourceQuota만 있으면 requests/limits 미선언 Pod가 admission에서 거부된다(`must specify resource requirements`). LimitRange가 먼저 기본값을 주입하므로 Phase D → Phase E 순서가 중요하다.

**server dry-run은 API 서버 admission 단계까지 포함한 검증이다.** manifest 문법 오류뿐 아니라 LimitRange 위반, ResourceQuota 초과도 dry-run 단계에서 미리 확인된다.

---

## 🔮 9. 다음 작업

| Phase | 작업 | 비고 |
|---|---|---|
| Ingress B | Grafana · Argo · MLflow · Filebrowser HTTPS + Basic Auth | 관리자 서비스 인증 강화 |
| — | YOLOv8 Serving Basic Auth 또는 API Key 적용 | 현재 인증 없음 |
| — | node-exporter PriorityClass system-node-critical 적용 | DaemonSet 보류 항목 |
| — | Let's Encrypt DNS-01 전환 | 본인 도메인 확보 후 |

---

## 📎 10. 관련 문서

- `docs/journal/5.서비스_노출_및_운영_안정화/4_28_PriorityClass_ResourceQuota_Phase_A_B_B1.md` — Phase A/B/B-1 상세 (선행 작업)
- `docs/incidents/4_28_grafana_chart_upgrade.md` — Phase B 도중 발생한 monitoring chart 사고
- `docs/overview/current-architecture.md` — Section 12(PriorityClass) / Section 13(LimitRange) / Section 14(ResourceQuota)

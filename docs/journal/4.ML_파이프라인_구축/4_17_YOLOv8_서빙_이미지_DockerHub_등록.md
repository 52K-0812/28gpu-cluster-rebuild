# YOLOv8 서빙 이미지 DockerHub 등록 및 nodeSelector 완화

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-17
> **작업 목적:** `yolov8-serving:local-v1` 이미지를 DockerHub에 push하여 단일 노드 로컬 의존성 제거. nodeSelector `kubernetes.io/hostname` 고정 해제로 2080Ti 풀 전체에서 스케줄 가능한 구조로 전환.
> **대상 서버:** master-01
> **작업 환경:** Kubernetes v1.29, ai-team 네임스페이스, nerdctl v1.7.6, DockerHub (`1jkim`)
> **선행 작업:** 4_16 서빙 이미지화 완료 (local-v1 빌드, 2080ti-gpu-04 로컬 load, Deployment 교체)
> **최종 결과:** DockerHub push 완료, Deployment 이미지 교체, nodeSelector 완화, 전 엔드포인트 정상 확인.

---

## 🏗️ 2. 작업 흐름

```
[master-01] nerdctl login → nerdctl tag → nerdctl push
    ↓
DockerHub 1jkim/yolov8-serving:v1 등록
    ↓
Deployment 백업 → image 교체 → rollout
    ↓
검증 (/health, /predict)
    ↓
nodeSelector kubernetes.io/hostname 제거 → rollout
    ↓
최종 검증
```

---

## 📐 3. 아키텍처 설계 결정

### 3-1. 왜 DockerHub인가 (로컬 containerd 대신)

| 항목 | 로컬 containerd | DockerHub public |
|---|---|---|
| 노드 장애 시 복구 | 재load 필요 | 자동 pull |
| 스케줄 범위 | hostname 고정 필수 | gpu-type만으로 충분 |
| 신규 노드 추가 | 수동 이미지 전달 필요 | 자동 |
| 비용 | 없음 | public 무료 |

> ⭐ **설계 결정:** public 레지스트리 사용으로 로컬 이미지 의존성 완전 제거. hostname nodeSelector 즉시 완화. Private 레지스트리 사용 시 imagePullSecret 설정 필요 — 현재는 Public 유지.

---

## 🔧 4. 실행 절차

### 4-1. nerdctl 인증 설정

```bash
# 1차 시도 — 실패
sudo nerdctl login docker.io
# FATA: expected acArg to be "docker.io", got "registry-1.docker.io"

# 2차 시도 — 엔드포인트 명시
sudo nerdctl login https://index.docker.io/v1
# Login Succeeded
```

> ⭐ **핵심:** `nerdctl login docker.io`는 호스트 문자열 불일치로 실패. `https://index.docker.io/v1`을 명시해야 한다.

### 4-2. config.json 키 교정

로그인은 성공했으나 push 시 `insufficient_scope: authorization failed` 반복.
원인: `/root/.docker/config.json`의 auths 키가 `index.docker.io`로 저장되어 push scope 불일치.

```bash
sudo cp /root/.docker/config.json /root/.docker/config.json.bak
sudo sed -i 's#"index.docker.io"#"https://index.docker.io/v1/"#' /root/.docker/config.json
sudo cat /root/.docker/config.json
```

교정 후 config.json:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "..."
    }
  }
}
```

> ⭐ **핵심:** nerdctl은 push 시 `https://index.docker.io/v1/` 키로 인증 정보를 조회한다. 로그인 성공과 push 권한은 별개로 동작하므로 config.json 키 형식을 직접 확인해야 한다.

### 4-3. DockerHub 저장소 생성

push 이전에 DockerHub 웹에서 `1jkim/yolov8-serving` 저장소를 Public으로 생성해야 한다.
저장소가 없으면 동일한 `insufficient_scope` 에러가 발생한다.

> ⚠️ **주의:** Private 저장소로 생성하면 K8s에서 imagePullSecret 설정이 필요해진다. 반드시 Public으로 생성할 것.

### 4-4. 이미지 태그 및 push

```bash
sudo nerdctl tag yolov8-serving:local-v1 1jkim/yolov8-serving:v1
sudo nerdctl images | grep yolov8-serving
# yolov8-serving          local-v1   bef57c5a53a9   7 hours ago   13.7 GiB
# 1jkim/yolov8-serving    v1         bef57c5a53a9   just now      13.7 GiB

sudo nerdctl push 1jkim/yolov8-serving:v1
# elapsed: 203.1s  total: 13.7K  → push 완료
```

### 4-5. Deployment 백업 및 이미지 교체

```bash
mkdir -p ~/backup/yolov8-serving
kubectl get deploy yolov8-serving -n ai-team -o yaml \
  > ~/backup/yolov8-serving/deploy-before-dockerhub-image.yaml

kubectl set image deployment/yolov8-serving \
  serving=1jkim/yolov8-serving:v1 -n ai-team
kubectl rollout status deployment/yolov8-serving -n ai-team
# deployment "yolov8-serving" successfully rolled out
```

### 4-6. 중간 검증 (nodeSelector 완화 전)

```bash
kubectl get pods -n ai-team -l app=yolov8-serving -o wide
# NAME                              READY   STATUS    NODE
# yolov8-serving-5f9d548c6b-6576d   1/1     Running   2080ti-gpu-04

kubectl get pod -n ai-team -l app=yolov8-serving \
  -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
# 1jkim/yolov8-serving:v1

curl -s http://WORKER-NODE-IP:SERVING-NODEPORT/health
# {"status":"ok","demo_model":"yolov8n-coco (80 classes)","champion_ready":true,"champion_version":"5"}

curl -s -X POST http://WORKER-NODE-IP:SERVING-NODEPORT/predict \
  -F "file=@/home/ubuntu/actions-runner/.../basic.png" \
  | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get("model"), d.get("count"))'
# visdrone-yolov8@champion (v5) 0
```

### 4-7. nodeSelector hostname 고정 제거

```bash
kubectl patch deployment yolov8-serving -n ai-team --type='json' -p='[
  {"op":"remove","path":"/spec/template/spec/nodeSelector/kubernetes.io~1hostname"}
]'
kubectl rollout status deployment/yolov8-serving -n ai-team
# deployment "yolov8-serving" successfully rolled out

kubectl get pods -n ai-team -l app=yolov8-serving -o wide
# 신규 Pod: Running on 2080ti-gpu-04 (재스케줄 결과)
```

---

## ✅ 5. 최종 검증 결과

```bash
kubectl get pod -n ai-team -l app=yolov8-serving \
  -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}{.items[0].spec.nodeSelector}{"\n"}'
# 1jkim/yolov8-serving:v1
# {"gpu-type":"2080ti"}

curl -s http://WORKER-NODE-IP:SERVING-NODEPORT/health
# {"status":"ok","demo_model":"yolov8n-coco (80 classes)","champion_ready":true,"champion_version":"5"}
```

| 항목           | 결과                                       |
| ------------ | ---------------------------------------- |
| Pod 상태       | Running / 2080ti-gpu-04                  |
| 이미지          | `1jkim/yolov8-serving:v1`                |
| nodeSelector | `{"gpu-type":"2080ti"}` (hostname 고정 없음) |
| `/health`    | champion_ready=true, version=5           |
| `/predict`   | visdrone-yolov8@champion (v5) 정상         |

---

## 📋 6. 주요 변경 요약

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 이미지 저장 위치 | 2080ti-gpu-04 로컬 containerd (k8s.io namespace) | DockerHub `1jkim/yolov8-serving:v1` (public) |
| Deployment 이미지 | `yolov8-serving:local-v1` | `1jkim/yolov8-serving:v1` |
| nodeSelector | `gpu-type: 2080ti` + `kubernetes.io/hostname: 2080ti-gpu-04` | `gpu-type: 2080ti` (hostname 고정 제거) |

---

## 💡 7. 핵심 인사이트

**nerdctl login은 성공해도 push가 실패할 수 있다.** 원인은 `/root/.docker/config.json`의 auths 키 형식. nerdctl은 push 시 `https://index.docker.io/v1/` 키를 조회하지만, 로그인 시 `index.docker.io`로 저장되는 경우가 있다. 로그인 성공 후 push 실패 시 config.json을 먼저 확인할 것.

**DockerHub 저장소 미생성 상태에서의 push는 `insufficient_scope`로 실패한다.** 저장소 부재와 권한 부재가 동일한 에러 메시지로 나타나므로 혼동하기 쉽다. push 전에 반드시 웹에서 저장소 존재 여부를 확인할 것.

**hostname nodeSelector는 로컬 이미지 의존성이 사라지는 시점에 즉시 제거해야 한다.** 이미지가 레지스트리에 올라간 이후에도 hostname 고정이 남아 있으면 해당 노드 장애 시 서비스가 복구되지 않는다.


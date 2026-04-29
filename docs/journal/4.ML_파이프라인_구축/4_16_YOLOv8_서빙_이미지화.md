# YOLOv8 서빙 이미지화 (pip install 제거)

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-16
> **작업 목적:** FastAPI 서빙의 런타임 `pip install` 제거 및 커스텀 이미지 전환. 재시작마다 설치하는 방식 → 이미지 내장 방식으로 개선.
> **대상 서버:** master-01 (빌드/제어), 2080ti-gpu-04 (서빙 노드)
> **작업 환경:** Kubernetes v1.29, ai-team 네임스페이스, containerd, nerdctl v1.7.6, buildkit v0.14.1
> **최종 결과:** 이미지화 완료. `/health`, `/predict-demo`, `/predict` 전 엔드포인트 정상.

---

## 🏗️ 2. 작업 흐름

```text
[master-01] buildkit 설치 → nerdctl build → yolov8-serving:local-v1
    │ nerdctl save → 6.2GB tar
    ▼
[scp] master-01 → 2080ti-gpu-04:/tmp/
    │ sudo nerdctl --namespace k8s.io load
    ▼
[K8s Deployment 교체]
    image 변경 + command/args 제거 + /app ConfigMap 마운트 제거
    nodeSelector: kubernetes.io/hostname=2080ti-gpu-04 추가
    ▼
[검증] /health + /predict-demo + /predict 정상 확인
```

---

## 📐 3. 아키텍처 설계 결정

### 3-1. 왜 이미지화인가 (런타임 pip install 대신)

| 항목 | 런타임 pip install | 이미지 내장 |
|---|---|---|
| 기동 시간 | 수 분 (패키지 설치) | 수 초 |
| 재현성 | 네트워크 상태에 의존 | 빌드 시점 고정 |
| 오프라인 동작 | 불가 | 가능 |
| 버전 일관성 | 패키지 버전 미보장 | Dockerfile에 고정 |

> ⭐ **설계 결정:** 이미지 내장 방식으로 전환. 코드 변경 시 이미지 재빌드(`local-v2`, `local-v3`)로 반영. ConfigMap 코드 마운트는 이미지 기반 배포에서 사용 금지.

---

## 🔧 4. 환경 구성

### 4-1. nerdctl + buildkit 설치 (master-01)

containerd 환경에서 `docker build`는 불가. `nerdctl` + `buildkit`으로 대체.

```bash
# nerdctl 설치
cd /tmp
wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-1.7.6-linux-amd64.tar.gz
sudo tar -xf nerdctl-1.7.6-linux-amd64.tar.gz -C /usr/local/bin/

# buildkit 설치 — nerdctl build 필수 의존성
wget https://github.com/moby/buildkit/releases/download/v0.14.1/buildkit-v0.14.1.linux-amd64.tar.gz
sudo tar -xf buildkit-v0.14.1.linux-amd64.tar.gz -C /usr/local/
sudo mkdir -p /run/buildkit
sudo nohup /usr/local/bin/buildkitd >/tmp/buildkitd.log 2>&1 &

# 확인
buildctl --version
sudo nerdctl version
```

### 4-2. nerdctl 설치 (2080ti-gpu-04 — 서빙 노드)

```bash
# master-01에서 실행
scp /tmp/nerdctl-1.7.6-linux-amd64.tar.gz ubuntu@WORKER-IP:/tmp/
ssh ubuntu@WORKER-IP \
  'sudo tar -xf /tmp/nerdctl-1.7.6-linux-amd64.tar.gz -C /usr/local/bin/ && \
   sudo nerdctl version'
```

---

## 🛠️ 5. 이미지 빌드

### 5-1. 작업 디렉터리 및 파일 준비

```bash
mkdir -p ~/serving-image && cd ~/serving-image

# 현재 운영 ConfigMap에서 main.py 추출
kubectl get configmap yolov8-serving-code -n ai-team \
  -o jsonpath='{.data.main\.py}' > main.py
```

### 5-2. Dockerfile

```dockerfile
FROM ultralytics/ultralytics:8.1.0

RUN pip install --no-cache-dir \
    fastapi==0.111.0 \
    uvicorn==0.30.1 \
    python-multipart==0.0.9 \
    mlflow \
    psycopg2-binary

COPY main.py /app/main.py
WORKDIR /app
EXPOSE 8080
CMD ["python", "/app/main.py"]
```

### 5-3. 빌드 실행

```bash
cd ~/serving-image
sudo nerdctl build -t yolov8-serving:local-v1 .
```

---

## 📦 6. 이미지 전달 (master-01 → 2080ti-gpu-04)

```bash
# tar 저장
mkdir -p ~/image-export
sudo nerdctl save -o ~/image-export/yolov8-serving-local-v1.tar yolov8-serving:local-v1
# 결과: 6.2GB

# 서빙 노드로 전송
scp ~/image-export/yolov8-serving-local-v1.tar ubuntu@WORKER-IP:/tmp/

# k8s.io namespace로 load — 이 옵션이 없으면 K8s가 이미지를 인식하지 못함
ssh ubuntu@WORKER-IP \
  'sudo nerdctl --namespace k8s.io load -i /tmp/yolov8-serving-local-v1.tar && \
   sudo nerdctl --namespace k8s.io images | grep yolov8-serving'
```

> ⭐ **핵심 학습:** `nerdctl load` 기본 namespace는 `default`. K8s kubelet은 `k8s.io` namespace만 참조. `--namespace k8s.io` 없이 load하면 이미지가 있어도 `ImagePullBackOff` 발생.

---

## 🚀 7. Deployment 교체

### 7-1. 배포 전 백업

```bash
mkdir -p ~/backup/yolov8-serving
kubectl get deploy yolov8-serving -n ai-team -o yaml \
  > ~/backup/yolov8-serving/deploy-before-image-change.yaml
```

### 7-2. 이미지 + 실행 방식 교체

```bash
# 이미지 교체
kubectl set image deployment/yolov8-serving serving=yolov8-serving:local-v1 -n ai-team

# command/args 제거 (pip install 런타임 설치 방식 제거)
kubectl patch deployment yolov8-serving -n ai-team --type='json' -p='[
  {"op":"remove","path":"/spec/template/spec/containers/0/command"},
  {"op":"remove","path":"/spec/template/spec/containers/0/args"}
]'

# /app ConfigMap 마운트 제거 (read-only 마운트로 인한 모델 다운로드 실패 방지)
kubectl patch deployment yolov8-serving -n ai-team --type='json' -p='[
  {"op":"remove","path":"/spec/template/spec/containers/0/volumeMounts/0"},
  {"op":"remove","path":"/spec/template/spec/volumes/0"}
]'

# nodeSelector에 hostname 고정 추가 (로컬 이미지 노드 고정 필수)
kubectl patch deployment yolov8-serving -n ai-team --type='merge' \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"2080ti-gpu-04"}}}}}'

# 롤아웃 확인
kubectl rollout status deployment/yolov8-serving -n ai-team
```

---

## ✅ 8. 검증 결과

```bash
# Pod 상태
kubectl get pods -n ai-team -l app=yolov8-serving -o wide
# 기대값: Running / 2080ti-gpu-04

# health 확인
curl -s http://WORKER-IP:SERVING-NODEPORT/health
# {"status":"ok","demo_model":"yolov8n-coco (80 classes)","champion_ready":true,"champion_version":"5"}

# /predict 추론 확인
curl -s -X POST http://WORKER-IP:SERVING-NODEPORT/predict -F "file=@test.png" | \
  python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get("model"), d.get("count"))'
# visdrone-yolov8@champion (v5)  0
```

| 엔드포인트 | 결과 |
|---|---|
| `GET /health` | ✅ champion_ready=true, version=5 |
| `POST /predict-demo` | ✅ COCO 80클래스 정상 |
| `POST /predict` | ✅ visdrone champion v5 정상 |

---

## 💡 9. 핵심 인사이트

**`nerdctl --namespace k8s.io` 없이 load하면 K8s가 이미지를 못 찾는다.** containerd는 namespace별로 이미지를 격리 관리한다. `nerdctl load`는 `default` namespace에 올리지만, kubelet은 `k8s.io` namespace만 본다.

**ConfigMap을 `/app`에 마운트하면 해당 경로가 read-only가 된다.** ultralytics는 `YOLO("yolov8n.pt")` 호출 시 현재 디렉토리에 모델 파일 다운로드를 시도한다. `/app`이 read-only면 쓰기 실패 → CrashLoopBackOff. 이미지화 이후 ConfigMap 마운트 자체가 불필요하다.

**`gpu-type=2080ti`만 nodeSelector로 쓰면 재스케줄 시 다른 2080Ti 노드로 이동한다.** 이미지가 특정 노드 로컬에만 있는 구조에서는 반드시 `kubernetes.io/hostname`을 함께 고정해야 한다.

**buildkit은 nerdctl build의 필수 의존성이다.** `buildkitd`가 실행 중이지 않으면 `no buildkit host is available` 에러로 빌드 자체가 불가하다.

---

## 📋 10. 주요 변경 요약

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 이미지 | `ultralytics/ultralytics:8.1.0` | `yolov8-serving:local-v1` |
| 실행 방식 | `pip install ... && python /app/main.py` | `CMD ["python", "/app/main.py"]` (이미지 내장) |
| `/app` 마운트 | ConfigMap read-only 마운트 | **제거** (이미지 내 `/app/main.py` 직접 사용) |
| nodeSelector | `gpu-type: 2080ti` | `gpu-type: 2080ti` + `kubernetes.io/hostname: 2080ti-gpu-04` |
| 이미지 저장 위치 | DockerHub public | 2080ti-gpu-04 로컬 containerd (k8s.io namespace) |

---

## 🔜 11. 미해결 / 후속 작업

| 항목                            | 상태                | 비고                                                       |
| ----------------------------- | ----------------- | -------------------------------------------------------- |
| DockerHub push                | ✅ 완료 (2026-04-17) | `1jkim/yolov8-serving:v1` push, hostname nodeSelector 제거 |
| ConfigMap yolov8-serving-code | 잔존                | 정리 가능하나 현재 미삭제                                           |
| buildkitd 서비스 등록              | 미완료               | 현재 nohup 백그라운드. 재부팅 시 재실행 필요                             |

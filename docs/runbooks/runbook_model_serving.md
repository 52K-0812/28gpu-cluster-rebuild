# Runbook — FastAPI YOLOv8 모델 서빙

> **목적:** YOLOv8 추론 서버의 배포 · 이미지 관리 · 검증 · 장애 대응 절차
> **최초 구축:** 2026-04-13
> **이미지화 전환:** 2026-04-16
> **관리 네임스페이스:** `ai-team`
> **담당:** 관리자 (master-01)

---

## 1. 📋 현재 배포 대상

| 항목 | 내용 |
|---|---|
| 엔드포인트 | `POST /predict-demo` (COCO) · `POST /predict` (champion) · `GET /health` |
| 배포 이미지 | `1jkim/yolov8-serving:v1` (DockerHub public) |
| 이미지 저장 위치 | DockerHub — 모든 노드에서 자동 pull 가능 |
| 배포 노드 | 2080Ti 풀 자동 스케줄 (`gpu-type: 2080ti`, hostname 고정 없음) |
| GPU | 2080Ti × 1 |
| 추론 속도 | 77ms |
| 접속 | `http://TAILSCALE-IP:30600` (Tailscale) |

**서빙 구조:**
```
[브라우저]
    │ 웹캠 캡처 or 이미지 업로드
    ▼
[FastAPI 웹 UI — ai-team 네임스페이스 / 2080ti-gpu-04]
    ├→ POST /predict-demo → YOLOv8n COCO 80클래스 추론
    └→ POST /predict      → visdrone-yolov8@champion 추론
    └→ 바운딩박스 + 클래스명 + confidence 반환
```

> **설계 원칙:** 추론은 2080Ti 1장, 학습은 V100 4장 전용으로 역할 분리. V100을 학습 Job 예약 자원으로 보존.

> **이미지 관리:** `1jkim/yolov8-serving:v1` (DockerHub public). 모든 2080Ti 노드에서 자동 pull. 코드 변경 시 이미지 재빌드 후 DockerHub push → Deployment 이미지 교체로 반영.

---

## 2. ✅ 사전 점검

배포 또는 재배포 전 아래 항목을 확인한다.

```bash
# 1. 네임스페이스 확인
kubectl get ns ai-team

# 2. 2080ti-gpu-04 GPU 가용량 확인
kubectl get node 2080ti-gpu-04 -o custom-columns=\
NAME:.metadata.name,\
GPU:.status.allocatable."nvidia\.com/gpu"

# 3. 기존 배포 상태 확인
kubectl get deploy,svc -n ai-team | grep yolov8

# 4. NFS StorageClass 정상 여부
kubectl get storageclass
```

---

## 3. 🐳 이미지 관리

### 현재 이미지 확인

```bash
kubectl get deploy yolov8-serving -n ai-team \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# 기대값: 1jkim/yolov8-serving:v1
```

### 이미지 재빌드 (코드 변경 시)

```bash
# master-01에서 buildkitd 실행 확인
pgrep buildkitd || sudo nohup /usr/local/bin/buildkitd >/tmp/buildkitd.log 2>&1 &

# 코드 수정 후 이미지 재빌드 (버전 번호 올릴 것)
cd ~/serving-image
# main.py 수정
sudo nerdctl build -t yolov8-serving:local-v2 .

# DockerHub 태그 및 push
sudo nerdctl tag yolov8-serving:local-v2 1jkim/yolov8-serving:v2
sudo nerdctl push 1jkim/yolov8-serving:v2

# Deployment 이미지 교체
kubectl set image deployment/yolov8-serving serving=1jkim/yolov8-serving:v2 -n ai-team
kubectl rollout status deployment/yolov8-serving -n ai-team
```

> **buildkitd 주의:** 현재 nohup 백그라운드 실행 중. master-01 재부팅 시 수동 재실행 필요.
> ```bash
> sudo nohup /usr/local/bin/buildkitd >/tmp/buildkitd.log 2>&1 &
> ```

---

## 4. 🚀 배포 절차 (신규 또는 전체 재배포)

```bash
kubectl apply -f - <<'EOF'
# ─── Deployment ─────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yolov8-serving
  namespace: ai-team
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yolov8-serving
  template:
    metadata:
      labels:
        app: yolov8-serving
    spec:
      nodeSelector:
        gpu-type: 2080ti
      containers:
      - name: serving
        image: 1jkim/yolov8-serving:v1
        ports:
        - containerPort: 8080
        resources:
          limits:
            nvidia.com/gpu: "1"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 10
        volumeMounts:
        - name: mlflow-artifacts
          mountPath: /mnt/mlflow-artifacts
        - name: datasets
          mountPath: /mnt/datasets
      volumes:
      - name: mlflow-artifacts
        persistentVolumeClaim:
          claimName: mlflow-artifacts-pvc
      - name: datasets
        persistentVolumeClaim:
          claimName: nfs-datasets-pvc
---
# ─── Service ────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: yolov8-serving
  namespace: ai-team
spec:
  type: NodePort
  selector:
    app: yolov8-serving
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30600
EOF
```

---

## 5. 🔍 검증 절차

```bash
# 1. Pod Running 확인
kubectl get pods -n ai-team -l app=yolov8-serving -o wide
# 기대값: 1/1 Running / 2080ti-gpu-04

# 2. 서버 기동 로그 확인
kubectl logs -n ai-team -l app=yolov8-serving --tail=10
# 기대값: INFO: Uvicorn running on http://0.0.0.0:8080

# 3. health 확인
curl http://TAILSCALE-IP:30600/health
# 기대값: {"status":"ok","demo_model":"yolov8n-coco (80 classes)","champion_ready":true,"champion_version":"5"}

# 4. /predict-demo 추론 테스트
curl -X POST http://TAILSCALE-IP:30600/predict-demo \
  -F "file=@test_image.jpg" | python3 -m json.tool
# 기대값: count + detections(COCO 클래스) + image(base64)

# 5. /predict 추론 테스트
curl -s -X POST http://TAILSCALE-IP:30600/predict \
  -F "file=@test_image.jpg" | \
  python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get("model"), d.get("count"))'
# 기대값: visdrone-yolov8@champion (v5)  N
```

---

## 6. ↩️ 롤백 절차

```bash
# 이전 버전으로 롤백
kubectl rollout undo deployment/yolov8-serving -n ai-team
kubectl rollout status deployment/yolov8-serving -n ai-team
kubectl get pods -n ai-team -l app=yolov8-serving -o wide

# 특정 revision으로 롤백
kubectl rollout history deployment/yolov8-serving -n ai-team
kubectl rollout undo deployment/yolov8-serving -n ai-team --to-revision=N
```

---

## 7. 🚨 장애 대응

### Pod가 Running이 되지 않는 경우

```bash
kubectl describe pod -n ai-team -l app=yolov8-serving | tail -30
kubectl logs -n ai-team -l app=yolov8-serving --tail=50
```

| 증상 | 원인 | 조치 |
|---|---|---|
| `ImagePullBackOff` | DockerHub 접근 불가 또는 이미지명 오타 | `kubectl describe pod` 에러 확인, 네트워크 상태 점검 |
| `CrashLoopBackOff` | 코드 오류 또는 볼륨 마운트 문제 | `kubectl logs` 에러 확인 |
| `Pending` | 2080Ti GPU 자원 없음 | `kubectl get pods -n ai-team -o wide`로 점유 Pod 확인 |
| `OOMKilled` | GPU 메모리 부족 | 다른 2080Ti Pod 점유 확인 후 정리 |

### 웹캠 접근 불가 (HTTP 환경)

브라우저 `getUserMedia()` API는 HTTPS 또는 localhost에서만 동작한다. HTTP 접속 시 카메라 권한이 차단된다.

**임시 해결 (Chrome 플래그):**
```
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```
`http://TAILSCALE-IP:30600` 추가 → Enabled → Chrome 재시작

> **근본 해결:** Ingress + TLS 적용 (미완료, 다음 작업 예정)

### champion 모델 미반영

```bash
# champion alias 현황 확인
curl http://TAILSCALE-IP:30600/health
# champion_ready: false → champion 로드 실패

# 즉시 재로드 시도
curl -X POST http://TAILSCALE-IP:30600/reload-champion

# 또는 Pod 재시작
kubectl rollout restart deployment/yolov8-serving -n ai-team
```

### 추론 응답 없음 / 500 에러

```bash
# 실시간 로그로 에러 확인
kubectl logs -n ai-team -l app=yolov8-serving -f

# Pod 재시작
kubectl rollout restart deployment/yolov8-serving -n ai-team
```

---

## 8. 🗃️ [Superseded] 이전 배포 방식

> **2026-04-16 이전 방식. 현재 미사용.**

```yaml
# 이전 방식 — 런타임 pip install (현재 제거됨)
image: ultralytics/ultralytics:8.1.0
command: ["/bin/sh", "-c"]
args:
- pip install fastapi uvicorn python-multipart mlflow psycopg2-binary -q && python /app/main.py
volumeMounts:
- name: app-code       # ConfigMap read-only 마운트 → 모델 다운로드 실패 유발
  mountPath: /app
```

**문제점:**
- Pod 재시작마다 pip install 재실행 (약 2~3분 지연)
- 네트워크 장애 시 의존성 설치 실패로 서비스 불가
- ConfigMap `/app` 마운트 read-only로 인해 모델 파일 다운로드 실패
- 재현성 없음 (pip 버전 drift 가능성)

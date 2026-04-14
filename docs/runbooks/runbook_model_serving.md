# Runbook — FastAPI YOLOv8 모델 서빙

> **목적:** YOLOv8 추론 서버의 배포 · 검증 · 장애 대응 절차
> **최초 구축:** 2026-04-13
> **관리 네임스페이스:** `ai-team`
> **담당:** 관리자 (master-01)

---

## 현재 배포 대상

| 항목 | 내용 |
|---|---|
| 모델 | YOLOv8n COCO pretrained (80 클래스) |
| 베이스 이미지 | `ultralytics/ultralytics:8.1.0` |
| 배포 노드 | 2080ti-gpu-04 (`WORKER-IP-06`, `gpu-type: 2080ti` nodeSelector) |
| GPU | 2080Ti × 1 |
| 추론 속도 | 77ms |
| 접속 | `http://TAILSCALE-IP:30600` (Tailscale) |
| 엔드포인트 | `GET /` 웹 UI · `POST /predict` 추론 · `GET /health` 헬스체크 |

**서빙 구조:**
```
[브라우저]
    │ 웹캠 캡처 or 이미지 업로드
    ▼
[FastAPI 웹 UI — ai-team 네임스페이스]
    └→ POST /predict → YOLOv8n GPU 추론
    └→ 바운딩박스 + 클래스명 + confidence 반환
```

> **설계 원칙:** 추론은 2080Ti 1장, 학습은 V100 4장 전용으로 역할 분리. V100을 학습 Job 예약 자원으로 보존.

---

## 사전 점검

배포 또는 재배포 전 아래 항목을 확인한다.

```bash
# 1. 네임스페이스 확인
kubectl get ns ai-team

# 2. 2080Ti 노드 GPU 가용량 확인
kubectl get nodes -l gpu-type=2080ti -o custom-columns=\
NAME:.metadata.name,\
GPU:.status.allocatable."nvidia\.com/gpu"

# 3. 기존 배포 상태 확인 (재배포 시)
kubectl get deploy,svc -n ai-team | grep yolov8

# 4. NFS StorageClass 정상 여부
kubectl get storageclass
```

---

## 배포 절차

### Step 1 — ConfigMap + Deployment + Service 적용

> **참고:** containerd 환경이라 `docker build` 불가. 앱 코드는 ConfigMap으로 주입하고, 의존성은 컨테이너 시작 시 `pip install`로 설치한다.

```bash
kubectl apply -f - <<'EOF'
# ─── ConfigMap (main.py) ────────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: yolov8-serving-code
  namespace: ai-team
data:
  main.py: |
    from fastapi import FastAPI, File, UploadFile
    from fastapi.responses import HTMLResponse, JSONResponse
    import uvicorn, cv2, numpy as np, base64
    from ultralytics import YOLO

    app = FastAPI(title="YOLOv8 Serving")
    model = YOLO("yolov8n.pt")

    @app.get("/", response_class=HTMLResponse)
    async def index(): return HTML

    @app.post("/predict")
    async def predict(file: UploadFile = File(...)):
        contents = await file.read()
        img = cv2.imdecode(np.frombuffer(contents, np.uint8), cv2.IMREAD_COLOR)
        results = model(img, device=0)[0]
        _, buffer = cv2.imencode('.jpg', results.plot(), [cv2.IMWRITE_JPEG_QUALITY, 90])
        return JSONResponse({
            "count": len(results.boxes),
            "detections": [{"class": model.names[int(b.cls)], "confidence": round(float(b.conf), 3)}
                           for b in results.boxes],
            "image": base64.b64encode(buffer).decode()
        })

    @app.get("/health")
    async def health(): return {"status": "ok", "model": "yolov8n", "classes": 80}

    if __name__ == "__main__": uvicorn.run(app, host="0.0.0.0", port=8080)
---
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
        image: ultralytics/ultralytics:8.1.0
        command: ["/bin/sh", "-c"]
        args:
        - pip install fastapi uvicorn python-multipart -q && python /app/main.py
        ports:
        - containerPort: 8080
        resources:
          limits:
            nvidia.com/gpu: "1"
        volumeMounts:
        - name: app-code
          mountPath: /app
      volumes:
      - name: app-code
        configMap:
          name: yolov8-serving-code
---
# ─── Service ────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: yolov8-serving
  namespace: ai-team
  annotations:
    metallb.universe.tf/loadBalancerIPs: "MASTER-IP"
spec:
  type: LoadBalancer
  selector:
    app: yolov8-serving
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30600
EOF
```

### Step 2 — 이미지 풀링 대기

`ultralytics:8.1.0` 이미지는 약 10GB로 최초 풀링에 10분 내외 소요된다. `ContainerCreating` 상태는 정상이므로 대기한다.

```bash
# 풀링 진행 상황 확인
kubectl describe pod -n ai-team -l app=yolov8-serving | grep -A5 Events
# "Pulling image"  → 정상 진행 중
# "Started container" → 풀링 완료, 기동 시작
```

---

## 검증 절차

```bash
# 1. Pod Running 확인
kubectl get pods -n ai-team | grep yolov8-serving
# 기대값: yolov8-serving-xxxx   1/1   Running

# 2. 서버 기동 로그 확인
kubectl logs -n ai-team -l app=yolov8-serving --tail=10
# 기대값:
# INFO:     Application startup complete.
# INFO:     Uvicorn running on http://0.0.0.0:8080

# 3. 헬스체크
curl http://TAILSCALE-IP:30600/health
# 기대값: {"status":"ok","model":"yolov8n","classes":80}

# 4. 추론 테스트
curl -X POST http://TAILSCALE-IP:30600/predict \
  -F "file=@test_image.jpg" | python3 -m json.tool
# 기대값: detections 배열 + count + image(base64)
```

---

## 롤백 절차

### ConfigMap 코드 수정 후 롤백

```bash
# 이전 버전 ConfigMap으로 교체 후 Pod 재시작
kubectl apply -f yolov8-serving-code-prev.yaml
kubectl rollout restart deployment/yolov8-serving -n ai-team
```

### 전체 삭제 후 재배포

```bash
kubectl delete deploy yolov8-serving -n ai-team
kubectl delete svc yolov8-serving -n ai-team
kubectl delete configmap yolov8-serving-code -n ai-team

# 재배포 — 위 배포 절차 Step 1 재실행
```

---

## 장애 대응

### Pod가 Running이 되지 않는 경우

```bash
kubectl describe pod -n ai-team -l app=yolov8-serving | tail -20
```

| 증상 | 원인 | 조치 |
|---|---|---|
| `ContainerCreating` 10분 이상 | 이미지 풀링 중 (10GB) | 대기. Events에 `Pulling image` 확인 |
| `OOMKilled` | GPU 메모리 부족 | 다른 2080Ti Pod 점유 확인 후 정리 |
| `Pending` | 2080ti 노드 GPU 자원 없음 | `kubectl get pods -n ai-team -o wide`로 점유 Pod 확인 |
| `CrashLoopBackOff` | pip install 실패 또는 코드 오류 | `kubectl logs`로 오류 메시지 확인 |

### 웹캠 접근 불가 (HTTP 환경)

브라우저 `getUserMedia()` API는 HTTPS 또는 localhost에서만 동작한다. HTTP 접속 시 카메라 권한이 차단된다.

**임시 해결 (Chrome 플래그):**
```
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```
`http://TAILSCALE-IP:30600` 추가 → Enabled → Chrome 재시작

> **근본 해결:** Ingress + TLS 적용 (미완료, 다음 작업 예정)

### 추론 응답 없음 / 500 에러

```bash
# 실시간 로그로 에러 확인
kubectl logs -n ai-team -l app=yolov8-serving -f

# Pod 재시작
kubectl rollout restart deployment/yolov8-serving -n ai-team
```

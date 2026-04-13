# FastAPI YOLOv8 모델 서빙

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-13
> **작업 목적:** 학습된 YOLOv8n (COCO pretrained) 모델을 K8s Deployment로 서빙하고, 웹캠 실시간 캡처 및 이미지 업로드 추론이 가능한 웹 UI를 제공한다.
> **대상 서버:** 2080ti-gpu-04 ((control-plane-public-ip)), master-01 ((control-plane-public-ip))
> **작업 환경:** Kubernetes v1.29, ai-team 네임스페이스, containerd
> **최종 결과:** YOLOv8 서빙 웹 UI 접속 확인, 웹캠 캡처 → GPU 추론 → 바운딩박스 결과 77ms 응답 확인

---

## 🏗️ 2. 작업 흐름

```
[브라우저]
    │ 웹캠 캡처 or 이미지 업로드
    ▼
[FastAPI + HTML/JS 웹 UI (ai-team 네임스페이스)]
    ├→ GET  /          → 웹 UI 페이지
    ├→ POST /predict   → 이미지 → YOLOv8n 추론 → 결과 이미지(base64) + JSON
    └→ GET  /health    → 서버 상태 확인
    ▼
[YOLOv8n COCO — 2080Ti GPU 추론]
    └→ 바운딩박스 + 클래스명 + confidence 반환

[접속]
    Tailscale: http://(vpn-endpoint)30600
    내부:      http://(control-plane-public-ip):30600
```

---

## 📐 3. 아키텍처 설계 결정

### 3.1 모델 선택 — YOLOv8n COCO pretrained

| 항목           | VisDrone best.pt  | YOLOv8n COCO pretrained    |
| -------------- | ----------------- | -------------------------- |
| 탐지 클래스    | 드론 영상 10개    | 일상 물체 80개             |
| 웹캠 테스트    | ❌ 드론 시점 특화 | ✅ 일상 환경에서 바로 동작 |
| 별도 학습 필요 | ❌ 이미 학습됨    | ❌ 공식 pretrained 사용    |

웹캠 기반 데모 목적이므로 COCO pretrained를 선택했다. 모델은 컨테이너 시작 시 ultralytics가 자동으로 다운로드한다.

### 3.2 이미지 빌드 없는 배포 전략

containerd 기반 클러스터라 `docker build`를 사용할 수 없다. 커스텀 이미지 빌드 없이:

- **베이스 이미지:** `ultralytics/ultralytics:8.1.0` (YOLOv8 + CUDA 포함)
- **FastAPI 의존성:** 컨테이너 시작 시 `pip install fastapi uvicorn python-multipart`
- **앱 코드:** ConfigMap으로 `main.py` 주입 → `/app/main.py` 마운트

MLflow 때 `psycopg2-binary` pip install 패턴과 동일하다.

### 3.3 GPU 배치 — 2080Ti (추론 전용)

학습은 V100 4장 DDP, 추론은 2080Ti 1장으로 분리했다. 추론은 단일 이미지 처리라 V100이 필요 없고, V100은 학습 Job 전용으로 남겨둔다.

---

## 📦 4. 전체 배포 YAML (ConfigMap + Deployment + Service)

```bash
kubectl apply -f - <<'EOF'
# ─── 1. ConfigMap (main.py) ─────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: yolov8-serving-code
  namespace: ai-team
data:
  main.py: |
    from fastapi import FastAPI, File, UploadFile
    from fastapi.responses import HTMLResponse, JSONResponse
    import uvicorn
    import cv2
    import numpy as np
    from ultralytics import YOLO
    import base64
    from PIL import Image

    app = FastAPI(title="YOLOv8 Serving")
    model = YOLO("yolov8n.pt")

    # (HTML 생략 — 전체 코드는 ConfigMap 참고)

    @app.get("/", response_class=HTMLResponse)
    async def index():
        return HTML

    @app.post("/predict")
    async def predict(file: UploadFile = File(...)):
        contents = await file.read()
        nparr = np.frombuffer(contents, np.uint8)
        img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
        results = model(img, device=0)[0]
        annotated = results.plot()
        _, buffer = cv2.imencode('.jpg', annotated, [cv2.IMWRITE_JPEG_QUALITY, 90])
        img_b64 = base64.b64encode(buffer).decode()
        detections = []
        for box in results.boxes:
            detections.append({
                "class": model.names[int(box.cls)],
                "confidence": round(float(box.conf), 3)
            })
        return JSONResponse({
            "count": len(detections),
            "detections": detections,
            "image": img_b64
        })

    @app.get("/health")
    async def health():
        return {"status": "ok", "model": "yolov8n", "classes": 80}

    if __name__ == "__main__":
        uvicorn.run(app, host="0.0.0.0", port=8080)
---
# ─── 2. Deployment ──────────────────────────────────────────────
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
        - |
          pip install fastapi uvicorn python-multipart -q && \
          python /app/main.py
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
# ─── 3. Service ─────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: yolov8-serving
  namespace: ai-team
  annotations:
    metallb.universe.tf/loadBalancerIPs: "(control-plane-public-ip)"
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

---

## 📊 5. 배포 확인

```bash
# Pod 상태
kubectl get pods -n ai-team | grep yolov8-serving

# 로그 확인 (pip install + uvicorn 시작)
kubectl logs -n ai-team -l app=yolov8-serving -f

# 정상 출력
# INFO:     Started server process [59]
# INFO:     Application startup complete.
# INFO:     Uvicorn running on http://0.0.0.0:8080
```

---

## ✅ 6. 검증 결과

| 항목          | 결과                                      |
| ------------- | ----------------------------------------- |
| 배포 노드     | 2080ti-gpu-04 ((control-plane-public-ip)) |
| GPU           | 2080Ti × 1                                |
| 추론 시간     | **77ms**                                  |
| 탐지 결과     | laptop 57.8% (노트북 화면 정확 탐지)      |
| 웹캠          | Chrome 플래그 예외 추가 후 정상 동작      |
| 이미지 업로드 | 드래그 앤 드롭 포함 정상 동작             |

---

## 🛠️ 7. 트러블슈팅

### 문제 1: 이미지 풀링 지연 (ContainerCreating 장시간 유지)

**원인:** `ultralytics:8.1.0` 이미지가 약 10GB로 풀링 시간이 길다. 오류가 아닌 정상 상태.

**확인 방법:**

```bash
kubectl describe pod -n ai-team -l app=yolov8-serving | grep -A5 Events
# "Pulling image" 메시지 확인 → 정상
```

**해결:** 10분 내외 대기 후 Running 전환 확인.

### 문제 2: 웹캠 카메라 접근 불가

**원인:** 브라우저의 `getUserMedia()` API는 **HTTPS 또는 localhost**에서만 동작한다. `http://(vpn-endpoint)30600` HTTP 접속 시 카메라 권한 차단.

**해결 (빠른 방법 — Chrome 플래그):**

```
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

`http://(vpn-endpoint)30600` 추가 → Enabled → Chrome 재시작

> **근본 해결:** HTTPS 인증서 적용 (self-signed + nginx). 추후 CI/CD 구성 시 함께 적용 예정.

---

## 💡 8. 핵심 인사이트

**ConfigMap으로 앱 코드를 주입하면 이미지 빌드 없이 배포할 수 있다.** containerd 기반 클러스터에서 `docker build`를 사용할 수 없는 제약을 ConfigMap 볼륨 마운트로 우회했다. 코드 수정 시 ConfigMap만 업데이트하고 Pod를 재시작하면 된다.

**추론은 2080Ti, 학습은 V100으로 역할을 분리했다.** 단일 이미지 추론은 대용량 GPU가 필요 없다. V100 4장을 학습 전용으로 남겨두고 추론 서빙은 2080Ti 1장으로 처리해 GPU 자원을 효율적으로 분배했다.

**77ms 추론 속도는 2080Ti GPU 연산 덕분이다.** CPU 추론 대비 10배 이상 빠른 속도로, 실시간 웹캠 자동 추론(0.5~2초 인터벌)이 가능한 수준이다.

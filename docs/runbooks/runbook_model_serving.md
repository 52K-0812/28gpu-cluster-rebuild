# Runbook — FastAPI YOLOv8 모델 서빙

> **목적:** YOLOv8 추론 서버의 배포 · 검증 · 장애 대응 절차  
> **최초 구축:** 2026-04-13  
> **최종 업데이트:** 2026-04-15  
> **관리 네임스페이스:** `ai-team`  
> **담당:** 관리자 (`master-01`)

---

## 현재 배포 대상

| 항목 | 내용 |
|---|---|
| 데모 모델 | `YOLOv8n COCO pretrained` (80 클래스) |
| 운영 모델 | `MLflow Registry alias champion` → NAS 모델 파일 매핑 |
| 베이스 이미지 | `ultralytics/ultralytics:8.1.0` |
| 배포 노드 | `2080ti-gpu-04` (`gpu-type: 2080ti`) |
| GPU | `2080Ti × 1` |
| 접속 | `http://TAILSCALE-IP:30600` |
| 주요 엔드포인트 | `GET /` · `POST /predict-demo` · `POST /predict` · `POST /reload-champion` · `GET /health` |
| 현재 배포 방식 | ConfigMap으로 `main.py` 주입 + 컨테이너 시작 시 `pip install` |
| 다음 개선 작업 | Serving 이미지화로 런타임 `pip install` 제거 |

### 현재 서빙 구조

```text
[브라우저]
    │ 파일 업로드 / 웹캠 입력
    ▼
[FastAPI 웹 UI — ai-team]
    ├─ POST /predict-demo → YOLOv8n COCO 추론
    ├─ POST /predict      → MLflow champion 모델 추론
    ├─ POST /reload-champion → 최신 champion 재로드
    └─ GET /health        → demo/champion 상태 점검
```

### 설계 원칙

- 추론은 `2080Ti 1장`에서 수행하고, `V100 4장`은 학습 Job 용도로 보존한다.
- 데모 품질과 운영 검증 목적이 다르므로 엔드포인트를 분리한다.
- 운영 모델은 MLflow alias `champion` 기준으로 관리한다.
- 현재는 ConfigMap 방식으로 운영 중이며, 다음 단계에서 이미지 기반 배포로 전환한다.

---

## 엔드포인트 역할

| 엔드포인트 | 역할 | 모델 |
|---|---|---|
| `GET /` | 웹 UI 제공 | - |
| `POST /predict-demo` | 웹캠/일반 이미지 데모 추론 | `yolov8n.pt` (COCO) |
| `POST /predict` | 운영용 추론 | `visdrone-yolov8@champion` 에 해당하는 NAS 모델 |
| `POST /reload-champion` | Pod 재시작 없이 최신 champion 재반영 | - |
| `GET /health` | 상태 점검 | demo/champion 동시 확인 |

### 모델 로딩 정책

#### 1. 데모 모델

- 서버 시작 시 `YOLO("yolov8n.pt")` 로드
- COCO 80클래스 기준
- 웹캠 데모, 일반 물체 탐지 확인 용도

#### 2. 운영 champion 모델

- MLflow Registry에서 `visdrone-yolov8@champion` alias 조회
- alias가 가리키는 version 번호 확인
- NAS 경로 `/mnt/datasets/models/visdrone-v{version}.pt` 우선 로드
- 필요 시 MLflow artifact 경로를 폴백으로 사용

예시:

```text
MLflow alias "champion" → version 3
NAS 파일 경로         → /mnt/datasets/models/visdrone-v3.pt
```

---

## 사전 점검

배포 또는 재배포 전 아래 항목을 확인한다.

```bash
# 1. 네임스페이스 확인
kubectl get ns ai-team

# 2. 2080Ti 노드 GPU 가용량 확인
kubectl get nodes -l gpu-type=2080ti -o custom-columns=\\
NAME:.metadata.name,\\
GPU:.status.allocatable."nvidia\\.com/gpu"

# 3. 기존 serving 배포 상태 확인
kubectl get deploy,svc,pod -n ai-team | grep yolov8

# 4. champion 모델 파일 존재 여부 확인
ls -lh /mnt/datasets/models/visdrone-v*.pt

# 5. MLflow Registry alias 확인 필요 시
# (환경에 맞는 mlflow 접속 설정 후)
python - <<'PY2'
from mlflow.tracking import MlflowClient
client = MlflowClient()
mv = client.get_model_version_by_alias("visdrone-yolov8", "champion")
print("champion version:", mv.version)
print("source:", mv.source)
PY2
```

---

## 현재 배포 절차

### Step 1 — ConfigMap + Deployment + Service 적용

> **참고**  
> 현재 클러스터는 아직 Serving 이미지화 전 단계다.  
> 따라서 앱 코드는 ConfigMap으로 주입하고, 의존성은 컨테이너 시작 시 설치한다.  
> 이후 이미지화 단계가 완료되면 이 절차는 대체될 예정이다.

```bash
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: yolov8-serving-code
  namespace: ai-team
data:
  main.py: |
    # 최신 main.py 내용 반영
    # 핵심 요구사항:
    # 1) /predict-demo → COCO demo 모델
    # 2) /predict      → MLflow champion 모델
    # 3) /reload-champion 지원
    # 4) /health 에 champion 상태 표시
---
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
          pip install fastapi uvicorn python-multipart mlflow psycopg2-binary -q && \
          python /app/main.py
        ports:
        - containerPort: 8080
        resources:
          limits:
            nvidia.com/gpu: "1"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 20
        volumeMounts:
        - name: app-code
          mountPath: /app
        - name: datasets
          mountPath: /mnt/datasets
        - name: mlflow-artifacts
          mountPath: /mnt/mlflow-artifacts
      volumes:
      - name: app-code
        configMap:
          name: yolov8-serving-code
      - name: datasets
        persistentVolumeClaim:
          claimName: datasets-pvc
      - name: mlflow-artifacts
        persistentVolumeClaim:
          claimName: mlflow-artifacts-pvc
---
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
YAML
```

### Step 2 — Pod 기동 확인

```bash
kubectl rollout status deployment/yolov8-serving -n ai-team
kubectl get pods -n ai-team | grep yolov8-serving
kubectl logs -n ai-team -l app=yolov8-serving --tail=30
```

기대 로그 예시:

```text
✅ COCO demo 모델 로드 완료
✅ champion v3 NAS 로드 완료: /mnt/datasets/models/visdrone-v3.pt
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8080
```

---

## 검증 절차

### 1. 헬스체크

```bash
curl http://TAILSCALE-IP:30600/health | python3 -m json.tool
```

기대 응답 예시:

```json
{
  "status": "ok",
  "demo_model": "yolov8n-coco",
  "champion_ready": true,
  "champion_version": "3"
}
```

### 2. 데모 엔드포인트 검증

```bash
curl -X POST http://TAILSCALE-IP:30600/predict-demo \
  -F "file=@test.jpg" | python3 -m json.tool
```

확인 포인트:
- COCO 클래스 기준 탐지 결과가 나와야 한다.
- 웹캠/일반 이미지 데모 용도로 동작해야 한다.

### 3. 운영 엔드포인트 검증

```bash
curl -X POST http://TAILSCALE-IP:30600/predict \
  -F "file=@test.jpg" | python3 -m json.tool
```

확인 포인트:
- champion 모델 기준 추론이 수행되어야 한다.
- `/predict-demo` 와 다른 탐지 결과가 나올 수 있다.

### 4. champion 재로드 검증

```bash
curl -X POST http://TAILSCALE-IP:30600/reload-champion | python3 -m json.tool
```

확인 포인트:
- Pod 재시작 없이 최신 alias 대상 모델을 다시 불러와야 한다.
- 이후 `/health` 에서 champion version 변경 여부를 확인한다.

---

## ConfigMap 업데이트 절차

현재 운영 방식에서 `main.py` 를 수정할 때 사용한다.

```bash
kubectl create configmap yolov8-serving-code \
  --from-file=main.py=~/main.py \
  -n ai-team \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/yolov8-serving -n ai-team
kubectl rollout status deployment/yolov8-serving -n ai-team
kubectl logs -n ai-team -l app=yolov8-serving --tail=20
```

Windows 로컬에서 전송 예시:

```powershell
scp C:\Users\wwdnj\Downloads\main.py ubuntu@112.76.56.151:~/
```

---

## 롤백 절차

### 1. ConfigMap 기준 롤백

```bash
kubectl apply -f yolov8-serving-code-prev.yaml
kubectl rollout restart deployment/yolov8-serving -n ai-team
kubectl rollout status deployment/yolov8-serving -n ai-team
```

### 2. 전체 삭제 후 재배포

```bash
kubectl delete deploy yolov8-serving -n ai-team
kubectl delete svc yolov8-serving -n ai-team
kubectl delete configmap yolov8-serving-code -n ai-team

# 이후 현재 배포 절차 Step 1 재실행
```

---

## 장애 대응

### Pod가 Running이 되지 않는 경우

```bash
kubectl describe pod -n ai-team -l app=yolov8-serving | tail -30
```

| 증상 | 원인 | 조치 |
|---|---|---|
| `ContainerCreating` 장시간 지속 | 대용량 이미지 풀링 또는 볼륨 마운트 대기 | `describe pod` 이벤트 확인 |
| `Pending` | 2080Ti 노드 GPU 자원 부족 | 다른 Pod 점유 확인 후 정리 |
| `CrashLoopBackOff` | `pip install` 실패, Python 코드 오류, champion 로드 오류 | `kubectl logs` 로 상세 오류 확인 |
| `Readiness probe failed` | `/health` 가 champion 초기화 전에 실패 | `initialDelaySeconds` 조정 및 초기 로그 확인 |

### champion 모델 로드 실패

증상 예시:
- `/health` 에서 `champion_ready: false`
- 로그에 NAS 파일 없음 또는 alias 조회 실패 표시

확인 절차:

```bash
ls -lh /mnt/datasets/models/visdrone-v*.pt
kubectl logs -n ai-team -l app=yolov8-serving --tail=100
```

점검 포인트:
- MLflow alias `champion` 이 올바른 version 을 가리키는지
- `/mnt/datasets/models/visdrone-v{version}.pt` 파일이 실제 존재하는지
- `mlflow-artifacts-pvc` 가 정상 마운트되었는지

### 추론 응답 없음 / 500 에러

```bash
kubectl logs -n ai-team -l app=yolov8-serving -f
kubectl rollout restart deployment/yolov8-serving -n ai-team
```

### 웹캠 접근 불가 (HTTP 환경)

브라우저 `getUserMedia()` API는 HTTPS 또는 localhost 에서만 동작한다. HTTP 접속 시 카메라 권한이 차단될 수 있다.

Chrome 임시 해결:

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

`http://TAILSCALE-IP:30600` 추가 → Enabled → Chrome 재시작

> 근본 해결은 Ingress + TLS 적용이다.

---

## 다음 작업 예정

### Serving 이미지화

현재 serving Pod 는 시작 시마다 아래 패키지를 런타임 설치한다.

- `fastapi`
- `uvicorn`
- `python-multipart`
- `mlflow`
- `psycopg2-binary`

이 방식은 다음 문제가 있다.

- Pod 재시작마다 설치 시간 발생
- 재현성 저하
- 장애 지점 증가

따라서 다음 단계에서 아래 방향으로 전환한다.

```text
현재:
ConfigMap 주입 + 컨테이너 시작 시 pip install

변경 예정:
Dockerfile/nerdctl 기반 serving 이미지 빌드
→ 이미지 push
→ Deployment image 교체
→ command/args 제거
```

이미지화 완료 후에는 이 runbook 에서 ConfigMap 중심 절차를 별도 문서로 분리하고,
본 문서는 이미지 기반 배포 절차 중심으로 다시 정리한다.

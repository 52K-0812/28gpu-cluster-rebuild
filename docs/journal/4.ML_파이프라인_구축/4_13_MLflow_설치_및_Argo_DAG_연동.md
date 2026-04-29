# MLflow 설치 및 Argo DAG 연동

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-13
> **작업 목적:** MLflow Tracking Server를 K8s에 배포하고, 기존 Argo DAG 파이프라인(yolov8-dag-pipeline)에 실험 로깅을 연동한다. 학습 실행마다 파라미터·메트릭·아티팩트가 자동으로 기록되어 버전별 성능 비교가 가능한 환경을 구성한다.
> **대상 서버:** master-01, NAS
> **작업 환경:** Kubernetes v1.29, Helm, mlflow 네임스페이스, ai-team 네임스페이스
> **최종 결과:** MLflow UI 접속 확인, Argo DAG 실행 후 params 105개 + metrics 7개 자동 기록 완료

---

## 🏗️ 2. 작업 흐름

```text
[MLflow Tracking Server (mlflow 네임스페이스)]
    ├→ Backend: PostgreSQL (K8s Deployment)
    │           └→ PVC → NAS (실험 메타데이터)
    └→ Artifact: NFS PVC → NAS /data/mlflow-artifacts/

[Argo DAG — yolov8-dag-pipeline]
    ├→ train 단계
    │     └→ mlflow.log_params() → run_id → NAS 파일로 전달
    └→ evaluate 단계
          └→ mlflow.log_metrics() + mlflow.log_artifact(best.pt)

[접속]
    Tailscale: http://TAILSCALE_HOST:30010
    내부:      http://LB_PUBLIC_IP:30010
```

---

## 📐 3. 아키텍처 설계 결정

### 3.1 왜 PostgreSQL인가 (SQLite 대신)

| 항목            | SQLite       | PostgreSQL        |
| --------------- | ------------ | ----------------- |
| 동시 쓰기       | ❌ lock 발생 | ✅ 다중 연결 지원 |
| 다중 학습 Job   | ❌ 충돌 가능 | ✅ 안전           |
| K8s 환경 적합성 | ❌ 파일 기반 | ✅ 서비스 기반    |

V100 4장 + 2080Ti 23장 환경에서 동시 학습 Job이 발생할 수 있으므로 PostgreSQL을 선택했다.

### 3.2 Artifact 저장소 — NFS PVC

NAS `/data/` 경로를 NFS PVC로 마운트해 아티팩트를 저장한다. 어느 GPU 노드에서 학습해도 동일한 경로로 best.pt가 저장된다.

### 3.3 run_id 파일 전달 패턴

Argo DAG에서 train → evaluate 단계 간 MLflow run을 연속으로 이어받기 위해 `mlflow_run_id.txt`를 NAS 공유 경로에 저장하는 방식을 사용했다. Argo의 output parameter 방식 대신 파일 기반 전달을 택한 이유는 기존 NFS PVC 마운트 구조를 그대로 활용할 수 있기 때문이다.

---

## 📦 4. Step 1 — MetalLB IP 추가

```bash
# 161 핑 테스트 (응답 없으면 사용 가능)
ping -c 3 LB_PUBLIC_IP

# MetalLB 풀에 161 추가
kubectl patch ipaddresspool main-pool -n metallb-system --type=json -p='[
  {"op": "replace", "path": "/spec/addresses", "value": ["LB_PUBLIC_IP/32", "LB_PUBLIC_IP/32", "LB_PUBLIC_IP/32"]}
]'

# 확인
kubectl get ipaddresspool main-pool -n metallb-system -o jsonpath='{.spec.addresses}'
```

---

## 📦 5. Step 2 — MLflow 전체 배포 (PostgreSQL + Tracking Server + PVC)

```bash
kubectl create namespace mlflow
```

```bash
kubectl apply -f - <<'EOF'
# ─── 1. NFS PVC (Artifact 저장소) ───────────────────────────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mlflow-artifacts-pvc
  namespace: mlflow
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 100Gi
---
# ─── 2. PostgreSQL PVC ──────────────────────────────────────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mlflow-postgres-pvc
  namespace: mlflow
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
---
# ─── 3. PostgreSQL Secret ───────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: mlflow-postgres-secret
  namespace: mlflow
type: Opaque
stringData:
  POSTGRES_DB: mlflow
  POSTGRES_USER: mlflow
  POSTGRES_PASSWORD: YOUR_PASSWORD
---
# ─── 4. PostgreSQL Deployment ───────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-postgres
  namespace: mlflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow-postgres
  template:
    metadata:
      labels:
        app: mlflow-postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: mlflow-postgres-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: mlflow-postgres-pvc
---
# ─── 5. PostgreSQL Service ──────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: mlflow-postgres
  namespace: mlflow
spec:
  selector:
    app: mlflow-postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# ─── 6. MLflow Tracking Server Deployment ───────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
  namespace: mlflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow-server
  template:
    metadata:
      labels:
        app: mlflow-server
    spec:
      initContainers:
      - name: wait-for-postgres
        image: busybox
        command: ['sh', '-c', 'until nc -z mlflow-postgres 5432; do echo waiting; sleep 2; done']
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.13.0
        command: ["/bin/sh", "-c"]
        args:
          - |
            pip install psycopg2-binary -q && \
            mlflow server \
              --backend-store-uri postgresql://mlflow:YOUR_PASSWORD@mlflow-postgres:5432/mlflow \
              --default-artifact-root /mnt/mlflow-artifacts \
              --host 0.0.0.0 \
              --port 5000
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: artifacts
          mountPath: /mnt/mlflow-artifacts
      volumes:
      - name: artifacts
        persistentVolumeClaim:
          claimName: mlflow-artifacts-pvc
---
# ─── 7. MLflow Service (MetalLB) ────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: mlflow-server
  namespace: mlflow
  annotations:
    metallb.universe.tf/loadBalancerIPs: "LB_PUBLIC_IP"
spec:
  type: LoadBalancer
  selector:
    app: mlflow-server
  ports:
  - port: 5000
    targetPort: 5000
EOF
```

### Pod 상태 확인

```bash
kubectl get pods -n mlflow -w

# 정상 출력
# mlflow-postgres-xxxx   1/1   Running
# mlflow-server-xxxx     1/1   Running
```

---

## 🌐 6. Step 3 — NodePort 고정 + systemd port-forward

### 6.1 NodePort 30010으로 고정

```bash
kubectl patch svc mlflow-server -n mlflow --type=json -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30010}
]'
```

### 6.2 systemd 서비스 등록

```bash
sudo nano /etc/systemd/system/kubectl-mlflow-forward.service
```

```ini
[Unit]
Description=kubectl port-forward for MLflow
After=network.target

[Service]
Restart=always
RestartSec=5
ExecStart=/usr/bin/kubectl port-forward svc/mlflow-server 30010:5000 -n mlflow --address 0.0.0.0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable kubectl-mlflow-forward.service
sudo systemctl start kubectl-mlflow-forward.service
sudo systemctl status kubectl-mlflow-forward.service
```

**접속:**

- Tailscale: `http://TAILSCALE_HOST:30010`
- 내부: `http://LB_PUBLIC_IP:30010`

---

## 🔗 7. Step 4 — Argo DAG MLflow 연동 (WorkflowTemplate 업데이트)

### 핵심 변경 사항

| 단계        | 변경 전     | 변경 후                                      |
| ----------- | ----------- | -------------------------------------------- |
| train       | 학습만 실행 | mlflow.log_params() + run_id 파일 저장       |
| evaluate    | mAP 출력만  | mlflow.log_metrics() + log_artifact(best.pt) |
| MLflow 주소 | -           | cluster.local 내부 DNS 사용                  |

### MLflow 내부 DNS 주소

```text
http://mlflow-server.mlflow.svc.cluster.local:5000
```

K8s 내부 DNS를 사용하므로 외부 IP나 NodePort 불필요.

### train 단계 핵심 코드

```python
import mlflow

MLFLOW_URI = "http://mlflow-server.mlflow.svc.cluster.local:5000"
mlflow.set_tracking_uri(MLFLOW_URI)
mlflow.set_experiment("yolov8-visdrone")

with mlflow.start_run(run_name=f"train-{version}") as run:
    mlflow.log_params({
        "epochs": epochs,
        "batch_size": batch,
        "model": "yolov8n",
        "dataset": "visdrone",
        "device": "0,1,2,3",
        "imgsz": 640,
        "model_version": version
    })

    # YOLO 학습 (ultralytics가 추가 params 자동 로깅)
    model = YOLO('yolov8n.pt')
    results = model.train(...)

    # run_id를 NAS에 저장 → evaluate 단계에서 이어받기
    run_id = run.info.run_id
    with open('/mnt/datasets/runs/dag-train/mlflow_run_id.txt', 'w') as f:
        f.write(run_id)
```

### evaluate 단계 핵심 코드

```python
# train이 저장한 run_id로 동일 run에 이어서 로깅
with open(run_id_file) as f:
    run_id = f.read().strip()

with mlflow.start_run(run_id=run_id):
    metrics = model.val(...)
    mlflow.log_metrics({
        "mAP50":   round(metrics.box.map50, 4),
        "mAP5095": round(metrics.box.map, 4)
    })
    mlflow.log_artifact(best_pt, artifact_path="weights")
```

### WorkflowTemplate 적용

```bash
kubectl apply -f yolov8-dag-pipeline.yaml
kubectl get workflowtemplate yolov8-dag-pipeline -n ai-team
```

---

## ✅ 8. 검증 결과

### 8.1 MLflow UI

| 항목      | 결과            |
| --------- | --------------- |
| 실험명    | yolov8-visdrone |
| Run명     | dag-train8      |
| 소요 시간 | 18.5min         |
| 상태      | ✅ Succeeded    |

### 8.2 기록된 데이터

| 분류       | 항목               | 값                                  |
| ---------- | ------------------ | ----------------------------------- |
| Metrics    | metrics/mAP50B     | 0.2752                              |
| Metrics    | metrics/mAP50-95B  | 0.1589                              |
| Metrics    | metrics/precisionB | 0.3743                              |
| Metrics    | metrics/recallB    | 0.2857                              |
| Metrics    | val/box_loss       | 1.41419                             |
| Metrics    | val/cls_loss       | 1.07261                             |
| Metrics    | val/dfl_loss       | 0.92512                             |
| Parameters | 총 105개           | epochs, batch, augmentation 등 전체 |

---

## 🛠️ 9. 트러블슈팅

### 문제 1: `No module named 'psycopg2'`

```text
2026/04/13 02:23:59 ERROR mlflow.cli: No module named 'psycopg2'
```

**원인:** `ghcr.io/mlflow/mlflow` 공식 이미지에 psycopg2가 포함되지 않음.

**시도한 방법:** `bitnami/mlflow:2.13.0` 교체 → bitnami에 mlflow 이미지 자체가 없음 (`count: 0`).

**해결:** 공식 이미지 유지 + 컨테이너 시작 시 pip install 추가

```yaml
command: ['/bin/sh', '-c']
args:
  - |
    pip install psycopg2-binary -q && \
    mlflow server --backend-store-uri postgresql://...
```

> **교훈:** 공식 mlflow 이미지는 PostgreSQL 드라이버를 포함하지 않는다. 커스텀 이미지 빌드 없이 해결하려면 entrypoint에 pip install을 추가하는 것이 가장 빠르다.

### 문제 2: MetalLB IP 직접 접근 불가

**원인:** Tailscale VPN은 `100.x.x.x` 대역만 터널링. MetalLB가 ARP로 광고하는 `LB_PUBLIC_IP` 대역은 Tailscale subnet router 없이는 외부에서 도달 불가.

**해결:** NodePort + systemd port-forward 방식으로 접속 (JupyterHub, Argo와 동일 패턴 유지)

```text
LB_PUBLIC_IP:30010  → 서버실 내부 접속
TAILSCALE_HOST:30010   → Tailscale VPN 외부 접속
```

---

## 💡 10. 핵심 인사이트

**train과 evaluate를 하나의 MLflow run으로 묶어야 한다.** 두 단계를 별개 run으로 기록하면 MLflow UI에서 파라미터와 메트릭이 분리되어 실험 단위 비교가 불가능하다. run_id를 NAS 파일로 전달해 동일 run에 이어 기록하는 패턴이 Argo DAG 환경에서 가장 단순하고 안정적이다.

**K8s 내부 DNS를 쓰면 IP 관리가 필요 없다.** `mlflow-server.mlflow.svc.cluster.local:5000`은 어느 노드의 Pod에서도 동일하게 동작한다. NodePort나 MetalLB IP를 하드코딩하면 서비스 변경 시 WorkflowTemplate도 수정해야 하는 의존성이 생긴다.

**ultralytics는 mlflow.autolog()를 지원한다.** 별도 log_params 코드 없이도 학습에 사용된 하이퍼파라미터 전체(105개)가 자동으로 기록된다. 커스텀 메트릭(mAP50, mAP5095)만 추가로 log_metrics()로 명시하면 충분하다.

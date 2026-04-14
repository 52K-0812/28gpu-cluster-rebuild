# RBAC + Namespace + JupyterHub 구축 보고서

> [!WARNING]
> **상태: Superseded**
> 이 문서는 2026-03-30 최초 구축 당시의 작업 기록입니다.
> 여기서 설명하는 `kubectl port-forward` 기반 접속 구조는 이후 다중 접속 장애 원인으로 판명되어 전면 재설계되었습니다.
>
> **현재 유효한 구조:** [JupyterHub 다중 접속 장애 및 서비스 재설계 ⭐](../../incidents/4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md) 참조  
> **현재 아키텍처 요약:** [overview](../../overview/current-architecture.md) 참조

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-03-30
> **작업 목적**: 28-GPU 클러스터 위에 AI 팀원 5명이 독립적으로 Python / TensorFlow / CUDA 환경을 사용할 수 있도록 네임스페이스 격리, RBAC 권한 구조, JupyterHub 플랫폼을 구축.
> **대상 서버**: master-01
> **작업 환경**: Kubernetes v1.29, Helm v3, Tailscale VPN
> **최종 결과**: 팀원 5명 계정 생성 완료 및 JupyterLab 브라우저 접속 성공

---

## 🏗️ 2. 작업 구성 (Architecture)

```
[팀원 브라우저]
      │ Tailscale VPN
      ▼
[master-01 :8000]
      │ kubectl port-forward
      ▼
[proxy-public Service]
      │
      ▼
[JupyterHub (ai-team 네임스페이스)]
      │ 사용자 로그인 시 Pod 자동 생성
      ▼
[singleuser Pod → GPU 노드 자동 배정]
      │
      ▼
[NFS PVC → NAS 28TB 영구 저장]
```

---

## 🚀 3. Step 1 — Namespace 생성

### 3.1 ai-team 네임스페이스 생성

```bash
kubectl create namespace ai-team
kubectl get namespace ai-team
```

**결과:**

```
NAME      STATUS   AGE
ai-team   Active   2d11h
```

---

## 🔐 4. Step 2 — RBAC 구성

### 4.1 Role 생성 (ai-team-(팀이름))

팀원이 `ai-team` 네임스페이스 내에서 할 수 있는 작업 범위를 정의했다. Pod 조회/생성/삭제, Job 제출, PVC 접근을 허용하고 클러스터 전체 접근은 차단.

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ai-team
  name: ai-team-
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "services", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "delete"]
EOF
```

### 4.2 ServiceAccount 생성 (팀원 5명)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-01
  namespace: ai-team
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-02
  namespace: ai-team
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-03
  namespace: ai-team
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-04
  namespace: ai-team
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-05
  namespace: ai-team
EOF
```

### 4.3 RoleBinding (Role ↔ ServiceAccount 연결)

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ai-team-binding
  namespace: ai-team
subjects:
- kind: ServiceAccount
  name: user-01
  namespace: ai-team
- kind: ServiceAccount
  name: user-02
  namespace: ai-team
- kind: ServiceAccount
  name: user-03
  namespace: ai-team
- kind: ServiceAccount
  name: user-04
  namespace: ai-team
- kind: ServiceAccount
  name: user-05
  namespace: ai-team
roleRef:
  kind: Role
  name: ai-team-(팀이름)
  apiGroup: rbac.authorization.k8s.io
EOF
```

**최종 확인:**

```bash
kubectl get role,rolebinding,serviceaccount -n ai-team
```

---

## 📦 5. Step 3 — JupyterHub 설치

### 5.1 Helm repo 추가

```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
```

### 5.2 secretToken 생성

```bash
openssl rand -hex 32
```

> **보안 주의:** 생성된 토큰은 values 파일에만 보관하고 외부 공유 금지.

### 5.3 values 파일 작성 (jupyterhub-values.yaml)

```yaml
proxy:
  secretToken: '[생성된 토큰값]'
  service:
    type: ClusterIP # Tailscale port-forward 방식 사용

hub:
  config:
    JupyterHub:
      authenticator_class: dummy
    DummyAuthenticator:
      password: 'YOUR_PASSWORD'
    Authenticator:
      allowed_users:
        - user-01
        - user-02
        - user-03
        - user-04
        - user-05

singleuser:
  image:
    name: jupyter/tensorflow-notebook
    tag: latest
  storage:
    type: dynamic
    dynamic:
      storageClass: nfs-client # NAS 28TB 연결
    capacity: 50Gi
  cpu:
    limit: 4
    guarantee: 1
  memory:
    limit: 16G
    guarantee: 4G
```

### 5.4 JupyterHub 배포

```bash
helm install jupyterhub jupyterhub/jupyterhub \
  --namespace ai-team \
  --values jupyterhub-values.yaml \
  --timeout 10m
```

**설치 결과:**

- Helm chart version: 4.3.3
- JupyterHub version: 5.4.4

**Pod 상태 확인:**

```
NAME                              READY   STATUS    AGE
continuous-image-puller-*         1/1     Running   5m46s  (× 6, 전 노드)
hub-54595b9fd6-m9ssd              1/1     Running   5m46s
proxy-555fb58c86-nkdlt            1/1     Running   5m46s
user-scheduler-57666cd58b-*       1/1     Running   5m46s  (× 2)
```

전체 Running 상태 진입 완료.

---

## 🌍 6. Step 4 — Tailscale 외부 접속 연결

기존 Tailscale 구성(master-01)을 활용하여 systemd port-forward 서비스를 추가했다.

```bash
sudo tee /etc/systemd/system/kubectl-jupyterhub.service <<'EOF'
[Unit]
Description=kubectl port-forward JupyterHub
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/bin/kubectl port-forward svc/proxy-public \
  -n ai-team --address 0.0.0.0 8000:80
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kubectl-jupyterhub
```

**접속 주소:**

| 경로        | 주소                                  |
| ----------- | ------------------------------------- |
| Tailscale   | http://TAILSCALE_HOST:8000             |
| 내부망 직접 | http://LB_PUBLIC_IP:8000 |

---

## ✅ 7. 결과 및 팀원 접속 정보

- JupyterLab 브라우저 접속 및 로그인 정상 확인.
- Notebook / Console / Terminal 전체 기능 진입 가능.

![자료사진](../images/스크린샷_2026-03-30_172818.png)

---

## 🔬 8. GPU 연결 검증

### 8.1 트러블슈팅: 이미지 호환성 문제

초기 이미지(`jupyter/tensorflow-notebook:latest`)로 설치 후 GPU 인식 실패. 원인 분석 및 해결 과정:

| 시도 | 이미지                                             | 결과                | 원인                      |
| ---- | -------------------------------------------------- | ------------------- | ------------------------- |
| 1차  | `jupyter/tensorflow-notebook:latest`               | ❌ GPU 미인식       | CUDA 드라이버 미포함      |
| 2차  | `pangeo/pytorch-notebook:cuda-12.2.0`              | ❌ ImagePullBackOff | 존재하지 않는 태그        |
| 3차  | `cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04` | ✅ GPU 인식 성공    | CUDA 12.0 포함, 하위 호환 |

> **핵심 교훈:** JupyterHub 이미지는 반드시 CUDA 드라이버가 포함된 전용 이미지를 사용해야 한다. `jupyter/tensorflow-notebook`은 CPU 전용 이미지임.

### 8.2 Pod 배치 확인 (K8s 자동 스케줄링)

```bash
kubectl get pods -n ai-team -o wide | grep jupyter
# jupyter-user-01   1/1   Running   0   39s   192.168.202.230   2080ti-gpu-02
```

K8s 스케줄러가 빈 GPU 노드를 자동으로 선택하여 배치. 1차 접속 시 `v100-gpu-01`, 2차 접속 시 `2080ti-gpu-02` — 상황에 따라 다른 노드 선택.

### 8.3 nvidia-smi 확인

```
NVIDIA-SMI 580.126.09   Driver Version: 580.126.09   CUDA Version: 13.0
GPU: NVIDIA GeForce RTX 2080 Ti   Memory: 11264MiB   Temp: 34°C
```

### 8.4 TensorFlow GPU 인식 확인

```bash
python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
# [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]
```

✅ TensorFlow GPU 인식 완료.

---

## 9. 핵심 인사이트

- **설계 순서가 결과를 만든다** — 단순한 툴 설치가 아니라 네임스페이스 격리 → RBAC 권한 설계 → 플랫폼 배포 → GPU 검증의 4단 구조를 순서대로 쌓았기 때문에 팀원 5명이 서로 독립된 CUDA 환경을 가질 수 있었다.
- **이미지 호환성 문제는 분석으로 해결한다** — JupyterHub 이미지 호환성 문제를 만났을 때 당황하지 않고 원인을 분석하여 검증된 이미지(`cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04`)로 교체했다. 실전에서 이미지 문제는 흔하고, 원인 분석 없이 이것저것 시도하면 시간만 낭비된다.
- **K8s 스케줄러 자동 배정 검증까지가 완성이다** — V100과 2080Ti 혼합 환경에서 스케줄러가 GPU를 자동으로 배정하는 구조를 실제 MNIST 학습으로 검증했다. 팀원이 브라우저만으로 GPU 환경에 접근할 수 있는 AI 연구 플랫폼이 완성된 시점이다.

---

## 🗒️ 10. 향후 계획

- [ ] 팀원별 개인 비밀번호 변경 안내
- [ ] YOLOv8 학습 Job 실행 (VisDrone 데이터셋)
- [ ] kubeconfig 파일 팀원 배포 (kubectl 직접 접근 필요 시)

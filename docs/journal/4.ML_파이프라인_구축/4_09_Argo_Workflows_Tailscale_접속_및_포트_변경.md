# Argo Workflows Tailscale 접속 및 포트 변경

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-09
> **작업 목적:** Argo Workflows 서비스 포트를 2746에서 30500으로 변경하고, Tailscale VPN을 통해 외부에서 접속 가능하도록 systemd port-forward 서비스를 구성한다.
> **대상 서버:** master-01 (LB_PUBLIC_IP / Tailscale: TAILSCALE_HOST)
> **작업 환경:** Kubernetes v1.29, Helm, Argo Workflows, Tailscale
> **최종 결과:** `http://TAILSCALE_HOST:30500` 으로 Argo UI Tailscale 접속 완료

---

## 🏗️ 2. 작업 흐름

```
[Helm values 수정 → servicePort: 30500]
        │
        ▼
[helm upgrade → MetalLB IP 30500 포트 반영]
        │
        ▼
[systemd port-forward 서비스 생성]
        │ KUBECONFIG 환경변수 설정 필요 (root 실행 시)
        ▼
[Tailscale → master-01:30500 → Argo UI]
```

---

## 🔧 3. 작업 상세

### 3-1. 기존 Helm values 확인

```bash
helm get values argo-workflows -n argo > argo-values.yaml
cat argo-values.yaml
```

**기존 values:**

```yaml
controller:
  workflowNamespaces:
    - argo
    - ai-team
server:
  authModes:
    - server
  extraArgs:
    - --auth-mode=server
workflow:
  rbac:
    create: true
  serviceAccount:
    create: true
```

---

### 3-2. servicePort 추가 및 Helm upgrade

`argo-values.yaml` 의 `server:` 블록에 포트 설정 추가:

```yaml
controller:
  workflowNamespaces:
    - argo
    - ai-team
server:
  authModes:
    - server
  extraArgs:
    - --auth-mode=server
  servicePort: 30500 # 추가
  serviceType: LoadBalancer # 추가
workflow:
  rbac:
    create: true
  serviceAccount:
    create: true
```

적용:

```bash
helm upgrade argo-workflows argo/argo-workflows \
  -n argo \
  -f argo-values.yaml
```

확인:

```bash
kubectl get svc -n argo
# argo-workflows-server   LoadBalancer   10.111.109.193   LB_PUBLIC_IP   30500:32567/TCP
```

---

### 3-3. systemd port-forward 서비스 생성

```bash
sudo nano /etc/systemd/system/kubectl-argo-forward.service
```

```ini
[Unit]
Description=kubectl port-forward for Argo Workflows
After=network.target

[Service]
Environment="KUBECONFIG=/home/ubuntu/.kube/config"
ExecStart=/usr/bin/kubectl port-forward svc/argo-workflows-server 30500:30500 -n argo --address 0.0.0.0
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

> **⚠️ 핵심 포인트:** `User=root` 로 실행 시 kubeconfig를 자동으로 찾지 못해 `localhost:8080 connection refused` 에러가 발생한다.
> `Environment="KUBECONFIG=/home/ubuntu/.kube/config"` 를 반드시 명시해야 한다.

적용 및 활성화:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kubectl-argo-forward.service
sudo systemctl status kubectl-argo-forward.service
```

**정상 상태 확인:**

```
● kubectl-argo-forward.service - kubectl port-forward for Argo Workflows
     Active: active (running)
```

---

### 3-4. --secure=false 확인

Argo Server는 기본적으로 HTTPS 리다이렉트가 걸릴 수 있다. HTTP 접속을 위해 아래 확인:

```bash
kubectl get deployment argo-workflows-server -n argo -o yaml | grep secure
# - --secure=false  ← 이 옵션이 있어야 HTTP 접속 가능
```

없을 경우 `argo-values.yaml` 의 `extraArgs` 에 추가 후 재배포:

```yaml
server:
  extraArgs:
    - --auth-mode=server
    - --secure=false
```

---

## ✅ 4. 최종 접속 정보

| 접속 방법            | 주소                          |
| -------------------- | ----------------------------- |
| **Tailscale (권장)** | `http://TAILSCALE_HOST:30500` |
| **MetalLB (내부망)** | `http://LB_PUBLIC_IP:30500`   |

---

## 5. 핵심 인사이트

- **Helm upgrade는 워크플로우 리소스를 건드리지 않는다** — WorkflowTemplate, Workflow 실행 기록은 쿠버네티스 CRD로 저장되므로 Helm 재배포와 무관하게 보존된다.
- **systemd에서 root로 kubectl 실행 시 KUBECONFIG 명시 필수** — root 계정은 `/home/ubuntu/.kube/config` 를 자동 참조하지 않아 반드시 `Environment` 로 경로를 지정해야 한다.
- **Argo UI 네임스페이스 필터** — 템플릿과 워크플로우가 `ai-team` 네임스페이스에 있으므로, UI 상단 드롭다운에서 `ai-team` 으로 전환해야 목록이 표시된다.

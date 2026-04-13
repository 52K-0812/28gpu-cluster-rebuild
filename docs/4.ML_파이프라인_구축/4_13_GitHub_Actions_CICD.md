# GitHub Actions CI/CD — Argo Workflow 자동 트리거

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-13
> **작업 목적:** GitHub 프라이빗 레포에 코드를 push하면 Self-hosted Runner가 감지하여 Argo Workflow를 자동으로 트리거하는 CI/CD 파이프라인을 구성한다.
> **대상 서버:** master-01 (LB_PUBLIC_IP)
> **작업 환경:** Kubernetes v1.29, GitHub Actions, Self-hosted Runner, Argo Workflows v4.0.4
> **최종 결과:** git push → GitHub Actions 트리거 (16s) → Argo cicd-pipeline 자동 생성 및 실행 확인

---

## 🏗️ 2. 작업 흐름

```
[개발자 로컬]
    │ git push → main 브랜치
    ▼
[GitHub (프라이빗 레포: 28gpu-cluster-cicd)]
    │ Actions 트리거
    ▼
[Self-hosted Runner (master-01)]
    │ kubectl 실행 (로컬 클러스터 직접 접근)
    ▼
[Argo Workflows]
    │ cicd-pipeline-xxxx Workflow 생성
    ▼
[DAG 파이프라인 실행]
    └→ validate-data → train → evaluate → save-model
```

---

## 📐 3. 아키텍처 설계 결정

### 3.1 레포지토리 분리 전략

| 항목               | 포트폴리오 레포 (퍼블릭) | CI/CD 레포 (프라이빗)             |
| ------------------ | ------------------------ | --------------------------------- |
| 레포명             | 28gpu-cluster-rebuild    | 28gpu-cluster-cicd                |
| 가시성             | Public                   | Private                           |
| 용도               | 포트폴리오 문서, README  | Actions workflow, 트리거 스크립트 |
| Self-hosted Runner | ❌                       | ✅                                |

퍼블릭 레포에 Self-hosted Runner를 붙이면 외부 PR에서 악성 코드 실행 위험이 있다. GitHub 공식 문서도 퍼블릭 레포 Self-hosted Runner를 비권장한다. 포트폴리오 공개성을 유지하면서 보안을 확보하기 위해 레포를 분리했다.

### 3.2 Self-hosted Runner 선택 이유

| 항목          | GitHub-hosted Runner | Self-hosted Runner          |
| ------------- | -------------------- | --------------------------- |
| 클러스터 접근 | ❌ VPN/터널링 필요   | ✅ 로컬 kubectl 직접 사용   |
| 비용          | 프라이빗 레포 유료   | ✅ 무료                     |
| 실무 표준     | 퍼블릭 프로젝트      | ✅ 온프레미스 클러스터 표준 |
| 설정 복잡도   | 낮음                 | 중간                        |

온프레미스 클러스터는 외부에서 K8s API 접근이 막혀있는 경우가 대부분이다. Self-hosted Runner를 master-01에 설치하면 kubectl이 로컬에서 직접 실행되므로 네트워크 문제가 없다.

### 3.3 workflow_dispatch — 수동 트리거 지원

`on: workflow_dispatch`를 추가해 GitHub UI에서 epochs, batch_size, model_version을 직접 입력하고 수동으로 파이프라인을 트리거할 수 있도록 했다. push 자동 트리거와 수동 트리거 두 가지를 모두 지원한다.

---

## 📦 4. Step 1 — 프라이빗 레포 생성

GitHub → New repository

- Name: `28gpu-cluster-cicd`
- Visibility: **Private**
- README, .gitignore, license: 없음

---

## 📦 5. Step 2 — Self-hosted Runner 설치

GitHub → 레포 → Settings → Actions → Runners → New self-hosted runner → Linux

```bash
# master-01에서 실행
mkdir actions-runner && cd actions-runner

# Runner 다운로드
curl -o actions-runner-linux-x64-2.333.1.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-x64-2.333.1.tar.gz

tar xzf ./actions-runner-linux-x64-2.333.1.tar.gz

# Runner 등록 (GitHub 페이지에서 토큰 발급)
./config.sh --url https://github.com/YOUR_GITHUB_USERNAME/28gpu-cluster-cicd \
            --token [GITHUB_발급_토큰]

# 입력값
# Runner group: Enter (Default)
# Runner name: master-01
# labels: Enter (skip)
# work folder: Enter (default)
```

### systemd 서비스로 등록 (재부팅 후에도 유지)

```bash
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

**정상 출력:**

```
● actions.runner.YOUR_GITHUB_USERNAME-28gpu-cluster-cicd.master-01.service
     Active: active (running)
```

> **주의:** `./run.sh`는 포그라운드 실행이라 터미널 닫으면 죽는다. 반드시 systemd 서비스로 등록해야 재부팅 후에도 살아있다.

---

## 📦 6. Step 3 — GitHub Actions Workflow 파일

```bash
cd ~
git clone https://[토큰]@github.com/YOUR_GITHUB_USERNAME/28gpu-cluster-cicd.git
cd 28gpu-cluster-cicd
mkdir -p .github/workflows
```

```bash
cat > .github/workflows/argo-trigger.yaml <<'EOF'
name: YOLOv8 Training Pipeline

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      epochs:
        description: 'Training epochs'
        default: '10'
      batch_size:
        description: 'Batch size'
        default: '16'
      model_version:
        description: 'Model version (v1, v2, ...)'
        default: 'v3'

jobs:
  trigger-argo:
    runs-on: self-hosted
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check cluster status
        run: |
          kubectl get nodes
          kubectl get pods -n ai-team | grep -E "Running|Error"

      - name: Trigger Argo Workflow
        run: |
          EPOCHS=${{ github.event.inputs.epochs || '10' }}
          BATCH=${{ github.event.inputs.batch_size || '16' }}
          VERSION=${{ github.event.inputs.model_version || 'v3' }}

          kubectl create -f - <<YAML
          apiVersion: argoproj.io/v1alpha1
          kind: Workflow
          metadata:
            generateName: cicd-pipeline-
            namespace: ai-team
          spec:
            workflowTemplateRef:
              name: yolov8-dag-pipeline
            arguments:
              parameters:
              - name: epochs
                value: "${EPOCHS}"
              - name: batch-size
                value: "${BATCH}"
              - name: model-version
                value: "${VERSION}"
          YAML

      - name: Get Workflow status
        run: |
          sleep 5
          kubectl get workflows -n ai-team --sort-by=.metadata.creationTimestamp | tail -3
EOF
```

```bash
git config --global user.email "[이메일]"
git config --global user.name "YOUR_GITHUB_USERNAME"
git add .
git commit -m "feat: Add Argo Workflow CI/CD trigger"
git push origin main
```

> **PAT 권한:** push 시 `workflow` scope가 없으면 거부된다. Personal Access Token 발급 시 `repo` + `workflow` 두 개 모두 체크해야 한다.

---

## ✅ 7. 검증 결과

### 7.1 GitHub Actions 실행 결과

| Step                  | 상태             | 소요 시간 |
| --------------------- | ---------------- | --------- |
| Set up job            | ✅ Succeeded     | 2s        |
| Checkout              | ✅ Succeeded     | 1s        |
| Check cluster status  | ✅ Succeeded     | 1s        |
| Trigger Argo Workflow | ✅ Succeeded     | 0s        |
| Get Workflow status   | ✅ Succeeded     | 5s        |
| Complete job          | ✅ Succeeded     | 0s        |
| **전체**              | ✅ **Succeeded** | **16s**   |

### 7.2 Argo Workflow 자동 생성 확인

```bash
kubectl get workflows -n ai-team --sort-by=.metadata.creationTimestamp | tail -3
```

```
yolov8-dag-pipeline-fxgn5   Succeeded   5d
yolov8-dag-pipeline-fwd6q   Succeeded   160m
cicd-pipeline-lk7wf         Running     2m
```

`cicd-pipeline-lk7wf` — push 트리거로 자동 생성된 Workflow 확인.

### 7.3 Argo UI 확인

`http://TAILSCALE_HOST:30500` → `cicd-pipeline-lk7wf` DAG 실시간 실행 확인

```
✅ validate-data → 🔵 train (실행 중) → evaluate → save-model
```

---

## 🛠️ 8. 트러블슈팅

### 문제 1: workflow scope 없이 push 거부

```
refusing to allow a Personal Access Token to create or update workflow
`.github/workflows/argo-trigger.yaml` without `workflow` scope
```

**원인:** PAT에 `workflow` 권한이 없음.

**해결:** GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic) → `repo` + `workflow` 두 개 체크

```bash
git remote set-url origin https://YOUR_GITHUB_USERNAME:[새토큰]@github.com/YOUR_GITHUB_USERNAME/28gpu-cluster-cicd.git
git push origin main
```

### 문제 2: git push 시 Authentication failed

```
remote: Invalid username or token.
fatal: Authentication failed
```

**원인:** GitHub은 HTTPS 클론 시 일반 비밀번호 인증 불가. PAT를 URL에 직접 포함해야 함.

**해결:**

```bash
git remote set-url origin https://YOUR_GITHUB_USERNAME:[PAT]@github.com/YOUR_GITHUB_USERNAME/28gpu-cluster-cicd.git
```

---

## 💡 9. 핵심 인사이트

**온프레미스 클러스터 CI/CD는 Self-hosted Runner가 표준이다.** GitHub-hosted Runner는 외부 네트워크에서 온프레미스 K8s API에 접근할 수 없다. Self-hosted Runner를 Control Plane 노드에 설치하면 kubectl이 로컬에서 직접 실행되어 네트워크 문제가 없다. 실무 MLOps 팀도 내부 GitLab/GitHub Enterprise + Self-hosted Runner 조합을 표준으로 사용한다.

**퍼블릭 레포와 프라이빗 레포를 분리하면 보안과 포트폴리오 공개성을 동시에 확보할 수 있다.** 퍼블릭 레포에 Self-hosted Runner를 붙이면 외부 PR에서 악성 코드가 실행될 수 있다. 포트폴리오는 퍼블릭으로 유지해 채용담당자가 바로 열람할 수 있도록 하고, CI/CD 워크플로우는 프라이빗 레포에서 관리한다.

**workflow_dispatch로 수동 파라미터 입력을 지원하면 실험 재현성이 높아진다.** epochs, batch_size, model_version을 GitHub UI에서 직접 입력하고 트리거할 수 있어, 코드 수정 없이 다양한 학습 조건을 실험할 수 있다.

# Current Architecture

> **최종 업데이트:** 2026-04-13
> **상태:** 운영 중
> **목적:** 클러스터의 현재 유효한 구조를 한 장으로 정리한 문서. 날짜별 작업 기록은 `journal/`, 운영 절차는 `runbooks/`, 장애 기록은 `incidents/` 참조.

---

## 노드 구성

| 노드            | 역할                 | GPU        | IP                                      |
| ------------- | ------------------ | ---------- | --------------------------------------- |
| master-01     | Control Plane      | —          | `MASTER-IP`                             |
| master-02     | Worker (시스템 파드 전담) | —          | `WORKER-IP-02`                          |
| v100-gpu-01   | Worker (학습 전용)     | V100 × 4   | `WORKER-IP-03`                          |
| 2080ti-gpu-02 | Worker             | 2080Ti × 8 | `WORKER-IP-04`                          |
| 2080ti-gpu-03 | Worker             | 2080Ti × 7 | `WORKER-IP-05`                          |
| 2080ti-gpu-04 | Worker (서빙 전용)     | 2080Ti × 8 | `WORKER-IP-06`                          |
| NAS (nas-01)  | 스토리지               | —          | `MASTER-IP` (1G) / `10.10.10.157` (10G) |

- **K8s 버전:** v1.29.15
- **OS:** Ubuntu 22.04.5 LTS
- **총 GPU:** V100 4장 + 2080Ti 23장 = 27장
- **Container Runtime:** containerd
- **CNI:** Calico v3.27

---

## 네트워크 구조

```
[외부 접속]
    Tailscale VPN (TAILSCALE-IP) → master-01

[관리망 — 1GbE, MASTER-IP 대역]
    master-01 ↔ master-02 ↔ GPU 노드 ↔ NAS
    K8s API, NFS 제어, 일반 트래픽

[데이터망 — 10GbE, 10.10.10.x]
    GPU 노드(153~156) ↔ NAS(157)
    학습 데이터 전송, NFS 실제 I/O
    목표 속도: 9Gbps 이상
```

**핵심 설계:** master-01은 1G망만 연결됨. NFS Provisioner 제어는 1G 주소(`MASTER-IP`)로, GPU 노드의 실제 데이터 접근은 10G망(`10.10.10.x`)으로 분리.

---

## 스토리지

| 항목 | 내용 |
|---|---|
| NAS 용량 | 28TB (`/data`) |
| StorageClass | `nfs-client` (default) |
| NFS Provisioner | master-01에 고정 배치 (nodeSelector) |
| 주요 경로 | `/data/datasets/` — 학습 데이터 및 모델 |
| | `/data/mlflow-artifacts/` — MLflow 아티팩트 |
| | `/data/backups/etcd/` — etcd 스냅샷 (7일 보존) |

---

## 서비스 현황

| 서비스                 | 네임스페이스     | 접속 주소                       | 상태  |
| ------------------- | ---------- | --------------------------- | --- |
| JupyterHub          | ai-team    | `http://TAILSCALE-IP:30000` | ✅   |
| MLflow              | mlflow     | `http://TAILSCALE-IP:30010` | ✅   |
| Grafana             | monitoring | `http://TAILSCALE-IP:30300` | ✅   |
| Prometheus          | monitoring | `http://TAILSCALE-IP:30310` | ✅   |
| Portainer           | portainer  | `http://TAILSCALE-IP:30320` | ✅   |
| Filebrowser (NAS)   | monitoring | `http://TAILSCALE-IP:30340` | ✅   |
| Argo Workflows UI   | argo       | `http://TAILSCALE-IP:30500` | ✅   |
| FastAPI 서빙 (YOLOv8) | ai-team    | `http://TAILSCALE-IP:30600` | ✅   |

---

## JupyterHub

- **Helm Chart:** 4.3.3
- **이미지:** `cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04`
- **인증:** DummyAuthenticator (현재) → GitHub OAuth 교체 예정
- **GPU 스케줄링:** V100 / 2080Ti 자동 배정 (nodeSelector + label)
- **팀원 계정:** member-01 ~ member-05 (RBAC, ai-team 네임스페이스 격리)
- **접속:** Tailscale → `http://TAILSCALE-IP:30000`
- **master-01 taint:** `node-role.kubernetes.io/control-plane:NoSchedule` (파드 스케줄링 차단)
- **master-02 배치 파드:** Prometheus, Grafana, Portainer, Alertmanager, JupyterHub hub/proxy

---

## ML 파이프라인 (Argo Workflows)

```
[Git push → GitHub Actions (Self-hosted Runner on master-01)]
    │ 16s
    ▼
[Argo cicd-pipeline 자동 트리거]
    │
    ▼
[DAG 파이프라인]
    validate-data → train → evaluate → save-model

[학습 환경]
    - 모델: YOLOv8n
    - 데이터셋: VisDrone (드론 시점 객체 탐지)
    - GPU: V100 × 4 (DDP)
    - 결과: visdrone-v1.pt / v2.pt → NAS /data/datasets/models/
```

---

## MLflow

- **Backend:** PostgreSQL (K8s Deployment + NFS PVC)
- **Artifact 저장소:** NFS PVC → NAS `/data/mlflow-artifacts/`
- **Argo 연동:** train 단계에서 `mlflow.log_params()`, evaluate 단계에서 `mlflow.log_metrics()` + `best.pt` 자동 기록
- **기록 규모:** params 105개 + metrics 7개 (학습 실행당)
- **접속:** `http://TAILSCALE-IP:30010`

---

## 모델 서빙 (FastAPI)

- **모델:** YOLOv8n COCO pretrained (80 클래스)
- **배포 방식:** K8s Deployment (ai-team 네임스페이스)
- **GPU:** 2080Ti × 1 (2080ti-gpu-04 고정)
- **추론 속도:** 77ms
- **엔드포인트:**
  - `GET /` — 웹 UI (웹캠 캡처 / 이미지 업로드)
  - `POST /predict` — 이미지 추론 → bbox + 클래스 + confidence 반환
  - `GET /health` — 서버 상태
- **접속:** `http://TAILSCALE-IP:30600`

> **현재 한계:** HTTP 서빙 → 웹캠 `getUserMedia()` Chrome 플래그 예외 필요. HTTPS(Ingress + TLS) 적용 예정.

---

## CI/CD

```
[개발자] → git push → main
    │
[GitHub Actions (Private Repo: 28gpu-cluster-cicd)]
    │ Self-hosted Runner (master-01)
    │ 트리거 시간: ~16s
    ▼
[Argo Workflows — cicd-pipeline 자동 생성]
    └→ validate-data → train → evaluate → save-model
```

- **Runner 위치:** master-01 (로컬 kubectl 직접 접근)
- **수동 트리거:** `workflow_dispatch` (epochs, batch_size, model_version 파라미터 입력 가능)
- **레포 분리 이유:** 퍼블릭 포트폴리오 레포에 Self-hosted Runner를 붙이면 외부 PR 악성 코드 실행 위험 → 프라이빗 레포 분리

---

## 모니터링

| 항목 | 내용 |
|---|---|
| Prometheus | kube-prometheus-stack (Helm) |
| Grafana | GPU 대시보드 #12239 (DCGM Exporter) |
| GPU 메트릭 | DCGM Exporter — 온도 / 전력 / 메모리 / 클럭 |
| 알람 | Alertmanager → Gmail SMTP |
| 알람 룰 | GPU 온도 80°C 초과 / 메모리 사용률 90% 초과 / GPU Idle |

---

## 백업 / DR

- **방식:** 호스트 crontab (K8s CronJob 아님 — 클러스터 장애 시 CronJob도 불가하므로)
- **스케줄:** 매일 02:00
- **저장:** NAS `/data/backups/etcd/` (10GbE)
- **보존:** 7일 자동 삭제
- **DR 검증:** 복구 절차 문서화 + 스냅샷 무결성 검증 완료 (2026-04-13)
- **복원 절차:** `runbooks/runbook_etcd_restore.md` 참조

---

## 미해결 과제 / 한계

| 항목 | 현황 | 비고 |
|---|---|---|
| HTTPS / Ingress | 미적용 | 웹캠 HTTP 제약 존재. TLS 적용 예정 |
| JupyterHub 인증 | DummyAuthenticator | GitHub OAuth 교체 예정 |
| 이미지 빌드 | pip install 런타임 설치 | 전용 이미지 빌드 환경 필요 |
| ResourceQuota | 미적용 | 팀원 GPU 점유 제한 없음 |
| K8s 업그레이드 | v1.29 (EOL 예정) | v1.31 업그레이드 문서화 완료, 실작업 미진행 |
| Canary 배포 | 미적용 | 다음 작업 예정 |

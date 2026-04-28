# Current Architecture

> **최종 업데이트:** 2026-04-28
> **상태:** 운영 중
> **목적:** 클러스터의 현재 유효한 구조를 한 장으로 정리한 문서. 날짜별 작업 기록은 `journal/`, 운영 절차는 `runbooks/`, 장애 기록은 `incidents/` 참조.

---

## 1. 🖥️ 노드 구성

| 노드            | 역할                 | GPU        | IP              |
| ------------- | ------------------ | ---------- | --------------- |
| master-01     | Control Plane        | —          | `<MASTER-IP>`   |
| master-02     | Worker (시스템 파드 전담) | —          | `<WORKER-IP-02>` |
| v100-gpu-01   | Worker (학습 전용)     | V100 × 4   | `<WORKER-IP-03>` |
| 2080ti-gpu-02 | Worker             | 2080Ti × 8 | `<WORKER-IP-04>` |
| 2080ti-gpu-03 | Worker             | 2080Ti × 7 | `<WORKER-IP-05>` |
| 2080ti-gpu-04 | Worker             | 2080Ti × 8 | `<WORKER-IP-06>` |
| NAS (nas-01)  | 스토리지               | —          | `<MASTER-IP>` (1G) / `10.10.10.157` (10G) |

- **K8s 버전:** v1.29.15
- **OS:** Ubuntu 22.04.5 LTS
- **총 GPU:** V100 4장 + 2080Ti 23장 = 27장
- **Container Runtime:** containerd
- **CNI:** Calico v3.27

> **설계 근거:** 3_31 네트워크 장애(MetalLB ARP 충돌 · Calico 연쇄 장애) 이후 SoC(Separation of Concerns) 원칙 적용. master-01은 Control Plane 전용, master-02는 시스템 파드 전담으로 역할 고정.

---

## 2. 🌐 네트워크 구조

```
[외부 접속]
    Tailscale VPN (<TAILSCALE-IP>) → master-01
    NGINX Ingress (<LB-INGRESS-IP>) → 캠퍼스 내부망 직접 접속

[관리망 — 1GbE, <CAMPUS-IP-RANGE>]
    master-01 ↔ master-02 ↔ GPU 노드 ↔ NAS
    K8s API, NFS 제어, 일반 트래픽

[데이터망 — 10GbE, 10.10.10.x]
    GPU 노드(153~156) ↔ NAS(157)
    학습 데이터 전송, NFS 실제 I/O
    목표 속도: 9Gbps 이상
```

**핵심 설계:**
- master-01은 1G망만 연결됨. NFS Provisioner 제어는 1G 주소로, GPU 노드의 실제 데이터 접근은 10G망(10.10.10.x)으로 분리.
- MetalLB L2 모드: IP Pool은 캠퍼스 스위치에 허용 등록된 대역만 사용. 노드 자체 IP를 Pool에 포함하지 않음(3_31 장애 교훈).

**MetalLB IP 배정:**

| IP | 용도 |
|---|---|
| `<LB-JUPYTERHUB-IP>` | JupyterHub proxy-public (현재 Ingress 백엔드 전환으로 직접 노출 안 함) |
| `<LB-ARGO-IP>` | Argo Workflows UI |
| `<LB-INGRESS-IP>` | NGINX Ingress Controller — 외부 진입점 일원화 |

---

## 3. 💾 스토리지

| 항목 | 내용 |
|---|---|
| NAS 용량 | 28TB (`/data`) |
| StorageClass | `nfs-client` (default) |
| NFS Provisioner | master-01에 고정 배치 (nodeSelector) |
| 주요 경로 | `/data/datasets/` — 학습 데이터 및 모델 |
| | `/data/mlflow-artifacts/` — MLflow 아티팩트 |
| | `/data/backups/etcd/` — etcd 스냅샷 (7일 보존) |

---

## 4. 🔌 서비스 현황

### 외부 사용자 서비스 (Ingress 경유 HTTPS)

| 서비스 | 네임스페이스 | 접속 주소 | 인증 | 상태 |
|---|---|---|---|---|
| JupyterHub | ai-team | `https://hub.<LB-INGRESS-IP>.nip.io` | GitHub OAuth | ✅ |
| YOLOv8 Serving | ai-team | `https://serving.<LB-INGRESS-IP>.nip.io` | 없음 (보호 정책 적용 예정) | ✅ |

### 관리자 서비스 (NodePort — 인증 강화 예정)

| 서비스 | 네임스페이스 | 접속 방식 | 상태 |
|---|---|---|---|
| Grafana | monitoring | NodePort `<MASTER-IP>:30300` | ✅ |
| Prometheus | monitoring | NodePort `<MASTER-IP>:30310` | ✅ (내부 전용) |
| Portainer | portainer | NodePort `<MASTER-IP>:30320` | ✅ |
| Filebrowser (NAS) | monitoring | NodePort `<MASTER-IP>:30340` | ✅ |
| MLflow | mlflow | NodePort `<MASTER-IP>:30010` | ✅ |
| Argo Workflows UI | argo | `<LB-ARGO-IP>:30500` | ✅ |

> **보안 현황:** 관리자 서비스는 현재 캠퍼스 내부망 + Tailscale 접근만 허용. Ingress Phase B에서 Basic Auth 적용 예정.

---

## 5. 📓 JupyterHub

- **Helm Chart:** 4.3.3
- **이미지:** `cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04`
- **인증:** GitHubOAuthenticator (oauthenticator 17.4.0) — 2026-04-27 전환 (DummyAuthenticator 제거)
- **GPU 스케줄링:** V100 / 2080Ti 자동 배정 (nodeSelector + label)
- **접속:** `https://hub.<LB-INGRESS-IP>.nip.io` (Ingress + cert-manager TLS)
- **master-01 taint:** `node-role.kubernetes.io/control-plane:NoSchedule` (파드 스케줄링 차단)
- **master-02 배치 파드:** JupyterHub hub/proxy, Prometheus, Grafana, Portainer, Alertmanager, MLflow, Argo Controller

### 인증 구성

| 항목 | 내용 |
|---|---|
| Authenticator | `GitHubOAuthenticator` |
| Client ID / Secret | K8s Secret `jupyterhub-github-oauth` (ai-team ns) — 환경변수 주입 |
| OAuth Callback URL | `https://hub.<LB-INGRESS-IP>.nip.io/hub/oauth_callback` |
| 관리자 계정 | `52K-0812` |
| 허용 사용자 | `52K-0812`, `Yeeeho`, `Da-Woon-J` |
| 이전 인증 방식 | ~~DummyAuthenticator~~ — 2026-04-27 제거 |

### 현재 PVC 구성 (ai-team ns)

| PVC | 용도 |
|---|---|
| `claim-x-52k-0812---63033fa0` | 52K-0812 홈 디렉터리 (50Gi) |
| `claim-yeeeho` | Yeeeho 홈 디렉터리 (50Gi) |
| `claim-da-woon-j` | Da-Woon-J 홈 디렉터리 (50Gi) — 최초 서버 기동 시 자동 생성 예정 |
| `hub-db-dir` | JupyterHub SQLite DB (1Gi) |
| `ai-datasets` | 공유 데이터셋 (100Gi, RWX) |
| `mlflow-artifacts-pvc` | MLflow 아티팩트 (100Gi, RWX) |
| `nfs-datasets-pvc` | NAS 전체 마운트 (500Gi, RWX) |

---

## 6. 🌐 Ingress + TLS

- **컨트롤러:** NGINX Ingress Controller (master-02 nodeSelector 고정 배치)
- **TLS:** cert-manager (cluster-ca self-signed — nip.io 도메인. Let's Encrypt HTTP-01은 캠퍼스 외부 도달성 제약으로 중단, DNS-01 방식으로 별도 Phase 예정)
- **외부 진입점:** MetalLB `<LB-INGRESS-IP>` (L2 ARP, 캠퍼스 내부망)
- **도메인 패턴:** `{서비스}.{LB-INGRESS-IP-DASHED}.nip.io`
- **완료 일자:** 2026-04-27
- **효과:** HTTPS 적용으로 웹캠 getUserMedia() 제약 해소 + GitHub OAuth 선결조건 충족

현재 Ingress 라우팅 대상:

| 도메인 | 백엔드 서비스 |
|---|---|
| `hub.<LB-INGRESS-IP>.nip.io` | JupyterHub proxy-public |
| `serving.<LB-INGRESS-IP>.nip.io` | FastAPI YOLOv8 서빙 |

---

## 7. 🔄 ML 파이프라인 (Argo Workflows)

```
[Git push → GitHub Actions (Self-hosted Runner on master-01)]
    │ ~16s
    ▼
[Argo cicd-pipeline 자동 트리거]
    │
    ▼
[DAG 파이프라인]
    validate-data → train → evaluate → save-model
                                           │
                                           ▼
                                    MLflow Registry 등록
                                    alias "champion" 지정
                                           │
                                           ▼
                                    FastAPI /predict 반영

[학습 환경]
    - 모델: YOLOv8n
    - 데이터셋: VisDrone (드론 시점 객체 탐지)
    - GPU: V100 × 4 (DDP)
    - 결과: visdrone-v{version}.pt → NAS /data/datasets/models/
```

---

## 8. 📊 MLflow

- **Backend:** PostgreSQL (K8s Deployment + NFS PVC)
- **Artifact 저장소:** NFS PVC → NAS `/data/mlflow-artifacts/`
- **배포 방식:** kubectl 직접 배포 (Helm 비관리)
- **Argo 연동:**
  - train 단계: `mlflow.log_params()` (autolog — 105개 자동 기록)
  - evaluate 단계: `mlflow.log_metrics()` + `best.pt` artifact 연동 로직
  - save-model 단계: Registry 등록 + alias "champion" 지정
- **Model Registry:** `visdrone-yolov8` 모델명, alias 기반 버전 관리
- **접속:** NodePort `<MASTER-IP>:30010` (관리자 전용, Ingress Phase B 대상)

---

## 9. 🚀 모델 서빙 (FastAPI)

- **배포 방식:** K8s Deployment (ai-team 네임스페이스)
- **배포 이미지:** `1jkim/yolov8-serving:v1` (DockerHub public) — 2026-04-17 전환
- **GPU:** 2080Ti × 1
- **추론 속도:** 77ms
- **nodeSelector:** `gpu-type=2080ti` (hostname 고정 없음 — 2080Ti 풀 전체 자동 스케줄)
- **접속:** `https://serving.<LB-INGRESS-IP>.nip.io` (Ingress + TLS)

### 엔드포인트

| 엔드포인트 | 모델 | 설명 |
|---|---|---|
| `GET /` | — | 웹 UI |
| `POST /predict-demo` | YOLOv8n COCO (80 클래스) | 웹캠 데모용 |
| `POST /predict` | MLflow champion (VisDrone) | E2E MLOps 운영용 |
| `POST /reload-champion` | — | Pod 재시작 없이 최신 champion 반영 |
| `GET /health` | — | 서버 상태 + champion 버전 확인 |

### 웹 UI

- 루트(`/`) 페이지에 웹 UI 제공
- 파일 업로드 / 웹캠 캡처 모드 지원
- `/predict-demo` / `/predict` 엔드포인트 선택 가능
- 헤더 상태 배지로 `COCO · 80cls` / `champion · v{버전}` 표시
- 추론 결과 이미지, 탐지 카드, confidence bar 표시

### health 응답 예시

```json
{
  "status": "ok",
  "demo_model": "yolov8n-coco (80 classes)",
  "champion_ready": true,
  "champion_version": "7"
}
```

### champion 모델 로드 방식

```
MLflow alias "champion" 조회
    └→ version 번호 확인
    └→ 1순위: NAS /mnt/datasets/models/visdrone-v{version}.pt 직접 로드
    └→ 2순위: MLflow artifact 경로 폴백 (예비 경로)
```

### 볼륨 마운트

| 마운트 경로 | PVC | 용도 |
|---|---|---|
| `/mnt/datasets` | nfs-datasets-pvc | 모델 파일 직접 접근 |
| `/mnt/mlflow-artifacts` | mlflow-artifacts-pvc (ai-team) | artifact 폴백 |

---

## 10. ⚙️ CI/CD

```
[개발자] → git push → main
    │
[GitHub Actions (Private Repo)]
    │ Self-hosted Runner (master-01)
    │ 트리거 시간: ~16s
    ▼
[Argo Workflows — cicd-pipeline 자동 생성]
    └→ validate-data → train → evaluate → save-model
                                              └→ MLflow champion 자동 갱신
```

- **Runner 위치:** master-01 (로컬 kubectl 직접 접근)
- **레포:** Private 레포 분리 운영 (퍼블릭 레포에 Self-hosted Runner 금지 — 외부 PR 악성 코드 실행 위험)
- **수동 트리거:** `workflow_dispatch` (epochs, batch_size, model_version 파라미터 입력 가능)

---

## 11. 📡 모니터링

| 항목 | 내용 |
|---|---|
| Prometheus | kube-prometheus-stack (Helm, **chart 82.15.1 고정**) |
| Grafana | GPU 대시보드 #12239 (DCGM Exporter), PVC: `grafana-pvc` (10Gi) |
| GPU 메트릭 | DCGM Exporter — 온도 / 전력 / 메모리 / 클럭 |
| 알람 | Alertmanager → Gmail SMTP |
| 알람 룰 | GPU 온도 85°C 초과 / 메모리 사용률 95% 초과 / GPU Idle |

> **운영 주의:** monitoring Helm upgrade 시 반드시 `--version 82.15.1` 플래그로 chart 버전을 고정할 것. 미지정 시 chart 자동 업그레이드로 Grafana NodePort(30300), Prometheus NodePort(30310), Grafana PVC 설정이 재렌더링되어 초기화될 수 있음. (INC-2026-04-28 참조)

---

## 12. 🛡️ 워크로드 우선순위 (PriorityClass)

2026-04-28 도입. 자원 경합 시 시스템 파드 보호 및 서빙/학습 워크로드 계층화를 위해 4계층 체계를 운영한다.

| PriorityClass | Value | 적용 대상 |
|---|---|---|
| `system-cluster-critical` | 2000000000 | cert-manager, argo-workflows controller/server, Prometheus, Alertmanager, Grafana, kube-state-metrics, Prometheus Operator, ingress-nginx, JupyterHub hub/proxy/user-scheduler, MLflow server/postgres |
| `serving-critical` | 1000000 | yolov8-serving **(Phase C 적용 예정)** |
| `training-normal` | 100 | Argo workflow steps **(Phase C 적용 예정)** |
| (default = 0) | 0 | JupyterHub singleuser, 기타 |

**보류 항목:**
- `prometheus-node-exporter` (DaemonSet) — `system-node-critical`이 더 적합, 별도 작업 예정
- `filebrowser` — kubectl 직접 배포, 별도 작업 예정

---

## 13. 🔒 백업 / DR

- **방식:** 호스트 crontab (K8s CronJob 아님 — 클러스터 장애 시 CronJob도 불가하므로)
- **스케줄:** 매일 02:00
- **저장:** NAS `/data/backups/etcd/` (10GbE)
- **보존:** 7일 자동 삭제
- **DR 검증:** 복구 절차 문서화 + 스냅샷 무결성 검증 완료 (2026-04-13)
- **복원 절차:** `runbooks/runbook_etcd_restore.md` 참조

---

## 14. ⚠️ 미해결 과제 / 한계

| 항목 | 현황 | 비고 |
|---|---|---|
| HTTPS / Ingress | ✅ 적용 완료 | hub / serving Ingress + TLS (nip.io) 운영 중 |
| JupyterHub 인증 | ✅ GitHub OAuth | GitHubOAuthenticator 전환 완료 (2026-04-27) |
| PriorityClass 도입 | ✅ 완료 (2026-04-28) | Phase A/B/B-1 완료. Phase C~E 진행 예정 |
| **ResourceQuota** | **미적용** | **Phase D/E 예정. 팀원 GPU 점유 제한 없음** |
| 관리자 서비스 인증 | NodePort 노출, 인증 없음 | Ingress Phase B에서 Basic Auth 적용 예정 (Grafana, MLflow, Argo, Filebrowser) |
| YOLOv8 Serving 보호 | 인증 없음 | Basic Auth 또는 API Key 적용 예정 |
| Portainer 외부 노출 | NodePort | Tailscale 경유 고정 유지 (Ingress 노출 계획 없음) |
| Grafana datasource 이중 정의 | 잠재 위험 | `additionalDataSources`와 `sidecar.datasources` 양쪽에 Prometheus 정의됨. chart 업그레이드 시 재발 가능. (INC-2026-04-28) |
| monitoring Helm chart 버전 | 82.15.1 고정 운영 | 84.1.2 업그레이드 시 NodePort/PVC 재렌더링 문제 확인됨. 업그레이드 전 values 정리 필요 |
| Da-Woon-J 로그인 검증 | 미검증 | 첫 로그인 시 PVC `claim-da-woon-j` 자동 생성 예정 |
| node-exporter PriorityClass | 미적용 | `system-node-critical` 적용 예정 |
| JupyterHub backup 파일 권한 | 664 (권고: 600) | `~/backup/jupyterhub/*.yaml` — proxy.secretToken 평문 포함 |
| K8s 업그레이드 | v1.29 (EOL 예정) | v1.31 업그레이드 runbook 작성 예정, 실작업 미진행 |
| buildkitd 서비스 등록 | nohup 백그라운드 | master-01 재부팅 시 재실행 필요 |

---

## 변경 이력

| 날짜 | 변경 내용 |
|---|---|
| 2026-03-31 | master-01 NoSchedule taint 적용, master-02 시스템 파드 전담 설계 확정 |
| 2026-04-13 | MLflow, GitHub Actions CI/CD, etcd DR 검증 반영 |
| 2026-04-15 | FastAPI champion serving, MLflow alias 기반 운영 흐름 반영 |
| 2026-04-17 | 서빙 이미지 DockerHub 전환 (`1jkim/yolov8-serving:v1`), 2080ti-gpu-04 hostname nodeSelector 고정 해제 |
| 2026-04-27 | Ingress + TLS 완료 (NGINX Ingress, cert-manager, cluster-ca self-signed, MetalLB LB-INGRESS-IP), GitHub OAuth 완료 (DummyAuth 제거), 서비스 분류 체계 정비 |
| 2026-04-28 | PriorityClass 4계층 도입 (Phase A/B/B-1 완료), monitoring chart 버전 82.15.1 고정 운영 결정, Grafana PVC 복구, INC-2026-04-28 발생 및 복구 |

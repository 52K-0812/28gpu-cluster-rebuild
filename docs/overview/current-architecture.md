# Current Architecture

> **최종 업데이트:** 2026-04-27
> **상태:** 운영 중
> **목적:** 클러스터의 현재 유효한 구조를 한 장으로 정리한 문서. 날짜별 작업 기록은 `journal/`, 운영 절차는 `runbooks/`, 장애 기록은 `incidents/` 참조.

---

## 1. 🖥️ 노드 구성

| 노드            | 역할                 | GPU        | IP                                      |
| ------------- | ------------------ | ---------- | --------------------------------------- |
| master-01     | Control Plane      | —          | `MASTER-IP`                             |
| master-02     | Worker (시스템 파드 전담) | —          | `WORKER-IP-02`                          |
| v100-gpu-01   | Worker (학습 전용)     | V100 × 4   | `WORKER-IP-03`                          |
| 2080ti-gpu-02 | Worker             | 2080Ti × 8 | `WORKER-IP-04`                          |
| 2080ti-gpu-03 | Worker             | 2080Ti × 7 | `WORKER-IP-05`                          |
| 2080ti-gpu-04 | Worker             | 2080Ti × 8 | `WORKER-IP-06`                          |
| NAS (nas-01)  | 스토리지               | —          | `MASTER-IP` (1G) / `10.10.10.157` (10G) |

- **K8s 버전:** v1.29.15
- **OS:** Ubuntu 22.04.5 LTS
- **총 GPU:** V100 4장 + 2080Ti 23장 = 27장
- **Container Runtime:** containerd
- **CNI:** Calico v3.27

> **2026-04-17 변경:** `2080ti-gpu-04` 역할에서 "서빙 전용" 표기 제거. 서빙 이미지 DockerHub 등록 후 hostname nodeSelector 해제로 2080Ti 풀 전체에서 자동 스케줄링.

---

## 2. 🌐 네트워크 구조

```
[외부 접속]
    Tailscale VPN (TAILSCALE-IP) → master-01

[Ingress Gateway — 2026-04-27 신규]
    NGINX Ingress Controller (master-02 고정)
    LoadBalancer IP: INGRESS-LB-IP
    TLS: cluster-ca self-signed (cert-manager)
    host 기반 라우팅 (nip.io wildcard DNS)

[관리망 — 1GbE, MASTER-IP 대역]
    master-01 ↔ master-02 ↔ GPU 노드 ↔ NAS
    K8s API, NFS 제어, 일반 트래픽

[데이터망 — 10GbE, 10.10.10.x]
    GPU 노드(153~156) ↔ NAS(157)
    학습 데이터 전송, NFS 실제 I/O
    목표 속도: 9Gbps 이상
```

**핵심 설계:** master-01은 1G망만 연결됨. NFS Provisioner 제어는 1G 주소(`MASTER-IP`)로, GPU 노드의 실제 데이터 접근은 10G망(`10.10.10.x`)으로 분리.

### MetalLB IP 할당 현황

| IP | 서비스 | 노출 방식 |
|---|---|---|
| `JUPYTERHUB-LB-IP` | proxy-public (JupyterHub LoadBalancer) | 기존 직접 노출 (fallback) |
| `ARGO-LB-IP` | argo-workflows-server | NodePort 30500 + LoadBalancer |
| `MLFLOW-LB-IP` | mlflow-server | NodePort 30010 + LoadBalancer |
| `INGRESS-LB-IP` | **ingress-nginx-controller (신규)** | host 기반 통합 게이트웨이 |

> ⛔ MetalLB L2 IP는 캠퍼스 LAN 내부에서만 ARP 도달 가능. 외부 접속은 Tailscale 또는 학내 클라이언트만 가능.

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

### 4-1. Ingress 기반 (신규, host 라우팅)

| 서비스 | 네임스페이스 | 외부 host | TLS | 그룹 |
|---|---|---|---|---|
| YOLOv8 FastAPI | ai-team | `https://serving.INGRESS-LB-IP.nip.io` | self-signed | 팀원 |
| JupyterHub | ai-team | `https://hub.INGRESS-LB-IP.nip.io` | self-signed | 팀원 |

### 4-2. 기존 직접 노출 (fallback 경로 유지)

| 서비스 | 네임스페이스 | 학내 접속 | Tailscale 접속 | 상태 |
|---|---|---|---|---|
| JupyterHub | ai-team | `http://JUPYTERHUB-LB-IP` (LB) | `http://TAILSCALE-IP:30000` | ✅ |
| MLflow | mlflow | `http://MLFLOW-LB-IP:5000` (LB) | `http://TAILSCALE-IP:30010` | ✅ |
| Grafana | monitoring | — | `http://TAILSCALE-IP:30300` | ✅ |
| Prometheus | monitoring | — | `http://TAILSCALE-IP:30310` | ✅ |
| Portainer | portainer | — | `http://TAILSCALE-IP:30320` | ✅ |
| Filebrowser (NAS) | monitoring | — | `http://TAILSCALE-IP:30340` | ✅ |
| Argo Workflows UI | argo | `http://ARGO-LB-IP:2746` (LB) | `http://TAILSCALE-IP:30500` | ✅ |
| FastAPI 서빙 (YOLOv8) | ai-team | — | `http://TAILSCALE-IP:30600` | ✅ |

> **Phase A 완료 후 정책:** Ingress 도입과 동시에 기존 NodePort/LoadBalancer 경로는 모두 fallback으로 유지. Ingress 장애 시 복구 채널 + Tailscale 경유 접근 안정성 확보 목적. Phase C에서 정리 예정.

### 4-3. 향후 Ingress 통합 예정 (Phase B)

| 그룹    | 서비스            | 예정 host                          | 인증 정책                          |
| ----- | -------------- | -------------------------------- | ------------------------------ |
| 팀원    | (현재 모두 통합 완료)  | —                                | —                              |
| 관리자   | Grafana        | `grafana.INGRESS-LB-IP.nip.io`   | Basic Auth + Grafana admin     |
| 관리자   | Argo Workflows | `argo.INGRESS-LB-IP.nip.io`      | Basic Auth + Argo 토큰           |
| 관리자   | Portainer      | `portainer.INGRESS-LB-IP.nip.io` | Basic Auth + Portainer admin   |
| 관리자   | Filebrowser    | `files.INGRESS-LB-IP.nip.io`     | Basic Auth + Filebrowser admin |
| 제한 공개 | MLflow         | `mlflow.INGRESS-LB-IP.nip.io`    | 접근 정책 검토 후 결정                  |
| 미노출   | Prometheus     | —                                | Grafana 데이터소스로만 접근             |

---

## 5. 📓 JupyterHub

- **Helm Chart:** 4.3.3
- **이미지:** `cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04`
- **인증:** 교육용 임시 인증 (현재) → GitHub OAuth 교체 예정 (HTTPS 선결 조건 충족 완료)
- **GPU 스케줄링:** V100 / 2080Ti 자동 배정 (nodeSelector + label)
- **팀원 계정:** 팀원 계정 5개 (RBAC, ai-team 네임스페이스 격리)
- **master-01 taint:** `node-role.kubernetes.io/control-plane:NoSchedule` (파드 스케줄링 차단)
- **master-02 배치 파드:** Prometheus, Grafana, Portainer, Alertmanager, JupyterHub hub/proxy

### 접속 경로 (2026-04-27 갱신)

| 경로 | 용도 | 인증 |
|---|---|---|
| `https://hub.INGRESS-LB-IP.nip.io` (Ingress, 신규) | 메인 접속 경로 |교육용 임시 인증  |
| `http://JUPYTERHUB-LB-IP` (LoadBalancer, fallback) | Ingress 장애 시 우회 | 동일 |
| `http://TAILSCALE-IP:30000` (NodePort, 외부 VPN) | 원격 접근 | 동일 |

### Ingress 설정

```yaml
host: hub.INGRESS-LB-IP.nip.io
backend: proxy-public:80
TLS: jupyterhub-tls (cluster-ca self-signed, DNS SAN 포함)
WebSocket annotation:
  proxy-read-timeout: 3600
  proxy-send-timeout: 3600
  proxy-body-size: 64m
```

> ⭐ **WebSocket 검증 통과:** Notebook 셀 실행(`import sys; sys.version`) 정상 동작 확인. 4_01 incident 패턴 재현 없음.

---

## 6. 🔄 ML 파이프라인 (Argo Workflows)

```
[Git push → GitHub Actions (Self-hosted Runner on master-01)]
    │ 16s
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

## 7. 📊 MLflow

- **Backend:** PostgreSQL (K8s Deployment + NFS PVC)
- **Artifact 저장소:** NFS PVC → NAS `/data/mlflow-artifacts/`
- **Argo 연동:**
  - train 단계: `mlflow.log_params()` (autolog — 105개 자동 기록)
  - evaluate 단계: `mlflow.log_metrics()` + `best.pt` artifact 연동 로직 추가
  - save-model 단계: Registry 등록 + alias "champion" 지정
- **Model Registry:** `visdrone-yolov8` 모델명, alias 기반 버전 관리
- **접속:** `http://TAILSCALE-IP:30010` (Ingress 통합 Phase B 예정)

---

## 8. 🚀 모델 서빙 (FastAPI)

- **배포 방식:** K8s Deployment (ai-team 네임스페이스)
- **배포 이미지:** `1jkim/yolov8-serving:v1` (DockerHub public) — 2026-04-17 전환
- **GPU:** 2080Ti × 1
- **추론 속도:** 77ms
- **nodeSelector:** `gpu-type=2080ti` (`kubernetes.io/hostname` 고정 제거됨 — 2026-04-17)

### 접속 경로 (2026-04-27 갱신)

| 경로 | 용도 |
|---|---|
| `https://serving.INGRESS-LB-IP.nip.io` (Ingress, 신규) | 메인 접속, HTTPS |
| `http://TAILSCALE-IP:30600` (NodePort, fallback) | 원격 / Ingress 장애 우회 |

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
- 헤더 상태 배지로 `COCO · 80cls` / `champion · v5` 표시
- 추론 결과 이미지, 탐지 카드, confidence bar 표시

> ⭐ **HTTPS 적용 효과 (2026-04-27):** `getUserMedia()` 웹캠 API가 HTTPS 환경에서 Chrome 플래그 예외 없이 정상 동작. self-signed 인증서 한계로 브라우저 첫 접속 시 경고 화면 통과 필요.

### champion 모델 로드 방식

```
MLflow alias "champion" 조회
    └→ version 번호 확인 (현재 v5)
    └→ 1순위: NAS /mnt/datasets/models/visdrone-v{version}.pt 직접 로드
    └→ 2순위: MLflow artifact 경로 폴백 (예비 경로)
```

> **설계 원칙:** MLflow Model Registry는 alias/version 관리에 사용하고, 실제 서빙 모델 로드는 NAS 저장 모델 파일을 1순위로 사용한다. MLflow artifact 경로는 현재 폴백 용도로만 유지한다.

> **반영 방식:** FastAPI /predict는 MLflow alias "champion"의 version을 조회한 뒤, 해당 version에 대응하는 NAS 모델 파일을 로드하도록 구성했다. alias 변경만으로 즉시 반영되는 것은 아니며, 필요 시 `/reload-champion` 엔드포인트 호출 또는 Pod 재시작으로 최신 version을 반영할 수 있다.

### 볼륨 마운트

| 마운트 경로 | PVC | 용도 |
|---|---|---|
| `/mnt/datasets` | nfs-datasets-pvc | 모델 파일 직접 접근 |
| `/mnt/mlflow-artifacts` | mlflow-artifacts-pvc (ai-team) | artifact 폴백 |

> ~~`/app` ConfigMap 마운트~~ — **제거됨 (2026-04-16).** 이미지화 이후 `main.py`가 이미지 내장. ConfigMap read-only 마운트가 모델 다운로드 경로 쓰기 실패를 유발하여 제거.

### Ingress 설정

```yaml
host: serving.INGRESS-LB-IP.nip.io
backend: yolov8-serving:8080
TLS: yolov8-serving-tls (cluster-ca self-signed, DNS SAN 포함)
annotation:
  proxy-body-size: 10m
  proxy-read-timeout: 120
  proxy-send-timeout: 120
```

### health 응답 예시

```json
{
  "status": "ok",
  "demo_model": "yolov8n-coco (80 classes)",
  "champion_ready": true,
  "champion_version": "5"
}
```

> **이미지 레지스트리:** `1jkim/yolov8-serving:v1` (DockerHub public). 2026-04-17 push 완료. 모든 2080Ti 노드에서 pull 가능. 로컬 containerd 의존성 제거됨.

---

## 9. ⚙️ CI/CD

```
[개발자] → git push → main
    │
[GitHub Actions (Private Repo: 28gpu-cluster-cicd)]
    │ Self-hosted Runner (master-01)
    │ 트리거 시간: ~16s
    ▼
[Argo Workflows — cicd-pipeline 자동 생성]
    └→ validate-data → train → evaluate → save-model
                                              └→ MLflow champion 자동 갱신
```

- **Runner 위치:** master-01 (로컬 kubectl 직접 접근)
- **수동 트리거:** `workflow_dispatch` (epochs, batch_size, model_version 파라미터 입력 가능)
- **레포 분리 이유:** 퍼블릭 포트폴리오 레포에 Self-hosted Runner를 붙이면 외부 PR 악성 코드 실행 위험 → 프라이빗 레포 분리

---

## 10. 📡 모니터링

| 항목         | 내용                                         |
| ---------- | ------------------------------------------ |
| Prometheus | kube-prometheus-stack (Helm)               |
| Grafana    | GPU 대시보드 #12239 (DCGM Exporter)            |
| GPU 메트릭    | DCGM Exporter — 온도 / 전력 / 메모리 / 클럭         |
| 알람         | Alertmanager → Gmail SMTP                  |
| 알람 룰       | GPU 온도 85°C 초과 / 메모리 사용률 95% 초과 / GPU Idle |

---

## 11. 🔒 백업 / DR

- **방식:** 호스트 crontab (K8s CronJob 아님 — 클러스터 장애 시 CronJob도 불가하므로)
- **스케줄:** 매일 02:00
- **저장:** NAS `/data/backups/etcd/` (10GbE)
- **보존:** 7일 자동 삭제
- **DR 검증:** 복구 절차 문서화 + 스냅샷 무결성 검증 완료 (2026-04-13)
- **복원 절차:** `runbooks/runbook_etcd_restore.md` 참조

---

## 12. 🔐 Ingress + TLS (2026-04-27 신규)

### 컴포넌트

| 컴포넌트 | 버전 | 배치 노드 | 역할 |
|---|---|---|---|
| NGINX Ingress Controller | ingress-nginx Helm | master-02 | host 기반 라우팅 + TLS 종단 |
| cert-manager | v1.20.2 | master-02 | 인증서 발급 / 갱신 자동화 |
| selfsigned-bootstrap | ClusterIssuer | — | bootstrap CA용 |
| cluster-ca-issuer | ClusterIssuer | — | 실서비스 인증서 발급용 |

### 인증서 정책

- **현재:** cluster-ca self-signed (cert-manager 자동 갱신)
- **만료:** 1년 (`duration: 8760h`), 30일 전 자동 갱신 (`renewBefore: 720h`)
- **적용 인증서:**
  - `yolov8-serving-tls` → `serving.INGRESS-LB-IP.nip.io`
  - `jupyterhub-tls` → `hub.INGRESS-LB-IP.nip.io`
- **공인 인증서:** 미적용. 본인 도메인 확보 후 DNS-01 challenge로 별도 Phase에서 재시도 (`runbook_letsencrypt_dns01.md`)

### Let's Encrypt HTTP-01 시도 결과

2026-04-27 staging 발급 시도 → 실패 → 중단:
- ACME challenge가 cert-manager solver Pod로 라우팅되지 않음
- `Presented: false` + connection error 반복
- `nip.io` + 캠퍼스 환경 조합의 한계로 판정
- 상세: `journal/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md` §5

### 도메인 체계

```
INGRESS-LB-IP (Ingress Gateway)
 ├─ serving.INGRESS-LB-IP.nip.io  → YOLOv8 FastAPI (팀원)
 └─ hub.INGRESS-LB-IP.nip.io      → JupyterHub (팀원)

[Phase B 예정]
 ├─ grafana.INGRESS-LB-IP.nip.io  → Grafana (관리자, Basic Auth)
 ├─ argo.INGRESS-LB-IP.nip.io     → Argo Workflows (관리자, Basic Auth)
 ├─ portainer.INGRESS-LB-IP.nip.io → Portainer (관리자, Basic Auth)
 ├─ files.INGRESS-LB-IP.nip.io    → Filebrowser (관리자, Basic Auth)
 └─ mlflow.INGRESS-LB-IP.nip.io   → MLflow (공개)
```

---

## 13. ⚠️ 미해결 과제 / 한계

| 항목 | 현황 | 비고 |
|---|---|---|
| HTTPS / Ingress | ✅ 적용 (self-signed) | `serving`, `hub` 두 host 적용 완료. 공인 인증서는 별도 Phase. |
| Let's Encrypt 공인 인증서 | 미적용 | HTTP-01 실패 → 본인 도메인 + DNS-01로 별도 Phase에서 재시도. `runbook_letsencrypt_dns01.md` |
| JupyterHub 인증 | 교육용 임시 인증 | GitHub OAuth 교체 예정 (HTTPS 선결 조건 충족 완료) |
| 관리자 그룹 Ingress 통합 | 미적용 | Phase B 예정 (Grafana, Argo, Portainer, Filebrowser, MLflow) |
| ResourceQuota | 미적용 | 팀원 GPU 점유 제한 없음 |
| K8s 업그레이드 | v1.29 (EOL 예정) | v1.31 업그레이드 runbook 작성 예정, 실작업 미진행 |
| buildkitd 서비스 등록 | nohup 백그라운드 | master-01 재부팅 시 재실행 필요 |
| NodePort/LoadBalancer fallback | 유지 중 | Phase C에서 정리 예정 |
| Prometheus 직접 노출 | NodePort 30310 | Phase C에서 폐쇄, Grafana 데이터소스 전환 예정 |

---

## 14. 📎 관련 문서

### Journal (작업 기록)
- `journal/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md` — Phase A 작업 기록
- `journal/4_17_YOLOv8_서빙_이미지_DockerHub_등록.md` — 서빙 이미지화
- `journal/4_13_MLflow_설치_및_Argo_DAG_연동.md` — MLflow 도입

### Runbooks (운영 절차)
- `runbooks/runbook_ingress_tls.md` — Ingress 추가 표준 절차
- `runbooks/runbook_letsencrypt_dns01.md` — 공인 인증서 적용 절차
- `runbooks/runbook_argo_dag.md` — Argo DAG 운영
- `runbooks/runbook_etcd_restore.md` — etcd 복원
- `runbooks/runbook_model_serving.md` — 모델 서빙

### Incidents (장애 기록)
- `incidents/4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md` — WebSocket 충돌
- `incidents/3_31_네트워크_장애_및_클러스터_설계_개선.md` — SoC 원칙 도입 사유

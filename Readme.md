# 🖥️ 27GPU 온프레미스 클러스터 기반 ML 파이프라인 구축

> 사용 불가능한 상태였던 **27 GPU 레거시 서버 인프라를 단독으로 복구·재설계하고**
> Kubernetes 기반 AI 학습 클러스터 및 자동화된 ML 파이프라인으로 재구성했습니다.
>
> **단순 복구가 아닌, 기존 시스템을 폐기하고 새로운 아키텍처로 전환하여
> Git push → 자동 학습 실행이 가능한 환경까지 구축했습니다.**
>
> 온프레미스 환경에서 GPU 자원 관리, 네트워크 분리, 스토리지 연동, CI/CD 기반 학습 자동화, Ingress 기반 HTTPS 접속 구조까지 포함한 전체 인프라를 직접 구축했습니다.
>
> 프로젝트 시작 시점에는 인프라 운영 경험이 많지 않았지만 AI를 문제 해결 보조 도구로 활용해 실제 작업을 진행했고 부족한 개념은 이후 학습과 문서화를 통해 보완했습니다.

![자료사진](./docs/images/20260312_204425.jpg)

---

## 📌 프로젝트 개요

| 항목     | 내용                                                                                       |
| -------- | ------------------------------------------------------------------------------------------ |
| **기간** | 2026년 3월 12일 ~ 4월 28일(재구축 종료)                                                    |
| **역할** | 단독 수행 (시스템 복구 · 인프라 설계 · ML 파이프라인 구축 전 과정)                         |
| **환경** | 학교 서버실 온프레미스 GPU 클러스터                                                        |
| **목표** | 방치된 GPU 클러스터를 AI 학습 파이프라인과 서비스 접속 환경이 동작하는 MLOps 인프라로 전환 |

---

## 🔥 시작하게 된 계기

1학년 때 교수님께 "학과 서버실에 AI 훈련용 GPU 서버가 있다"는 말을 들었습니다. 2학년이 되어 직접 서버실을 찾아갔고, 5년간 업데이트가 멈춘 클러스터와 4페이지짜리 재기동 매뉴얼을 발견했습니다.

담당 업체는 이미 서비스를 종료한 상태였고, 유지보수도 불가능한 상황이었습니다. 교수님께

> "제가 복구해봐도 되겠습니까? 복구 과정에서 프로그램이 망가져도 상관없습니까?
> 그때는 새로 만들어보겠습니다"

라고 여쭤본 뒤 작업을 시작했고 구형 시스템을 되살리는 과정에서 **OS가 너무 구형이라 K8s + GPU 생태계 연동이 근본적으로 불가능하다**는 한계를 직접 확인했습니다.

단순 복구가 아닌 전면 재설계를 결정했고, Ubuntu 22.04 + Kubernetes 기반으로 클러스터를 새로 구축한 뒤 JupyterHub 팀 환경, YOLOv8 학습 파이프라인, Argo Workflows, MLflow, FastAPI 서빙, Ingress 기반 HTTPS 접속 구조까지 단계적으로 완성했습니다.

클러스터를 처음 만졌을 때는 리눅스 명령어도 낯설었습니다. AI를 문제 해결 도구로 적극 활용하며 빠르게 실행했고, 모든 작업을 저널 형식으로 기록해 쌓인 문서를 다시 학습 자료로 삼아 기초 개념까지 역방향으로 이해해 나갔습니다. 속도보다 기록을 먼저 챙겼고, 그 기록이 이 레포를 구성합니다.

![4장뿐이였던메뉴얼](./docs/images/KakaoTalk_20260413_143557870.jpg)

---

## ✅ 최종 결과

| 항목                  | 결과                                                                                                 |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| 총 GPU                | V100 4장 + 2080Ti 23장 = **27장 통합**                                                               |
| 스토리지              | **27.3TB** NFS 마운트 완료                                                                           |
| 내부 네트워크         | **전 노드 ~9Gbps** (1G → 10G, 약 10배)                                                               |
| 모니터링              | Grafana GPU 실시간 대시보드 (온도/전력/메모리)                                                       |
| Ingress Gateway       | NGINX Ingress Controller + cert-manager 도입, 단일 IP `INGRESS-LB-IP`에서 host 기반 라우팅 구성      |
| HTTPS 접속            | `https://hub.INGRESS-LB-IP.nip.io` / `https://serving.INGRESS-LB-IP.nip.io`                          |
| TLS                   | cluster-ca self-signed 인증서, DNS SAN 포함, cert-manager 관리                                       |
| 인증                  | GitHub OAuth 적용 완료 (DummyAuthenticator 제거 — 2026-04-27)                                        |
| 관리자 접속           | Grafana `:30300` / Prometheus `:30310` / Portainer `:30320` (Phase B Ingress 통합 예정)              |
| 팀 환경               | JupyterHub (GitHub 계정 기반 인증, GPU 자동 배정, NFS 홈 디렉토리, Notebook 셀 실행 검증 완료)       |
| GPU 검증              | TensorFlow / PyTorch CUDA 인식 · MNIST 학습 동작 검증                                                |
| ML 학습               | YOLOv8 COCO mAP@0.5 46.8% / VisDrone mAP@0.5 33.4% (V100 4장 DDP)                                    |
| 알람                  | GPU 온도/메모리/노드 다운 3종 → Gmail 자동 통보                                                      |
| 워크플로우            | Argo Workflows — 팀원 웹 UI로 학습 Job 제출 가능                                                     |
| MLflow                | 실험 추적 · params 105개 + metrics 자동 기록 · Registry alias(champion) 기반 버전 관리               |
| 모델 서빙             | FastAPI `/predict`(champion) · `/predict-demo`(COCO) · `https://serving.INGRESS-LB-IP.nip.io` · 77ms |
| 워크로드 우선순위     | PriorityClass 4계층 도입 완료 — 시스템 파드 eviction 보호, 서빙/학습 계층화                          |
| LimitRange            | `ai-team-default-compute` — 컨테이너 기본 request/limit 자동 주입, min/max 강제 (2026-04-28)         |
| ResourceQuota         | `ai-team-compute-quota` — GPU 16장 상한, CPU/Memory/Pod 네임스페이스 상한 적용 (2026-04-28)          |
| RBAC / Namespace 격리 | `ai-team` 네임스페이스 + RBAC 격리 환경 (학생 Job 제출 권한 분리)                                    |
| etcd 정기 백업        | 호스트 crontab → NAS 자동 저장, snapshot 무결성 DR 검증 완료                                         |
| Filebrowser           | NAS 웹 파일 탐색기 배포 (monitoring 네임스페이스)                                                    |
| GitHub Actions CI/CD  | 코드 push → Argo Workflow 자동 트리거 파이프라인 (16초 완료)                                         |
| 구축 기간             | **약 7주** (3/12 ~ 4/28)                                                                             |

---

## 🗺️ 작업 타임라인

```
Phase 1  레거시 복구 (3/12 ~ 3/19)
  └ GRUB 루트 복구 · K8s 인증서 갱신 · NAS RAID 재구성 · HPE ACPI 해결
  └ GPU 드라이버 업그레이드 시도 → Ubuntu 18.04 구조적 한계 확인 → 전면 재구축 결정 ⭐
  └ 상세 → docs/journal/1.cheetah-복구/

Phase 2  클러스터 재구축 (3/20 ~ 3/27)
  └ Ubuntu 22.04 클린 설치 · K8s v1.29 · Calico CNI · 워커노드 6대 Join
  └ 10G NIC 물리 이전 (master-02 → NAS, 전송속도 10배) ⭐ · GPU Operator · 모니터링 스택
  └ 상세 → docs/journal/2.GPU_클러스터_재구축/

Phase 3  팀 환경 구축 및 서비스 안정화 (3/30 ~ 4/2)
  └ ai-team RBAC + JupyterHub (5명 계정, GPU 자동 배정, CUDA 검증)
  └ port-forward 다중 접속 장애 → LoadBalancer 전환 재설계 ⭐ · 10GbE 라우팅 최적화
  └ 상세 → docs/journal/3.AI_학습_팀_환경_구축/

Phase 4  ML 파이프라인 및 서빙 구축 (4/2 ~ 4/17)
  └ YOLOv8 COCO / VisDrone 학습 Job · Argo DAG 4단계 파이프라인 · Alertmanager
  └ MLflow 실험 추적 + Argo 연동 · GitHub Actions CI/CD (git push → 16s 트리거)
  └ FastAPI 서빙 + 웹 UI · 서빙 이미지화 (pip install 제거) · DockerHub 등록
  └ 상세 → docs/journal/4.ML_파이프라인_구축/

Phase 5  서비스 노출 및 운영 안정화 (4/27 ~ 4/28)
  └ NGINX Ingress Controller + cert-manager 도입 · MetalLB 신규 IP INGRESS-LB-IP 할당
  └ nip.io wildcard DNS 기반 host 라우팅: hub / serving 분리
  └ cluster-ca self-signed TLS 인증서 재발급 · Fake Certificate 문제 해결
  └ JupyterHub WebSocket timeout annotation 적용 · Notebook 셀 실행 검증 완료 ⭐
  └ DummyAuthenticator 제거 → GitHub OAuth (GitHubOAuthenticator) 전환 완료 ⭐
  └ PriorityClass 4계층 도입 (Phase A/B/B-1) — 시스템 파드 eviction 보호 완료 ⭐
  └ PriorityClass Phase C — yolov8-serving serving-critical, Argo WorkflowTemplate training-normal 적용 완료 ⭐
  └ LimitRange Phase D — ai-team 컨테이너 기본 request/limit 자동 주입 (500m/1Gi req, 4/16Gi limit) ⭐
  └ ResourceQuota Phase E — ai-team GPU 16장 상한 포함 네임스페이스 자원 상한 확립 ⭐
  └ monitoring Helm chart 버전 82.15.1 고정 운영 결정 (INC-2026-04-28)
  └ 상세 → docs/journal/5.서비스_노출_및_운영_안정화/
```

---

## 💡 핵심 의사결정 5가지

### 1. 재구축 vs 패치 — "고칠 것인가, 새로 만들 것인가"

구형 CHEETAH 시스템을 일단 살린 뒤, V100 DGX 노드의 GPU 드라이버(418.87 → 535) 업그레이드를 직접 시도했습니다.

시도한 순서는 다음과 같습니다.

1. `apt` 방식 설치 시도 → Ubuntu 18.04 라이브러리 의존성 충돌로 실패
2. Runfile 수동 설치 방식으로 전환 → 드라이버 535 설치 성공, Docker GPU 인식 성공
3. K8s Device Plugin GPU 인식 시도 → 3가지 경로(DaemonSet 공식 매니페스트 / 수동 바이너리 배포 / 토폴로지 매니페스트 커스터마이즈)로 시도했으나 `nvidia.com/gpu: 0` 상태 유지
4. **Ubuntu 18.04 라이브러리 경로 체계 문제로 K8s Device Plugin 연동이 근본적으로 불가능하다** 결론 도출
5. 전면 재구축 결정

작업 타임라인까지 분 단위로 기록한 작업일지: 3.19 GPU 드라이버 업데이트

> 무작정 버리지 않고, 세 가지 방법을 순서대로 시도한 뒤 직접 한계를 확인하고 나서 결정했습니다.

---

### 2. 10G 네트워크 병목 발견 → 하드웨어 재배치

클러스터 구축 중, NAS가 10G 스위치에 연결되어 있지만 실제로는 1G NIC만 탑재되어 있다는 점을 발견했습니다. 28TB 스토리지에서 GPU 노드로 학습 데이터를 전송할 때 병목이 발생한다는 것을 인식했고, master-01에 장착된 10G PCIe NIC를 NAS로 직접 이전하여 내부 10G 통신을 구성했습니다.

> 소프트웨어 설정이 아닌 **물리 하드웨어 재배치**로 해결한 사례입니다.

![자료사진](./docs/images/20260323_182524.jpg)

---

### 3. 완벽함보다 동작 — 2080Ti 1장 미인식 문제

2080Ti 1장이 인식되지 않아 케이블 교체 및 PCIe 슬롯 변경을 시도했습니다. 슬롯 교체까지 가려면 추가 시간이 상당히 필요하다고 판단하여, **27장으로 먼저 클러스터를 완성하는 것을 선택**했습니다.

> 완벽한 28장보다 지금 동작하는 27장이 더 가치 있다는 트레이드오프 판단입니다.

![자료사진](./docs/images/20260324_140843.jpg)

---

### 4. kubectl Job → Argo Workflows — "팀원이 스스로 실험할 수 있는 환경"

YOLOv8 학습을 K8s Job으로 실행하는 것까지는 됐지만, 팀원이 실험 조건(epochs, batch size, GPU 타입)을 바꾸려면 매번 YAML을 수정하고 kubectl 명령어를 직접 입력해야 했습니다.

이 구조의 문제는 두 가지였습니다.

첫째, **운영 의존성** — 실험 조건을 바꿀 때마다 관리자가 직접 개입해야 했습니다.

둘째, **보안** — 팀원이 스스로 Job을 제출하려면 master-01 SSH 접근 권한, 즉 Ubuntu 관리자 계정을 공유해야 하는 상황이었습니다. 이는 클러스터 전체에 대한 접근권을 팀원에게 주는 것과 다름없었습니다.

Argo Workflows 도입으로 WorkflowTemplate을 파라미터화하고 웹 UI에서 Submit할 수 있도록 구성해, 팀원은 SSH 없이 브라우저에서 실험 조건만 입력하고 학습을 제출할 수 있게 됐습니다.

> 인프라 구축에서 그치지 않고, 팀원이 안전하게 독립적으로 실험할 수 있는 환경을 만드는 것까지가 MLOps의 첫 번째 레이어라고 판단했습니다.

---

### 5. 개별 포트 노출 → Ingress Gateway — "서비스를 어떻게 안정적으로 노출할 것인가"

JupyterHub, FastAPI, MLflow, Argo처럼 서비스가 늘어나면서 각 서비스마다 NodePort 또는 LoadBalancer IP를 따로 관리하는 방식은 운영 복잡도가 커지는 구조가 되었습니다. 특히 JupyterHub는 단순 HTTP 접속뿐 아니라 Notebook 커널 통신에 WebSocket이 필요하기 때문에, 접속 경로가 불안정하면 로그인은 되지만 셀 실행에서 장애가 발생할 수 있었습니다.

이를 해결하기 위해 NGINX Ingress Controller와 cert-manager를 도입하고, `INGRESS-LB-IP` 단일 IP에서 host 기반으로 서비스를 분기하는 구조로 전환했습니다.

```text
hub.INGRESS-LB-IP.nip.io      → JupyterHub
serving.INGRESS-LB-IP.nip.io  → YOLOv8 FastAPI
```

처음에는 Let's Encrypt HTTP-01도 검토했지만, 캠퍼스 네트워크 환경과 ACME challenge 라우팅 제약으로 staging 발급이 안정적으로 완료되지 않았습니다. 따라서 Phase A에서는 `cluster-ca` 기반 self-signed TLS를 유지하고, 공인 인증서는 본인 도메인 확보 후 DNS-01 방식으로 별도 Phase에서 적용하기로 결정했습니다.

> 서비스 노출 구조를 "개별 포트 접속"에서 "단일 Ingress Gateway + host 기반 라우팅"으로 바꾸면서, 팀원이 사용할 HTTPS 접속 경로와 운영자가 확장할 수 있는 표준 패턴을 함께 확보했습니다.

---

## 🏗️ 클러스터 구성

| 노드          | 역할                                             | GPU        | IP                                   |
| ------------- | ------------------------------------------------ | ---------- | ------------------------------------ |
| master-01     | Control Plane                                    | —          | `MASTER-IP`                          |
| master-02     | Worker (시스템 파드 + Ingress/cert-manager 전담) | —          | `WORKER-IP-02`                       |
| v100-gpu-01   | Worker (학습 전용)                               | V100 × 4   | `WORKER-IP-03`                       |
| 2080ti-gpu-02 | Worker                                           | 2080Ti × 8 | `WORKER-IP-04`                       |
| 2080ti-gpu-03 | Worker                                           | 2080Ti × 7 | `WORKER-IP-05`                       |
| 2080ti-gpu-04 | Worker                                           | 2080Ti × 8 | `WORKER-IP-06`                       |
| NAS (nas-01)  | 스토리지                                         | —          | `NAS-IP` (1G) / `<NAS-10G-IP>` (10G) |

- **총 GPU:** V100 4장 + 2080Ti 23장 = **27장**
- **K8s 버전:** v1.29.15
- **OS:** Ubuntu 22.04.5 LTS

> 구축 당시(2026년 3월) GPU Operator · Calico · MetalLB 등 핵심 애드온의 호환성이 충분히 검증된 버전으로 v1.29를 선택했습니다. 최신 버전보다 패치가 안정적으로 쌓인 버전을 택하는 것이 온프레미스 GPU 클러스터 초기 구축에서 리스크를 줄이는 합리적인 판단이었습니다.

---

## 📦 기술 스택

| 영역                  | 기술                                                  |
| --------------------- | ----------------------------------------------------- |
| **OS**                | Ubuntu 22.04.5 LTS                                    |
| **Container Runtime** | Containerd                                            |
| **Orchestration**     | Kubernetes v1.29.15                                   |
| **CNI**               | Calico v3.27                                          |
| **GPU 관리**          | NVIDIA GPU Operator                                   |
| **스토리지**          | NFS (nfs-subdir-external-provisioner)                 |
| **로드밸런서**        | MetalLB v0.13.12                                      |
| **Ingress**           | NGINX Ingress Controller                              |
| **TLS 관리**          | cert-manager v1.20.2 + cluster-ca self-signed         |
| **DNS 라우팅**        | nip.io wildcard DNS 기반 host 라우팅                  |
| **모니터링**          | Prometheus + Grafana (kube-prometheus-stack)          |
| **GPU 모니터링**      | DCGM Exporter (Grafana Dashboard #12239)              |
| **관리 UI**           | Portainer                                             |
| **원격 접속**         | Tailscale VPN                                         |
| **RAID**              | AVAGO 3108 MegaRAID (RAID 1 + RAID 5)                 |
| **팀 환경**           | JupyterHub (Helm chart 4.3.3 / gpu-jupyter CUDA 12.0) |
| **인증**              | GitHub OAuth (GitHubOAuthenticator)                   |
| **ML 프레임워크**     | PyTorch, TensorFlow, Ultralytics YOLOv8               |
| **워크플로우**        | Argo Workflows (Helm v4.0.4)                          |
| **실험 추적**         | MLflow (PostgreSQL 백엔드, NFS Artifact 저장소)       |
| **알람**              | Alertmanager (Gmail SMTP, PrometheusRule CRD)         |

---

## 🧭 기술 선택 근거

| 항목             | 검토한 대안           | 선택                       | 선택 근거                                                                                             |
| ---------------- | --------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------- |
| 오케스트레이터   | Slurm, Nomad, Ray     | **Kubernetes**             | Argo · MLflow 등 MLOps 도구가 K8s 전제로 설계됨. 구직시장 범용성. Slurm은 컨테이너 재현성 관리에 약점 |
| CNI              | Cilium, Flannel       | **Calico**                 | 온프레미스 BGP 지원, 운영 안정성, 공식 문서·커뮤니티 자료 성숙도                                      |
| 스토리지         | Ceph, Longhorn        | **NFS**                    | 기존 NAS 하드웨어 직접 활용, 구현 단순성 우선. Ceph는 별도 클러스터 필요                              |
| 컨테이너 런타임  | CRI-O                 | **containerd**             | GPU Operator 공식 권장 런타임                                                                         |
| ML 파이프라인    | Kubeflow, Prefect     | **Argo Workflows**         | K8s 네이티브, 경량, 학습 진입장벽 낮음. Kubeflow는 현 규모 대비 설치 복잡도 과함                      |
| etcd 백업 자동화 | K8s CronJob           | **호스트 crontab**         | K8s 장애 시에도 백업이 동작해야 하므로 K8s에 의존하지 않는 호스트 레벨 선택                           |
| 서비스 노출      | 개별 NodePort/LB      | **NGINX Ingress**          | 단일 IP에서 host 기반으로 여러 서비스를 분리하고 TLS 종단을 통합 관리                                 |
| 인증서 정책      | Let's Encrypt HTTP-01 | **cluster-ca self-signed** | 캠퍼스 네트워크의 외부 도달성 제약으로 HTTP-01 중단. 본인 도메인 확보 후 DNS-01을 별도 Phase로 분리   |
| JupyterHub 인증  | DummyAuthenticator    | **GitHub OAuth**           | DummyAuth는 인증 없이 누구나 접근 가능. GitHub 계정 기반 접근 제어로 팀원 전용 환경 보호              |

---

## 🛠️ 주요 트러블슈팅

> 인프라 복구 2건 + MLOps 파이프라인 운영 중 발생한 3건으로 구성했습니다. 상세 내용은 각 문서를 참고하세요.

---

### [인프라] GRUB 단일 사용자 모드 — 인증 정보 전량 분실 상황 복구

교수님 승인 하에 물리 접근 권한으로 복구를 진행했습니다. 모든 서버의 인증 정보가 유실된 상태에서 데이터 보존을 위해 포맷 대신 GRUB 단일 사용자 모드를 선택했습니다. 재부팅 시 커널 파라미터에 `rw init=/bin/bash`를 주입해 루트 쉘에 진입하고 패스워드를 재설정했습니다.

```bash
# 수정 전: ... ro quiet splash
# 수정 후: ... rw init=/bin/bash
passwd ubuntu && sync && reboot -f
```

---

### [인프라] NAS RAID 복구 — 컨트롤러 "두더지 잡기 현상"

0번 SSD 장애 감지 후 MegaRAID BIOS 직접 진입. 컨트롤러가 OS 미러링 복구를 위해 데이터용 HDD를 무단 점유해 `Rebuild` 상태로 만들어버리는 현상을 발견했습니다.

핵심은 세 단계의 선행 작업이었습니다. 이 순서가 틀리면 이후 어떤 작업도 불가능합니다.

1. `PD Mgmt` → 0번 SSD 선택 → `F2` → **Make Unconfigured Good** — `Unconfigured Bad` 낙인을 제거하지 않으면 컨트롤러가 해당 드라이브를 완전히 무시해 이후 작업이 불가
2. `Foreign Config` → **Clear** — 이전 RAID 잔류 정보를 소거하지 않으면 컨트롤러가 구형 구성을 복원하려 시도
3. `Rebuild` 중인 10TB HDD → `F2` → **Stop Rebuild → Make UG** — HDD를 자유 상태로 풀지 않으면 RAID 5 재생성 목록에 나타나지 않음

세 단계 완료 후 0번 SSD를 `Global Hot Spare`로 지정 → OS 미러링 자동 복구, RAID 5 27.3TB 재생성.

![자료사진](./docs/images/20260316_130432.jpg)
![자료사진](./docs/images/20260316_184714.jpg)
![자료사진](./docs/images/20260316_191154.jpg)

---

### [MLOps] PyTorch + V100 호환성 — CUDA no kernel image error

V100 학습 Job 실행 시 `CUDA error: no kernel image is available for execution on the device` 발생. 원인은 `ultralytics:latest`의 PyTorch 2.11이 V100 Compute Capability 7.0을 미지원하는 것이었습니다. `ultralytics:8.1.0` (PyTorch 2.1.0)으로 이미지를 교체해 해결했습니다.

| 이미지             | PyTorch | V100 (CC 7.0) |
| ------------------ | ------- | ------------- |
| ultralytics:latest | 2.11.0  | ❌            |
| ultralytics:8.1.0  | 2.1.0   | ✅            |

> GPU 버전 호환성은 CUDA 버전이 아니라 **PyTorch 빌드 타겟 Compute Capability** 기준으로 확인해야 한다.

---

### [MLOps] DDP 멀티GPU — shared memory 부족

V100 4장 DDP 학습 Job에서 `RuntimeError: No space left on device` 발생. K8s Pod 기본 `/dev/shm` 크기(64MB)가 멀티GPU 워커 프로세스 간 데이터 공유에 부족한 것이 원인이었습니다. YAML에 `emptyDir: medium: Memory, sizeLimit: 16Gi` 볼륨을 추가해 해결했습니다.

```yaml
- name: dshm
  emptyDir:
    medium: Memory
    sizeLimit: 16Gi
```

> DDP Job은 `/dev/shm` 설정이 필수다. Argo WorkflowTemplate 수정 후에는 Retry가 아닌 새 Submit으로 제출해야 변경사항이 반영된다.

---

### [MLOps] JupyterHub 다중 접속 — 세션 끊김 및 서비스 구조 재설계

팀원 2명이 동시 접속하면 한 명의 세션이 끊기는 현상 발생. Pod와 MetalLB는 정상이었지만 `kubectl port-forward`를 systemd 서비스로 등록해 팀 서버로 운용하는 구조가 문제였습니다. `port-forward`는 단일 프로세스가 모든 요청을 중계하는 1인용 임시 터널이라 동시 접속 시 WebSocket 충돌이 필연적으로 발생합니다.

추가로 MetalLB IP Pool에 master-01 IP(151)가 포함되어 ARP 충돌 → speaker 89회 재시작 → Calico CrashLoopBackOff까지 연쇄 장애로 이어졌습니다.

port-forward를 전면 제거하고 `proxy-public`을 LoadBalancer 타입으로 전환, MetalLB Pool을 기존 허용 IP로 교체해 해결했습니다.

```bash
# port-forward 전면 제거
sudo pkill -f "kubectl port-forward"
sudo systemctl disable kubectl-jupyterhub.service
```

> `kubectl port-forward`는 개발자 1인용 디버깅 도구다. 다중 사용자 환경에 systemd 서비스로 등록하면 WebSocket 세션 충돌이 필연적으로 발생한다. MetalLB IP Pool에 노드 자체 IP를 절대 포함하지 마라.

---

## 🏁 이후 방향

이 레포는 **재구축 완료 시점(2026-04-28)**을 기준으로 작성되었습니다.
운영 단계 기록은 별도 레포에서 이어집니다.

- 운영 레포: (링크 추가 예정)
- 포트폴리오 압축 레포: (링크 추가 예정)

---

## 🗂️ 문서 구조

```text
📁 docs/
├── 📁 overview/
│   ├── 📄 cluster-diagram.md            # 클러스터 구성도
│   ├── 📄 current-architecture.md       # 현재 유효한 클러스터 구조 한 장 요약
│   └── 📄 timeline.md                   # 작업 타임라인 상세 (날짜별 풀버전)
│
├── 📁 journal/                          # 날짜순 작업 기록 (증거 문서)
│   ├── 📁 1.cheetah-복구/               # Phase 1 — 레거시 시스템 복구 시도
│   │   ├── 📄 3_12_AI_클러스터_복구_1~5
│   │   ├── 📄 3_16_NAS_복구_1~2
│   │   ├── 📄 3_17_HPE_ACPI_장애
│   │   └── 📄 3_19_GPU_드라이버_업데이트  ⭐ 재구축 결정 근거
│   │
│   ├── 📁 2.GPU_클러스터_재구축/         # Phase 2 — Ubuntu 22.04 + K8s 신규 구축
│   │   ├── 📄 0_시스템_재구축_개요
│   │   ├── 📄 1_백업 ~ 6_워커노드_합류
│   │   └── 📄 7_GPU_클러스터_완전_구축_가이드
│   │
│   ├── 📁 3.AI_학습_팀_환경_구축/        # Phase 3 — JupyterHub · RBAC · 네트워크
│   │   ├── 📄 3_30_RBAC_Namespace_JupyterHub_구축  ⚠️ [superseded]
│   │   ├── 📄 4_01_Grafana_ServiceAccount_분리_및_보안_강화
│   │   └── 📄 4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정
│   │
│   ├── 📁 4.ML_파이프라인_구축/          # Phase 4 — YOLOv8 · Argo · MLflow · CI/CD
│   │   ├── 📄 4_02_YOLOv8_COCO_학습_및_웹캠_추론_테스트
│   │   ├── 📄 4_03_YOLOv8_VisDrone_멀티GPU_학습_Job
│   │   ├── 📄 4_04_YOLOv8_VisDrone_학습_결과_보고서
│   │   ├── 📄 4_06_Argo_Workflows_설치_및_WorkflowTemplate_구성
│   │   ├── 📄 4_07_Alertmanager_이메일_알람_구성
│   │   ├── 📄 4_09_Argo_Workflows_Tailscale_접속_및_포트_변경
│   │   ├── 📄 4_13_MLflow_설치_및_Argo_DAG_연동
│   │   ├── 📄 4_13_GitHub_Actions_CICD
│   │   ├── 📄 4_15_MLflow_alias_FastAPI_엔드포인트_분리
│   │   ├── 📄 4_16_YOLOv8_서빙_이미지화
│   │   └── 📄 4_17_YOLOv8_서빙_이미지_DockerHub_등록
│   │
│   └── 📁 5.서비스_노출_및_운영_안정화/  # Phase 5 — Ingress · TLS · OAuth · 자원 거버넌스
│       ├── 📄 4_27_Ingress_TLS_도입_및_host_기반_라우팅
│       ├── 📄 4_27_JupyterHub_GitHub_OAuth_전환
│       ├── 📄 4_28_PriorityClass_ResourceQuota_Phase_A_B_B1  ⭐ PriorityClass 4계층 · 시스템 파드 보호
│       └── 📄 4_28_PriorityClass_ResourceQuota_Phase_C_D_E  ⭐ 워크로드 priority · LimitRange · ResourceQuota
│
├── 📁 runbooks/                         # 운영 절차 · 복구 매뉴얼
│   ├── 📄 runbook_etcd_restore.md       # etcd 백업 · 스냅샷 복원 DR 검증
│   ├── 📄 runbook_argo_dag.md           # DAG 파이프라인 구성 · 운영
│   ├── 📄 runbook_model_serving.md      # FastAPI 서빙 배포 · 운영
│   ├── 📄 runbook_ingress_tls.md        # Ingress + TLS 서비스 추가 표준 절차
│   └── 📄 runbook_letsencrypt_dns01.md  # DNS-01 기반 공인 인증서 전환 절차
│
├── 📁 incidents/                        # 장애 기록 · 트러블슈팅
│   ├── 📄 3_17_HPE_ACPI_장애
│   ├── 📄 3_19_GPU_드라이버_업데이트    ⭐ 재구축 결정 근거 (journal과 교차 수록)
│   ├── 📄 3_23_10G_NIC_이전             ⭐ 하드웨어 재배치로 10G 개통
│   ├── 📄 3_23_2080Ti_GPU_미인식        ⭐ 27장 확정 판단
│   ├── 📄 3_27_NAS_NIC_접촉불량         ⭐ rev ff 진단 · PCIe 재삽입
│   ├── 📄 3_27_Grafana_DCGM_No_data
│   ├── 📄 3_27_Portainer_Tailscale_접속불가
│   ├── 📄 3_31_네트워크_장애_및_클러스터_설계_개선  ⭐ MetalLB ARP 충돌 · Calico 연쇄 장애
│   ├── 📄 4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선  ⭐ port-forward 제거 · 재설계
│   ├── 📄 4_01_Grafana_대시보드_미표시_및_RBAC_권한_문제
│   ├── 📄 4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정
│   ├── 📄 4_09_Alertmanager_Silence_DaemonSet_알람_억제
│   ├── 📄 4_16_incidents_이미지화_장애_3건   ⭐ containerd namespace 격리 · ConfigMap read-only 충돌 · nodeSelector 범위 불일치
│   └── 📄 4_28_grafana_chart_upgrade   ⭐ helm --version 미지정 → chart 자동 업그레이드 · CrashLoop · NodePort 변경
│
└── 📁 images/
```

---

## 📁 작업 문서

> 현재 클러스터 구조 한 장 요약 → [**overview/current-architecture.md**](docs/overview/current-architecture.md)
> 운영 절차 → [**docs/runbooks/**](docs/runbooks/)

### 1. CHEETAH 클러스터 복구

| 문서                                                                                  | 내용                                   |
| ------------------------------------------------------------------------------------- | -------------------------------------- |
| [AI 클러스터 복구 1](docs/journal/1.cheetah-복구/3_12_AI_클러스터_복구_1.md)          | OS 루트 권한 복구 및 패스워드 초기화   |
| [AI 클러스터 복구 2](docs/journal/1.cheetah-복구/3_12_AI_클러스터_복구_2.md)          | K8s 인증서 갱신 및 노드 동기화         |
| [AI 클러스터 복구 3](docs/journal/1.cheetah-복구/3_12_AI_클러스터_복구_3.md)          | Elasticsearch/Kibana 메모리 이슈       |
| [AI 클러스터 복구 4](docs/journal/1.cheetah-복구/3_12_AI_클러스터_복구_4.md)          | Kafka/Zookeeper 연동 이슈              |
| [AI 클러스터 복구 5](docs/journal/1.cheetah-복구/3_12_AI_클러스터_복구_5.md)          | MetalLB LoadBalancer 복구              |
| [NAS 복구 1](docs/journal/1.cheetah-복구/3_16_NAS_복구_1.md)                          | RAID 재구성 및 드라이브 장애 해결      |
| [NAS 복구 2](docs/journal/1.cheetah-복구/3_16_NAS_복구_2.md)                          | NFS 연결 및 K8s PV 복구                |
| [HPE ACPI 장애](docs/journal/1.cheetah-복구/3_17_HPE_ACPI_장애.md)                    | 커널-펌웨어 호환성 문제 해결           |
| [GPU 드라이버 업데이트 ⭐](docs/journal/1.cheetah-복구/3_19_GPU_드라이버_업데이트.md) | 재구축 결정의 근거가 된 핵심 판단 문서 |

### 2. GPU 클러스터 재구축

| 문서                                                                                                      | 내용                              |
| --------------------------------------------------------------------------------------------------------- | --------------------------------- |
| [데이터 백업](docs/journal/2.GPU_클러스터_재구축/1_백업.md)                                               | 레거시 시스템 데이터 NAS 아카이빙 |
| [Ubuntu OS 재설치](docs/journal/2.GPU_클러스터_재구축/2_우분투_OS_재설치.md)                              | LVM 제거, 고정 IP, 클린 설치      |
| [K8s 엔진 설치](docs/journal/2.GPU_클러스터_재구축/3_K8s_엔진_설치.md)                                    | Containerd, kubeadm, kubectl      |
| [K8s 클러스터 초기화](docs/journal/2.GPU_클러스터_재구축/4_K8s_클러스터_초기화.md)                        | kubeadm init, Join Token 발급     |
| [Calico CNI 설치](docs/journal/2.GPU_클러스터_재구축/5_Calico_CNI_설치.md)                                | Pod 네트워크 구성                 |
| [워커노드 구성 및 클러스터 합류](docs/journal/2.GPU_클러스터_재구축/6_워커_노드_구성_및_클러스터_합류.md) | 6대 워커노드 원샷 설치 스크립트   |
| [GPU 클러스터 완전 구축 가이드](docs/journal/2.GPU_클러스터_재구축/7_GPU_클러스터_완전_구축_가이드.md)    | 전체 재구축 상세 가이드           |

### 3. AI 팀 환경 구축

| 문서                                                                                                                             | 내용                                                      |
| -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| [RBAC + JupyterHub 구축 ⚠️(superseded)](docs/journal/3.AI_학습_팀_환경_구축/3_30_RBAC_Namespace_JupyterHub_구축.md)              | RBAC 5명 계정 · JupyterHub · GPU 자동 배정 · CUDA 검증    |
| [Grafana ServiceAccount 분리 및 보안 강화](docs/journal/3.AI_학습_팀_환경_구축/4_01_Grafana_ServiceAccount_분리_및_보안_강화.md) | grafana-sa 생성 · 최소 권한 원칙 적용                     |
| [GPU 노드 10GbE 라우팅 최적화 ⭐](docs/journal/3.AI_학습_팀_환경_구축/4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정.md)        | Pod 통신 경로 1GbE → 10GbE 우선 라우팅 · v100 케이블 교체 |

### 4. ML 파이프라인 구축

| 문서                                                                                                                                      | 내용                                                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [YOLOv8 COCO 학습 및 웹캠 추론 ⭐](docs/journal/4.ML_파이프라인_구축/4_02_YOLOv8_COCO_학습_및_웹캠_추론_테스트.md)                        | K8s Job 파이프라인 · mAP@0.5 46.8% · 실시간 추론 성공                                         |
| [YOLOv8 VisDrone 멀티GPU 학습 Job ⭐](docs/journal/4.ML_파이프라인_구축/4_03_YOLOv8_VisDrone_멀티GPU_학습_Job.md)                         | V100 4장 DDP · VisDrone 드론 Object Detection 학습                                            |
| [YOLOv8 VisDrone 학습 결과 보고서](docs/journal/4.ML_파이프라인_구축/4_04_YOLOv8_VisDrone_학습_결과_보고서.md)                            | 학습 결과 분석 · mAP@0.5 33.4% · 소형 객체 한계 분석                                          |
| [Argo Workflows 설치 및 WorkflowTemplate 구성 ⭐](docs/journal/4.ML_파이프라인_구축/4_06_Argo_Workflows_설치_및_WorkflowTemplate_구성.md) | Helm 설치 · MetalLB UI 노출 · 팀원 웹 UI Job 제출 환경 완성                                   |
| [Alertmanager 이메일 알람 구성 ⭐](docs/journal/4.ML_파이프라인_구축/4_07_Alertmanager_이메일_알람_구성.md)                               | Gmail SMTP 연동 · GPU 온도/메모리/노드 다운 알람 3종 · 실제 수신 확인                         |
| [Argo DAG 파이프라인 구성 ⭐](docs/runbooks/runbook_argo_dag.md)                                                                          | 데이터검증→학습→평가→버전별저장 4단계 · visdrone-v1.pt/v2.pt 버전 관리                        |
| [Argo Workflows Tailscale 접속 및 포트 변경](docs/journal/4.ML_파이프라인_구축/4_09_Argo_Workflows_Tailscale_접속_및_포트_변경.md)        | 포트 2746 → 30500 · systemd port-forward · Tailscale 외부 접속                                |
| [MLflow 설치 및 Argo DAG 연동 ⭐](docs/journal/4.ML_파이프라인_구축/4_13_MLflow_설치_및_Argo_DAG_연동.md)                                 | PostgreSQL 백엔드 · params 105개 + metrics 자동 기록 · 버전별 모델 저장                       |
| [GitHub Actions CI/CD ⭐](docs/journal/4.ML_파이프라인_구축/4_13_GitHub_Actions_CICD.md)                                                  | Self-hosted Runner · git push → Argo Workflow 자동 트리거 · 16s 완료                          |
| [MLflow alias + FastAPI 엔드포인트 분리 ⭐](docs/journal/4.ML_파이프라인_구축/4_15_MLflow_alias_FastAPI_엔드포인트_분리.md)               | Registry alias(champion) · NAS 우선 로드 · /predict-demo / /predict 분리 · 웹 UI 구축         |
| [YOLOv8 서빙 이미지화 ⭐](docs/journal/4.ML_파이프라인_구축/4_16_YOLOv8_서빙_이미지화.md)                                                 | nerdctl + buildkit 빌드 · pip install 런타임 제거 · ConfigMap 마운트 해제 · 장애 3건 해소     |
| [서빙 이미지 DockerHub 등록 ⭐](docs/journal/4.ML_파이프라인_구축/4_17_YOLOv8_서빙_이미지_DockerHub_등록.md)                              | `1jkim/yolov8-serving:v1` push · hostname nodeSelector 고정 해제 · 2080Ti 풀 전체 스케줄 가능 |
| [etcd 백업 및 DR 검증 ⭐](docs/runbooks/runbook_etcd_restore.md)                                                                          | 호스트 crontab 자동 백업 · NAS 저장 · snapshot 무결성 검증                                    |

### 5. 서비스 노출 및 운영 안정화

| 문서                                                                                                                                                               | 내용                                                                                                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| [Ingress + TLS 도입 및 host 기반 라우팅 ⭐](docs/journal/5.서비스_노출_및_운영_안정화/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md)                                | NGINX Ingress Controller · cert-manager · host 라우팅 · self-signed TLS · JupyterHub WebSocket 검증                   |
| [JupyterHub GitHub OAuth 전환 ⭐](docs/journal/5.서비스_노출_및_운영_안정화/4_27_JupyterHub_GitHub_OAuth_전환.md)                                                  | DummyAuthenticator 제거 · GitHubOAuthenticator 적용 · allowed_users 설정                                              |
| [PriorityClass 도입 Phase A/B/B-1 ⭐](docs/journal/5.서비스_노출_및_운영_안정화/4_28_PriorityClass_ResourceQuota_Phase_A_B_B1.md)                                  | PriorityClass 4계층 정의 · 시스템 파드 system-cluster-critical 적용 · MLflow 보강 · JupyterHub extraPodSpec 방식      |
| [PriorityClass Phase C + LimitRange Phase D + ResourceQuota Phase E ⭐](docs/journal/5.서비스_노출_및_운영_안정화/4_28_PriorityClass_ResourceQuota_Phase_C_D_E.md) | serving-critical(yolov8-serving) · training-normal(Argo WorkflowTemplate) · LimitRange · ResourceQuota(GPU 16장 상한) |
| [Ingress + TLS Runbook](docs/runbooks/runbook_ingress_tls.md)                                                                                                      | 신규 서비스를 host 기반 Ingress + cluster-ca TLS로 노출하는 표준 절차                                                 |
| [Let's Encrypt DNS-01 Runbook](docs/runbooks/runbook_letsencrypt_dns01.md)                                                                                         | 본인 도메인 + Cloudflare DNS API 기반 공인 인증서 전환 절차                                                           |

### 6. 장애 기록 (incidents)

| 문서                                                                                                                  | 내용                                                                                                                   |
| --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [10G NIC 이전 ⭐](docs/incidents/3_23_10G_NIC_이전.md)                                                                | 병목 발견 → 하드웨어 재배치 → 10G 개통                                                                                 |
| [2080Ti GPU 미인식 ⭐](docs/incidents/3_23_2080Ti_GPU_미인식.md)                                                      | 8장 중 1장 미인식 → 27장 확정 판단                                                                                     |
| [NAS NIC 접촉불량 ⭐](docs/incidents/3_27_NAS_NIC_접촉불량.md)                                                        | rev ff 진단 → PCIe 재삽입 → 9.41Gbps 복구                                                                              |
| [Grafana DCGM No data](docs/incidents/3_27_Grafana_DCGM_No_data.md)                                                   | ServiceMonitor 라벨 불일치 → json replace 패치                                                                         |
| [Portainer Tailscale 접속불가](docs/incidents/3_27_Portainer_Tailscale_접속불가.md)                                   | port-forward 좀비 상태 → 서비스 재시작                                                                                 |
| [네트워크 장애 및 클러스터 설계 개선 ⭐](docs/incidents/3_31_네트워크_장애_및_클러스터_설계_개선.md)                  | MetalLB ARP 충돌 · Calico 연쇄 장애 · IP Pool 재설계                                                                   |
| [JupyterHub 다중 접속 장애 및 서비스 재설계 ⭐](docs/incidents/4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md) | port-forward 제거 · MetalLB IP 재설계 · 공용/관리자 접속 분리                                                          |
| [Grafana 대시보드 미표시 및 RBAC 권한 문제](docs/incidents/4_01_Grafana_대시보드_미표시_및_RBAC_권한_문제.md)         | 물리 네트워크 단절 + RBAC 권한 부족 동시 진단                                                                          |
| [Alertmanager Silence — DaemonSet 오탐 억제](docs/incidents/4_09_Alertmanager_Silence_DaemonSet_알람_억제.md)         | continuous-image-puller 오탐 분석 · PrometheusRule 대신 Silence 적용                                                   |
| [이미지화 장애 3건 ⭐](docs/incidents/4_16_incidents_이미지화_장애_3건.md)                                            | containerd namespace 격리 · ConfigMap read-only 충돌 · nodeSelector 범위 불일치                                        |
| [Grafana chart 의도치 않은 업그레이드 ⭐](docs/incidents/4_28_grafana_chart_upgrade.md)                               | helm upgrade 버전 미고정 → chart 자동 업그레이드 → CrashLoopBackOff · NodePort 변경 · PVC 미마운트 · 동일 세션 내 복구 |

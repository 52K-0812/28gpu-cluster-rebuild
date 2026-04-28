# 클러스터 구조 다이어그램

> **기준일:** 2026-04-28
> **K8s:** v1.29.15 / Ubuntu 22.04.5 LTS / Calico v3.27

---

## 1. 🖥️ 노드 구성 및 역할 분리

| 노드 | 역할 | 배치 파드 / 용도 | GPU | IP |
|---|---|---|---|---|
| master-01 | Control Plane 전용 | etcd · apiserver · scheduler · controller-manager<br>GitHub Actions Runner · etcd 백업 crontab<br>**NoSchedule taint 적용** | — | `<MASTER-IP>` |
| master-02 | 시스템 파드 전담 Worker | Prometheus · Grafana · Portainer · Alertmanager<br>JupyterHub hub/proxy · MLflow · Argo Controller<br>**NGINX Ingress Controller** | — | `<WORKER-IP-02>` |
| v100-gpu-01 | Worker (학습 전용) | Argo DAG 학습 Job (V100 × 4 DDP) | V100 × 4 | 10.10.10.153 |
| 2080ti-gpu-02 | Worker | 학습 · 서빙 워크로드 | 2080Ti × 8 | 10.10.10.154 |
| 2080ti-gpu-03 | Worker | 학습 · 서빙 워크로드 | 2080Ti × 7 | 10.10.10.155 |
| 2080ti-gpu-04 | Worker | 학습 · 서빙 워크로드 | 2080Ti × 8 | 10.10.10.156 |
| NAS (nas-01) | 스토리지 | 28TB NFS 공유 스토리지 | — | `<MASTER-IP>` (1G) / 10.10.10.157 (10G) |

> **총 GPU:** V100 × 4 + 2080Ti × 23 = **27장**
> **설계 근거:** 3_31 네트워크 장애 후 SoC 원칙 적용

---

## 2. 🌐 네트워크 토폴로지

```mermaid
graph TD
    EXT1["🌐 Tailscale VPN\n<TAILSCALE-IP>"]
    EXT2["🌐 캠퍼스 내부망\nNGINX Ingress\n<LB-INGRESS-IP>"]

    subgraph MGMT["관리망 — 1GbE (Cisco Catalyst 2960G)"]
        M01["master-01\nControl Plane 전용\nNoSchedule taint\nGitHub Runner · etcd 백업"]
        M02["master-02\n시스템 파드 전담\nPrometheus · Grafana · Portainer\nMLflow · Argo · NGINX Ingress"]
        NAS1G["NAS (nas-01)\n28TB\n1G: <MASTER-IP>"]
    end

    subgraph DATA["데이터망 — 10GbE (NETGEAR XS508M)"]
        G1["v100-gpu-01\nV100 × 4\n학습 전용\n10.10.10.153"]
        G2["2080ti-gpu-02\n2080Ti × 8\n10.10.10.154"]
        G3["2080ti-gpu-03\n2080Ti × 7\n10.10.10.155"]
        G4["2080ti-gpu-04\n2080Ti × 8\n10.10.10.156"]
        NAS10G["NAS (nas-01)\n10G: 10.10.10.157"]
    end

    EXT1 -->|VPN| M01
    EXT2 -->|MetalLB L2| M02
    M01 --- M02
    M01 --- NAS1G
    M02 --- NAS1G

    G1 -->|10GbE| NAS10G
    G2 -->|10GbE| NAS10G
    G3 -->|10GbE| NAS10G
    G4 -->|10GbE| NAS10G

    style M01 fill:#dbeafe,stroke:#2563eb
    style M02 fill:#dbeafe,stroke:#2563eb
    style NAS1G fill:#dcfce7,stroke:#16a34a
    style NAS10G fill:#dcfce7,stroke:#16a34a
    style G1 fill:#fef3c7,stroke:#d97706
    style G2 fill:#fee2e2,stroke:#dc2626
    style G3 fill:#fee2e2,stroke:#dc2626
    style G4 fill:#fee2e2,stroke:#dc2626
    style EXT2 fill:#ede9fe,stroke:#7c3aed
```

---

## 3. 🔄 ML 파이프라인 흐름

```mermaid
graph LR
    DEV["개발자<br/>git push → main"]

    subgraph CICD["CI/CD (master-01)"]
        GHA["GitHub Actions<br/>Self-hosted Runner<br/>~16s 트리거"]
    end

    subgraph ARGO["Argo Workflows DAG (ai-team NS)"]
        VAL["validate-data<br/>VisDrone 존재 확인"]
        TRAIN["train<br/>V100 × 4 DDP<br/>YOLOv8n"]
        EVAL["evaluate<br/>mAP 측정<br/>MLflow metrics 기록"]
        SAVE["save-model<br/>NAS 저장<br/>Registry 등록"]
        VAL --> TRAIN --> EVAL --> SAVE
    end

    subgraph MLFLOW["MLflow (mlflow NS)"]
        PG["PostgreSQL<br/>메타데이터 백엔드"]
        ART["NFS PVC<br/>/data/mlflow-artifacts/"]
        REG["Model Registry<br/>visdrone-yolov8<br/>alias: champion"]
        PG --- ART
        PG --- REG
    end

    NAS["NAS<br/>/data/datasets/models/<br/>visdrone-vN.pt"]

    subgraph SERVE["Serving (ai-team NS)"]
        API["FastAPI<br/>predict-demo: COCO<br/>predict: champion"]
    end

    DEV --> GHA --> VAL
    TRAIN -->|"log_params 105개"| PG
    EVAL -->|"log_metrics 7개<br/>log_artifact best.pt"| PG
    SAVE --> NAS
    SAVE --> REG
    REG --> API
    NAS --> API

    style GHA fill:#dbeafe,stroke:#2563eb
    style VAL fill:#fef3c7,stroke:#d97706
    style TRAIN fill:#fef3c7,stroke:#d97706
    style EVAL fill:#fef3c7,stroke:#d97706
    style SAVE fill:#dcfce7,stroke:#16a34a
    style PG fill:#ede9fe,stroke:#7c3aed
    style ART fill:#dcfce7,stroke:#16a34a
    style REG fill:#ede9fe,stroke:#7c3aed
    style NAS fill:#dcfce7,stroke:#16a34a
    style API fill:#dbeafe,stroke:#2563eb
```

---

## 4. 🔌 서비스 맵

```mermaid
graph TD
    VPN["Tailscale VPN\n<TAILSCALE-IP>"]
    INGRESS["NGINX Ingress\n<LB-INGRESS-IP>\ncert-manager TLS"]

    subgraph EXTERNAL["외부 사용자 서비스 — HTTPS Ingress 경유"]
        JH["JupyterHub\nhub.<LB-INGRESS-IP>.nip.io\nGitHub OAuth ✅"]
        FA["FastAPI YOLOv8\nserving.<LB-INGRESS-IP>.nip.io\n77ms · 2080Ti × 1\n⚠️ 인증 미적용"]
    end

    subgraph ADMIN["관리자 서비스 — NodePort (Ingress Phase B 대상)"]
        MLF["MLflow\n:30010\nPostgreSQL 백엔드"]
        GF["Grafana\n:30300\nGPU 대시보드"]
        FB["Filebrowser\n:30340\nNAS 웹 탐색기"]
        PORT["Portainer\n:30320\nTailscale 전용"]
        ARGOUI["Argo UI\n<LB-ARGO-IP>:2746"]
    end

    subgraph MONITORING["모니터링 내부"]
        PROM["Prometheus\n:30310\n내부 전용"]
        AM["Alertmanager\nGmail SMTP"]
        GF --- PROM
        PROM --- AM
    end

    INGRESS --> JH
    INGRESS --> FA
    VPN --> MLF
    VPN --> GF
    VPN --> FB
    VPN --> PORT
    VPN --> ARGOUI

    style JH fill:#dbeafe,stroke:#2563eb
    style FA fill:#dbeafe,stroke:#2563eb
    style MLF fill:#ede9fe,stroke:#7c3aed
    style ARGOUI fill:#fef3c7,stroke:#d97706
    style GF fill:#dcfce7,stroke:#16a34a
    style PROM fill:#dcfce7,stroke:#16a34a
    style FB fill:#dcfce7,stroke:#16a34a
    style AM fill:#dcfce7,stroke:#16a34a
    style PORT fill:#f1f5f9,stroke:#64748b
    style INGRESS fill:#ede9fe,stroke:#7c3aed
```

---

## 5. 💾 스토리지 구조

```mermaid
graph TD
    NAS["NAS (nas-01)\n28TB · /data"]

    subgraph PROV["K8s 스토리지 레이어"]
        SC["StorageClass: nfs-client\nNFS Provisioner on master-01"]
    end

    subgraph PVCS["PVC 목록"]
        P1["nfs-datasets-pvc\n/data/datasets/\nVisDrone · 모델 · runs"]
        P2["mlflow-artifacts-pvc\n/data/mlflow-artifacts/\n100Gi"]
        P3["mlflow-postgres-pvc\nPostgreSQL 데이터\n10Gi"]
        P4["etcd 백업\n/data/backups/etcd/\n7일 보존 · crontab 02:00"]
        P5["JupyterHub PVC\nclaim-{username}\n사용자별 자동 생성"]
    end

    subgraph CONSUMERS["사용 주체"]
        C1["Argo DAG\n학습 Job"]
        C2["MLflow\nTracking Server"]
        C3["PostgreSQL\nDeployment"]
        C4["master-01\ncrontab"]
        C5["JupyterHub\n사용자 노트북"]
    end

    NAS --> SC
    SC --> P1
    SC --> P2
    SC --> P3
    SC --> P5
    NAS -->|"10GbE 직접"| P4

    P1 --> C1
    P2 --> C2
    P3 --> C3
    P4 --> C4
    P5 --> C5

    style NAS fill:#dcfce7,stroke:#16a34a
    style SC fill:#dbeafe,stroke:#2563eb
    style P1 fill:#fef3c7,stroke:#d97706
    style P2 fill:#ede9fe,stroke:#7c3aed
    style P3 fill:#ede9fe,stroke:#7c3aed
    style P4 fill:#dcfce7,stroke:#16a34a
    style P5 fill:#dbeafe,stroke:#2563eb
```

---

## 6. 🔢 PriorityClass 계층 (2026-04-28 도입)

| 클래스명 | value | 적용 대상 |
|---|---|---|
| system-cluster-critical | 2,000,000,000 | kube-system 시스템 파드 |
| serving-critical | 1,000,000 | yolov8-serving |
| training-normal | 100 | Argo DAG 학습 Job |
| (default) | 0 | 나머지 모든 파드 |

> **설계 원칙:** serving-critical은 학습 Job보다 항상 선점 우위. 스케줄러가 GPU 부족 시 training-normal Pod를 선점하여 serving Pod 보호.

---

## 변경 이력

| 날짜 | 변경 내용 |
|---|---|
| 2026-03-31 | master-01 NoSchedule taint 적용, master-02 시스템 파드 전담 설계 확정 |
| 2026-04-13 | MLflow, GitHub Actions CI/CD, etcd DR 검증 반영 |
| 2026-04-15 | FastAPI champion serving, MLflow alias 기반 운영 흐름 반영 |
| 2026-04-17 | 서빙 이미지 DockerHub 전환 (`1jkim/yolov8-serving:v1`), 2080ti-gpu-04 hostname nodeSelector 고정 해제 |
| 2026-04-27 | NGINX Ingress + cert-manager TLS 완료, GitHub OAuth 완료, 서비스 맵 3분류(외부/관리자/내부) 체계로 재편, JupyterHub PVC 사용자별 자동 생성 반영 |
| 2026-04-28 | PriorityClass 4계층 도입 (serving-critical/training-normal), INC-2026-04-28 Grafana 차트 업그레이드 장애 복구 |

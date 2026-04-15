# 클러스터 구조 다이어그램

> **기준일:** 2026-04-15
> **K8s:** v1.29.15 / Ubuntu 22.04.5 LTS / Calico v3.27

---

## 1. 노드 구성 및 역할 분리

| 노드 | 역할 | 배치 파드 / 용도 | GPU | IP |
|---|---|---|---|---|
| master-01 | Control Plane 전용 | etcd · apiserver · scheduler · controller-manager<br>GitHub Actions Runner · etcd 백업 crontab<br>**NoSchedule taint 적용** | — | MASTER-IP |
| master-02 | 시스템 파드 전담 Worker | Prometheus · Grafana · Portainer · Alertmanager<br>JupyterHub hub/proxy · MLflow · Argo Controller | — | WORKER-IP-02 |
| v100-gpu-01 | Worker (학습 전용) | Argo DAG 학습 Job (V100 × 4 DDP) | V100 × 4 | 10.10.10.153 |
| 2080ti-gpu-02 | Worker | 학습 워크로드 | 2080Ti × 8 | 10.10.10.154 |
| 2080ti-gpu-03 | Worker | 학습 워크로드 | 2080Ti × 7 | 10.10.10.155 |
| 2080ti-gpu-04 | Worker (서빙 전용) | FastAPI YOLOv8 서빙 (2080Ti × 1) | 2080Ti × 8 | 10.10.10.156 |
| NAS (nas-01) | 스토리지 | 28TB NFS 공유 스토리지 | — | MASTER-IP (1G) / 10.10.10.157 (10G) |

> **총 GPU:** V100 × 4 + 2080Ti × 23 = **27장**
> **설계 근거:** 3_31 네트워크 장애 후 SoC 원칙 적용

---

## 2. 네트워크 토폴로지

```mermaid
graph TD
    EXT["🌐 Tailscale VPN\n100.76.84.68"]

    subgraph MGMT["관리망 — 1GbE (Cisco Catalyst 2960G)"]
        M01["master-01\nControl Plane 전용\nNoSchedule taint\nGitHub Runner · etcd 백업"]
        M02["master-02\n시스템 파드 전담\nPrometheus · Grafana\nPortainer · MLflow · Argo"]
        NAS1G["NAS (nas-01)\n28TB\n1G: MASTER-IP"]
    end

    subgraph DATA["데이터망 — 10GbE (NETGEAR XS508M)"]
        G1["v100-gpu-01\nV100 × 4\n학습 전용\n10.10.10.153"]
        G2["2080ti-gpu-02\n2080Ti × 8\n10.10.10.154"]
        G3["2080ti-gpu-03\n2080Ti × 7\n10.10.10.155"]
        G4["2080ti-gpu-04\n2080Ti × 8\n서빙 전용\n10.10.10.156"]
        NAS10G["NAS (nas-01)\n10G: 10.10.10.157"]
    end

    EXT -->|VPN| M01
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
```

---

## 3. ML 파이프라인 흐름

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

    subgraph MLFLOW["MLflow (mlflow NS :30010)"]
        PG["PostgreSQL<br/>메타데이터 백엔드"]
        ART["NFS PVC<br/>/data/mlflow-artifacts/"]
        REG["Model Registry<br/>visdrone-yolov8<br/>alias: champion"]
        PG --- ART
        PG --- REG
    end

    NAS["NAS<br/>/data/datasets/models/<br/>visdrone-vN.pt"]

    subgraph SERVE["Serving (ai-team NS :30600)"]
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

## 4. 서비스 맵

```mermaid
graph TD
    VPN["Tailscale VPN\n100.76.84.68"]

    subgraph AITEAM["ai-team NS"]
        JH["JupyterHub\n:30000\nDummyAuth → OAuth 예정"]
        FA["FastAPI YOLOv8\n:30600\n77ms · 2080Ti × 1"]
    end

    subgraph MLFLOWNS["mlflow NS"]
        MLF["MLflow\n:30010\nPostgreSQL 백엔드"]
    end

    subgraph ARGONS["argo NS"]
        ARGOUI["Argo UI\n:30500"]
    end

    subgraph MONITORING["monitoring NS"]
        GF["Grafana\n:30300\nGPU 대시보드"]
        PROM["Prometheus\n:30310"]
        FB["Filebrowser\n:30340\nNAS 웹 탐색기"]
        AM["Alertmanager\nGmail SMTP"]
        GF --- PROM
        PROM --- AM
    end

    subgraph PORTAINER["portainer NS"]
        PORT["Portainer\n:30320"]
    end

    VPN --> JH
    VPN --> FA
    VPN --> MLF
    VPN --> ARGOUI
    VPN --> GF
    VPN --> FB
    VPN --> PORT

    style JH fill:#dbeafe,stroke:#2563eb
    style FA fill:#dbeafe,stroke:#2563eb
    style MLF fill:#ede9fe,stroke:#7c3aed
    style ARGOUI fill:#fef3c7,stroke:#d97706
    style GF fill:#dcfce7,stroke:#16a34a
    style PROM fill:#dcfce7,stroke:#16a34a
    style FB fill:#dcfce7,stroke:#16a34a
    style AM fill:#dcfce7,stroke:#16a34a
    style PORT fill:#f1f5f9,stroke:#64748b
```

---

## 5. 스토리지 구조

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
    end

    subgraph CONSUMERS["사용 주체"]
        C1["Argo DAG\n학습 Job"]
        C2["MLflow\nTracking Server"]
        C3["PostgreSQL\nDeployment"]
        C4["master-01\ncrontab"]
    end

    NAS --> SC
    SC --> P1
    SC --> P2
    SC --> P3
    NAS -->|"10GbE 직접"| P4

    P1 --> C1
    P2 --> C2
    P3 --> C3
    P4 --> C4

    style NAS fill:#dcfce7,stroke:#16a34a
    style SC fill:#dbeafe,stroke:#2563eb
    style P1 fill:#fef3c7,stroke:#d97706
    style P2 fill:#ede9fe,stroke:#7c3aed
    style P3 fill:#ede9fe,stroke:#7c3aed
    style P4 fill:#dcfce7,stroke:#16a34a
```

---

## 변경 이력

| 날짜 | 변경 내용 |
|---|---|
| 2026-03-31 | master-01 NoSchedule taint 적용, master-02 시스템 파드 전담 설계 확정 |
| 2026-04-13 | MLflow, GitHub Actions CI/CD, etcd DR 검증 반영 |
| 2026-04-15 | FastAPI champion serving, MLflow alias 기반 운영 흐름 반영 |

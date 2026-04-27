# 클러스터 구조 다이어그램

> **기준일:** 2026-04-27
> **K8s:** v1.29.15 / Ubuntu 22.04.5 LTS / Calico v3.27

---

## 1. 🖥️ 노드 구성 및 역할 분리

| 노드 | 역할 | 배치 파드 / 용도 | GPU | IP |
|---|---|---|---|---|
| master-01 | Control Plane 전용 | etcd · apiserver · scheduler · controller-manager<br>GitHub Actions Runner · etcd 백업 crontab<br>**NoSchedule taint 적용** | — | MASTER-IP |
| master-02 | 시스템 파드 전담 Worker | Prometheus · Grafana · Portainer · Alertmanager<br>JupyterHub hub/proxy · MLflow · Argo Controller<br>**NGINX Ingress Controller · cert-manager (2026-04-27 신규)** | — | WORKER-IP-02 |
| v100-gpu-01 | Worker (학습 전용) | Argo DAG 학습 Job (V100 × 4 DDP) | V100 × 4 | 10.10.10.153 |
| 2080ti-gpu-02 | Worker | 학습 워크로드 | 2080Ti × 8 | 10.10.10.154 |
| 2080ti-gpu-03 | Worker | 학습 워크로드 | 2080Ti × 7 | 10.10.10.155 |
| 2080ti-gpu-04 | Worker | 학습 / 서빙 자동 스케줄 | 2080Ti × 8 | 10.10.10.156 |
| NAS (nas-01) | 스토리지 | 28TB NFS 공유 스토리지 | — | MASTER-IP (1G) / 10.10.10.157 (10G) |

> **총 GPU:** V100 × 4 + 2080Ti × 23 = **27장**
> **설계 근거:** 3_31 네트워크 장애 후 SoC 원칙 적용

---

## 2. 🌐 네트워크 토폴로지

```mermaid
graph TD
    EXT["🌐 Tailscale VPN<br/>TAILSCALE-IP"]
    LAN["🌐 캠퍼스 LAN<br/>112.76.56.x"]

    INGRESS["🔐 Ingress Gateway<br/>INGRESS-LB-IP<br/>NGINX Ingress + cert-manager<br/>host 기반 라우팅 + TLS"]

    subgraph MGMT["관리망 — 1GbE (Cisco Catalyst 2960G)"]
        M01["master-01<br/>Control Plane 전용<br/>NoSchedule taint<br/>GitHub Runner · etcd 백업"]
        M02["master-02<br/>시스템 파드 전담<br/>Prometheus · Grafana<br/>Portainer · MLflow · Argo<br/>Ingress Controller · cert-manager"]
        NAS1G["NAS (nas-01)<br/>28TB<br/>1G: MASTER-IP"]
    end

    subgraph DATA["데이터망 — 10GbE (NETGEAR XS508M)"]
        G1["v100-gpu-01<br/>V100 × 4<br/>학습 전용<br/>10.10.10.153"]
        G2["2080ti-gpu-02<br/>2080Ti × 8<br/>10.10.10.154"]
        G3["2080ti-gpu-03<br/>2080Ti × 7<br/>10.10.10.155"]
        G4["2080ti-gpu-04<br/>2080Ti × 8<br/>10.10.10.156"]
        NAS10G["NAS (nas-01)<br/>10G: 10.10.10.157"]
    end

    EXT -->|VPN| M01
    LAN -->|HTTPS 443| INGRESS
    INGRESS -.->|backend 라우팅| M02
    M01 --- M02
    M01 --- NAS1G
    M02 --- NAS1G

    G1 -->|10GbE| NAS10G
    G2 -->|10GbE| NAS10G
    G3 -->|10GbE| NAS10G
    G4 -->|10GbE| NAS10G

    style M01 fill:#dbeafe,stroke:#2563eb
    style M02 fill:#dbeafe,stroke:#2563eb
    style INGRESS fill:#fce7f3,stroke:#db2777
    style NAS1G fill:#dcfce7,stroke:#16a34a
    style NAS10G fill:#dcfce7,stroke:#16a34a
    style G1 fill:#fef3c7,stroke:#d97706
    style G2 fill:#fee2e2,stroke:#dc2626
    style G3 fill:#fee2e2,stroke:#dc2626
    style G4 fill:#fee2e2,stroke:#dc2626
```

---

## 3. 🔐 Ingress 라우팅 구조 (2026-04-27 신규)

```mermaid
graph LR
    USER["👤 사용자<br/>브라우저"]

    DNS["🌐 nip.io wildcard DNS<br/>*.INGRESS-LB-IP.nip.io<br/>→ INGRESS-LB-IP"]

    subgraph GATEWAY["Ingress Gateway (master-02)"]
        NGX["NGINX Ingress Controller<br/>LoadBalancer INGRESS-LB-IP<br/>:80 / :443"]
        CM["cert-manager<br/>cluster-ca-issuer<br/>self-signed TLS"]
    end

    subgraph INGRESSES["Ingress Resources"]
        I1["serving.INGRESS-LB-IP.nip.io<br/>tls: yolov8-serving-tls"]
        I2["hub.INGRESS-LB-IP.nip.io<br/>tls: jupyterhub-tls"]
    end

    subgraph BACKENDS["Backend Services (ai-team NS)"]
        S1["yolov8-serving:8080<br/>FastAPI"]
        S2["proxy-public:80<br/>JupyterHub Proxy"]
    end

    USER -->|"https://*.nip.io"| DNS
    DNS --> NGX
    CM -.->|"인증서 발급/갱신"| NGX
    NGX --> I1
    NGX --> I2
    I1 --> S1
    I2 --> S2

    style NGX fill:#fce7f3,stroke:#db2777
    style CM fill:#fce7f3,stroke:#db2777
    style I1 fill:#dbeafe,stroke:#2563eb
    style I2 fill:#dbeafe,stroke:#2563eb
    style S1 fill:#dcfce7,stroke:#16a34a
    style S2 fill:#dcfce7,stroke:#16a34a
```

> **TLS 인증서:** cluster-ca self-signed (cert-manager 자동 갱신, 1년 만료, 30일 전 자동 갱신)
> **Phase B 예정:** Grafana, Argo, Portainer, Filebrowser, MLflow Ingress 추가 예정 (관리자 그룹은 Basic Auth 추가)

---

## 4. 🔄 ML 파이프라인 흐름

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

    subgraph SERVE["Serving (ai-team NS)"]
        API["FastAPI<br/>predict-demo: COCO<br/>predict: champion<br/>https://serving.*.nip.io"]
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

## 5. 🔌 서비스 맵

```mermaid
graph TD
    VPN["🌐 Tailscale VPN<br/>TAILSCALE-IP"]
    LAN["🌐 캠퍼스 LAN<br/>112.76.56.x"]
    INGRESS["🔐 Ingress Gateway<br/>INGRESS-LB-IP<br/>host 기반 + TLS"]

    subgraph AITEAM["ai-team NS"]
        JH["JupyterHub<br/>:30000 (NodePort, fallback)<br/>JUPYTERHUB-LB-IP (LB, fallback)<br/>hub.*.nip.io (Ingress, 메인)"]
        FA["FastAPI YOLOv8<br/>:30600 (NodePort, fallback)<br/>serving.*.nip.io (Ingress, 메인)<br/>77ms · 2080Ti × 1"]
    end

    subgraph MLFLOWNS["mlflow NS"]
        MLF["MLflow<br/>:30010 (NodePort)<br/>MLFLOW-LB-IP (LB)<br/>(Phase B 예정: mlflow.*.nip.io)"]
    end

    subgraph ARGONS["argo NS"]
        ARGOUI["Argo UI<br/>:30500 (NodePort)<br/>ARGO-LB-IP (LB)<br/>(Phase B 예정: argo.*.nip.io)"]
    end

    subgraph MONITORING["monitoring NS"]
        GF["Grafana :30300<br/>(Phase B: grafana.*.nip.io + Basic Auth)"]
        PROM["Prometheus :30310<br/>(Phase C: 폐쇄 예정)"]
        FB["Filebrowser :30340<br/>(Phase B: files.*.nip.io + Basic Auth)"]
        AM["Alertmanager<br/>Gmail SMTP"]
        GF --- PROM
        PROM --- AM
    end

    subgraph PORTAINER["portainer NS"]
        PORT["Portainer :30320<br/>(Phase B: portainer.*.nip.io + Basic Auth)"]
    end

    LAN --> INGRESS
    INGRESS -->|"hub.*.nip.io"| JH
    INGRESS -->|"serving.*.nip.io"| FA

    VPN --> JH
    VPN --> FA
    VPN --> MLF
    VPN --> ARGOUI
    VPN --> GF
    VPN --> FB
    VPN --> PORT

    style INGRESS fill:#fce7f3,stroke:#db2777
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

> **2026-04-27 변경:** 팀원 그룹(JupyterHub, YOLOv8) Ingress 통합 완료. 관리자/공개 그룹은 Phase B 예정. 기존 NodePort/LoadBalancer 경로는 fallback으로 유지.

---

## 6. 💾 스토리지 구조

```mermaid
graph TD
    NAS["NAS (nas-01)<br/>28TB · /data"]

    subgraph PROV["K8s 스토리지 레이어"]
        SC["StorageClass: nfs-client<br/>NFS Provisioner on master-01"]
    end

    subgraph PVCS["PVC 목록"]
        P1["nfs-datasets-pvc<br/>/data/datasets/<br/>VisDrone · 모델 · runs"]
        P2["mlflow-artifacts-pvc<br/>/data/mlflow-artifacts/<br/>100Gi"]
        P3["mlflow-postgres-pvc<br/>PostgreSQL 데이터<br/>10Gi"]
        P4["etcd 백업<br/>/data/backups/etcd/<br/>7일 보존 · crontab 02:00"]
    end

    subgraph CONSUMERS["사용 주체"]
        C1["Argo DAG<br/>학습 Job"]
        C2["MLflow<br/>Tracking Server"]
        C3["PostgreSQL<br/>Deployment"]
        C4["master-01<br/>crontab"]
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
| 2026-04-17 | 서빙 이미지 DockerHub 전환 (`1jkim/yolov8-serving:v1`), nodeSelector hostname 고정 제거 반영 |
| 2026-04-27 | **Ingress + TLS 도입**: NGINX Ingress Controller (master-02) + cert-manager + cluster-ca self-signed. host 기반 라우팅으로 `serving`, `hub` 통합. MetalLB IP 162 추가. Phase A 완료. |

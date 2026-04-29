# GPU 노드 네트워크 최적화

## 🗂️ 1. 작업 개요

> **작업일:** 2026-04-02
> **목적:** GPU 노드 간 Pod 통신을 10GbE 로 우선 라우팅하여 DCGM exporter 안정성 및 학습 성능 확보
> **대상:** v100-gpu-01, 2080ti-gpu-02~04 (GPU 워커 노드 전체)
> **작업 환경:** Kubernetes v1.29, Ubuntu 22.04
> **원인:** GPU 노드 기본 라우트가 1GbE 인터페이스로 설정 → Calico Pod 통신 느림
> **최종 결과:** 전 GPU 노드 Pod 네트워크를 10GbE 인터페이스로 우선 라우팅하도록 metric 조정

---

## 2. 문제 배경

### 2.1 기존 문제 (3/27 DCGM 데이터 미표시)

- Grafana에서 DCGM exporter 메트릭이 간헐적으로 `No data`
- 원인: Prometheus → DCGM exporter 스크래핑이 **1GbE 경로**로 가면서 타임아웃 발생

### 2.2 근본 원인

GPU 노드들이 **기본 라우트를 1GbE 인터페이스로 설정**되어 있었음:

```bash
# 작업 전 상태
v100-gpu-01:    default via LB_PUBLIC_IP dev enp2s0f0 (1GbE)
2080ti-gpu-02:  default via LB_PUBLIC_IP dev eno1 (1GbE)
2080ti-gpu-03:  default via LB_PUBLIC_IP dev eno1 (1GbE)
2080ti-gpu-04:  default via LB_PUBLIC_IP dev eno1 (1GbE)
```

→ Calico Pod 간 통신(`192.168.x.x` 대역)도 1GbE로 라우팅됨

---

## 3. 하드웨어 구성 확인

### 3.1 노드별 네트워크 인터페이스

| 노드          | 1GbE 인터페이스 | 10GbE 인터페이스 | 비고        |
| ------------- | --------------- | ---------------- | ----------- |
| v100-gpu-01   | `enp2s0f0`      | `enp2s0f1`       | DGX Station |
| 2080ti-gpu-02 | `eno1`          | `eno2`           | Supermicro  |
| 2080ti-gpu-03 | `eno1`          | `eno2`           | Supermicro  |
| 2080ti-gpu-04 | `eno1`          | `eno2`           | Supermicro  |

### 3.2 인터페이스 속도 확인

**초기 상태:**

```bash
# v100-gpu-01
ethtool enp2s0f0 | grep Speed
# → Speed: 100Mb/s ⚠️ 비정상!

ethtool enp2s0f1 | grep Speed
# → Speed: 10000Mb/s ✅
```

**문제 발견:** `enp2s0f0`가 100Mbps로 협상됨 (정상: 1000Mbps)

**원인:** Cat5e 케이블은 맞지만 단선 추정

**해결:** 케이블 교체 후 1000Mbps로 복구

---

## 4. 해결 과정

### 4.1 케이블 문제 해결 (v100-gpu-01)

**🔍 증상:**

```bash
ethtool enp2s0f0
# Supported link modes: 100baseT/Full, 1000baseT/Full, 10000baseT/Full
# Speed: 100Mb/s  ← NIC는 1G 지원하는데 100M으로 협상됨
# Auto-negotiation: on
```

**🛠️ 조치:**

- Cat5e 불량 케이블 → **Cat5e 정상 케이블로 교체**

**✅ 결과:**

```bash
ethtool enp2s0f0 | grep Speed
# → Speed: 1000Mb/s ✅
```

---

### 4.2 netplan 라우팅 규칙 생성

Calico Pod 통신 대역(`192.168.0.0/16`)을 10GbE 인터페이스로 우선 라우팅하도록 설정.

#### v100-gpu-01용 설정 파일

```yaml
# /etc/netplan/99-k8s-routes.yaml
network:
  version: 2
  ethernets:
    enp2s0f1:
      routes:
        - to: 192.168.0.0/16
          via: 0.0.0.0
          metric: 50
          scope: link
```

#### 2080ti-gpu-02~04용 설정 파일

```yaml
# /etc/netplan/99-k8s-routes.yaml
network:
  version: 2
  ethernets:
    eno2:
      routes:
        - to: 192.168.0.0/16
          via: 0.0.0.0
          metric: 50
          scope: link
```

---

### 4.3 배포 및 적용

**1단계: master-01에서 설정 파일 생성**

```bash
# v100용
cat <<'EOF' > /tmp/99-k8s-routes-v100.yaml
network:
  version: 2
  ethernets:
    enp2s0f1:
      routes:
        - to: 192.168.0.0/16
          via: 0.0.0.0
          metric: 50
          scope: link
EOF

# 2080ti용
cat <<'EOF' > /tmp/99-k8s-routes-2080ti.yaml
network:
  version: 2
  ethernets:
    eno2:
      routes:
        - to: 192.168.0.0/16
          via: 0.0.0.0
          metric: 50
          scope: link
EOF
```

**2단계: v100-gpu-01 배포**

```bash
scp /tmp/99-k8s-routes-v100.yaml ubuntu@LB_PUBLIC_IP:/tmp/
ssh ubuntu@LB_PUBLIC_IP "sudo cp /tmp/99-k8s-routes-v100.yaml /etc/netplan/99-k8s-routes.yaml && sudo netplan apply"
```

**3단계: 2080ti-gpu-02~04 배포**

```bash
# 2080ti-gpu-02
scp /tmp/99-k8s-routes-2080ti.yaml ubuntu@LB_PUBLIC_IP:/tmp/
ssh ubuntu@LB_PUBLIC_IP "sudo cp /tmp/99-k8s-routes-2080ti.yaml /etc/netplan/99-k8s-routes.yaml && sudo netplan apply"

# 2080ti-gpu-03
scp /tmp/99-k8s-routes-2080ti.yaml ubuntu@LB_PUBLIC_IP:/tmp/
ssh ubuntu@LB_PUBLIC_IP "sudo cp /tmp/99-k8s-routes-2080ti.yaml /etc/netplan/99-k8s-routes.yaml && sudo netplan apply"

# 2080ti-gpu-04
scp /tmp/99-k8s-routes-2080ti.yaml ubuntu@LB_PUBLIC_IP:/tmp/
ssh ubuntu@LB_PUBLIC_IP "sudo cp /tmp/99-k8s-routes-2080ti.yaml /etc/netplan/99-k8s-routes.yaml && sudo netplan apply"
```

**주의:** `Permissions for /etc/netplan/99-k8s-routes.yaml are too open` 경고는 무시 가능 (동작에 영향 없음)

---

### 4.4 라우팅 규칙 검증

**확인 명령어:**

```bash
ssh ubuntu@LB_PUBLIC_IP "ip route show | grep 192.168.0.0/16"
ssh ubuntu@LB_PUBLIC_IP "ip route show | grep 192.168.0.0/16"
ssh ubuntu@LB_PUBLIC_IP "ip route show | grep 192.168.0.0/16"
ssh ubuntu@LB_PUBLIC_IP "ip route show | grep 192.168.0.0/16"
```

**결과:**

```text
v100-gpu-01:    192.168.0.0/16 dev enp2s0f1 proto static scope link metric 50 ✅
2080ti-gpu-02:  192.168.0.0/16 dev eno2 proto static scope link metric 50 ✅
2080ti-gpu-03:  192.168.0.0/16 dev eno2 proto static scope link metric 50 ✅
2080ti-gpu-04:  192.168.0.0/16 dev eno2 proto static scope link metric 50 ✅
```

→ **Calico Pod 간 통신이 모두 10GbE 인터페이스로 라우팅됨**

---

### 4.5 DCGM Exporter 상태 확인

```bash
kubectl get pods -n gpu-operator | grep dcgm
```

**결과:**

```bash
nvidia-dcgm-exporter-8jbsm   1/1   Running   1 (46h ago)      8d
nvidia-dcgm-exporter-ll8g2   1/1   Running   1 (46h ago)      8d
nvidia-dcgm-exporter-ptmbd   1/1   Running   4 (27h ago)      8d
nvidia-dcgm-exporter-rfwj8   1/1   Running   11 (7m47s ago)   8d
```

→ **전체 Running, 재시작 횟수 안정적**

---

## 5. 최종 네트워크 구성

### 5.1 트래픽 경로 분리

| 트래픽 유형              | 인터페이스                 | 대역             | 속도   |
| ------------------------ | -------------------------- | ---------------- | ------ |
| **외부 인터넷 / 학교망** | 1GbE (`enp2s0f0`, `eno1`)  | `0.0.0.0/0`      | 1Gbps  |
| **Calico Pod 간 통신**   | 10GbE (`enp2s0f1`, `eno2`) | `192.168.0.0/16` | 10Gbps |
| **NAS 스토리지 접근**    | 10GbE (`enp2s0f1`, `eno2`) | NFS 마운트       | 10Gbps |

### 5.2 노드별 최종 구성

**v100-gpu-01 (DGX Station)**

- `enp2s0f0` (1GbE): 외부 통신
- `enp2s0f1` (10GbE): Calico Pod 통신, NAS 연결

**2080ti-gpu-02~04 (Supermicro)**

- `eno1` (1GbE): 외부 통신
- `eno2` (10GbE): Calico Pod 통신, NAS 연결

---

## 6. 핵심 인사이트

### 6.1 케이블 카테고리 중요성

- **Cat5**: 최대 100Mbps — 절대 사용 금지
- **Cat5e**: 최대 1Gbps — 1G 연결용
- **Cat6/6a**: 최대 10Gbps — 10G 연결용

→ **인터페이스는 10G 지원해도 케이블이 Cat5면 100M으로 협상됨**

### 6.2 Auto-negotiation 실패 대응

NIC와 스위치 모두 1Gbps 지원하는데 100Mbps로 협상되는 경우:

1. **케이블부터 확인** (Cat5 → Cat5e/6 교체)
2. 케이블 정상인데도 100M이면 스위치 포트 설정 확인
3. 마지막 수단: `ethtool -s` 강제 설정 (권장하지 않음)

### 6.3 Kubernetes 네트워크 최적화

- **CNI(Calico) Pod 통신**은 별도 대역(`192.168.0.0/16`) 사용
- 이 대역을 10GbE 인터페이스로 명시적 라우팅하면:
  - Prometheus 스크래핑 안정성 확보
  - 분산 학습 시 GPU 간 통신 속도 최대화
  - NAS 스토리지 I/O 성능 향상

### 6.4 netplan 라우팅 규칙 설계

```yaml
routes:
  - to: 192.168.0.0/16 # Calico Pod 대역
    via: 0.0.0.0 # 직접 연결 (게이트웨이 없음)
    metric: 50 # 우선순위 (낮을수록 높음, 기본 라우트는 100)
    scope: link # 로컬 링크 범위
```

→ **기본 라우트(`0.0.0.0/0`)는 1GbE 유지, Calico 대역만 10GbE로 분리**

---

## 7. 빠른 진단 명령어 레퍼런스

```bash
# 1. 인터페이스 속도 확인
ethtool <인터페이스명> | grep Speed

# 2. 라우팅 테이블 확인
ip route show

# 3. Calico 대역 라우팅 확인
ip route show | grep 192.168.0.0/16

# 4. DCGM exporter 상태 확인
kubectl get pods -n gpu-operator | grep dcgm

# 5. netplan 설정 적용
sudo netplan apply

# 6. 네트워크 인터페이스 목록
ip link show
```

---

## 8. 관련 문서

- `3_27_Grafana_DCGM_No_data.md` — DCGM 데이터 미표시 원인 분석
- `3_31_네트워크_장애_및_클러스터_설계_개선.md` — MetalLB/Calico 네트워크 장애
- `4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md` — MetalLB IP Pool 충돌 해결

## 9. 향후 작업

- [x] GPU 노드 10GbE 라우팅 설정 완료
- [x] YOLOv8 학습 Job 실행 시 네트워크 대역폭 측정
- [ ] NAS NIC 브라켓 도착 후 물리 고정 작업

---

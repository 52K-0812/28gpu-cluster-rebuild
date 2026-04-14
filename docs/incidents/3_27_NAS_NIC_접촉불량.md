# 🛡️ [트러블슈팅] NAS 10G NIC (BCM57810) 접촉불량 — rev ff 진단 및 복구

> **작업 일자:** 2026-03-27
> **대상:** nas-01 (Supermicro X11SSL-F), Broadcom BCM57810 10G NIC
> **환경:** Ubuntu 22.04 LTS, 1G 관리망(eno1) + 10G 데이터망(BCM57810) 이중화 구성
> **증상:** NAS 10G NIC 완전 먹통 (`lspci` rev ff, 인터페이스 미인식)
> **결과:** PCIe 물리 재삽입 + netplan 인터페이스명 수정으로 10G 복구 (9.41 Gbps 확인)

---

## 1. 개요

| 항목          | 내용                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------ |
| **목적**      | nas-01 10G NIC 펌웨어 크래시로 완전 먹통 상태 진단 및 물리 재삽입으로 복구                 |
| **대상**      | nas-01 (Ubuntu 22.04 LTS), Broadcom BCM57810, Supermicro X11SSL-F                          |
| **구성**      | 1G 관리망: eno1 → LB_PUBLIC_IP / 10G 데이터망: BCM57810                       |
| **핵심 전략** | `rev ff` 진단으로 소프트웨어 복구 불가 확인 → PCIe 물리 재삽입 → netplan 인터페이스명 수정 |
| **발생일**    | 2026-03-27                                                                                 |

---

## 2. 문제 현상

- 10G NIC 포트 링크 LED 미점등, 케이블 연결에도 신호 없음
- `enp2s0f0` 인터페이스 `NO-CARRIER` 상태 — 10G 통신 완전 불가
- GPU 노드(153~156)의 10G → NAS 데이터 전송 불가
- 콘솔에 아래 에러가 초당 수회 반복 출력

```
bnx2x 0000:02:00.0 enp2s0f0: MDC/MDIO access timeout
bnx2x: [bnx2x_state_wait:312(enp2s0f0)]timeout waiting for state
bnx2x: [bnx2x_func_stop:9129(enp2s0f0)]FUNC_STOP ramrod failed
bnx2x: [bnx2x_fw_command:3055(enp2s0f0)]FW failed to respond!
```

---

## 3. 원인 분석

| 원인            | 내용                                                                                                       |
| --------------- | ---------------------------------------------------------------------------------------------------------- |
| **직접 원인**   | PCIe 슬롯에 브라켓 없이 NIC 카드 장착 → 랜선 연결/제거 시 카드 흔들림 → PCIe 접촉 불량 발생                |
| **시스템 반응** | 접촉 불량으로 PCI 설정 공간 접근 불가 → 펌웨어가 `D3cold` 전원 상태에서 `D0`으로 복귀 실패 → `rev ff` 진입 |

**판단 근거 — PCI 상태 비교:**

```bash
# 장애 시
02:00.0 Ethernet controller: BCM57810 (rev ff)   # rev ff = PCI 통신 완전 불가
!!! Unknown header type 7f                        # PCI 헤더 읽기 실패

# 복구 후
01:00.0 Ethernet controller: BCM57810 (rev 10)   # 정상
```

`ipmitool sdr list` 결과 CPU 34°C, 팬 정상 → 하드웨어 고장 아님, 접촉 불량으로 판단.

---

## 4. 해결 과정

### 4-1 소프트웨어 진단

**🛠️ 사용 명령어:**

```bash
ip link show enp2s0f0                  # NO-CARRIER, state DOWN 확인
sudo dmesg | tail -30 | grep bnx2x    # MDC/MDIO access timeout 반복 확인
sudo lspci -vv | grep -A5 "02:00"     # rev ff, Unknown header type 7f 확인
sudo ipmitool sdr list | grep -iE "fan|temp"  # CPU 34°C, FAN 정상 확인
```

---

### 4-2 드라이버 재로드 시도 → 실패

**🛠️ 사용 명령어:**

```bash
sudo ip link set enp2s0f0 down
sudo rmmod bnx2x
sudo modprobe bnx2x
# "Cannot find device enp2s0f0" → 소프트웨어로 복구 불가 확인
```

---

### 4-3 부팅 안전 조치 — bnx2x 블랙리스트 등록

물리 재삽입 작업 전 부팅 시 bnx2x 크래시로 인한 부팅 중단 방지.

**🛠️ 사용 명령어:**

```bash
echo "blacklist bnx2x" | sudo tee /etc/modprobe.d/blacklist-bnx2x.conf
sudo update-initramfs -u
sudo shutdown -h now
```

---

### 4-4 물리 NIC 재삽입 및 슬롯 변경

1. 파워 케이블까지 완전 분리 (잔류 전류 방지) → 30초 대기
2. BCM57810 카드를 슬롯에서 완전히 분리 후 **꽉 눌러 재삽입**
3. RAID 카드(AOC-S3108)와 슬롯 위치 교체

![NIC 위치교체](20260327_171926.jpg)

| 구분    | 슬롯                             |
| ------- | -------------------------------- |
| 변경 전 | BCM57810 → CPU SLOT6 (X8 in X16) |
| 변경 후 | BCM57810 → PCH SLOT4 (X4 in X8)  |

4. 파워 케이블 연결 후 부팅

---

### 4-5 블랙리스트 해제 및 드라이버 로드

**🛠️ 사용 명령어:**

```bash
sudo rm /etc/modprobe.d/blacklist-bnx2x.conf
sudo update-initramfs -u
sudo modprobe bnx2x
sleep 5
ip link show
# enp1s0f0, enp1s0f1 인터페이스 정상 인식 확인
```

---

### 4-6 netplan 인터페이스명 수정

슬롯 변경으로 PCI 주소가 `02:00` → `01:00`으로 바뀌어 인터페이스명 변경됨.

**🛠️ 사용 명령어:**

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
# 수정 전: enp2s0f0 → 수정 후: enp1s0f0
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - LB_PUBLIC_IP/24
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      routes:
        - to: default
          via: LB_PUBLIC_IP
    enp1s0f0: # 변경된 인터페이스명
      addresses:
        - (storage-backend-ip)/24
      mtu: 9000
```

```bash
sudo netplan apply
ip addr show enp1s0f0  # inet (storage-backend-ip)/24 확인
```

---

### 4-7 RAID 및 디스크 마운트 확인

슬롯 위치 변경 후 RAID 설정은 카드 자체 메모리에 저장되므로 별도 조치 불필요.

**🛠️ 사용 명령어:**

```bash
lsblk
df -h
# /dev/sdb1 → /data 28TB 정상 마운트 확인
```

---

## 5. 결과

**🛠️ 10G 속도 검증:**

```bash
iperf3 -s                       # NAS에서 수신기 실행
iperf3 -c (storage-backend-ip)          # v100-gpu-01(153)에서 테스트
# [ 5]  0.00-10.00 sec  11.0 GBytes  9.41 Gbits/sec ✅
```

| 항목     | 결과                                   |
| -------- | -------------------------------------- |
| enp1s0f0 | UP ✅ ((storage-backend-ip), mtu 9000) |
| /data    | 28TB 마운트 정상 ✅                    |
| RAID     | 이상 없음 ✅                           |
| 10G 속도 | **9.41 Gbits/sec** ✅                  |

---

## 6. 핵심 인사이트

- **`rev ff`는 소프트웨어 문제가 아니다** — PCI 장치가 `rev ff`를 반환하면 드라이버 재로드로는 해결 불가. 물리적 접촉 불량을 먼저 의심하고 재삽입으로 접근해야 한다.
- **슬롯 변경 시 인터페이스명이 바뀐다** — PCIe 슬롯 주소 변경 → 인터페이스명 변경 → netplan 반드시 업데이트. `ip link show`로 확인 필수.
- **RAID 슬롯 변경은 무해하다** — RAID 설정은 카드 자체 메모리에 저장되므로 슬롯을 옮겨도 디스크/마운트에 영향 없음.
- **ipmitool로 하드웨어 고장 여부 먼저 확인** — 팬/온도 정상이면 물리 고장보다 접촉 불량 가능성이 높다.

---

## 7. 재발 방지 — BCM57810 브라켓 장착 완료 (2026-04-04)

장애의 근본 원인이었던 브라켓 미장착 문제를 영구 해소했다. AliExpress에 주문한 BCM57810 전용 브라켓이 도착하여 NAS에 장착 완료.

| 항목            | 내용                                                   |
| --------------- | ------------------------------------------------------ |
| **브라켓 도착** | AliExpress (BCM57810 전용 풀하이트 브라켓)             |
| **장착 위치**   | NAS nas-01 PCH SLOT4                                   |
| **효과**        | 랜선 연결/제거 시 카드 고정 → PCIe 접촉 불량 원천 차단 |

### 브라켓 도착 및 장착 과정

![브라켓 도착](20260404_171729.jpg)

> _AliExpress에서 주문한 BCM57810 전용 브라켓 도착_

![브라켓 장착 완료](20260404_172138.jpg)

> _브라켓 장착 후 BCM57810 NIC — 포트 2개 정상 노출_

![NAS 내부 장착](20260404_172417.jpg)

> _NAS nas-01 서버 내부 — PCIe 슬롯에 브라켓과 함께 완전 고정된 상태_

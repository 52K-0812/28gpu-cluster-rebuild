# etcd 정기 백업 및 DR 검증

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-13
> **작업 목적:** etcd 스냅샷 자동 백업을 구성하고, 실제 복원 가능 여부를 검증한다.
> **대상 서버:** master-01, NAS
> **작업 환경:** Kubernetes v1.29, containerd (distroless etcd 컨테이너)
> **최종 결과:** 호스트 crontab 매일 02:00 자동 백업 · NAS 7일 보존 · 74MB 스냅샷 무결성 DR 검증 완료 ✅

---

## 🏗️ 2. 작업 흐름

```
[호스트 crontab — 매일 02:00]
    │
    ▼
[etcd 컨테이너 내부 — etcdctl snapshot save /tmp/]
    │ distroless 이미지라 cp 불가
    │ → /proc/{pid}/root 경로로 호스트에서 직접 복사
    ▼
[NAS /data/backups/etcd/etcd-snapshot-{TIMESTAMP}.db]
    │ 7일 이상 경과 시 자동 삭제
    ▼
[무결성 검증 — etcdctl snapshot status]
    └→ HASH / REVISION / TOTAL KEYS 정상 확인
```

---

## 📐 3. 설계 결정

### 3.1 왜 K8s CronJob이 아닌 호스트 crontab인가

| 항목 | K8s CronJob | 호스트 crontab |
|---|---|---|
| K8s 장애 시 동작 | ❌ 클러스터가 죽으면 CronJob도 불가 | ✅ 독립적으로 동작 |
| 설정 복잡도 | 높음 (RBAC, ServiceAccount, Volume) | 낮음 |
| etcd 의존성 | ❌ etcd가 죽으면 스케줄링 불가 | ✅ 없음 |

etcd 백업의 존재 이유는 "클러스터가 죽었을 때 복구"다. K8s에 의존하는 백업 자동화는 그 목적 자체를 달성하지 못한다.

### 3.2 distroless 컨테이너 대응 — `/proc/{pid}/root` 경로 활용

etcd 컨테이너는 distroless 이미지라 `cat`, `cp`, `rm` 등 기본 명령어가 없다. 컨테이너 내부 `/tmp`에 스냅샷을 저장한 뒤, 호스트에서 `/proc/{pid}/root` 경로로 직접 접근해 복사하는 방식으로 우회했다.

---

## 🔧 Step 1 — 사전 환경 확인

```bash
# etcd Pod 상태 확인
kubectl get pods -n kube-system | grep etcd

# 인증서 경로 확인
kubectl describe pod etcd-master-01 -n kube-system \
  | grep -E "listen-client|cert-file|key-file|trusted-ca"

# NAS 마운트 확인
df -h | grep /data
mount | grep nfs

# 백업 디렉토리 생성
sudo mkdir -p /data/backups/etcd
```

---

## 🔧 Step 2 — 수동 스냅샷 테스트

etcdctl이 호스트 PATH에 없어서 etcd 컨테이너 내부에서 직접 실행했다.

```bash
sudo crictl exec -it $(sudo crictl ps | grep etcd | awk '{print $1}') \
  etcdctl snapshot save /tmp/etcd-snapshot-test.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**결과:** 74MB 스냅샷 1초 내 저장 성공

---

## 🔧 Step 3 — 백업 스크립트 작성

```bash
sudo tee /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash

BACKUP_DIR="/data/backups/etcd"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_PATH="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"
RETAIN_DAYS=7

# etcd 컨테이너 ID 및 PID 획득
ETCD_CONTAINER=$(crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd | awk '{print $1}')
CONTAINER_PID=$(crictl --runtime-endpoint unix:///run/containerd/containerd.sock inspect ${ETCD_CONTAINER} \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['info']['pid'])")

# 스냅샷을 컨테이너 내부 /tmp에 저장
crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -i ${ETCD_CONTAINER} \
  etcdctl snapshot save /tmp/etcd-snapshot-${TIMESTAMP}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 호스트에서 /proc/{pid}/root 경로로 직접 복사 (distroless 대응)
sudo cp /proc/${CONTAINER_PID}/root/tmp/etcd-snapshot-${TIMESTAMP}.db ${SNAPSHOT_PATH}

# 7일 이상 된 백업 자동 삭제
find ${BACKUP_DIR} -name "*.db" -mtime +${RETAIN_DAYS} -delete

echo "[$(date)] etcd snapshot saved: ${SNAPSHOT_PATH} ($(du -h ${SNAPSHOT_PATH} | cut -f1))"
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh
```

---

## 🔧 Step 4 — crontab 자동화 등록

```bash
sudo crontab -e
# 추가:
# 0 2 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1

sudo touch /var/log/etcd-backup.log
sudo chmod 644 /var/log/etcd-backup.log
```

---

## 🔧 Step 5 — 스냅샷 무결성 검증 (DR 검증)

실제 restore는 클러스터가 중단되므로, `snapshot status`로 복원 가능 여부를 검증했다.

```bash
SNAPSHOT=$(ls -t /data/backups/etcd/*.db | head -1)
ETCD_CONTAINER=$(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd | awk '{print $1}')
CONTAINER_PID=$(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock inspect ${ETCD_CONTAINER} \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['info']['pid'])")

sudo cp $SNAPSHOT /proc/${CONTAINER_PID}/root/tmp/verify.db

sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -i ${ETCD_CONTAINER} \
  etcdctl snapshot status /tmp/verify.db --write-out=table
```

**검증 결과:**

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| ac37ac53 |  7881825 |       3090 |      74 MB |
+----------+----------+------------+------------+
```

HASH / REVISION / TOTAL KEYS 정상 — 복원 가능한 스냅샷 확인 ✅

---

## 🚨 실제 복원 절차 (장애 시 참고)

> ⚠️ 운영 중 클러스터에서 실행하면 서비스가 중단됩니다. 장애 상황에서만 사용하세요.

```bash
# 1. etcd Static Pod 중지
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. 기존 etcd 데이터 백업
sudo mv /var/lib/etcd /var/lib/etcd.bak

# 3. 스냅샷 복원
sudo etcdutl snapshot restore /data/backups/etcd/etcd-snapshot-{TIMESTAMP}.db \
  --data-dir=/var/lib/etcd \
  --name=master-01 \
  --initial-cluster=master-01=https://MASTER-IP:2380 \
  --initial-advertise-peer-urls=https://MASTER-IP:2380

# 4. etcd 재시작
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 5. 클러스터 상태 확인
kubectl get nodes
```

---

## ✅ 결과 요약

| 항목 | 내용 |
|---|---|
| 백업 위치 | `/data/backups/etcd/` (NAS 10GbE 마운트) |
| 자동 실행 | 매일 02:00 (호스트 crontab) |
| 보존 기간 | 7일 자동 삭제 |
| 스냅샷 크기 | 74MB |
| 무결성 검증 | HASH / REVISION / KEYS 정상 확인 ✅ |
| 로그 | `/var/log/etcd-backup.log` |

> **설계 원칙:** K8s가 죽어도 백업은 살아있어야 한다. etcd 백업은 K8s에 의존하지 않는 호스트 레벨에서 동작해야 한다.

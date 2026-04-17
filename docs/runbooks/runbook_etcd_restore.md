# Runbook — etcd 백업 및 DR 복구

> **목적:** etcd 자동 백업 운영 · 무결성 검증 · 클러스터 복구 절차
> **최초 구축:** 2026-04-13
> **관리 노드:** master-01
> **담당:** 관리자 (master-01)

---

## 1. 📋 현재 배포 대상

| 항목 | 내용 |
|---|---|
| etcd 버전 | v3.5.16 (kubeadm v1.29 기본 설치) |
| 백업 방식 | 호스트 crontab (K8s 장애 시에도 독립 동작) |
| 스케줄 | 매일 02:00 (master-01 로컬 시간) |
| 저장 위치 | NAS `/data/backups/etcd/` (10GbE 마운트) |
| 보존 기간 | 7일 자동 삭제 |
| 스냅샷 크기 | ~74MB |
| 로그 | `/var/log/etcd-backup.log` |

> **설계 원칙:** etcd 백업은 K8s에 의존해서는 안 된다. 클러스터가 죽은 상황에서 복구가 필요하기 때문에, K8s CronJob이 아닌 호스트 crontab으로 자동화한다.

---

## 2. ✅ 사전 점검

```bash
# 1. etcd Pod 상태 확인
kubectl get pods -n kube-system | grep etcd
# 기대값: etcd-master-01   1/1   Running

# 2. NAS 마운트 확인
df -h | grep /data
mount | grep nfs
# 기대값: /data 경로에 NAS가 마운트된 상태

# 3. 백업 디렉토리 확인
ls -lh /data/backups/etcd/
# 기대값: 최신 스냅샷 파일 존재

# 4. crontab 등록 확인
sudo crontab -l | grep etcd
# 기대값: 0 2 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1

# 5. 백업 스크립트 존재 확인
ls -lh /usr/local/bin/etcd-backup.sh
```

---

## 3. 🛠️ 백업 스크립트 배포 절차

> 최초 설치 또는 스크립트 재배포 시에만 실행한다.

### Step 1 — 백업 디렉토리 생성

```bash
sudo mkdir -p /data/backups/etcd
```

### Step 2 — 백업 스크립트 작성

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

# 호스트에서 /proc/{pid}/root 경로로 직접 복사 (distroless 컨테이너 대응)
sudo cp /proc/${CONTAINER_PID}/root/tmp/etcd-snapshot-${TIMESTAMP}.db ${SNAPSHOT_PATH}

# 7일 이상 된 백업 자동 삭제
find ${BACKUP_DIR} -name "*.db" -mtime +${RETAIN_DAYS} -delete

echo "[$(date)] etcd snapshot saved: ${SNAPSHOT_PATH} ($(du -h ${SNAPSHOT_PATH} | cut -f1))"
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh
```

> **참고 — distroless 컨테이너 대응:** etcd 컨테이너는 distroless 이미지라 `cp` 등 기본 명령어가 없다. 컨테이너 내부 `/tmp`에 스냅샷을 저장한 뒤 호스트에서 `/proc/{pid}/root` 경로로 직접 복사한다.

### Step 3 — crontab 자동화 등록

```bash
sudo crontab -e
# 아래 줄 추가:
# 0 2 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1

sudo touch /var/log/etcd-backup.log
sudo chmod 644 /var/log/etcd-backup.log
```

---

## 4. 🔍 검증 절차

### 수동 스냅샷 즉시 실행

```bash
sudo /usr/local/bin/etcd-backup.sh
# 기대값: etcd snapshot saved: /data/backups/etcd/etcd-snapshot-YYYYMMDD_HHMMSS.db (74M)
```

### 스냅샷 무결성 검증 (DR 검증)

```bash
# 최신 스냅샷 선택
SNAPSHOT=$(ls -t /data/backups/etcd/*.db | head -1)
ETCD_CONTAINER=$(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd | awk '{print $1}')
CONTAINER_PID=$(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock inspect ${ETCD_CONTAINER} \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['info']['pid'])")

sudo cp $SNAPSHOT /proc/${CONTAINER_PID}/root/tmp/verify.db

sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -i ${ETCD_CONTAINER} \
  etcdctl snapshot status /tmp/verify.db --write-out=table
```

**기대값:**
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| ac37ac53 |  7881825 |       3090 |      74 MB |
+----------+----------+------------+------------+
```

HASH / REVISION / TOTAL KEYS 세 값이 모두 정상이면 복원 가능한 스냅샷이다.

### 백업 로그 확인

```bash
tail -20 /var/log/etcd-backup.log
# 기대값: [날짜시간] etcd snapshot saved: ... (74M)
```

---

## 5. 🔴 DR 복구 절차

> ⚠️ **운영 중 클러스터에서 실행하면 서비스가 중단된다. 클러스터 완전 장애 시에만 수행한다.**

```bash
# 1. 복구할 스냅샷 확인
ls -lh /data/backups/etcd/

# 2. etcd Static Pod 중지
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 3. 기존 etcd 데이터 백업
sudo mv /var/lib/etcd /var/lib/etcd.bak

# 4. 스냅샷 복원
sudo etcdutl snapshot restore /data/backups/etcd/etcd-snapshot-{TIMESTAMP}.db \
  --data-dir=/var/lib/etcd \
  --name=master-01 \
  --initial-cluster=master-01=https://MASTER-IP:2380 \
  --initial-advertise-peer-urls=https://MASTER-IP:2380

# 5. etcd 재시작
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 6. 클러스터 복구 확인
kubectl get nodes
kubectl get pods -n kube-system
```

---

## 6. 🚨 장애 대응

### 백업이 실행되지 않는 경우

```bash
# crontab 등록 확인
sudo crontab -l | grep etcd

# 스크립트 수동 실행으로 오류 확인
sudo /usr/local/bin/etcd-backup.sh
```

| 증상 | 원인 | 조치 |
|---|---|---|
| 스냅샷 파일 없음 | crontab 미등록 | `sudo crontab -e`로 재등록 |
| `crictl: command not found` | containerd 소켓 경로 불일치 | `sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps` 로 직접 확인 |
| `etcdctl: snapshot save failed` | 인증서 경로 오류 | `kubectl describe pod etcd-master-01 -n kube-system`에서 cert 경로 재확인 |
| NAS 마운트 없음 | NFS 연결 끊김 | `sudo mount -a` 후 재시도 |

### 스냅샷 무결성 검증 실패

```
Error: snapshot file doesn't exist
```

```bash
# 스냅샷 파일 직접 확인
ls -lh /data/backups/etcd/

# NAS 마운트 재확인
mount | grep nfs
df -h | grep /data
```

# 🗂️ Filebrowser — NAS 파일 탐색기 구축

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-13
> **작업 목적:** NAS(`/data`) 파일을 웹 GUI로 탐색·업로드·다운로드할 수 있는 파일 탐색기를 구축한다.
> **대상 서버:** master-01 ((control-plane-public-ip)), monitoring 네임스페이스, NAS ((control-plane-public-ip))
> **작업 환경:** Kubernetes v1.29, monitoring 네임스페이스
> **최종 결과:** `http://(control-plane-public-ip):30340` (학교망) / `http://(vpn-endpoint)30340` (Tailscale) 접속 완료

---

## 📐 2. 설계 결정

### 왜 Filebrowser인가

| 도구                 | 장단점                                                                           |
| -------------------- | -------------------------------------------------------------------------------- |
| **Filebrowser** ✅   | 경량 단일 바이너리, NFS 마운트 그대로 웹 UI 제공, 업로드/다운로드/권한 관리 지원 |
| JupyterHub 파일 패널 | 이미 존재하나 업로드/다운로드 불편, 팀원 전용 홈 디렉토리만 접근 가능            |
| Portainer 볼륨 뷰어  | 파일 내용 탐색 불가, 관리 기능 제한적                                            |

### 포트 번호 선택

기존 포트 체계 `303xx` (모니터링/관리자 대역)에 편입 — **30340** 사용.

| 대역      | 역할                   |
| --------- | ---------------------- |
| 30300     | Grafana                |
| 30310     | Prometheus             |
| 30320     | Portainer              |
| 30330     | Alertmanager UI        |
| **30340** | **Filebrowser** ← 신규 |

### 네임스페이스

관리자 전용 도구이므로 `monitoring` 네임스페이스에 배치.

---

## 🏗️ 3. 아키텍처

```
관리자 (Tailscale / 학교망)
        │
        ▼
NodePort :30340
        │
        ▼
filebrowser Pod (monitoring 네임스페이스)
    ├── /srv  ← nfs-pv-filebrowser (NAS (control-plane-public-ip):/data)
    └── /database ← filebrowser-db-pvc (계정/설정 영구 보존)
```

### PV/PVC 구조

| 이름                  | 타입                         | 용도                    |
| --------------------- | ---------------------------- | ----------------------- |
| `nfs-pv-filebrowser`  | Static PV                    | NAS `/data` 직접 마운트 |
| `nfs-pvc-filebrowser` | PVC (monitoring)             | 위 PV 바인딩            |
| `filebrowser-db-pvc`  | Dynamic PVC (nfs-client 1Gi) | 계정·설정 DB 영구 보존  |

> **Static PV를 새로 만든 이유:** `nfs-pv-data`는 이미 `default/nfs-pvc-data`에 1:1 바인딩되어 있어 다른 네임스페이스에서 재사용 불가. 동일한 NAS 경로를 바라보는 별도 PV를 생성했다.

---

## 📦 4. 배포 매니페스트

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-filebrowser
spec:
  capacity:
    storage: 28Ti
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: (control-plane-public-ip)
    path: /data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-filebrowser
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ''
  volumeName: nfs-pv-filebrowser
  resources:
    requests:
      storage: 28Ti
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: filebrowser
  namespace: monitoring
  labels:
    app: filebrowser
spec:
  replicas: 1
  selector:
    matchLabels:
      app: filebrowser
  template:
    metadata:
      labels:
        app: filebrowser
    spec:
      containers:
        - name: filebrowser
          image: filebrowser/filebrowser:latest
          args:
            - --database=/database/filebrowser.db
            - --root=/srv
            - --port=8080
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: nas-data
              mountPath: /srv
            - name: database
              mountPath: /database
          resources:
            requests:
              cpu: '100m'
              memory: '128Mi'
            limits:
              cpu: '500m'
              memory: '512Mi'
      volumes:
        - name: nas-data
          persistentVolumeClaim:
            claimName: nfs-pvc-filebrowser
        - name: database
          persistentVolumeClaim:
            claimName: filebrowser-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: filebrowser
  namespace: monitoring
  labels:
    app: filebrowser
spec:
  type: NodePort
  selector:
    app: filebrowser
  ports:
    - name: http
      port: 80
      targetPort: 8080
      nodePort: 30340
```

---

## 🛠️ 5. 트러블슈팅

### 문제 1: PVC `nfs-pvc-data` not found

**증상:** Pod Pending — `persistentvolumeclaim "nfs-pvc-data" not found`

**원인:** 처음에 `default` 네임스페이스의 `nfs-pvc-data`를 그대로 참조했으나, PVC는 네임스페이스를 넘어 공유되지 않음.

**해결:** `monitoring` 네임스페이스 전용 PV/PVC(`nfs-pv-filebrowser` / `nfs-pvc-filebrowser`) 별도 생성.

---

### 문제 2: `listen tcp :80: bind: permission denied`

**증상:** CrashLoopBackOff — `listen tcp :80: bind: permission denied`

**원인:** filebrowser 컨테이너가 non-root 유저로 실행되어 1024 이하 포트 바인딩 불가.

**해결:** `--port=8080` 인자로 포트 변경, Service의 `targetPort`도 8080으로 수정.

```bash
kubectl patch deployment filebrowser -n monitoring --type=json -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/args",
   "value": ["--database=/database/filebrowser.db", "--root=/srv", "--port=8080"]},
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0/containerPort", "value": 8080}
]'
kubectl patch svc filebrowser -n monitoring --type=json -p='[
  {"op": "replace", "path": "/spec/ports/0/targetPort", "value": 8080}
]'
```

---

### 문제 3: 초기 로그인 실패 (admin/admin 불가)

**증상:** `/api/login: 403` — `admin/admin` 로그인 불가

**원인:** filebrowser가 최초 실행 시 랜덤 비밀번호를 자동 생성함. 초기값이 `admin/admin`이 아님.

**확인 방법:** DB 초기화 후 로그에서 생성된 비밀번호 확인.

```bash
# DB 삭제 후 재시작
kubectl exec -n monitoring deployment/filebrowser -- rm /database/filebrowser.db
kubectl rollout restart deployment/filebrowser -n monitoring

# 로그에서 초기 비밀번호 확인
kubectl logs -n monitoring -l app=filebrowser | grep "randomly generated"
# 출력 예: User 'admin' initialized with randomly generated password: (초기 비밀번호)
```

> **운영 원칙:** 최초 로그인 후 즉시 비밀번호 변경 필수 (Settings → User Management).

---

## ✅ 6. 결과

| 항목                   | 결과                                                                            |
| ---------------------- | ------------------------------------------------------------------------------- |
| **접속**               | `http://(control-plane-public-ip):30340` 정상 접속                              |
| **NAS 탐색**           | `/data/datasets/`, `/data/models/`, `/data/backups/` 등 전체 디렉토리 탐색 가능 |
| **계정 영구 보존**     | `filebrowser-db-pvc` PVC로 Pod 재시작 후에도 계정 유지                          |
| **레거시 데이터 정리** | `cheetah-volume-*`, `lost+found`, 테스트 PVC 폴더 삭제 완료                     |

---

## 💡 7. 핵심 인사이트

- **PV는 네임스페이스 리소스가 아니지만 1:1 바인딩이다.** 이미 바인딩된 PV를 다른 네임스페이스 PVC에서 재사용할 수 없다. 동일 NAS 경로를 여러 네임스페이스에서 써야 한다면 Static PV를 네임스페이스별로 별도 생성해야 한다.
- **컨테이너 포트 80은 non-root 환경에서 바인딩 불가다.** 8080 이상의 포트를 사용하고 Service의 `targetPort`만 맞춰주면 NodePort 번호에는 영향 없다.
- **filebrowser 초기 비밀번호는 랜덤 생성이다.** `admin/admin`이 아니며, DB 초기화 후 로그에서 확인해야 한다.

---

## 📋 8. 운영 참고

### 상태 확인

```bash
kubectl get pod -n monitoring -l app=filebrowser
kubectl get pvc -n monitoring | grep filebrowser
kubectl logs -n monitoring -l app=filebrowser --tail=20
```

### 재시작

```bash
kubectl rollout restart deployment/filebrowser -n monitoring
```

### 계정 초기화가 필요한 경우

```bash
kubectl exec -n monitoring deployment/filebrowser -- rm /database/filebrowser.db
kubectl rollout restart deployment/filebrowser -n monitoring
kubectl logs -n monitoring -l app=filebrowser | grep "randomly generated"
```

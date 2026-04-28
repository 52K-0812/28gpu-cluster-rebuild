# 문서 정합성 점검 결과

> **점검 일시:** 2026-04-28
> **점검자:** Claude Code
> **기준 문서:** `docs/overview/current-architecture.md`
> **kubectl 데이터 출처:** master-01 직접 실행

---

## 요약

| 그룹 | 총 항목 | 정상 | 주의 | 불일치 |
|---|---|---|---|---|
| A. 실제 상태 대조 | 10 | 5 | 3 | 2 |
| B. 문서 간 교차 검증 | 4 | 0 | 1 | 3 |
| C. 미해결 과제 점검 | 6 | 5 | 0 | 1 |
| D. 문서 품질 점검 | 4 | 1 | 2 | 1 |
| **합계** | **24** | **11** | **6** | **7** |

---

## 불일치 / 주의 항목 (즉시 조치 필요)

### [불일치] A-3 — yolov8-serving PriorityClass 이미 serving-critical 적용됨

- **발견 위치:** `docs/overview/current-architecture.md` §12, `docs/journal/5.서비스_노출_및_운영_안정화/4_28_PriorityClass_ResourceQuota_Phase_A_B_B1.md` §7
- **현재 문서 내용:** `serving-critical (1000000) — yolov8-serving (Phase C 적용 예정)`
- **실제 상태:**
  ```
  yolov8-serving-7dcd96c8bf-5jfb4   serving-critical
  ```
- **수정 제안:**
  - `current-architecture.md` §12 PriorityClass 표에서 yolov8-serving 항목을 "Phase C 적용 예정" → "적용 완료"로 변경
  - §14 미해결 과제에서 `PriorityClass 도입` 항목 비고를 "Phase A/B/B-1 완료. Phase C~E 진행 예정" → "Phase A/B/B-1/C(serving) 완료. training-normal, LimitRange, ResourceQuota 미적용"으로 갱신
  - 4_28 journal §7 현황 요약도 동일하게 갱신 필요

---

### [불일치] A-9 — MLflow 서비스 타입 및 MetalLB IP 배정표 미기재

- **발견 위치:** `docs/overview/current-architecture.md` §4, §2 MetalLB 배정표
- **현재 문서 내용:** MLflow 접속 방식 — `NodePort <MASTER-IP>:30010`
- **실제 상태:**
  ```
  mlflow-server   LoadBalancer   10.96.118.226   112.76.56.161   5000:30010/TCP
  ```
  서비스 타입이 **LoadBalancer**이며 EXTERNAL-IP `112.76.56.161` 할당. NodePort 30010도 병존.
- **수정 제안:**
  - §2 MetalLB IP 배정표에 MLflow IP(`112.76.56.161`) 항목 추가
  - §4 관리자 서비스 표에서 MLflow 접속 방식을 `LoadBalancer <LB-MLFLOW-IP>:5000 (NodePort 30010 병존)`으로 갱신
  - MetalLB Pool 전체 IP 배정 정리:
    - `112.76.56.158` — JupyterHub proxy-public (Ingress 전환 후 직접 노출 안 함)
    - `112.76.56.160` — Argo Workflows UI
    - `112.76.56.161` — MLflow (미기재)
    - `112.76.56.162` — NGINX Ingress Controller

---

### [불일치] B-3 / D-3 — runbook_model_serving.md 서빙 접속 주소 미갱신

- **발견 위치:** `docs/runbooks/runbook_model_serving.md`
- **현재 문서 내용:** `http://TAILSCALE-IP:30600` (NodePort 방식)
- **실제 상태:** Ingress 전환 완료 (`https://serving.112-76-56-162.nip.io`), NodePort 30600은 병존 중
- **수정 제안:**
  - runbook 접속 주소를 Ingress URL 기준으로 갱신하고, NodePort는 fallback 용도로 併記:
    ```
    기본 접속: https://serving.<LB-INGRESS-IP>.nip.io (Ingress + TLS)
    fallback:  http://<MASTER-IP>:30600 (NodePort, Ingress 장애 시)
    ```
  - 변경 이력에 "2026-04-27 Ingress 전환 완료" 항목 추가

---

### [불일치] B-4 — timeline.md 2026-04-17 이후 작업 내역 누락

- **발견 위치:** `docs/overview/timeline.md`
- **현재 문서 내용:** 2026-04-17까지 기록 (DockerHub 등록)
- **실제 상태 (누락 항목):**
  - 2026-04-27: NGINX Ingress + cert-manager 도입, JupyterHub GitHub OAuth 전환
  - 2026-04-28: PriorityClass 4계층 도입 (Phase A/B/B-1/C-serving 완료), INC-2026-04-28 발생 및 복구
- **수정 제안:** timeline.md에 아래 항목 추가
  ```
  2026-04-27  Ingress + TLS 도입 (NGINX Ingress, cert-manager, cluster-ca, MetalLB 162)
  2026-04-27  JupyterHub GitHub OAuth 전환 (DummyAuthenticator 제거)
  2026-04-28  PriorityClass 4계층 도입 (Phase A/B/B-1 완료, serving-critical 적용 완료)
  2026-04-28  INC-2026-04-28: monitoring chart 의도치 않은 업그레이드 → 복구
  ```

---

### [불일치] B-2 — current-architecture.md §12와 실제 Phase C 완료 상태 불일치

(A-3 항목과 연계. current-architecture.md §12 표, §14 미해결 과제, Phase C 설명 모두 갱신 필요.)

---

### ~~[불일치] C-1 — JupyterHub backup 파일 권한 664 (보안 위험)~~ ✅ 해소 (2026-04-28)

- **발견 위치:** `/home/ubuntu/backup/jupyterhub/*.yaml`
- **조치 전 상태:** 권한 `664` — 같은 그룹 사용자가 읽기 가능. `proxy.secretToken` 평문 포함.
- **조치 내용:** `chmod 600 ~/backup/jupyterhub/*.yaml` 실행 완료
- **조치 후 상태:**
  ```
  -rw------- ubuntu ubuntu  values-before-add-members-20260427-0924.yaml
  -rw------- ubuntu ubuntu  values-before-ingress-host-tls.yaml
  -rw------- ubuntu ubuntu  values-before-oauth-20260427-0807.yaml
  ```
- **잔여 과제:** 신규 백업 생성 시에도 `600` 유지되도록 백업 스크립트에 `chmod 600` 추가 권고.

---

### [불일치] D-3 — runbook_letsencrypt_dns01.md 현재 상태 미명시

- **발견 위치:** `docs/runbooks/runbook_letsencrypt_dns01.md`
- **현재 문서 내용:** HTTP-01 실패 원인 분석 포함, DNS-01 Phase B 계획 문서
- **문제:** 이 runbook이 아직 실행되지 않은 **계획 문서**임을 명시하지 않음. 처음 읽는 운영자가 현재 운영 중인 절차로 오해할 수 있음.
- **수정 제안:** 문서 최상단에 상태 표시 추가:
  ```
  > ⚠️ 상태: 계획 단계 (미완료). 현재 클러스터는 cluster-ca self-signed 인증서로 운영 중.
  > 공인 인증서 전환은 본인 도메인 확보 후 이 runbook을 참고하여 진행.
  ```

---

## 주의 항목

### [주의] A-1 — GPU label 검증 불완전

kubectl describe 출력이 grep에 의해 잘려 `gpu-type=v100`, `gpu-type=2080ti` 레이블 존재 여부를 확인하지 못했다. 현재 yolov8-serving이 `gpu-type=2080ti` nodeSelector로 스케줄되어 정상 동작 중이므로 레이블은 존재하는 것으로 추정되나, 별도로 확인을 권고한다.

```bash
kubectl get nodes --show-labels | grep -E "gpu-type|hostname"
```

---

### [주의] A-5 — MetalLB Pool 최초 설정 annotation과 현재 spec 불일치

annotation의 `last-applied-configuration`에는 `112.76.56.151/32`만 있으나 현재 spec에는 4개 IP가 등록되어 있다. `kubectl apply` 없이 `kubectl patch`로만 변경했기 때문에 발생한 drift이다. 기능 이상은 없으나 `kubectl apply`로 관리 시 annotation 기준으로 덮어쓸 위험이 있다.

**권고:** MetalLB IPAddressPool을 현재 spec 기준으로 YAML 파일을 재작성하고 `kubectl apply`로 동기화.

---

### [주의] A-7 — 문서에 미기재된 PVC

아래 PVC가 실제 존재하지만 current-architecture.md에 명시되지 않았다.

| 네임스페이스 | PVC | 용도 추정 |
|---|---|---|
| mlflow | `mlflow-postgres-pvc` | PostgreSQL 데이터 (10Gi) |
| monitoring | `filebrowser-db-pvc` | Filebrowser DB (1Gi) |
| monitoring | `nfs-pvc-filebrowser` | NAS 전체 마운트 (28Ti) |

current-architecture.md 해당 섹션에 추가를 권고한다.

---

### ~~[주의] A-8 — etcd crontab 등록 계정 불명확~~ ✅ 확인 완료 (2026-04-28)

`sudo crontab -l` 실행 결과 root 계정 crontab에 등록 확인:

```
0 2 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1
```

매일 02:00 root 권한으로 실행 중. ubuntu 계정 crontab이 비어있었던 것은 root로 등록했기 때문. 정상 운영 중.

**문서 반영 필요:** `current-architecture.md §13`, `runbook_etcd_restore.md`의 "호스트 crontab" 표현을 "root crontab (`sudo crontab -e`)"으로 명시 권고.

---

### [주의] B-1 — cluster-diagram.md 기준일 미갱신

- 파일 기준일: 2026-04-27
- 2026-04-28 변경 사항 미반영: PriorityClass 4계층 구조, monitoring chart 버전 고정 운영 결정
- current-architecture.md §12(PriorityClass)와 동기화 필요

---

### [주의] D-1 — 플레이스홀더 마스킹 정책 미명시

current-architecture.md 등에 `<MASTER-IP>`, `<LB-INGRESS-IP>` 등 플레이스홀더가 의도적으로 사용되어 있으나, 어디에도 "보안 목적 IP 마스킹" 정책이 명시되지 않았다. 처음 보는 기여자가 미완성 문서로 오해할 수 있다.

**실제 IP 매핑 (이번 점검 결과 확인):**

| 플레이스홀더 | 실제 값 | 용도 |
|---|---|---|
| `<MASTER-IP>` | 112.76.56.151 | master-01 관리망 |
| `<WORKER-IP-02>` | 112.76.56.152 | master-02 |
| `<WORKER-IP-03>` | 112.76.56.153 | v100-gpu-01 |
| `<WORKER-IP-04>` | 112.76.56.154 | 2080ti-gpu-02 |
| `<WORKER-IP-05>` | 112.76.56.155 | 2080ti-gpu-03 |
| `<WORKER-IP-06>` | 112.76.56.156 | 2080ti-gpu-04 |
| `<LB-INGRESS-IP>` | 112.76.56.162 | NGINX Ingress |
| `<LB-ARGO-IP>` | 112.76.56.160 | Argo UI |
| `<LB-JUPYTERHUB-IP>` | 112.76.56.158 | JupyterHub proxy (직접 노출 없음) |
| (미기재) | 112.76.56.161 | MLflow LoadBalancer |

**권고:** README 또는 각 문서 상단에 마스킹 정책 1줄 추가.
```
> IP 주소는 보안 목적으로 플레이스홀더로 표기함. 실제 값은 master-01 ~/.kube/config 및 클러스터 설정 참조.
```

---

## 정상 확인 항목

| 항목 | 결과 |
|---|---|
| A-1 노드 구성 (hostname, taint, 수량) | ✅ 6노드 일치, master-01 NoSchedule taint 확인 |
| A-2 PriorityClass 등록 | ✅ system-cluster-critical / serving-critical / training-normal 모두 존재 |
| A-3 cert-manager PriorityClass | ✅ 3개 Pod 전원 system-cluster-critical |
| A-3 argo PriorityClass | ✅ 2개 Pod 전원 system-cluster-critical |
| A-3 monitoring PriorityClass | ✅ 5개 핵심 Pod system-cluster-critical, node-exporter/filebrowser 보류 문서 일치 |
| A-3 ingress-nginx PriorityClass | ✅ system-cluster-critical |
| A-3 mlflow PriorityClass | ✅ postgres/server 모두 system-cluster-critical |
| A-3 JupyterHub hub/proxy/user-scheduler | ✅ system-cluster-critical |
| A-3 singleuser (jupyter-x-52k-0812) | ✅ priority 없음 (default=0, 문서 일치) |
| A-4 Ingress 라우팅 | ✅ hub/serving 모두 112.76.56.162으로 정상 라우팅 |
| A-6 JupyterHub 인증 | ✅ GitHubOAuthenticator, 허용 사용자 3명, DummyAuth 완전 제거 |
| A-7 nfs-client StorageClass default | ✅ |
| A-7 ai-team PVC 전체 Bound | ✅ (claim-da-woon-j 미존재 = 예정 항목 일치) |
| A-9 MLflow Running 상태 | ✅ postgres/server 1/1 Running |
| A-10 GitHub Actions Runner | ✅ systemd 서비스로 등록, 정상 동작 |
| A-10 buildkitd 상태 | ✅ nohup 백그라운드 실행 중, 문서(미해결 과제) 일치 |
| A-8 etcd 백업 파일 | ✅ 2026-04-21~28 8개 파일, 7일 보존 정책 일치, 02:00 실행 |
| C-2 ResourceQuota 미적용 | ✅ 없음, 문서 일치 |
| C-3 NodePort 서비스 외부 노출 | ✅ 문서 일치 (인증 없음 — 미해결 과제로 기재됨) |
| C-4 YOLOv8 Serving Basic Auth 미적용 | ✅ 문서 일치 (예정 항목) |
| C-5 K8s 버전 v1.29.15 | ✅ |
| C-6 Da-Woon-J PVC 미존재 | ✅ 예정 항목 일치 |
| D-4 INC-2026-04-28 장애 기록 | ✅ docs/incidents/4_28_grafana_chart_upgrade.md 존재 |
| Monitoring Helm chart 버전 | ✅ kube-prometheus-stack-82.15.1 고정 확인 (REVISION: 13) |
| Certificate 상태 | ✅ jupyterhub-tls, yolov8-serving-tls 모두 READY=True |

---

## 즉시 조치 권고 (보안 / 운영 위험)

### ~~🔴 보안 위험 — backup 파일 권한 즉시 수정~~ ✅ 완료 (2026-04-28)

`chmod 600 ~/backup/jupyterhub/*.yaml` 실행 완료. 3개 파일 모두 `-rw-------` 확인.

---

### 🟠 운영 위험 — MetalLB IPAddressPool YAML 재동기화

annotation drift로 `kubectl apply` 시 MLflow, Argo IP가 삭제될 위험이 있다. 현재 spec 기준으로 YAML을 재작성 후 `kubectl apply`로 동기화를 권고한다.

```yaml
# ~/quota-priority/metallb-pool.yaml (현재 상태 기준)
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: main-pool
  namespace: metallb-system
spec:
  addresses:
  - 112.76.56.158/32   # JupyterHub proxy-public
  - 112.76.56.160/32   # Argo Workflows UI
  - 112.76.56.161/32   # MLflow
  - 112.76.56.162/32   # NGINX Ingress Controller
```

---

### ~~🟡 운영 위험 — etcd crontab 등록 계정 확인 필요~~ ✅ 확인 완료 (2026-04-28)

root crontab에 `0 2 * * * /usr/local/bin/etcd-backup.sh` 등록 확인. 정상 운영 중.

---

## 문서 수정 우선순위 요약

| 우선순위 | 파일 | 수정 내용 |
|---|---|---|
| 즉시 | 터미널 | `chmod 600 ~/backup/jupyterhub/*.yaml` |
| 높음 | `current-architecture.md` §12, §14 | yolov8-serving serving-critical 적용 완료 반영 |
| 높음 | `current-architecture.md` §2, §4 | MLflow LoadBalancer IP 배정표 추가 |
| 높음 | `timeline.md` | 2026-04-27/28 작업 항목 추가 |
| 중간 | `runbook_model_serving.md` | 서빙 접속 주소 Ingress URL로 갱신 |
| 중간 | `runbook_letsencrypt_dns01.md` | 미완료 계획 문서 상태 명시 |
| 중간 | `cluster-diagram.md` | 기준일 2026-04-28로 갱신, PriorityClass 반영 |
| 낮음 | `current-architecture.md` §7, §8 | mlflow-postgres-pvc 등 미기재 PVC 추가 |
| 낮음 | README 또는 각 문서 | 플레이스홀더 마스킹 정책 1줄 추가 |

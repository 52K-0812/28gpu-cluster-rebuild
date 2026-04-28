# 문서 정합성 점검 결과

> **점검 일시:** 2026-04-28
> **점검자:** Claude Code
> **기준 문서:** `docs/overview/current-architecture.md`
> **kubectl 데이터 출처:** master-01 직접 실행
> **최종 업데이트:** 2026-04-28 (조치 완료 항목 정리)

---

## 요약

| 구분 | 항목 수 |
|---|---|
| 전체 점검 항목 | 24 |
| 조치 완료 | 22 |
| **미완료 (잔여)** | **2** |

---

## 미완료 항목

### 🟠 A-5 — MetalLB IPAddressPool YAML 재동기화 (운영 위험)

annotation drift로 `kubectl apply` 시 MLflow(161), Argo(160) IP가 삭제될 위험. 현재 spec 기준으로 YAML 재작성 후 동기화 필요.

```yaml
# master-01: ~/quota-priority/metallb-pool.yaml
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

```bash
kubectl apply -f ~/quota-priority/metallb-pool.yaml
```

> **타이밍 주의:** 적용 순간 LoadBalancer 서비스 IP 재할당으로 수 초 끊길 수 있음. Phase C 완료 후 한가한 시점에 진행 권고.

---

### ⚪ A-1 — GPU 레이블 검증 (확인 필요)

kubectl describe 출력이 잘려 `gpu-type=v100`, `gpu-type=2080ti` 레이블 존재 여부를 직접 확인하지 못했다. 현재 yolov8-serving이 정상 동작 중이므로 레이블은 존재하는 것으로 추정되나, 아래 명령으로 확인 권고.

```bash
kubectl get nodes --show-labels | grep -E "gpu-type|hostname"
```

---

## 조치 완료 이력

| 항목 | 내용 | 완료일 |
|---|---|---|
| A-3 / B-2 | yolov8-serving serving-critical 반영 (current-architecture.md §12, §14) | 2026-04-28 |
| A-9 | MLflow LoadBalancer IP 배정표 추가 (§2, §4) | 2026-04-28 |
| A-7 | 누락 PVC 3건 추가 (filebrowser-db-pvc, nfs-pvc-filebrowser, mlflow-postgres-pvc) | 2026-04-28 |
| A-8 | etcd crontab 소유자 root 확인 → current-architecture.md, runbook_etcd_restore.md 명시 | 2026-04-28 |
| B-1 | cluster-diagram.md 기준일 2026-04-28 갱신, PriorityClass 섹션 추가 | 2026-04-28 |
| B-3 / D-3 | runbook_model_serving.md 서빙 URL → Ingress URL 갱신, fallback NodePort 병기 | 2026-04-28 |
| B-4 | timeline.md Phase 5 (4/27~4/28) 추가 | 2026-04-28 |
| C-1 | JupyterHub backup 파일 권한 664 → 600 (`chmod 600 ~/backup/jupyterhub/*.yaml`) | 2026-04-28 |
| D-1 | current-architecture.md 상단 IP 마스킹 정책 주석 추가 | 2026-04-28 |
| D-3 | runbook_letsencrypt_dns01.md 미완료 상태 배너 추가 | 2026-04-28 |

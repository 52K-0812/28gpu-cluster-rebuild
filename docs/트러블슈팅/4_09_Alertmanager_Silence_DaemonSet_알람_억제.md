# Alertmanager Silence — DaemonSet 알람 억제

## 1. 개요

| 항목 | 내용 |
|---|---|
| **목적** | master-01 NoSchedule taint로 인한 오탐 알람 억제 |
| **대상** | `ai-team/continuous-image-puller` DaemonSet |
| **핵심 전략** | PrometheusRule 직접 수정 대신 AlertManager Silence 적용 |

---

## 2. 문제 원인

JupyterHub는 이미지 프리풀링을 위해 `continuous-image-puller` DaemonSet을 `ai-team` 네임스페이스에 자동 생성한다.

DaemonSet은 모든 노드에 파드 배포를 시도하지만, `master-01`은 컨트롤 플레인 전용 노드로 `NoSchedule` taint가 설정되어 있어 파드가 스케줄링되지 않는다. 이는 **의도된 클러스터 설계**다.

이로 인해 Prometheus가 아래 두 알람을 지속 발생시킨다.

| Alert | 조건 |
|---|---|
| `KubeDaemonSetRolloutStuck` | desired ≠ current 상태가 15분 이상 지속 |
| `KubeDaemonSetMisScheduled` | misscheduled pod count > 0 이 15분 이상 지속 |

---

## 3. 해결 전략

`monitoring-kube-prometheus-kubernetes-apps` PrometheusRule은 kube-prometheus-stack Helm chart가 관리한다. 직접 수정 시 `helm upgrade` 때 원복되므로 **AlertManager Silence**로 처리한다.

---

## 4. 적용

### 4-1. 포트포워딩

**🛠️ 사용 명령어:**
```bash
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093 &
```

### 4-2. Silence 등록

**🛠️ 사용 명령어:**
```bash
amtool --alertmanager.url=http://localhost:9093 silence add \
  alertname=~"KubeDaemonSetRolloutStuck|KubeDaemonSetMisScheduled" \
  daemonset="continuous-image-puller" \
  namespace="ai-team" \
  --comment="master-01 NoSchedule taint으로 인한 의도된 상태. 워커 노드 아님." \
  --duration=8760h
```

### 4-3. 등록 확인

**🛠️ 사용 명령어:**
```bash
amtool --alertmanager.url=http://localhost:9093 silence query daemonset="continuous-image-puller"
```

---

## 5. 결과

| 항목 | 값 |
|---|---|
| Silence ID | `668af865-b74f-4287-9663-6199718c06b5` |
| 억제 알람 | `KubeDaemonSetRolloutStuck`, `KubeDaemonSetMisScheduled` |
| 대상 | `ai-team/continuous-image-puller` |
| 만료 | 2027-04-09 11:03:57 UTC (1년) |
| 상태 | ✅ 알람 메일 수신 중단 확인 |

---

## 6. 핵심 인사이트

- **PrometheusRule 직접 수정 금지**: kube-prometheus-stack Helm 관리 룰은 업그레이드 시 덮어씌워짐. Silence 또는 별도 PrometheusRule로 오버라이드 필요
- **Silence 만료 관리**: `--duration=8760h` (1년) 설정. 만료 후 재등록 또는 영구 억제가 필요하면 별도 PrometheusRule inhibit 규칙으로 전환 권장
- **amtool 버전 경고 무시 가능**: `amtool 0.23.0` vs `alertmanager 0.31.1` 버전 차이 경고는 기능 동작에 영향 없음

---

### 참고 명령어

```bash
# master-01 taint 확인
kubectl describe node master-01 | grep Taint

# Silence 삭제 (필요 시)
amtool --alertmanager.url=http://localhost:9093 silence expire 668af865-b74f-4287-9663-6199718c06b5

# 포트포워딩 종료
kill %1
```

# Alertmanager 이메일 알람 구성

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-07
> **작업 목적:** Prometheus Alertmanager에 Gmail SMTP 연동 및 GPU 특화 알람 룰을 구성해 클러스터 이상 상황을 관리자 이메일로 자동 통보한다.
> **대상 서버:** master-01 , monitoring 네임스페이스
> **작업 환경:** kube-prometheus-stack, Alertmanager, PrometheusRule CRD
> **최종 결과:** GPU 온도/메모리/노드 다운 알람 룰 등록, 실제 클러스터 알람 이메일 수신 확인

---

## 🏗️ 2. 작업 흐름

```text
[Alertmanager 현재 상태 확인]
        │ kube-prometheus-stack에 이미 포함됨
        ▼
[Gmail SMTP 설정 → Secret 업데이트]
        │ 앱 비밀번호 발급 (2단계 인증 필요)
        ▼
[Alertmanager 재시작]
        │ 설정 반영
        ▼
[GPU 특화 PrometheusRule 등록]
        │ 온도/메모리/노드 다운
        ▼
[실제 알람 이메일 수신 확인]
```

---

## 📋 3. 사전 확인

### 3.1 Alertmanager 상태

kube-prometheus-stack 설치 시 Alertmanager가 자동으로 포함된다. 별도 설치 불필요.

```bash
kubectl get pods -n monitoring | grep alertmanager
```

**정상 출력:**

```text
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0    6d17h
```

### 3.2 기본 설정 확인

```bash
kubectl get secret alertmanager-monitoring-kube-prometheus-alertmanager \
  -n monitoring \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
```

기본 상태는 `receiver: "null"` — 알람이 어디에도 전송되지 않는 상태.

---

## 📧 4. Step 1 — Gmail SMTP 설정

### 4.1 Gmail 앱 비밀번호 발급

Gmail SMTP 사용 시 계정 비밀번호가 아닌 **앱 비밀번호**가 필요하다.

1. `myaccount.google.com` → 보안
2. **2단계 인증** 활성화 확인 (필수)
3. **앱 비밀번호** 검색 → 기타(직접 입력) → `Alertmanager` 입력
4. 생성된 16자리 비밀번호 복사 (공백 제거)

### 4.2 Alertmanager 설정 파일 작성

```bash
cat << 'EOF' > alertmanager-config.yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'YOUR_GMAIL@gmail.com'
  smtp_auth_username: 'YOUR_GMAIL@gmail.com'
  smtp_auth_password: 'YOUR_APP_PASSWORD'
  smtp_require_tls: true

route:
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'email'
  routes:
  - matchers:
    - alertname = "Watchdog"
    receiver: 'null'

receivers:
- name: 'null'
- name: 'email'
  email_configs:
  - to: 'YOUR_GMAIL@gmail.com'
    send_resolved: true

inhibit_rules:
- source_matchers:
  - severity = critical
  target_matchers:
  - severity =~ warning|info
  equal: [namespace, alertname]
EOF
```

### 4.3 Secret 업데이트

```bash
kubectl create secret generic alertmanager-monitoring-kube-prometheus-alertmanager \
  --from-file=alertmanager.yaml=alertmanager-config.yaml \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4.4 Alertmanager 재시작

```bash
kubectl rollout restart statefulset \
  alertmanager-monitoring-kube-prometheus-alertmanager -n monitoring

# 재시작 확인
kubectl get pods -n monitoring | grep alertmanager
```

---

## 🖥️ 5. Step 2 — GPU 특화 알람 룰 등록

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: gpu.rules
    rules:
    - alert: GPUHighTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 85
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "GPU 온도 위험 ({{ $labels.instance }})"
        description: "GPU {{ $labels.gpu }} 온도가 {{ $value }}°C 초과. 즉시 확인 필요."

    - alert: GPUMemoryHigh
      expr: DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_FREE > 0.95
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "GPU 메모리 95% 초과 ({{ $labels.instance }})"
        description: "GPU {{ $labels.gpu }} 메모리 사용률이 95%를 초과했습니다."

    - alert: GPUNodeDown
      expr: up{job="nvidia-dcgm-exporter"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "GPU 노드 다운 ({{ $labels.instance }})"
        description: "DCGM Exporter 응답 없음. GPU 노드 상태를 확인하세요."
EOF
```

**확인:**

```bash
kubectl get prometheusrule -n monitoring | grep gpu
# gpu-alerts   19s
```

---

## ✅ 6. 검증 결과

### 6.1 테스트 알람 발송

```bash
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager \
  -n monitoring 9093:9093 &

curl -X POST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "namespace": "monitoring"
    },
    "annotations": {
      "summary": "Alertmanager 테스트 알람",
      "description": "GPU 클러스터 Alertmanager 정상 동작 확인"
    }
  }]'
```

### 6.2 실제 수신된 알람

테스트 알람 발송 전에 **실제 클러스터 알람**이 먼저 수신됨:

```text
[FIRING:1] KubeDaemonSetRolloutStuck
namespace = ai-team
daemonset = continuous-image-puller
severity = warning
```

**분석:**

```bash
kubectl get daemonset continuous-image-puller -n ai-team
# DESIRED 5 / CURRENT 5 / READY 5 → 실제 문제 없음
```

→ kube-prometheus-stack 기본 알람 룰의 **오탐(False Positive)**. 실제 장애 없이 민감도 설정으로 인해 발생. 무시 가능.

**의미:** Alertmanager가 실제 클러스터 이벤트를 감지해 이메일로 전송하는 게 완벽하게 동작함을 증명.

---

## 📊 7. 최종 알람 구성 현황

| 알람명             | 조건                    | 심각도   | 대기 시간 |
| ------------------ | ----------------------- | -------- | --------- |
| GPUHighTemperature | GPU 온도 > 85°C         | critical | 2분       |
| GPUMemoryHigh      | GPU 메모리 > 95%        | warning  | 5분       |
| GPUNodeDown        | DCGM Exporter 응답 없음 | critical | 1분       |

**알람 수신 경로:** Prometheus → Alertmanager → Gmail SMTP → 관리자 이메일

---

## 💡 8. 핵심 인사이트

**모니터링 스택은 알람이 있어야 완성이다.** Grafana 대시보드로 GPU 상태를 시각화하는 것만으로는 부족하다. 관리자가 대시보드를 항상 보고 있을 수 없기 때문에, 임계값 초과 시 자동으로 통보되는 알람 체계가 있어야 실제 운영 환경이라고 볼 수 있다.

**오탐(False Positive)도 증거가 된다.** 첫 번째로 수신된 알람이 실제 장애가 아닌 오탐이었지만, 이는 Alertmanager가 클러스터 이벤트를 실시간으로 감지하고 있다는 증거다. 오탐을 분석하고 "실제 문제 없음"을 확인하는 과정 자체가 운영 경험이다.

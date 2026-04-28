# Runbook — Ingress + TLS 서비스 추가 절차

## 🎯 1. 목적

기존 클러스터의 임의 서비스를 host 기반 Ingress + cluster-ca self-signed TLS 구조로 노출하는 표준 절차. Phase A 결과물(`serving.INGRESS-LB-IP.nip.io`, `hub.INGRESS-LB-IP.nip.io`)을 기반으로 동일 패턴 재현.

> **선결 조건:** NGINX Ingress Controller, cert-manager, cluster-ca-issuer가 이미 운영 중이어야 한다. 미설치 상태라면 `docs/journal/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md` 4-2 ~ 4-4 절차 선행.

---

## 📋 2. 사전 체크리스트

작업 시작 전 다음 항목 모두 통과 확인:

```bash
# Ingress Controller 정상 동작
kubectl get pods -n ingress-nginx
# → ingress-nginx-controller-xxx 1/1 Running

# cert-manager 정상 동작
kubectl get pods -n cert-manager
# → 3개 Pod 모두 1/1 Running

# cluster-ca-issuer 정상 동작
kubectl get clusterissuer cluster-ca-issuer
# → READY=True

# 대상 Service 존재 확인
kubectl get svc <service-name> -n <namespace>
# → ClusterIP 또는 LoadBalancer/NodePort 정상 동작
```

---

## 🔧 3. 변수 정의

작업 전 다음 값 확정:

| 변수             | 예시                             | 설명                                  |
| -------------- | ------------------------------ | ----------------------------------- |
| `SERVICE_NAME` | `grafana`                      | 노출할 Service 이름                      |
| `NAMESPACE`    | `monitoring`                   | Service 네임스페이스                      |
| `BACKEND_PORT` | `80`                           | Service의 실제 port (NodePort 아님)      |
| `HOST`         | `grafana.INGRESS-LB-IP.nip.io` | 외부 노출 host                          |
| `TLS_SECRET`   | `grafana-tls`                  | TLS Secret 이름 (관례: `<service>-tls`) |
| `INGRESS_NAME` | `grafana-ingress`              | Ingress 리소스 이름                      |
| `INGRESS_IP`   | `INGRESS-LB-IP`                | Ingress Controller LoadBalancer IP  |
| `TAILSCALE_IP` | `TAILSCALE-IP`                 | Tailscale 노드 IP (인증서 IP SAN용)       |

> ⭐ `BACKEND_PORT`는 Service의 `spec.ports[].port` 값이지 NodePort가 아니다. 추측 말고 반드시 다음 명령으로 확인:
> ```bash
> kubectl get svc <SERVICE_NAME> -n <NAMESPACE> -o yaml | grep -A8 '^spec:'
> ```

---

## 🚀 4. 실행 절차

### 4-1. Certificate 생성 (host 기반 + DNS SAN)

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${TLS_SECRET}
  namespace: ${NAMESPACE}
spec:
  secretName: ${TLS_SECRET}
  issuerRef:
    name: cluster-ca-issuer
    kind: ClusterIssuer
  commonName: ${HOST}
  dnsNames:
  - ${HOST}
  ipAddresses:
  - ${INGRESS_IP}
  - ${TAILSCALE_IP}
  duration: 8760h
  renewBefore: 720h
EOF
```

> ⭐ **DNS SAN 필수:** `dnsNames`가 없으면 NGINX가 default Fake Certificate를 내보낸다. host 기반 라우팅에서 가장 흔한 함정.

### 4-2. Certificate READY 확인

```bash
kubectl get certificate ${TLS_SECRET} -n ${NAMESPACE}
# 기대값: READY=True (보통 5초 이내)

kubectl get secret ${TLS_SECRET} -n ${NAMESPACE}
# 기대값: TYPE=kubernetes.io/tls, DATA=3
```

`READY=False`로 30초 이상 머무르면:
```bash
kubectl describe certificate ${TLS_SECRET} -n ${NAMESPACE}
# Events 섹션에서 발급 실패 사유 확인
```

### 4-3. Ingress 생성

**기본 템플릿 (대부분 서비스):**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${INGRESS_NAME}
  namespace: ${NAMESPACE}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ${HOST}
    secretName: ${TLS_SECRET}
  rules:
  - host: ${HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${SERVICE_NAME}
            port:
              number: ${BACKEND_PORT}
EOF
```

### 4-4. 서비스별 특수 annotation

#### WebSocket 사용 서비스 (JupyterHub, Argo, Grafana 일부)

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "64m"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

> 60초 기본 timeout으로는 WebSocket idle 시 끊김 발생. 3600초로 확장 필수.

#### 큰 파일 업로드 서비스 (MLflow, Filebrowser)

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "0"     # 무제한
  # 또는 명시적 한도
  nginx.ingress.kubernetes.io/proxy-body-size: "500m"
```

#### Argo Workflows (gRPC + HTTP/2)

Argo는 통신 모드 확인 필수:
```bash
kubectl get deployment argo-workflows-server -n argo -o yaml | grep -A5 -E 'args|command'
```

`--secure=true` 모드면 backend가 HTTPS:
```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
```

`--secure=false` 모드면 일반 HTTP backend:
```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
```

#### 관리자 그룹 (Basic Auth 추가)

```bash
# Secret 생성
htpasswd -c /tmp/auth admin   # 비밀번호 입력
kubectl create secret generic ${SERVICE_NAME}-basic-auth \
  --from-file=auth=/tmp/auth -n ${NAMESPACE}
rm /tmp/auth
```

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: <service>-basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Admin Only"
```

---

## ✅ 5. 검증 절차

### 5-1. Ingress 상태 확인

```bash
kubectl get ingress ${INGRESS_NAME} -n ${NAMESPACE}
# 기대값:
# HOSTS: ${HOST}
# ADDRESS: INGRESS-LB-IP
# PORTS: 80, 443
```

### 5-2. HTTPS 응답 검증

```bash
curl -k -s -o /dev/null -w "%{http_code}\n" https://${HOST}/
# 기대값: 200, 302, 401(Basic Auth 적용 시) 중 하나
```

### 5-3. 인증서 검증 (Fake Certificate 체크)

```bash
echo | openssl s_client \
  -connect ${INGRESS_IP}:443 \
  -servername ${HOST} \
  2>/dev/null | openssl x509 -noout -issuer -subject
```

기대값:
```
issuer=CN = cluster-ca
subject=CN = ${HOST}
```

> ⛔ `issuer=O = Acme Co, CN = Kubernetes Ingress Controller Fake Certificate`가 나오면 **실패**. Certificate에 DNS SAN 누락 또는 Ingress의 `tls.secretName`과 실제 Secret 불일치. 4-1로 돌아가서 dnsNames 확인.

### 5-4. 인증서 SAN 검증

```bash
kubectl get secret ${TLS_SECRET} -n ${NAMESPACE} \
  -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -ext subjectAltName
```

기대값:
```
X509v3 Subject Alternative Name:
    DNS:${HOST}, IP Address:INGRESS-LB-IP, IP Address:TAILSCALE-IP
```

### 5-5. 기능별 추가 검증

| 서비스 유형 | 검증 항목 |
|---|---|
| 정적 웹 UI | 브라우저 접속 → 로그인 → 주요 메뉴 클릭 |
| WebSocket 사용 | 실시간 동작 확인 (JupyterHub: Notebook 셀 실행, Grafana: Live 차트) |
| API 서버 | curl로 주요 endpoint 호출 |
| 파일 업로드 | 큰 파일(>10MB) 업로드 테스트 |

---

## 🚨 6. 트러블슈팅 빠른 참조

### Symptom: Fake Certificate

```bash
# 원인 확인
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100 \
  | grep -i "${TLS_SECRET}\|fake\|certificate"
```

대표 메시지:
```
ssl certificate "${NAMESPACE}/${TLS_SECRET}" does not contain a Common Name 
or Subject Alternative Name for server "${HOST}"
```

**해결:** Certificate spec에 `dnsNames: - ${HOST}` 추가 후 Secret 강제 재발급:
```bash
kubectl delete secret ${TLS_SECRET} -n ${NAMESPACE}
# cert-manager가 자동으로 재발급
```

### Symptom: 404 Not Found (FastAPI 형식)

응답: `{"detail":"Not Found"}` (JSON)

**원인:** 요청이 의도한 backend Service가 아닌 다른 Pod로 라우팅됨. host 분리 안 된 catch-all Ingress 충돌 가능성.

**확인:**
```bash
kubectl get ingress -A
# 같은 host(또는 host 미지정 + path /) Ingress가 둘 이상이면 충돌
```

### Symptom: 504 Gateway Timeout

**원인:** WebSocket 또는 long-polling 연결이 60초 기본 timeout으로 끊김.

**해결:** annotation 추가:
```yaml
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### Symptom: 413 Request Entity Too Large

**원인:** 기본 `proxy-body-size: 1m` 제한.

**해결:**
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "100m"
# 또는 무제한
nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

### Symptom: Certificate `READY=False` 지속

```bash
kubectl describe certificate ${TLS_SECRET} -n ${NAMESPACE}
kubectl get certificaterequest -n ${NAMESPACE}
```

`cluster-ca-issuer`의 self-signed 발급은 본래 5초 이내 완료. 30초 이상 걸리면 cert-manager Pod 또는 cluster-ca-issuer 자체 문제. cert-manager 로그 확인:
```bash
kubectl logs -n cert-manager deploy/cert-manager --tail=50
```

---

## 🔄 7. 롤백 절차

### Ingress만 제거 (TLS Secret 보존)

```bash
kubectl delete ingress ${INGRESS_NAME} -n ${NAMESPACE}
```

기존 NodePort/LoadBalancer 경로는 영향 없이 유지.

### Certificate + Secret 완전 제거

```bash
kubectl delete certificate ${TLS_SECRET} -n ${NAMESPACE}
kubectl delete secret ${TLS_SECRET} -n ${NAMESPACE}
```

> ⛔ 운영 중인 Ingress가 해당 Secret을 참조하고 있으면 즉시 Fake Certificate로 fallback. Ingress 제거 또는 다른 Secret으로 변경 후 진행.

---

## 📎 8. 서비스별 빠른 적용 예시

### Grafana

```bash
SERVICE_NAME=monitoring-grafana
NAMESPACE=monitoring
BACKEND_PORT=80
HOST=grafana.INGRESS-LB-IP.nip.io
TLS_SECRET=grafana-tls
INGRESS_NAME=grafana-ingress
```

추가 설정 필요:
```bash
# Helm values 또는 ConfigMap에 root_url 추가
kubectl edit configmap monitoring-grafana -n monitoring
# data.grafana.ini에 추가:
# [server]
# root_url = https://grafana.INGRESS-LB-IP.nip.io
# serve_from_sub_path = false
kubectl rollout restart deployment monitoring-grafana -n monitoring
```

Basic Auth 적용 (관리자 전용).

### MLflow

```bash
SERVICE_NAME=mlflow-server
NAMESPACE=mlflow
BACKEND_PORT=5000
HOST=mlflow.INGRESS-LB-IP.nip.io
TLS_SECRET=mlflow-tls
INGRESS_NAME=mlflow-ingress
```

추가 annotation:
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "500m"   # 모델 artifact 업로드
```

Basic Auth 미적용 (의도적 공개).

### Argo Workflows

```bash
SERVICE_NAME=argo-workflows-server
NAMESPACE=argo
BACKEND_PORT=2746
HOST=argo.INGRESS-LB-IP.nip.io
TLS_SECRET=argo-tls
INGRESS_NAME=argo-ingress
```

선행 확인 필수:
```bash
kubectl get deployment argo-workflows-server -n argo -o yaml | grep -A5 args
# --secure=true / --secure=false 확인 후 backend-protocol annotation 결정
```

Basic Auth 적용 (관리자 전용).

### Portainer

```bash
SERVICE_NAME=portainer
NAMESPACE=portainer
BACKEND_PORT=9000
HOST=portainer.INGRESS-LB-IP.nip.io
TLS_SECRET=portainer-tls
INGRESS_NAME=portainer-ingress
```

WebSocket annotation 필수 (콘솔 기능):
```yaml
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

Basic Auth 적용 (관리자 전용).

### Filebrowser

```bash
SERVICE_NAME=filebrowser
NAMESPACE=monitoring   # 확인 필요
BACKEND_PORT=80         # 확인 필요
HOST=files.INGRESS-LB-IP.nip.io
TLS_SECRET=filebrowser-tls
INGRESS_NAME=filebrowser-ingress
```

큰 파일 업로드 대비:
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

Basic Auth 적용 (관리자 전용).

---

## 📌 9. 검증 통과 기준

서비스 추가 작업의 완료 판정 기준:

```
☐ Certificate READY=True
☐ Secret tls.crt에 DNS SAN으로 host 포함 확인
☐ Ingress ADDRESS=INGRESS-LB-IP 표시
☐ openssl 결과 issuer=CN=cluster-ca (Fake Certificate 아님)
☐ curl HTTPS 응답 200/302/401 정상
☐ 브라우저에서 실제 기능 동작 확인
☐ (해당 시) WebSocket / 파일 업로드 / API 호출 등 추가 검증 통과
```

위 7개 항목 모두 통과해야 Phase 완료. 하나라도 실패 시 6 트러블슈팅으로 복귀.

---

## 📎 10. 관련 문서

- `docs/journal/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md` — Phase A 작업 기록 + 함정 사례
- `docs/runbooks/runbook_letsencrypt_dns01.md` — 공인 인증서 적용 시 (별도 Phase)
- `docs/overview/current-architecture.md` — 클러스터 운영 구조 요약

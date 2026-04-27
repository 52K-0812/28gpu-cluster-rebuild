# Runbook — Let's Encrypt 공인 인증서 적용 (DNS-01 Challenge)

## 🎯 1. 목적

self-signed 인증서를 Let's Encrypt 공인 인증서로 전환하기 위한 절차서. Phase A에서 HTTP-01 challenge가 실패한 원인을 회피하기 위해 DNS-01 challenge 사용.

> **선결 조건:**
> - 본인 소유 도메인 1개 이상 (예: `mldemo.dev`, `28gpu-cluster.com`)
> - DNS API 자격증명 (Cloudflare 권장)
> - Phase A 완료 (NGINX Ingress + cert-manager + cluster-ca-issuer 운영 중)

---

## 🤔 2. 왜 DNS-01인가 (HTTP-01 거부 사유)

| 항목 | HTTP-01 | DNS-01 |
|---|---|---|
| 외부 도달성 요구 | `:80` 포트 외부 공개 필수 | 불필요 |
| 캠퍼스 NAT/방화벽 영향 | 큼 | 없음 |
| wildcard 인증서 발급 | 불가능 | 가능 |
| nip.io 호환성 | 가능 | **불가능** (DNS 권한 없음) |
| 본인 도메인 필요 | 불필요 (nip.io 가능) | **필수** |
| 캠퍼스 환경 적합성 | 낮음 (Phase A에서 실패) | 높음 |

⭐ **결정 사유:** Phase A에서 HTTP-01 challenge가 다음 두 가지 원인으로 실패:
1. ACME challenge 요청이 cert-manager solver Pod로 라우팅되지 않음 (`{"detail":"Not Found"}` FastAPI 응답)
2. 외부 ACME 서버에서 `error: connection`(validation data fetch 실패) 발생 — 캠퍼스 외부→내부 응답 패스 비대칭 가능성

DNS-01은 외부 도달성 무관하므로 캠퍼스 환경에서도 안정적으로 동작.

---

## 📋 3. 사전 준비

### 3-1. 도메인 확보

선택지:
- 신규 구매: Cloudflare Registrar (도메인 + DNS API 통합), 연 $9~10
- 기존 도메인 활용: DNS NS를 Cloudflare로 이전

> Cloudflare를 권장하는 이유: 무료 DNS API, cert-manager 공식 지원, Web UI에서 즉시 확인 가능.

### 3-2. Cloudflare API Token 생성

Cloudflare Dashboard → My Profile → API Tokens → Create Token

권한 설정:
```
Zone - DNS - Edit
Zone Resources: Include - Specific zone - <your-domain>
```

> ⛔ Global API Key 사용 금지. Zone DNS Edit 전용 토큰만 생성. 노출 시 피해 범위 최소화.

토큰 값 안전하게 보관 (한 번만 표시됨).

### 3-3. DNS Zone 확인

작업 전 도메인이 Cloudflare에 정상 등록되어 있는지 확인:

```bash
dig NS <your-domain>
# 기대값: NS 레코드에 *.cloudflare.com 표시
```

---

## 🔧 4. 변수 정의

| 변수 | 예시 | 설명 |
|---|---|---|
| `DOMAIN` | `mldemo.dev` | 본인 소유 도메인 |
| `EMAIL` | `your@email.com` | Let's Encrypt 등록 이메일 |
| `CF_API_TOKEN` | `xxx...` | Cloudflare API Token (Zone DNS Edit 권한) |
| `INGRESS_IP` | `INGRESS-LB-IP` | Ingress Controller LoadBalancer IP |

이번 runbook에서 발급할 인증서 대상:
- `serving.${DOMAIN}` → YOLOv8 서빙
- `hub.${DOMAIN}` → JupyterHub
- (Phase B 추가 시) `grafana.${DOMAIN}`, `mlflow.${DOMAIN}` 등

---

## 🚀 5. 실행 절차

### 5-1. Cloudflare API Token Secret 생성

```bash
kubectl create secret generic <dns-provider-api-token-secret> \
  -n cert-manager \
  --from-literal=api-token=${CF_API_TOKEN}
```

확인:
```bash
kubectl get secret <dns-provider-api-token-secret> -n cert-manager
# DATA=1
```

### 5-2. DNS A 레코드 생성

Cloudflare Dashboard → DNS → Records → Add Record:

```
Type: A
Name: serving
IPv4: INGRESS-LB-IP
Proxy: DNS only (회색 구름)   # ⛔ Proxied(주황 구름) 사용 금지
TTL: Auto
```

```
Type: A
Name: hub
IPv4: INGRESS-LB-IP
Proxy: DNS only
TTL: Auto
```

> ⛔ **Proxy는 반드시 DNS only.** Proxied 모드는 Cloudflare가 SSL 종단을 가져가서 cert-manager가 발급한 인증서가 무용지물이 된다.

검증:
```bash
dig serving.${DOMAIN} +short
# 기대값: INGRESS-LB-IP
dig hub.${DOMAIN} +short
# 기대값: INGRESS-LB-IP
```

> DNS 전파에 최대 5분 소요. 즉시 안 보이면 잠시 대기.

### 5-3. ClusterIssuer 생성 (DNS-01 + Cloudflare)

먼저 staging issuer로 검증:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-staging
spec:
  acme:
    email: ${EMAIL}
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-dns01-staging-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: <dns-provider-api-token-secret>
            key: api-token
EOF
```

검증:
```bash
kubectl get clusterissuer letsencrypt-dns01-staging
# READY=True
```

### 5-4. Staging 인증서 발급 테스트

> ⭐ production으로 바로 가지 말고 staging으로 먼저 검증. Let's Encrypt는 등록 도메인 기준 주당 50개 인증서 제한이 있다.

테스트용 Certificate:
```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: serving-tls-dns01-test
  namespace: ai-team
spec:
  secretName: serving-tls-dns01-test
  issuerRef:
    name: letsencrypt-dns01-staging
    kind: ClusterIssuer
  commonName: serving.${DOMAIN}
  dnsNames:
  - serving.${DOMAIN}
EOF
```

발급 진행 상황 모니터링:
```bash
kubectl get certificate serving-tls-dns01-test -n ai-team -w
# READY=True까지 보통 1~3분 소요 (DNS 전파 대기 포함)

kubectl get challenge -n ai-team
# State=valid 확인

kubectl describe certificate serving-tls-dns01-test -n ai-team
# Events에 "Issued" 메시지
```

> DNS-01은 cert-manager가 Cloudflare API로 `_acme-challenge.serving.${DOMAIN}` TXT 레코드를 자동 생성/삭제한다. Cloudflare Dashboard에서 일시적으로 TXT 레코드가 보였다 사라지는 것이 정상.

### 5-5. Staging 인증서 검증

```bash
# Secret 내용 확인
kubectl get secret serving-tls-dns01-test -n ai-team -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -issuer -subject -dates
# 기대값: issuer에 "(STAGING) Let's Encrypt" 포함
```

⛔ `issuer`에 `(STAGING)`이 **없으면** staging issuer가 아닌 다른 issuer가 발급한 것 → 잘못된 상태. 정리 후 재시도.

### 5-6. Staging 정리 및 Production 전환

staging 정리:
```bash
kubectl delete certificate serving-tls-dns01-test -n ai-team
kubectl delete secret serving-tls-dns01-test -n ai-team
```

Production ClusterIssuer 생성:
```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-prod
spec:
  acme:
    email: ${EMAIL}
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-dns01-prod-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: <dns-provider-api-token-secret>
            key: api-token
EOF
```

검증:
```bash
kubectl get clusterissuer letsencrypt-dns01-prod
# READY=True
```

### 5-7. 운영 서비스 인증서를 production으로 발급

기존 self-signed `yolov8-serving-tls`를 LE production으로 교체:

```bash
# 기존 self-signed Certificate 백업 (참고용)
kubectl get certificate yolov8-serving-tls -n ai-team -o yaml \
  > ~/backup/yolov8-serving-tls-selfsigned.yaml

# Production 발급 Certificate로 교체
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: yolov8-serving-tls
  namespace: ai-team
spec:
  secretName: yolov8-serving-tls
  issuerRef:
    name: letsencrypt-dns01-prod
    kind: ClusterIssuer
  commonName: serving.${DOMAIN}
  dnsNames:
  - serving.${DOMAIN}
  duration: 2160h         # 90일 (LE 표준)
  renewBefore: 360h       # 만료 15일 전 자동 갱신
EOF

# 강제 재발급 트리거
kubectl delete secret yolov8-serving-tls -n ai-team
```

> ⚠️ `commonName`과 `dnsNames`를 새 도메인으로 맞춰야 한다. 기존 `serving.INGRESS-LB-IP.nip.io`가 아닌 `serving.${DOMAIN}`이다.

JupyterHub TLS도 동일 방식으로 `hub.${DOMAIN}` 발급.

### 5-8. Ingress host 변경

Certificate가 새 도메인 기반이므로 Ingress의 `tls.hosts`와 `rules.host`도 변경 필요:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yolov8-serving-ingress
  namespace: ai-team
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - serving.${DOMAIN}
    secretName: yolov8-serving-tls
  rules:
  - host: serving.${DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yolov8-serving
            port:
              number: 8080
EOF
```

### 5-9. 최종 검증

```bash
# 인증서 issuer 확인 (production 표기 확인)
echo | openssl s_client -connect ${INGRESS_IP}:443 \
  -servername serving.${DOMAIN} 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

기대값:
```
issuer=C = US, O = Let's Encrypt, CN = R3
subject=CN = serving.<your-domain>
notBefore=...
notAfter=... (90일 이내)
```

> ⭐ `(STAGING)` 표기가 **없어야** 한다. 있으면 production이 아닌 staging 인증서가 나간 것.

브라우저 검증:
- `https://serving.${DOMAIN}` 접속 → 자물쇠 정상 (경고 없음)
- 인증서 상세 → "Let's Encrypt" 발급자 확인

---

## 🔄 6. 자동 갱신 동작 확인

cert-manager는 Certificate spec의 `renewBefore` 기준으로 자동 갱신:

```bash
# 갱신 예정 시점 확인
kubectl get certificate yolov8-serving-tls -n ai-team -o jsonpath='{.status.renewalTime}'
```

Let's Encrypt 인증서는 90일 만료, 보통 만료 30~60일 전 자동 갱신 권장. 위 spec의 `renewBefore: 360h`(15일)은 다소 늦으니 운영 안정성 확보 후 `renewBefore: 720h`(30일)로 조정 검토.

### 갱신 실패 알림 설정

```bash
# Alertmanager에 cert-manager 갱신 실패 룰 추가 (별도 작업)
# Prometheus 메트릭: certmanager_certificate_expiration_timestamp_seconds
```

---

## 🚨 7. 트러블슈팅

### Symptom: Challenge `pending` 장기 지속

```bash
kubectl describe challenge -n ai-team
```

대표 원인:
1. **Cloudflare API Token 권한 부족**
   - 해결: Token에 Zone DNS Edit 권한 + 해당 zone 명시 확인
2. **DNS 전파 지연**
   - 보통 1~5분, 최대 10분. cert-manager는 자동 재시도
3. **Cloudflare에 zone 등록 안 됨**
   - 해결: NS 레코드가 Cloudflare로 정상 위임됐는지 확인

```bash
# DNS 직접 확인
dig _acme-challenge.serving.${DOMAIN} TXT +short
# challenge 진행 중이면 임시 TXT 값 표시
```

### Symptom: Rate Limit 도달

```
acme: error: 429 :: POST :: https://acme-v02.api.letsencrypt.org/...
too many certificates already issued
```

**해결:**
- 동일 도메인 주당 50개 한도. 잠시 대기(7일).
- staging으로 먼저 검증하지 않고 production을 반복 시도하면 발생.
- 향후 절차: staging 검증 → production은 한 번에.

### Symptom: ARI 또는 OCSP 관련 경고

cert-manager v1.20+ 일부 메시지는 알림용. Certificate `READY=True`면 무시 가능.

### Symptom: 자동 갱신 실패

```bash
# 수동 갱신 트리거
kubectl annotate certificate yolov8-serving-tls -n ai-team \
  cert-manager.io/issue-temporary-certificate="true" --overwrite
kubectl delete secret yolov8-serving-tls -n ai-team
```

cert-manager가 새 ACME Order를 시작.

---

## 🔙 8. 롤백 (self-signed 복귀)

공인 인증서 적용 후 문제 발생 시 즉시 self-signed로 복귀:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: yolov8-serving-tls
  namespace: ai-team
spec:
  secretName: yolov8-serving-tls
  issuerRef:
    name: cluster-ca-issuer
    kind: ClusterIssuer
  commonName: serving.${DOMAIN}
  dnsNames:
  - serving.${DOMAIN}
  - serving.INGRESS-LB-IP.nip.io     # 기존 host 호환성 유지
  ipAddresses:
  - ${INGRESS_IP}
  duration: 8760h
  renewBefore: 720h
EOF

kubectl delete secret yolov8-serving-tls -n ai-team
```

Ingress의 host를 nip.io 기반으로 되돌리거나, 새 도메인과 nip.io 둘 다 추가하는 dual-host 구성 가능:

```yaml
spec:
  tls:
  - hosts:
    - serving.${DOMAIN}
    - serving.INGRESS-LB-IP.nip.io
    secretName: yolov8-serving-tls
  rules:
  - host: serving.${DOMAIN}
    http: { ... }
  - host: serving.INGRESS-LB-IP.nip.io
    http: { ... }
```

---

## 📌 9. Phase 진입 기준

이 runbook을 실행할 시점 판단:

```
☐ 본인 소유 도메인 확보됨
☐ Cloudflare(또는 동등 DNS API 제공자) 등록 완료
☐ DNS API Token 생성 완료
☐ Phase A self-signed 운영이 안정화됨 (최소 1주 무장애)
☐ 공인 인증서가 실질적으로 필요한 사유 발생 (예: 외부 데모, OAuth 적용 등)
```

⛔ 위 항목 충족 전에는 self-signed 유지가 정답이다. Let's Encrypt는 운영 부담을 동반하는 선택이지 default가 아니다.

---

## 📎 10. 관련 문서

- `docs/journal/4_27_Ingress_TLS_도입_및_host_기반_라우팅.md` — Phase A 작업 + HTTP-01 실패 사유
- `docs/runbooks/runbook_ingress_tls.md` — Ingress 추가 표준 절차
- cert-manager 공식 문서: https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/

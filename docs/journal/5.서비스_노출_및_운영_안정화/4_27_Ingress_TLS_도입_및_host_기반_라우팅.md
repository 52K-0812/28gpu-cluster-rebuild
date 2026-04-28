# Ingress + TLS 도입 및 host 기반 라우팅 전환

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-27
> **작업 목적:** NGINX Ingress Controller + cert-manager 도입으로 단일 IP(`INGRESS-LB-IP`)에서 host 기반 라우팅 + TLS 종단 처리 구조 확립. JupyterHub WebSocket 안정화 및 향후 OAuth callback URL 사용을 위한 HTTPS 선결 조건 확보.
> **대상 서버:** master-01 (작업 노드), master-02 (Ingress Controller / cert-manager 배치 노드)
> **작업 환경:** Kubernetes v1.29.15, MetalLB v0.13+, ingress-nginx Helm chart, cert-manager v1.20.2
> **선행 작업:** 4_17 YOLOv8 서빙 이미지 DockerHub 등록 완료, MetalLB Pool에 기존 서비스용 LoadBalancer IP들이 운영 중인 상태에서 신규 Ingress IP를 추가
> **최종 결과:** `serving.INGRESS-LB-IP.nip.io`, `hub.INGRESS-LB-IP.nip.io` 두 host 기반 Ingress + cluster-ca self-signed TLS 정상 동작. JupyterHub Notebook 셀 실행까지 검증 통과.

---

## 🏗️ 2. 작업 흐름

```
MetalLB Pool에 신규 Ingress IP 추가 → Ingress Controller 설치(master-02 고정)
    ↓
cert-manager 설치(master-02 고정) → cluster-ca 기반 self-signed Issuer 구성
    ↓
yolov8-serving / jupyterhub Certificate 생성 (IP SAN)
    ↓
yolov8-serving Ingress 생성 (host 미지정, catch-all)
    ↓
[설계 충돌 발견] JupyterHub Ingress 추가 시 라우팅 충돌 예상
    ↓
host 기반 분리 결정 → nip.io wildcard DNS 채택
    ↓
[Let's Encrypt staging 시도] HTTP-01 검증 실패 → 중단 결정
    ↓
self-signed 유지 + DNS SAN 포함 재발급
    ↓
JupyterHub Ingress 생성 (WebSocket annotation 포함)
    ↓
브라우저 로그인 + Notebook 셀 실행 검증 → Phase A 완료
```

---

## 📐 3. 아키텍처 설계 결정

### 3-1. Ingress Controller 노드 배치 — master-02 고정

3_31 네트워크 장애 이후 SoC(Separation of Concerns) 원칙에 따라 Control Plane(master-01)과 시스템 워크로드(master-02)를 분리 운영 중. Ingress Controller는 시스템 워크로드 범주이므로 master-02에 nodeSelector 고정.

```yaml
controller:
  nodeSelector:
    kubernetes.io/hostname: master-02
```

### 3-2. MetalLB IP 할당 방식 — annotation 단독 사용

```yaml
service:
  type: LoadBalancer
  annotations:
    metallb.universe.tf/loadBalancerIPs: "INGRESS-LB-IP"
```

`spec.loadBalancerIP` 필드는 K8s API 레벨에서 deprecated 예정. MetalLB v0.13+에서는 annotation으로만 제어하는 것이 표준이며, 두 방식 동시 사용 시 충돌 로그 또는 무시 동작이 발생할 수 있음. annotation 단독 사용으로 일원화.

### 3-3. defaultBackend nodeSelector 미적용

defaultBackend는 404 응답용 경량 컨테이너로, master-02 강제 배치 시 스케줄링 유연성만 감소. 기본 스케줄 정책 유지.

### 3-4. host 기반 분리 결정 — path 기반 거부 사유

| 방식 | 장점 | 단점 | 결정 |
|---|---|---|---|
| host 기반 | TLS 인증서 host별 분리 가능, JupyterHub `base_url` 변경 불필요 | 도메인 체계 필요 | ⭐ 채택 |
| path 기반 | 단일 host로 운영 가능 | JupyterHub `c.JupyterHub.base_url` 재설정 필요, WebSocket 라우팅 이슈 위험 | 거부 |

JupyterHub는 path prefix 변경 시 Hub 재구성이 필요하고 WebSocket 경로 매칭이 까다롭다. `docs/incidents/4_01` 장애가 WebSocket 충돌이었던 점을 고려하면 path 기반은 운영 리스크가 너무 크다.

### 3-5. wildcard DNS 선택 — sslip.io vs nip.io

| 항목 | sslip.io | nip.io |
|---|---|---|
| 운영 안정성 | 단일 운영자, 다운타임 이력 있음 | 더 오래 운영, 다운타임 사례 적음 |
| Let's Encrypt 호환성 | 동일 | 동일 (커뮤니티 표준) |
| 캠퍼스 방화벽 차단 사례 | 일부 보고됨 | 거의 없음 |

⭐ **결정:** `nip.io` 채택. `serving.INGRESS-LB-IP.nip.io`, `hub.INGRESS-LB-IP.nip.io` 형태로 wildcard DNS가 자동으로 해당 IP로 resolve.

### 3-6. TLS 인증서 정책 — cluster-ca self-signed 유지

Let's Encrypt staging HTTP-01 발급 시도 후 실패 → self-signed 유지 결정. 상세는 트러블슈팅 참고.

---

## 🔧 4. 실행 절차

### 4-1. MetalLB Pool에 162 추가

```bash
kubectl patch ipaddresspool main-pool -n metallb-system --type='json' -p='[
  {"op":"add","path":"/spec/addresses/-","value":"INGRESS-LB-IP/32"}
]'
```

검증 (실제 spec.addresses):
```
- JUPYTERHUB-LB-IP/32   # JupyterHub proxy-public (기존)
- ARGO-LB-IP/32   # Argo Workflows (기존)
- MLFLOW-LB-IP/32   # MLflow (기존)
- INGRESS-LB-IP/32   # Ingress Controller (신규)
```

> ⭐ **검증 함정:** 테스트 LoadBalancer Service에 ping 응답 확인은 부적절. selector가 가리키는 healthy endpoint가 0개면 MetalLB는 ARP 광고를 하지 않거나 응답하지 않는다. 검증은 EXTERNAL-IP 할당 여부 + 기존 IP 유지 여부로만 판정.

### 4-2. NGINX Ingress Controller 설치

```yaml
# ~/ingress-nginx-values.yaml
controller:
  nodeSelector:
    kubernetes.io/hostname: master-02
  service:
    type: LoadBalancer
    annotations:
      metallb.universe.tf/loadBalancerIPs: "INGRESS-LB-IP"
  replicaCount: 1
  admissionWebhooks:
    enabled: true
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  -f ~/ingress-nginx-values.yaml --timeout 5m
```

검증:
```
ingress-nginx-controller   1/1 Running   master-02
EXTERNAL-IP: INGRESS-LB-IP
PORT(S): 80, 443
```

### 4-3. cert-manager 설치

```yaml
# ~/cert-manager-values.yaml
installCRDs: true
nodeSelector:
  kubernetes.io/hostname: master-02
webhook:
  nodeSelector:
    kubernetes.io/hostname: master-02
cainjector:
  nodeSelector:
    kubernetes.io/hostname: master-02
```

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  -f ~/cert-manager-values.yaml --timeout 5m
```

검증: cert-manager / cert-manager-webhook / cert-manager-cainjector 3개 Pod 모두 master-02에서 Running.

### 4-4. cluster-ca 기반 self-signed Issuer 구성

```bash
# 1단계: bootstrap self-signed ClusterIssuer
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
EOF

# 2단계: cluster CA Certificate (자체 서명 → 다른 인증서를 서명할 CA 역할)
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: cluster-ca
  secretName: <cluster-ca-secret>
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
EOF

# 3단계: cluster-ca-issuer (실제 서비스 인증서 발급용)
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-ca-issuer
spec:
  ca:
    secretName: <cluster-ca-secret>
EOF
```

검증:
```
selfsigned-bootstrap    READY=True
selfsigned-ca           READY=True
cluster-ca-issuer       READY=True
```

### 4-5. host 기반 Certificate 재발급

> ⭐ **핵심 함정:** 초기 Certificate는 `ipAddresses`만 있고 `dnsNames`가 없어서 host 기반 라우팅에서 NGINX가 default Fake Certificate를 내보내는 문제 발생. DNS SAN 포함 재발급 필수.

```bash
# yolov8-serving-tls 재발급
kubectl apply -f - <<'EOF'
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
  commonName: serving.INGRESS-LB-IP.nip.io
  dnsNames:
  - serving.INGRESS-LB-IP.nip.io
  ipAddresses:
  - INGRESS-LB-IP
  - TAILSCALE-IP
  duration: 8760h
  renewBefore: 720h
EOF

# 강제 재발급
kubectl delete secret yolov8-serving-tls -n ai-team
```

JupyterHub TLS도 동일 방식으로 `hub.INGRESS-LB-IP.nip.io` DNS SAN 포함 재발급.

### 4-6. yolov8-serving Ingress

```yaml
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
    - serving.INGRESS-LB-IP.nip.io
    secretName: yolov8-serving-tls
  rules:
  - host: serving.INGRESS-LB-IP.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yolov8-serving
            port:
              number: 8080
```

### 4-7. JupyterHub Ingress (WebSocket annotation 포함)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jupyterhub-ingress
  namespace: ai-team
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "64m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - hub.INGRESS-LB-IP.nip.io
    secretName: jupyterhub-tls
  rules:
  - host: hub.INGRESS-LB-IP.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: proxy-public
            port:
              number: 80
```

> ⭐ **WebSocket annotation 처음부터 포함 결정:** `/hub/login`은 WebSocket 없이 200/302 응답이 와서 검증 단계에서 통과로 보이지만, 실제 사용자가 노트북 셀 실행 시 `/user/.../api/kernels/.../channels` WebSocket이 60초 기본 timeout으로 끊긴다. 후순위로 미루면 4_01 incident 패턴이 재현되므로 처음부터 `proxy-read-timeout: 3600` 적용.

---

## 🔥 5. 트러블슈팅

### 문제 1 — yolov8-serving Ingress catch-all 충돌

**증상:** 첫 yolov8-serving Ingress가 `host: 미지정 + path: /`로 생성됨. JupyterHub Ingress를 같은 패턴으로 추가 시 ingress-nginx가 어느 backend로 라우팅할지 결정 불가능한 상태.

**원인:** Ingress 라우팅 우선순위는 `creationTimestamp` 기준 + host/path 매칭 specificity. catch-all 두 개는 충돌 또는 임의 선택을 유발.

**해결:** host 기반 분리. yolov8-serving Ingress부터 `host: serving.INGRESS-LB-IP.nip.io` 명시로 재적용.

### 문제 2 — Let's Encrypt staging HTTP-01 발급 실패

**증상:**
```
Challenge State: invalid
Reason: Error getting validation data
Presented: false
```

ACME challenge token URL을 직접 curl 시 응답:
```
HTTP/1.1 404 Not Found
{"detail":"Not Found"}
```

**원인 분석:**
- `{"detail":"Not Found"}`는 cert-manager solver 응답이 아닌 FastAPI 기본 404 응답 형식
- ACME challenge 요청이 cert-manager solver Pod로 가지 않고 yolov8-serving FastAPI로 라우팅됨
- `acme.cert-manager.io/http01-edit-in-place: "true"` annotation을 적용해도 Ingress rules에 challenge path가 추가되지 않음
- `Presented: false` 상태가 핵심: cert-manager가 solver Ingress를 만들었지만 ingress-nginx가 라우팅 반영 실패
- 추가로 외부 ACME 서버 입장에서 `error: connection`이 발생 → 캠퍼스 외부 → 내부 응답 패스에 비대칭 라우팅 가능성

**시도한 방법:**
1. `cert-manager.io/cluster-issuer: letsencrypt-staging` annotation 적용 → 실패
2. `acme.cert-manager.io/http01-edit-in-place: "true"` 추가 → 실패
3. `nginx.ingress.kubernetes.io/ssl-redirect: "false"` 추가 → 실패

**결정:** Let's Encrypt HTTP-01 중단. self-signed 유지. 공인 인증서는 본인 도메인 확보 후 DNS-01 challenge로 별도 Phase에서 재시도.

> ⭐ **교훈:** `nip.io`는 DNS-01 challenge가 불가능(외부 DNS 권한 없음)하고, HTTP-01은 캠퍼스 네트워크 환경상 불안정. 정식 인증서가 필수가 아닌 한 self-signed가 가장 실용적이다.

### 문제 3 — Fake Certificate 응답

**증상:** Ingress, Certificate, Secret 모두 정상 상태인데 openssl s_client 결과:
```
issuer=O = Acme Co, CN = Kubernetes Ingress Controller Fake Certificate
```

**원인:** `yolov8-serving-tls` 인증서가 IP SAN만 포함하고 DNS SAN(`serving.INGRESS-LB-IP.nip.io`)이 없음. NGINX Ingress Controller는 host 매칭 인증서를 못 찾으면 자체 default Fake Certificate를 내보냄.

ingress-nginx 로그:
```
ssl certificate "ai-team/yolov8-serving-tls" does not contain a Common Name 
or Subject Alternative Name for server "serving.INGRESS-LB-IP.nip.io"
```

**해결:** Certificate spec에 `dnsNames` 추가 후 Secret 강제 재발급.
```bash
kubectl delete secret yolov8-serving-tls -n ai-team   # 강제 재발급 트리거
```

### 문제 4 — kubectl annotate 명령 줄바꿈 깨짐

**증상:**
```
error: at least one annotation update is required
-bash: acme.cert-manager.io/http01-edit-in-place-: No such file or directory
```

**원인:** 백슬래시(`\`) 뒤 공백/빈 줄이 들어가면 Bash가 line continuation을 인식하지 못함.

**해결:** annotation 제거 명령은 한 줄로 작성.
```bash
kubectl annotate ingress yolov8-serving-ingress -n ai-team acme.cert-manager.io/http01-edit-in-place- cert-manager.io/cluster-issuer- --overwrite
```

### 문제 5 — Certificate 자동 재생성

**증상:** staging Certificate를 삭제했는데 즉시 재생성됨.

**원인:** Ingress에 `cert-manager.io/cluster-issuer: letsencrypt-staging` annotation이 남아있어서 ingress-shim이 자동으로 Certificate를 재생성.

**해결:** Certificate 삭제 전 Ingress의 cert-manager 관련 annotation부터 모두 제거.

---

## ✅ 6. 최종 검증 결과

### 6-1. yolov8-serving

```bash
$ curl -k https://serving.INGRESS-LB-IP.nip.io/health
{"status":"ok","demo_model":"yolov8n-coco (80 classes)","champion_ready":true,"champion_version":"7"}

$ openssl s_client -connect INGRESS-LB-IP:443 \
    -servername serving.INGRESS-LB-IP.nip.io 2>/dev/null \
    | openssl x509 -noout -issuer -subject
issuer=CN = cluster-ca
subject=CN = serving.INGRESS-LB-IP.nip.io
```

인증서 SAN:
```
DNS:serving.INGRESS-LB-IP.nip.io
IP Address:INGRESS-LB-IP
IP Address:TAILSCALE-IP
```

### 6-2. JupyterHub

```bash
$ curl -k -s -o /dev/null -w "%{http_code}\n" https://hub.INGRESS-LB-IP.nip.io/hub/login
200

$ openssl s_client -connect INGRESS-LB-IP:443 \
    -servername hub.INGRESS-LB-IP.nip.io 2>/dev/null \
    | openssl x509 -noout -issuer -subject
issuer=CN = cluster-ca
subject=CN = hub.INGRESS-LB-IP.nip.io
```

### 6-3. WebSocket / 커널 검증 (핵심)

브라우저 → JupyterLab → Notebook 셀 실행:

```python
import sys
sys.version
```

결과:
```
'3.11.7 | packaged by conda-forge | (main, Dec 23 2023, 14:43:09) [GCC 12.3.0]'
```

> ⭐ 셀 실행 성공 = `/user/.../api/kernels/.../channels` WebSocket 연결 정상. 4_01 incident 재현 없음.

---

## 💡 7. 핵심 인사이트

### 인사이트 1: Ingress catch-all은 단일 서비스에서만 안전

`host: 미지정 + path: /` 패턴은 Ingress가 하나뿐일 때만 안전하다. 둘 이상 추가될 가능성이 있으면 처음부터 host 명시로 가야 한다. 첫 Ingress부터 host 기반으로 설계하는 것이 미래의 라우팅 충돌을 예방한다.

### 인사이트 2: Certificate SAN은 host와 일치해야 한다

cert-manager로 인증서를 생성해도 SAN이 실제 요청 host를 포함하지 않으면 NGINX는 인증서 자체를 거부하고 default Fake Certificate로 fallback한다. Certificate 생성 시 `dnsNames`와 Ingress의 `tls.hosts`가 정확히 일치해야 한다.

### 인사이트 3: Let's Encrypt HTTP-01은 nip.io + 캠퍼스 환경에서 불안정

`nip.io`는 외부 DNS라 DNS-01 challenge 불가능, HTTP-01만 옵션이지만 ACME 서버의 외부→내부 응답 패스가 캠퍼스 네트워크에서 안정적이지 않다. self-signed로도 host 기반 라우팅 + TLS 암호화 목적은 충분히 달성된다. 공인 인증서가 필요하면 본인 소유 도메인 + DNS-01이 정답이다.

### 인사이트 4: JupyterHub WebSocket은 처음부터 검증해야 한다

`/hub/login` HTTP 200 응답만으로는 검증 불충분. Notebook 셀 실행까지 통과해야 진짜 완료다. 후순위로 미뤄두면 4_01 incident가 재현된다. WebSocket annotation(`proxy-read-timeout: 3600`)은 처음부터 포함하는 것이 표준.

### 인사이트 5: Ingress + cert-manager 디버깅은 응답 내용으로 backend 추적

ACME challenge URL 응답이 `{"detail":"Not Found"}` JSON이면 FastAPI로 라우팅된 것. 텍스트 토큰 문자열이면 cert-manager solver로 라우팅된 것. 응답 형식만 봐도 backend가 어디인지 즉시 식별 가능하다.

---

## 📁 8. Decision Log

| 날짜 | 결정 | 사유 |
|---|---|---|
| 2026-04-27 | Ingress Controller master-02 고정 | 3_31 SoC 원칙 적용, 시스템 워크로드 분리 |
| 2026-04-27 | host 기반 라우팅 (path 거부) | JupyterHub `base_url` 변경 회피, WebSocket 안정화 |
| 2026-04-27 | nip.io wildcard DNS 채택 (sslip.io 거부) | 운영 안정성 + LE 호환성 + 방화벽 차단 사례 |
| 2026-04-27 | Let's Encrypt HTTP-01 중단, self-signed 유지 | Challenge `Presented: false` + connection error 반복, 캠퍼스 환경 한계 |
| 2026-04-27 | 공인 인증서는 본인 도메인 + DNS-01로 별도 Phase | nip.io는 DNS API 권한 없어 DNS-01 불가 |
| 2026-04-27 | NodePort/LoadBalancer 기존 경로 유지 | Ingress 장애 시 fallback, Phase C까지 유지 |
| 2026-04-27 | JupyterHub WebSocket annotation 처음부터 포함 | 4_01 incident 재현 방지 |
| 2026-04-27 | Certificate IP SAN 유지 | 백워드 호환성, 향후 정리 항목으로 인지 |

---

## 🔮 9. 다음 작업

### Phase B (예정)
- 관리자 그룹 Ingress 추가: Grafana, Argo Workflows, Portainer, Filebrowser
- 공개 그룹 Ingress 추가: MLflow
- 관리자 그룹에 ingress-nginx Basic Auth 적용

### Phase C (예정)
- NodePort 정리
- Prometheus 직접 노출 폐쇄 (Grafana 데이터소스로만 접근)

### 별도 Phase (장기)
- 본인 소유 도메인 확보 → DNS-01 challenge로 Let's Encrypt 공인 인증서 적용
- JupyterHub GitHub OAuth 교체 (HTTPS 선결 조건 충족 완료)

---

## 📎 10. 관련 문서

- `docs/runbooks/runbook_ingress_tls.md` — Ingress 추가 표준 절차서
- `docs/runbooks/runbook_letsencrypt_dns01.md` — 미래 공인 인증서 재시도 절차서
- `docs/incidents/4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md` — WebSocket 충돌 장애 기록
- `docs/incidents/3_31_네트워크_장애_및_클러스터_설계_개선.md` — SoC 원칙 도입 사유

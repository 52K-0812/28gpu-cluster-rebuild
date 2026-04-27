# JupyterHub GitHub OAuth 전환

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-27
> **작업 목적:** JupyterHub 인증 방식을 DummyAuthenticator에서 GitHubOAuthenticator로 전환. 허용 계정 기반 접근 제어 적용.
> **대상 서버:** master-01
> **작업 환경:** Kubernetes v1.29, ai-team 네임스페이스, JupyterHub Helm chart 4.3.3, oauthenticator 17.4.0
> **선행 작업:** Ingress + TLS 적용 완료 (`https://hub.112-76-56-162.nip.io` 운영 중)
> **최종 결과:** GitHubOAuthenticator 전환 완료. 허용 계정 3명 로그인 검증. 허용 외 계정 403 차단 확인. DummyAuth PVC 및 Hub DB 레코드 정리 완료.

---

## 🏗️ 2. 작업 흐름

```
oauthenticator 내장 확인 (Hub Pod exec)
    ↓
GitHub OAuth App 생성 (callback URL 설정)
    ↓
K8s Secret 등록 (jupyterhub-github-oauth)
    ↓
기존 values 백업
    ↓
full values 파일 작성 (DummyAuth 완전 제거)
    ↓
dry-run 검증 (DummyAuthenticator 미출현 확인)
    ↓
helm upgrade 실제 적용
    ↓
Hub 로그 검증 → 브라우저 로그인 검증 → 차단 검증
    ↓
팀원 계정 추가 (Yeeeho, Da-Woon-J)
    ↓
DummyAuth PVC / Hub DB 레코드 정리
```

---

## 📐 3. 아키텍처 설계 결정

### 3-1. full values 파일 방식 채택 (--reuse-values 미사용)

| 항목 | --reuse-values 방식 | full values 방식 |
|---|---|---|
| DummyAuthenticator 제거 | ❌ 기존 설정과 병합되어 잔존 | ✅ 완전 대체 |
| 관리 복잡도 | 낮음 (변경분만 작성) | 높음 (전체 값 관리) |
| 안전성 | 의도치 않은 값 잔존 위험 | 명시적 전체 제어 |

> ⭐ **설계 결정:** `--reuse-values`는 현재 release의 모든 커스텀 값을 병합하므로 제거 대상 설정(`DummyAuthenticator`)도 함께 살아남는다. DummyAuth를 완전히 제거하려면 full values 파일로 전체를 덮어써야 한다. dry-run에서 이를 확인 후 전환했다.

### 3-2. Client ID/Secret을 K8s Secret → 환경변수 주입

| 항목 | values 파일 평문 기재 | K8s Secret 주입 |
|---|---|---|
| helm get values 노출 | ❌ 평문 노출 | ✅ 비노출 |
| 백업 파일 유출 위험 | ❌ 있음 | ✅ 없음 |
| 운영 복잡도 | 낮음 | 다소 높음 |

> ⭐ **설계 결정:** `helm get values`로 values 파일이 노출될 경우 Client Secret이 평문으로 드러난다. Secret → 환경변수 주입 방식으로 민감값을 values 파일에서 분리했다.

---

## 🔧 4. 실행 절차

### 4-1. oauthenticator 내장 확인

```bash
kubectl exec -n ai-team deploy/hub -- python -c \
  'import oauthenticator; from oauthenticator.github import GitHubOAuthenticator; \
   print("oauthenticator version:", getattr(oauthenticator, "__version__", "unknown")); \
   print("GitHubOAuthenticator: OK")'
# oauthenticator version: 17.4.0
# GitHubOAuthenticator: OK
```

Hub 이미지에 oauthenticator 17.4.0 내장 확인. 추가 이미지 빌드 불필요.

### 4-2. GitHub OAuth App 생성

GitHub → Settings → Developer settings → OAuth Apps → New OAuth App

| 항목 | 값 |
|---|---|
| Application name | 28GPU Cluster JupyterHub |
| Homepage URL | `https://hub.112-76-56-162.nip.io` |
| Authorization callback URL | `https://hub.112-76-56-162.nip.io/hub/oauth_callback` |

### 4-3. K8s Secret 등록

```bash
read -r -s -p "GitHub Client ID: " GITHUB_CLIENT_ID; echo
read -r -s -p "GitHub Client Secret: " GITHUB_CLIENT_SECRET; echo

kubectl create secret generic jupyterhub-github-oauth \
  -n ai-team \
  --from-literal=client-id="${GITHUB_CLIENT_ID}" \
  --from-literal=client-secret="${GITHUB_CLIENT_SECRET}"

unset GITHUB_CLIENT_ID GITHUB_CLIENT_SECRET
```

```bash
kubectl describe secret jupyterhub-github-oauth -n ai-team
# Name:         jupyterhub-github-oauth
# Namespace:    ai-team
# Type:         Opaque
# Data
# ====
# client-id:      20 bytes
# client-secret:  40 bytes
```

### 4-4. 기존 values 백업

```bash
mkdir -p ~/backup/jupyterhub
helm get values jupyterhub -n ai-team \
  > ~/backup/jupyterhub/values-before-oauth-$(date +%Y%m%d-%H%M).yaml
```

> ⚠️ **주의:** `helm get values` 출력에 `proxy.secretToken` 평문이 포함된다. 백업 파일 권한을 600으로 조여둘 것.
> ```bash
> chmod 600 ~/backup/jupyterhub/*.yaml
> ```

### 4-5. full values 파일 작성

```bash
BACKUP_FILE="$(ls -t ~/backup/jupyterhub/values-before-oauth-*.yaml | head -1)"
PROXY_SECRET_TOKEN="$(awk '/secretToken:/ {print $2; exit}' "$BACKUP_FILE")"

cat > ~/jupyterhub-oauth-full-values.yaml <<EOF
hub:
  config:
    JupyterHub:
      authenticator_class: github
    GitHubOAuthenticator:
      oauth_callback_url: https://hub.112-76-56-162.nip.io/hub/oauth_callback
    Authenticator:
      allowed_users:
        - 52K-0812
      admin_users:
        - 52K-0812
  extraEnv:
    GITHUB_CLIENT_ID:
      valueFrom:
        secretKeyRef:
          name: jupyterhub-github-oauth
          key: client-id
    GITHUB_CLIENT_SECRET:
      valueFrom:
        secretKeyRef:
          name: jupyterhub-github-oauth
          key: client-secret
  extraConfig:
    00-github-oauth-secret: |
      import os
      c.GitHubOAuthenticator.client_id = os.environ["GITHUB_CLIENT_ID"]
      c.GitHubOAuthenticator.client_secret = os.environ["GITHUB_CLIENT_SECRET"]

proxy:
  secretToken: ${PROXY_SECRET_TOKEN}
  service:
    type: LoadBalancer

singleuser:
  cpu:
    guarantee: 1
    limit: 4
  extraResource:
    guarantees:
      nvidia.com/gpu: "1"
    limits:
      nvidia.com/gpu: "1"
  image:
    name: cschranz/gpu-jupyter
    tag: v1.6_cuda-12.0_ubuntu-22.04
  memory:
    guarantee: 4G
    limit: 16G
  storage:
    capacity: 50Gi
    dynamic:
      storageClass: nfs-client
    type: dynamic
EOF
```

### 4-6. dry-run 검증

```bash
helm upgrade jupyterhub jupyterhub/jupyterhub \
  --version 4.3.3 -n ai-team \
  -f ~/jupyterhub-oauth-full-values.yaml \
  --dry-run --debug 2>&1 | tee ~/jupyterhub-oauth-dryrun-clean.log

grep -nE "authenticator_class|GitHubOAuthenticator|allowed_users|admin_users|GITHUB_CLIENT|jupyterhub-github-oauth|DummyAuthenticator" \
  ~/jupyterhub-oauth-dryrun-clean.log | head -120
```

정상 기준: `authenticator_class: github`, `GitHubOAuthenticator`, `GITHUB_CLIENT_ID/SECRET` 확인. **`DummyAuthenticator` 미출현.**

### 4-7. helm upgrade 실제 적용

```bash
helm upgrade jupyterhub jupyterhub/jupyterhub \
  --version 4.3.3 -n ai-team \
  -f ~/jupyterhub-oauth-full-values.yaml \
  --timeout 5m
# Release "jupyterhub" has been upgraded.
# STATUS: deployed  REVISION: 11

kubectl rollout status deployment/hub -n ai-team --timeout=180s
# deployment "hub" successfully rolled out

kubectl logs -n ai-team deploy/hub --tail=150 | grep -iE "oauth|github|authenticator|dummy"
# Loading pass-through config section hub.config.GitHubOAuthenticator
# Loading extra config: 00-github-oauth-secret
# [I] Using Authenticator: oauthenticator.github.GitHubOAuthenticator-17.4.0
```

### 4-8. 팀원 계정 추가

```bash
cat > ~/jupyterhub-oauth-add-members.yaml <<'EOF'
hub:
  config:
    Authenticator:
      allowed_users:
        - 52K-0812
        - Yeeeho
        - Da-Woon-J
      admin_users:
        - 52K-0812
EOF

helm upgrade jupyterhub jupyterhub/jupyterhub \
  --version 4.3.3 -n ai-team \
  --reuse-values \
  -f ~/jupyterhub-oauth-add-members.yaml \
  --timeout 5m
# STATUS: deployed  REVISION: 12
```

> ⭐ **참고:** 팀원 추가는 authenticator_class 등 핵심 설정 변경이 없으므로 `--reuse-values` 방식 사용 가능. allowed_users 조각 파일만 적용하면 된다.

### 4-9. DummyAuth PVC 및 Hub DB 레코드 정리

```bash
kubectl delete pvc claim-member-01 claim-member-02 claim-member-03 \
  claim-member-04 claim-member-05 -n ai-team --ignore-not-found
```

`claim-member-01`은 `jupyter-member-01` Pod가 사용 중이어서 즉시 삭제되지 않고 Terminating 상태로 유지됨. Pod 종료 후 자동 삭제 완료.

```bash
kubectl delete pod jupyter-member-01 -n ai-team --ignore-not-found
kubectl get pvc -n ai-team
```

Hub Admin 페이지 (`https://hub.112-76-56-162.nip.io/hub/admin`) 에서 member-01~05 사용자 레코드 → Stop server → Delete user 순으로 정리 완료.

---

## ✅ 5. 최종 검증 결과

```bash
curl -k -s -o /dev/null -w "%{http_code}\n" https://hub.112-76-56-162.nip.io/hub/login
# 200

grep -iE "github|sign in" /tmp/jupyterhub-login.html
# href='/hub/oauth_login?next='>Sign in with GitHub</a>
```

| 항목 | 결과 |
|---|---|
| Hub 로그인 페이지 | HTTP 200, Sign in with GitHub 노출 |
| 52K-0812 로그인 | ✅ 성공, JupyterLab 진입, Python 3.11.7 kernel 정상 |
| Yeeeho 로그인 | ✅ 성공, PVC `claim-yeeeho` 자동 생성 |
| Da-Woon-J 로그인 | ⏳ 미검증 — 로그인 예정 (PVC는 최초 서버 기동 시 자동 생성) |
| 허용 외 계정 접근 | ✅ 403 Forbidden 차단 |
| DummyAuth PVC 정리 | ✅ claim-member-01~05 삭제 완료 |
| Hub DB 레코드 정리 | ✅ member-01~05 Admin UI에서 삭제 완료 |

---

## 📋 6. 주요 변경 요약

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 인증 방식 | DummyAuthenticator | GitHubOAuthenticator (oauthenticator 17.4.0) |
| 계정 관리 | member-01~05 / 공유 비밀번호 | GitHub username 기반 개인 계정 |
| 허용 사용자 | member-01~05 | 52K-0812, Yeeeho, Da-Woon-J |
| 관리자 | 없음 | 52K-0812 |
| Client 인증 정보 | 없음 | K8s Secret `jupyterhub-github-oauth` |
| 접속 URL | `http://TAILSCALE-IP:30000` | `https://hub.112-76-56-162.nip.io` |

---

## 💡 7. 핵심 인사이트

**`--reuse-values`로 인증 방식을 교체하려 하면 기존 설정이 병합되어 DummyAuthenticator가 잔존한다.** dry-run 단계에서 이를 확인하지 않으면 두 인증 방식이 혼재된 상태로 적용된다. 인증 방식 전환처럼 기존 설정을 완전히 제거해야 하는 경우에는 반드시 full values 파일 방식으로 적용해야 한다.

**`helm get values` 백업 파일에 `proxy.secretToken`이 평문으로 포함된다.** 백업 자체는 필수이지만 파일 권한(664 기본값)이 같은 그룹 사용자에게 읽히므로 즉시 600으로 조여야 한다.

**PVC 삭제 전 Hub DB 레코드를 함께 정리해야 한다.** PVC를 삭제해도 Hub SQLite DB에 사용자 레코드가 남아 있으면 해당 username으로 서버 기동 시도가 발생하여 새 PVC가 재생성될 수 있다. PVC 정리와 Admin UI 레코드 삭제는 함께 처리해야 완결된다.

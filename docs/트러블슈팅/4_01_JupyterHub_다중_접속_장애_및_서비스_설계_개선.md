# JupyterHub 다중 접속 장애 및 서비스 설계 개선

## 1. 개요

| 항목          | 내용                                                                                  |
| ------------- | ------------------------------------------------------------------------------------- |
| **목적**      | JupyterHub 팀 환경 동시 접속 세션 끊김 원인 규명 → 서비스 구조 전면 재설계            |
| **대상**      | master-01 (LB_PUBLIC_IP), ai-team 네임스페이스, 클러스터 전체 서비스     |
| **핵심 원인** | `kubectl port-forward` 남용 + MetalLB가 마스터 노드 IP 점거 + 비대칭 서비스 타입 혼용 |
| **작업일**    | 2026-04-01                                                                            |

---

## 2. 문제 현상

- 팀원 2명이 `http://LB_PUBLIC_IP:8000`으로 JupyterHub 접속 시 **한 명이 로그인하면 다른 한 명의 세션이 끊김**
- 세션 끊김이 서버 기동(Spawn) 단계가 아닌 **로그인 화면(Hub) 단계**에서 발생
- Portainer(`9443`), JupyterHub(`8000`) curl 요청 모두 응답 없음(hang)
- `kubectl logs` 및 `kubectl port-forward` 실행 시 `dial tcp LB_PUBLIC_IP:10250: i/o timeout` 발생
- MetalLB `speaker` Pod 수십~수백 회 재시작 (`RESTARTS: 89`)

---

## 3. 원인 분석

### 3.1 초기 구조 진단

| 단계 | 확인 명령어                            | 결과                                                                              |
| ---- | -------------------------------------- | --------------------------------------------------------------------------------- |
| 1    | `kubectl get svc -n ai-team`           | proxy-public이 `ClusterIP` — 외부 노출 없음                                       |
| 2    | `ps -ef \| grep port-forward`          | `kubectl port-forward svc/proxy-public ... 8000:80` 프로세스 동작 중              |
| 3    | `systemctl list-units \| grep jupyter` | `kubectl-jupyterhub.service`로 systemd 등록된 포트 포워딩 확인                    |
| 4    | `kubectl get svc -n portainer -o yaml` | Portainer `LoadBalancer` 타입, `EXTERNAL-IP: LB_PUBLIC_IP` 점거 상태 |
| 5    | `kubectl get pods -n metallb-system`   | speaker-nbg8s: 89회, speaker-66kvl: 42회 재시작                                   |
| 6    | `kubectl get pods -n calico-system`    | `calico-node` 다수 `Init:CrashLoopBackOff`                                        |

### 3.2 원인 1 — `kubectl port-forward`를 팀 서버로 운용

`proxy-public` 서비스가 `ClusterIP` 타입이었기 때문에 외부 접속용으로 `port-forward`를 systemd 서비스로 등록해 운용하고 있었다.

```
# 등록되어 있던 포트 포워딩들
kubectl port-forward svc/proxy-public -n ai-team --address 0.0.0.0 8000:80
kubectl port-forward svc/portainer -n portainer --address 0.0.0.0 9443:9443
kubectl port-forward svc/monitoring-grafana -n monitoring --address 0.0.0.0 3000:3000
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring --address 0.0.0.0 9090:9090
```

`port-forward`는 단일 프로세스가 모든 요청을 중계하는 **1인용 임시 터널**이다. 팀원 5명이 동시에 접속하면 WebSocket 연결이 충돌하며 기존 세션을 끊어버린다.

### 3.3 원인 2 — MetalLB가 마스터 노드 IP(`151`)를 점거

MetalLB IP Pool에 `LB_PUBLIC_IP`이 등록되어 있었고, Portainer가 이 IP를 `LoadBalancer` 타입으로 점거하고 있었다. `151`은 `master-01` OS 자체가 사용하는 메인 IP이기 때문에 다음 충돌이 발생했다.

```
master-01 OS:   "LB_PUBLIC_IP은 내 SSH/터미널 주소야"
MetalLB:        "Portainer에 할당됐으니 내가 ARP 응답할게"
학교 스위치:    "어? 151번 MAC이 갑자기 바뀌었네? 차단"
```

학교 네트워크 장비가 ARP를 거부하면서 `speaker` Pod들이 연속 재시작되었고, 그 여파로 Calico 네트워크 레이어까지 불안정해졌다.

### 3.4 원인 3 — 비대칭 서비스 타입 혼용

| 서비스                    | 타입                     | 문제                         |
| ------------------------- | ------------------------ | ---------------------------- |
| proxy-public (JupyterHub) | ClusterIP + port-forward | 1인용 터널, 동시 접속 불가   |
| portainer                 | LoadBalancer             | master-01 IP 점거 → ARP 충돌 |
| monitoring-grafana        | NodePort                 | 정상                         |
| prometheus                | NodePort                 | 정상                         |

`ClusterIP + port-forward`, `NodePort`, `LoadBalancer`가 혼재해 트래픽 경로 추적이 불가능한 구조였다.

### 3.5 장애 전파 경로

```
MetalLB가 master-01 IP(151) 점거 시도
    ↓
학교 스위치가 ARP 충돌로 패킷 드랍
    ↓
MetalLB speaker Pod 연속 재시작 (최대 89회)
    ↓
Calico-node Init:CrashLoopBackOff (install-cni 실패)
    ↓
노드 간 통신 불안정 (i/o timeout)
    ↓
port-forward 터널 불안정 + 5인 동시 접속 세션 충돌
    ↓
JupyterHub 다중 접속 시 세션 끊김
```

---

## 4. 해결 과정

### 4.1 systemd 포트 포워딩 서비스 영구 제거

**🔍 확인:**

```bash
systemctl list-units --type=service | grep jupyter
# 출력: kubectl-jupyterhub.service  loaded activating auto-restart kubectl port-forward JupyterHub
```

**🛠️ 조치:**

```bash
sudo systemctl stop kubectl-jupyterhub.service
sudo systemctl disable kubectl-jupyterhub.service

# 잔존 프로세스 일괄 종료
sudo pkill -f "kubectl port-forward"
```

**결과:** 포트 8000 해제. 이후 정식 서비스 포트로 교체 가능 상태.

---

### 4.2 Portainer LoadBalancer → NodePort 전환 (IP 반납)

**🔍 확인:**

```bash
kubectl get svc portainer -n portainer -o yaml
# type: LoadBalancer, EXTERNAL-IP: LB_PUBLIC_IP 확인
```

**🛠️ 조치:**

```bash
kubectl patch svc portainer -n portainer -p '{"spec": {"type": "NodePort"}}'
```

**결과:** MetalLB가 `151` IP 점거를 중단 → 학교 스위치와의 ARP 충돌 해소.

---

### 4.3 Calico 네트워크 재초기화

IP 충돌이 해소된 후 `CrashLoopBackOff` 상태의 Calico Pod들을 재생성했다.

**🛠️ 조치:**

```bash
kubectl delete pod -n calico-system --all

# 복구 모니터링
kubectl get pods -n calico-system -w
# → 전체 Running 전환 확인
```

**결과:** `calico-node` 전체 `Running` 복구 → 노드 간 통신 정상화.

---

### 4.4 JupyterHub proxy-public NodePort 재생성

기존 서비스에 중복 포트 항목(`8000`, `8080`)이 남아있어 `apply`가 실패했다. `delete` 후 재생성 방식을 택했다.

**🛠️ 조치:**

```bash
kubectl delete svc proxy-public -n ai-team

kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: proxy-public
  namespace: ai-team
spec:
  type: NodePort
  selector:
    app: jupyterhub
    component: proxy
    release: jupyterhub
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 8000
    nodePort: 30080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
EOF
```

**결과:** `NodePort 30080` 정식 개방. `sessionAffinity: ClientIP`로 3시간 세션 유지. 팀원 5명 동시 접속 확인.

---

### 4.5 MetalLB IP Pool 교체 — 구 클러스터 IP 재활용

장애의 근본 원인이 IP Pool에 마스터 노드 IP(`151`)가 포함된 것임을 확인했다. 구 클러스터에서 웹서비스 제공용으로 학교 스위치에 이미 허용 등록된 `LB_PUBLIC_IP`로 Pool을 교체하면 ARP 충돌 없이 LoadBalancer 타입을 사용할 수 있다고 판단했다.

**🔍 공석 확인:**

```bash
ping -c 5 LB_PUBLIC_IP
# → Destination Host Unreachable (미사용 IP 확인)
```

**🛠️ Pool 교체:**

```bash
kubectl patch ipaddresspool main-pool -n metallb-system \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/addresses", "value":["LB_PUBLIC_IP/32"]}]'

kubectl get ipaddresspools -n metallb-system
# ADDRESSES: ["LB_PUBLIC_IP/32"] 확인
```

**🔍 더미 서비스로 IP 할당 가능성 사전 검증:**

실서비스에 바로 적용하지 않고 더미 서비스를 먼저 띄워 학교 스위치가 `158` IP를 실제로 허용하는지 검증했다.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ip-test-service
  namespace: default
  annotations:
    metallb.universe.tf/loadBalancerIPs: LB_PUBLIC_IP
spec:
  selector:
    app: non-existent-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl get svc ip-test-service
# EXTERNAL-IP: LB_PUBLIC_IP → 즉시 할당 확인

kubectl delete svc ip-test-service
```

**결과:** `EXTERNAL-IP: LB_PUBLIC_IP` 정상 할당. 학교 스위치가 해당 IP를 허용하고 있음 검증 완료.

---

### 4.6 서비스 역할 분리 설계 확정

검증 완료 후 서비스 접근 경로를 목적에 따라 분리했다.

| 접근 대상     | 방식         | IP                          | 포트                     |
| ------------- | ------------ | --------------------------- | ------------------------ |
| 학과생 (공용) | LoadBalancer | `LB_PUBLIC_IP` | 80 (포트 번호 없이 접속) |
| 관리자 전용   | NodePort     | `LB_PUBLIC_IP` | `303xx` 시리즈           |

---

### 4.7 JupyterHub — LoadBalancer + 공개 IP 전환

```bash
# proxy-public을 LoadBalancer + 158 IP로 전환
kubectl patch svc proxy-public -n ai-team \
  -p '{"spec": {"type": "LoadBalancer", "loadBalancerIP": "LB_PUBLIC_IP"}}'

# 외부 포트를 80으로 변경 (포트 번호 없이 접속)
kubectl patch svc proxy-public -n ai-team \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/port", "value": 80}]'

# Tailscale 접속용 NodePort를 30000으로 정리
kubectl patch svc proxy-public -n ai-team \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30000}]'
```

**결과:** 학과생은 `http://LB_PUBLIC_IP` 입력만으로 접속 가능. Tailscale 원격 접속은 `http://TAILSCALE_HOST:30000`.

---

### 4.8 관리자 도구 포트 번호 체계 정규화

관리 도구 NodePort를 `303xx` 시리즈로 통일해 목적별 포트 번호 규칙을 만들었다.

```bash
# Prometheus: 30090 → 30310
kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30310}]'

# Portainer: 30900 → 30320
kubectl patch svc portainer -n portainer \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30320}]'
```

---

## 5. 최종 서비스 구성

```bash
kubectl get svc -A | grep -E 'NodePort|LoadBalancer'
```

```
ai-team     proxy-public                          LoadBalancer  10.97.60.206   LB_PUBLIC_IP  80:30000/TCP
monitoring  monitoring-grafana                    NodePort      10.104.33.176  <none>         3000:30356/TCP,80:30300/TCP
monitoring  monitoring-kube-prometheus-prometheus NodePort      10.97.237.241  <none>         9090:30310/TCP,8080:32311/TCP
portainer   portainer                             NodePort      10.102.122.237 <none>         9000:30320/TCP
```

### 공용 접속 (학과생)

| 서비스                 | 접속 주소                          | 비고                                  |
| ---------------------- | ---------------------------------- | ------------------------------------- |
| JupyterHub             | `http://LB_PUBLIC_IP` | 포트 번호 없음, 학교 내부망 직접 접속 |
| JupyterHub (Tailscale) | `http://TAILSCALE_HOST:30000`       | 외부/원격 접속                        |

### 관리자 전용 (151 + 303xx 포트 시리즈)

| 서비스     | 접속 주소                                | 용도                    |
| ---------- | ---------------------------------------- | ----------------------- |
| Grafana    | `http://LB_PUBLIC_IP:30300` | GPU/Pod 모니터링 시각화 |
| Prometheus | `http://LB_PUBLIC_IP:30310` | 원본 메트릭 조회        |
| Portainer  | `http://LB_PUBLIC_IP:30320` | 컨테이너/Pod 관리       |

---

## 6. 핵심 인사이트

- **`kubectl port-forward`는 개발자 1인용 디버깅 도구다.** 다중 사용자 환경에 systemd 서비스로 등록해 운용하면 WebSocket 세션 충돌과 병목이 필연적으로 발생한다. `NodePort` 또는 `Ingress`로 대체해야 한다.
- **MetalLB IP Pool에 노드 자신의 IP를 절대 포함하지 마라.** 노드 OS가 사용 중인 IP를 LoadBalancer에 할당하면 ARP 충돌이 발생해 학교·기업 네트워크 장비가 해당 포트를 차단하고 speaker Pod를 연속 재시작시킨다.
- **On-premise 환경에서 LoadBalancer IP는 네트워크 관리자가 허용한 대역만 사용해야 한다.** 학교 스위치는 Port Security(IP-MAC 바인딩)와 ARP Inspection을 적용하는 경우가 많다. 기존에 허용된 IP를 재활용하거나, 전산팀에 요청해 허용 대역을 확보해야 한다.
- **더미 서비스로 IP 할당 가능성을 먼저 검증하라.** 실서비스에 바로 적용하기 전에 테스트용 서비스를 띄워 `EXTERNAL-IP`가 `<pending>` 없이 즉시 할당되는지 확인하는 것이 안전한 순서다.
- **서비스 타입 비대칭 혼용은 장애 분석을 극도로 어렵게 만든다.** 동일 클러스터 내 서비스는 접근 목적(공용/관리자)에 따라 방식을 통일하고, 포트 번호도 규칙 있게 관리해야 한다.
- **Calico CrashLoopBackOff 시 먼저 `install-cni` 로그를 확인해라.** Calico 본체가 아니라 CNI 설치 초기화 단계에서 실패하는 경우가 많다.

---

## 7. 빠른 진단 명령어 레퍼런스

```bash
# 1. 실행 중인 port-forward 전체 확인
ps -ef | grep port-forward

# 2. systemd로 등록된 kubectl 서비스 확인
systemctl list-units --type=service | grep kubectl

# 3. LoadBalancer IP 점거 상태 확인
kubectl get svc -A | grep LoadBalancer

# 4. MetalLB speaker 재시작 횟수 확인 (10회 이상이면 IP 충돌 의심)
kubectl get pods -n metallb-system

# 5. MetalLB IP Pool 확인
kubectl get ipaddresspools -n metallb-system

# 6. Calico 상태 확인
kubectl get pods -n calico-system

# 7. Calico install-cni 실패 로그 확인
kubectl logs -n calico-system [calico-node-pod명] -c install-cni

# 8. 전체 NodePort/LoadBalancer 포트 한눈에 보기
kubectl get svc -A | grep -E 'NodePort|LoadBalancer'

# 9. 포트 포워딩 전체 강제 종료
sudo pkill -f "kubectl port-forward"
sudo systemctl stop kubectl-*.service
sudo systemctl disable kubectl-*.service
```

---

## 8. 핵심 인사이트

이번 작업은 **초기 임시방편이 팀 운영 단계에서 한계를 드러낸 전형적인 기술 부채 케이스**였다. `port-forward`를 systemd 서비스로 등록해 영구화한 것이 문제를 잠복시켰고, MetalLB IP Pool에 마스터 노드 자체 IP를 넣은 설정 오류가 Calico 네트워크 마비라는 2차 장애로 이어졌다.

장애 복구 이후 단순히 "동작하는 상태"로 되돌리는 데서 그치지 않고, 서비스 접근 경로를 **목적에 따라 구조적으로 재설계**했다는 점이 핵심이다. 구 클러스터에서 이미 학교 스위치에 허용 등록된 `LB_PUBLIC_IP`을 MetalLB Pool로 재활용하고, 더미 서비스로 IP 할당 가능성을 사전 검증한 뒤 실서비스에 적용한 순서가 안전했다.

최종 구조는 학과생용 공개 대문(`158`, 포트 없음)과 관리자 전용 통로(`151:303xx` 시리즈)를 명확히 분리한다. 포트 번호 체계(`30300/30310/30320`) 통일은 이후 팀원 온보딩과 추가 서비스 연동 시 혼선을 줄이는 운영 원칙이다.

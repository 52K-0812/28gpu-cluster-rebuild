# Calico CNI 설치 (Pod Network)

> **작업 일자:** 2026-03-20
> **대상 장비:** master-01 (MASTER_IP)
> **목적:** Pod 간 통신을 위한 네트워크 플러그인 설치

---

## 1. 작업 개요

Kubernetes는 기본적으로 Pod 간 네트워크를 제공하지 않습니다. CNI(Container Network Interface) 플러그인을 별도로 설치해야 노드 상태가 `NotReady`에서 `Ready`로 전환됩니다.

**Calico를 선택한 이유:**

| 항목              | 내용                                         |
| ----------------- | -------------------------------------------- |
| **안정성**        | 대규모 프로덕션 환경에서 검증된 CNI          |
| **성능**          | eBPF 기반 고성능 네트워크 처리               |
| **호환성**        | kubeadm 기본 권장 CNI 중 하나                |
| **네트워크 정책** | NetworkPolicy 지원으로 Pod 간 접근 제어 가능 |

---

## 2. Calico 설치

```bash
# Tigera Operator 설치 (Calico 관리 컨트롤러)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# 네트워크 리소스 설정 적용 (192.168.0.0/16 대역)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

> **192.168.0.0/16 대역 이유:** `kubeadm init` 시 `--pod-network-cidr=192.168.0.0/16`으로 설정한 대역과 반드시 일치해야 합니다. 대역이 다르면 Pod 간 통신이 불가합니다.

---

## 3. 설치 확인

```bash
# 모든 Pod가 Running 상태가 될 때까지 확인 (약 1~2분 소요)
kubectl get pods -A

# 노드 상태 확인
kubectl get nodes
```

**정상 결과:**

```
NAME        STATUS   ROLES           AGE
master-01   Ready    control-plane   Xm
```

`STATUS: Ready` — Calico가 정상적으로 네트워크를 구성했음을 의미합니다.

---

## 4. 작업 결과

- Calico v3.27.0 설치 완료
- master-01 노드 상태 `NotReady` → `Ready` 전환 확인
- 다음 단계: 마스터 노드 스케줄링 허용 (Taint 해제)

![자료사진](../images/Pasted image 20260320154933.png)

# 🛡️ [트러블슈팅] YOLOv8 서빙 이미지화 전환 중 장애 3건

> **발생 일자:** 2026-04-16
> **대상:** ai-team 네임스페이스, yolov8-serving Deployment, 2080ti-gpu-04
> **환경:** Kubernetes v1.29, containerd, nerdctl v1.7.6
> **작업 맥락:** `ultralytics:8.1.0` + 런타임 `pip install` → 커스텀 이미지(`yolov8-serving:local-v1`) 전환 작업 중 발생
> **결과:** 3건 모두 조치 완료. 서빙 정상 복구.
> **관련 문서:** `journal/4_16_YOLOv8_서빙_이미지화.md`, `runbooks/runbook_model_serving.md`

---

## 1. 장애 목록

| 번호 | 증상 | 원인 요약 | 심각도 |
|---|---|---|---|
| 1 | ImagePullBackOff | nerdctl load namespace 불일치 | 중간 |
| 2 | CrashLoopBackOff | ConfigMap read-only 마운트로 모델 다운로드 실패 | 중간 |
| 3 | rollout progress deadline 초과 | nodeSelector 미고정으로 다른 노드 스케줄 | 중간 |

> 3건 모두 기존 Running Pod는 유지된 상태에서 새 Pod에만 장애 발생. 서비스 중단 없음.

---

## 2. 장애 상세

### 2-1. ImagePullBackOff — nerdctl load namespace 불일치

**증상**

```text
Failed to pull image "yolov8-serving:local-v1": failed to resolve reference
"docker.io/library/yolov8-serving:local-v1": pull access denied,
repository does not exist or may require authorization
```

새 Pod가 2080ti-gpu-04에 스케줄됐음에도 이미지를 찾지 못하고 ImagePullBackOff 발생.

**원인**

`nerdctl load` 기본 namespace는 containerd의 `default`.
K8s kubelet은 containerd의 `k8s.io` namespace 이미지만 참조한다.
두 namespace가 격리되어 있어, `default`에 load한 이미지는 K8s에서 보이지 않음.

```text
containerd namespace
  ├── default       ← nerdctl load 기본값 (K8s 미사용)
  └── k8s.io        ← kubelet 참조 namespace
```

**조치**

```bash
# k8s.io namespace로 재load
ssh ubuntu@WORKER-IP \
  'sudo nerdctl --namespace k8s.io load -i /tmp/yolov8-serving-local-v1.tar'

# 확인
ssh ubuntu@WORKER-IP \
  'sudo nerdctl --namespace k8s.io images | grep yolov8-serving'
```

**재발 방지**

서빙 노드에 이미지 load 시 반드시 `--namespace k8s.io` 옵션 사용.
`runbook_model_serving.md` 이미지 관리 섹션에 명시 완료.

---

### 2-2. CrashLoopBackOff — ConfigMap read-only 마운트로 모델 다운로드 실패

**증상**

```text
Warning: Failed to create the file yolov8n.pt: Read-only file system
curl: (23) Failed writing body (0 != 16375)

ConnectionError: Download failure for
https://github.com/ultralytics/assets/releases/download/v8.1.0/yolov8n.pt.
Retry limit reached.
```

Pod가 ImagePull 성공 후 기동 직후 crash, CrashLoopBackOff 반복.

**원인**

기존 Deployment에 ConfigMap(`yolov8-serving-code`)이 `/app` 경로에 마운트되어 있었다.
ConfigMap 마운트는 read-only다.
ultralytics `YOLO("yolov8n.pt")` 호출 시 현재 디렉토리(`/app`)에 모델 파일 다운로드를 시도 → 쓰기 불가 → 예외 발생 → crash.

이미지화 이후 `main.py`가 이미지 내 `/app/main.py`에 내장되어 있어 ConfigMap 마운트 자체가 불필요한 상태였음.

**조치**

```bash
# /app ConfigMap volumeMount 및 volume 제거
kubectl patch deployment yolov8-serving -n ai-team --type='json' -p='[
  {"op":"remove","path":"/spec/template/spec/containers/0/volumeMounts/0"},
  {"op":"remove","path":"/spec/template/spec/volumes/0"}
]'
```

**재발 방지**

이미지 기반 배포에서는 ConfigMap 코드 마운트 사용 금지.
코드 변경 시 이미지 재빌드(`local-v2`, `local-v3` 등 버전 올릴 것)로만 반영.
ConfigMap은 환경변수나 설정값 주입 용도로만 사용.

---

### 2-3. rollout progress deadline 초과 — nodeSelector 미고정으로 다른 노드 스케줄

**증상**

```text
error: deployment "yolov8-serving" exceeded its progress deadline

NAME                              READY   STATUS             NODE
yolov8-serving-865cb84456-nm678   0/1     ImagePullBackOff   2080ti-gpu-03
yolov8-serving-6f4bfb86f5-hgp5s   1/1     Running            2080ti-gpu-04
```

새 Pod가 의도하지 않은 `2080ti-gpu-03`에 스케줄되어 이미지를 찾지 못함.

**원인**

Deployment nodeSelector가 `gpu-type: 2080ti`만 설정되어 있어
`2080ti-gpu-02`, `2080ti-gpu-03`, `2080ti-gpu-04` 어디로도 스케줄 가능한 상태.
이미지는 `2080ti-gpu-04` 로컬에만 존재 → 다른 노드 스케줄 시 즉시 ImagePullBackOff.

**조치**

```bash
# hostname nodeSelector 추가로 2080ti-gpu-04 고정
kubectl patch deployment yolov8-serving -n ai-team --type='merge' \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"2080ti-gpu-04"}}}}}'
```

**재발 방지**

로컬 이미지 보관 구조에서는 `kubernetes.io/hostname` nodeSelector 필수.
DockerHub 또는 레지스트리 push 이후에는 hostname 고정 완화 가능.
→ 2026-04-17 DockerHub push 완료 후 hostname 고정 제거됨.

---

## 3. 핵심 인사이트

**containerd namespace 격리는 직관적이지 않다.** `nerdctl images`로 이미지가 보여도 K8s가 못 찾는 상황이 발생한다. 이미지 load 전에 항상 namespace를 명시하는 습관이 필요하다.

**ConfigMap 마운트 경로와 이미지 내 경로가 충돌하면 read-only 문제가 숨겨진다.** 이미지화 전환 시 기존 마운트 구성을 함께 정리하지 않으면 예상치 못한 위치에서 장애가 발생한다.

**nodeSelector 범위와 이미지 가용 범위는 항상 일치해야 한다.** 이미지가 1개 노드 로컬에만 있는 상태에서 여러 노드로 스케줄될 수 있는 구성은 언제든 장애를 유발한다.

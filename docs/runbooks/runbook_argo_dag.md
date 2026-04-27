# Runbook — Argo Workflows DAG 파이프라인

> **목적:** YOLOv8 DAG 파이프라인의 실행 · 파라미터 변경 · 장애 대응 절차
> **최초 구축:** 2026-04-07
> **관리 네임스페이스:** `ai-team`
> **담당:** 관리자 (master-01)

---

## 1. 📋 현재 배포 대상

| 항목             | 내용                                                      |
| ---------------- | --------------------------------------------------------- |
| WorkflowTemplate | `yolov8-dag-pipeline` (ai-team 네임스페이스)              |
| Argo 버전        | v4.0.4                                                    |
| 파이프라인 구조  | validate-data → train → evaluate → save-model (4단계 DAG) |
| 학습 GPU         | V100 × 4 (DDP, `gpu-type: v100` nodeSelector)             |
| 데이터셋         | VisDrone (`nfs-datasets-pvc` → NAS `/data/datasets/`)     |
| 모델 저장 경로   | NAS `/data/datasets/models/visdrone-{version}.pt`         |
| UI 접속          | `http://TAILSCALE-IP:30500` (Tailscale)                   |

**파이프라인 흐름:**

```
[validate-data]
    │ VisDrone 데이터셋 존재 확인 — 실패 시 전체 중단
    ▼
[train]
    │ YOLOv8n V100 × 4 DDP 학습
    │ NAS /mnt/datasets/runs/dag-train/ 저장
    ▼
[evaluate]
    │ best.pt 로드 → val 평가 → mAP@0.5 추출
    │ eval_result.txt 저장
    ▼
[save-model]
    └ best.pt → /mnt/datasets/models/visdrone-{version}.pt
```

> **설계 원칙:** 각 단계 실패 시 이후 단계는 실행되지 않는다. 실패 지점은 Argo UI DAG 그래프에서 즉시 확인 가능.

---

## 2. ✅ 사전 점검

```bash
# 1. Argo Workflows Controller 상태
kubectl get pods -n argo | grep workflow-controller
# 기대값: workflow-controller-xxxx   1/1   Running

# 2. WorkflowTemplate 존재 확인
kubectl get workflowtemplate -n ai-team
# 기대값: yolov8-dag-pipeline 목록에 있어야 함

# 3. V100 노드 GPU 가용량 확인
kubectl get nodes -l gpu-type=v100 -o custom-columns=\
NAME:.metadata.name,GPU:.status.allocatable."nvidia\.com/gpu"
# 기대값: v100-gpu-01   4

# 4. NFS PVC 상태 확인
kubectl get pvc nfs-datasets-pvc -n ai-team
# 기대값: STATUS Bound

# 5. VisDrone 데이터셋 존재 확인 (NAS)
ls /data/datasets/visdrone/VisDrone2019-DET-train/images | wc -l
ls /data/datasets/visdrone/VisDrone2019-DET-val/images | wc -l
```

---

## 3. ▶️ 실행 절차

### Argo UI에서 실행 (권장)

1. `http://TAILSCALE-IP:30500` 접속
2. 왼쪽 메뉴 **Workflow Templates** → `ai-team` → `yolov8-dag-pipeline`
3. **Submit** 클릭
4. 파라미터 입력:

| 파라미터        | 기본값 | 설명                        |
| --------------- | ------ | --------------------------- |
| `epochs`        | `100`  | 학습 에포크 수              |
| `batch-size`    | `16`   | 배치 사이즈                 |
| `model-version` | `v1`   | 저장 버전명 (v1, v2, v3...) |

5. **+SUBMIT** 클릭

### kubectl로 실행

```bash
kubectl create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: yolov8-dag-
  namespace: ai-team
spec:
  workflowTemplateRef:
    name: yolov8-dag-pipeline
  arguments:
    parameters:
    - name: epochs
      value: "100"
    - name: batch-size
      value: "16"
    - name: model-version
      value: "v3"
EOF
```

### WorkflowTemplate 수정 후 재적용

> ⚠️ `kubectl patch` 방식은 volumes 인덱스 충돌로 오류가 발생할 수 있다. 반드시 파일로 관리한다.

```bash
kubectl delete workflowtemplate yolov8-dag-pipeline -n ai-team
kubectl apply -f yolov8-dag-pipeline.yaml
```

---

## 4. 📄 WorkflowTemplate YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: yolov8-dag-pipeline
  namespace: ai-team
spec:
  entrypoint: pipeline
  arguments:
    parameters:
      - name: epochs
        value: '100'
      - name: batch-size
        value: '16'
      - name: model-version
        value: 'v1'

  templates:
    - name: pipeline
      dag:
        tasks:
          - name: validate-data
            template: validate-data
          - name: train
            template: train
            dependencies: [validate-data]
            arguments:
              parameters:
                - name: epochs
                  value: '{{workflow.parameters.epochs}}'
                - name: batch-size
                  value: '{{workflow.parameters.batch-size}}'
          - name: evaluate
            template: evaluate
            dependencies: [train]
          - name: save-model
            template: save-model
            dependencies: [evaluate]
            arguments:
              parameters:
                - name: model-version
                  value: '{{workflow.parameters.model-version}}'

    - name: validate-data
      container:
        image: busybox
        command: [sh, -c]
        args:
          - |
            if [ -d "/mnt/datasets/visdrone/VisDrone2019-DET-train/images" ]; then
              echo "Train: $(ls /mnt/datasets/visdrone/VisDrone2019-DET-train/images | wc -l)장 확인"
            else echo "ERROR: Train 데이터셋 없음"; exit 1; fi
            if [ -d "/mnt/datasets/visdrone/VisDrone2019-DET-val/images" ]; then
              echo "Val: $(ls /mnt/datasets/visdrone/VisDrone2019-DET-val/images | wc -l)장 확인"
            else echo "ERROR: Val 데이터셋 없음"; exit 1; fi
        volumeMounts:
          - name: datasets
            mountPath: /mnt/datasets
      volumes:
        - name: datasets
          persistentVolumeClaim:
            claimName: nfs-datasets-pvc

    - name: train
      inputs:
        parameters:
          - name: epochs
          - name: batch-size
      nodeSelector:
        gpu-type: v100
      container:
        image: ultralytics/ultralytics:8.1.0
        command: [python, -c]
        args:
          - |
            from ultralytics import YOLO
            model = YOLO('yolov8n.pt')
            model.train(
              data='/mnt/datasets/visdrone/visdrone.yaml',
              epochs={{inputs.parameters.epochs}},
              batch={{inputs.parameters.batch-size}},
              imgsz=640, device='0,1,2,3',
              project='/mnt/datasets/runs', name='dag-train'
            )
        resources:
          limits:
            nvidia.com/gpu: '4'
        volumeMounts:
          - name: datasets
            mountPath: /mnt/datasets
          - name: dshm
            mountPath: /dev/shm
      volumes:
        - name: datasets
          persistentVolumeClaim:
            claimName: nfs-datasets-pvc
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 16Gi

    - name: evaluate
      nodeSelector:
        gpu-type: v100
      container:
        image: ultralytics/ultralytics:8.1.0
        command: [python, -c]
        args:
          - |
            import os
            best_pt = '/mnt/datasets/runs/dag-train/weights/best.pt'
            if not os.path.exists(best_pt):
              print("ERROR: best.pt 없음"); exit(1)
            from ultralytics import YOLO
            metrics = YOLO(best_pt).val(
              data='/mnt/datasets/visdrone/visdrone.yaml',
              imgsz=640, device=0
            )
            map50 = metrics.box.map50
            print(f"mAP@0.5: {map50:.4f}")
            open('/mnt/datasets/runs/dag-train/eval_result.txt', 'w').write(f"mAP50={map50:.4f}\n")
        resources:
          limits:
            nvidia.com/gpu: '1'
        volumeMounts:
          - name: datasets
            mountPath: /mnt/datasets
          - name: dshm
            mountPath: /dev/shm
      volumes:
        - name: datasets
          persistentVolumeClaim:
            claimName: nfs-datasets-pvc
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 8Gi

    - name: save-model
      inputs:
        parameters:
          - name: model-version
      container:
        image: busybox
        command: [sh, -c]
        args:
          - |
            VERSION={{inputs.parameters.model-version}}
            SRC="/mnt/datasets/runs/dag-train/weights/best.pt"
            DST="/mnt/datasets/models/visdrone-${VERSION}.pt"
            mkdir -p /mnt/datasets/models
            [ -f "$SRC" ] && cp "$SRC" "$DST" && echo "저장 완료: ${DST}" \
              || (echo "ERROR: best.pt 없음"; exit 1)
            ls -lh /mnt/datasets/models/
        volumeMounts:
          - name: datasets
            mountPath: /mnt/datasets
      volumes:
        - name: datasets
          persistentVolumeClaim:
            claimName: nfs-datasets-pvc
```

---

## 5. 🔍 검증 절차

```bash
# 1. Workflow 실행 상태 확인
kubectl get workflow -n ai-team
# 기대값: Running → Succeeded

# 2. 단계별 Pod 상태 확인
kubectl get pods -n ai-team -l workflows.argoproj.io/workflow=<workflow-name>

# 3. NAS 모델 저장 확인
ls -lh /data/datasets/models/
# 기대값: visdrone-v{N}.pt 파일 존재

# 4. mAP 결과 확인
cat /data/datasets/runs/dag-train/eval_result.txt
# 기대값: mAP50=0.XXXX
```

**단계별 정상 소요 시간 (epochs=1 기준):**

| 단계          | 소요 시간 |
| ------------- | --------- |
| validate-data | ~5s       |
| train         | ~3m       |
| evaluate      | ~30s      |
| save-model    | ~5s       |

---

## 6. ↩️ 롤백 절차

### 특정 단계만 재실행

Argo UI에서 실패한 단계 클릭 → **Retry** 버튼으로 해당 단계부터 재실행.

### 이전 버전 모델 복구

save-model은 버전명이 다르면 덮어쓰지 않는다. NAS에 버전별 파일이 보존되어 있다.

```bash
# 저장된 모델 버전 목록
ls -lh /data/datasets/models/

# 특정 버전을 현재 버전으로 복사
cp /data/datasets/models/visdrone-v1.pt /data/datasets/models/visdrone-latest.pt
```

### WorkflowTemplate 이전 버전 복구

```bash
# 현재 템플릿 백업
kubectl get workflowtemplate yolov8-dag-pipeline -n ai-team -o yaml > backup.yaml

# 이전 버전 파일로 복구
kubectl delete workflowtemplate yolov8-dag-pipeline -n ai-team
kubectl apply -f yolov8-dag-pipeline-prev.yaml
```

---

## 7. 🚨 장애 대응

### Workflow가 Pending에서 진행되지 않는 경우

```bash
kubectl describe workflow <workflow-name> -n ai-team | tail -20
```

| 증상                  | 원인                         | 조치                                             |
| --------------------- | ---------------------------- | ------------------------------------------------ |
| validate-data Pending | V100 GPU 자원 없음           | 실행 중인 학습 Job 확인 후 대기 또는 정리        |
| validate-data Failed  | VisDrone 데이터셋 없음       | `ls /data/datasets/visdrone/` 경로 확인          |
| train OOMKilled       | `/dev/shm` 부족 또는 GPU OOM | dshm sizeLimit 조정 또는 batch-size 축소         |
| evaluate Failed       | best.pt 없음                 | `ls /data/datasets/runs/dag-train/weights/` 확인 |

### shared memory 부족 오류

```
RuntimeError: unable to write to file </torch_...>: No space left on device
```

GPU를 사용하는 단계(train, evaluate) 모두 dshm 볼륨이 필요하다. YAML 누락 여부 확인 후 재적용.

> 체크포인트: GPU 컨테이너를 새로 추가할 때마다 `dshm` 볼륨 포함 여부를 반드시 확인한다.

```bash
kubectl delete workflowtemplate yolov8-dag-pipeline -n ai-team
kubectl apply -f yolov8-dag-pipeline.yaml
```

### Workflow Controller 장애

```bash
kubectl rollout restart deployment/workflow-controller -n argo
kubectl get pods -n argo -w
```

## 8.참고 사진
![](../images/스크린샷%202026-04-07%20172757.png)
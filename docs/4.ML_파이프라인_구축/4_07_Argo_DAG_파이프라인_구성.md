# Argo Workflows DAG 파이프라인 구성

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-07
> **작업 목적:** Argo Workflows DAG로 데이터 검증 → 학습 → 모델 평가 → 버전별 저장 파이프라인을 자동화한다. 단계별 의존성을 명시해 이전 단계 실패 시 다음 단계로 진행되지 않도록 설계한다.
> **대상 서버:** master-01 ((control-plane-public-ip)), v100-gpu-01 ((control-plane-public-ip)), NAS ((control-plane-public-ip))
> **작업 환경:** Kubernetes v1.29, Argo Workflows v4.0.4, ai-team 네임스페이스
> **최종 결과:** DAG 4단계 파이프라인 완주, visdrone-v1.pt / visdrone-v2.pt NAS 버전별 저장 확인

---

## 🏗️ 2. 작업 흐름

```
[validate-data]
    │ VisDrone 데이터셋 존재 확인
    │ 실패 시 전체 파이프라인 중단
    ▼
[train]
    │ YOLOv8n V100 ×4 DDP 학습
    │ NAS /mnt/datasets/runs/dag-train/ 저장
    ▼
[evaluate]
    │ best.pt 로드 → val 데이터셋 평가
    │ mAP@0.5 추출 → eval_result.txt 저장
    ▼
[save-model]
    │ best.pt → /mnt/datasets/models/visdrone-{version}.pt 복사
    │ 버전 파라미터로 v1, v2, v3... 관리
```

---

## 📐 3. DAG vs 기존 WorkflowTemplate 차이

| 항목      | 기존 WorkflowTemplate     | DAG 파이프라인            |
| --------- | ------------------------- | ------------------------- |
| 구조      | 단일 컨테이너             | 4단계 분리                |
| 실패 처리 | 전체 실패                 | 해당 단계에서 중단        |
| 모델 저장 | 학습 디렉토리에 자동 저장 | 버전별 명시적 저장        |
| 가시성    | 로그만 확인 가능          | UI에서 단계별 상태 시각화 |
| 재현성    | 수동 버전 관리            | 파라미터로 버전 자동 관리 |

---

## 🚀 4. WorkflowTemplate YAML

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
            echo "=== 데이터 검증 시작 ==="
            if [ -d "/mnt/datasets/visdrone/VisDrone2019-DET-train/images" ]; then
              COUNT=$(ls /mnt/datasets/visdrone/VisDrone2019-DET-train/images | wc -l)
              echo "Train 이미지: ${COUNT}장 확인"
            else
              echo "ERROR: Train 데이터셋 없음"
              exit 1
            fi
            if [ -d "/mnt/datasets/visdrone/VisDrone2019-DET-val/images" ]; then
              COUNT=$(ls /mnt/datasets/visdrone/VisDrone2019-DET-val/images | wc -l)
              echo "Val 이미지: ${COUNT}장 확인"
            else
              echo "ERROR: Val 데이터셋 없음"
              exit 1
            fi
            echo "=== 데이터 검증 완료 ==="
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
              imgsz=640,
              device='0,1,2,3',
              project='/mnt/datasets/runs',
              name='dag-train'
            )
            print("=== 학습 완료 ===")
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
            runs_dir = '/mnt/datasets/runs/dag-train'
            best_pt = f'{runs_dir}/weights/best.pt'
            if not os.path.exists(best_pt):
              print("ERROR: best.pt 없음")
              exit(1)
            from ultralytics import YOLO
            model = YOLO(best_pt)
            metrics = model.val(
              data='/mnt/datasets/visdrone/visdrone.yaml',
              imgsz=640,
              device=0
            )
            map50 = metrics.box.map50
            print(f"=== 모델 평가 완료 ===")
            print(f"mAP@0.5: {map50:.4f}")
            with open('/mnt/datasets/runs/dag-train/eval_result.txt', 'w') as f:
              f.write(f"mAP50={map50:.4f}\n")
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
            echo "=== 모델 저장 시작 ==="
            VERSION={{inputs.parameters.model-version}}
            SRC="/mnt/datasets/runs/dag-train/weights/best.pt"
            DST="/mnt/datasets/models/visdrone-${VERSION}.pt"
            mkdir -p /mnt/datasets/models
            if [ -f "$SRC" ]; then
              cp "$SRC" "$DST"
              echo "저장 완료: ${DST}"
              ls -lh /mnt/datasets/models/
            else
              echo "ERROR: best.pt 없음"
              exit 1
            fi
            echo "=== 모델 저장 완료 ==="
        volumeMounts:
          - name: datasets
            mountPath: /mnt/datasets
      volumes:
        - name: datasets
          persistentVolumeClaim:
            claimName: nfs-datasets-pvc
```

### 적용

```bash
# 로컬에서 master-01으로 파일 전송
scp yolov8-dag-pipeline.yaml ubuntu@(control-plane-public-ip):~/

# master-01에서 적용
kubectl apply -f yolov8-dag-pipeline.yaml
```

---

## 📊 5. Argo UI 사용법

### 파이프라인 제출

1. `http://(control-plane-public-ip):2746` 접속
2. **Workflow Templates** → ai-team → `yolov8-dag-pipeline`
3. **Submit** 클릭
4. 파라미터 입력:
   - `epochs`: 학습 에포크 수
   - `batch-size`: 배치 사이즈
   - `model-version`: 저장할 버전명 (v1, v2, v3...)
5. **+SUBMIT** 클릭

### DAG 시각화

Submit 후 Workflow 클릭하면 DAG 그래프가 실시간으로 업데이트됨:

```
● validate-data → ● train → ● evaluate → ● save-model
(초록: 완료 / 파랑: 실행 중 / 빨강: 실패)
```

---

## ✅ 6. 검증 결과

### 6.1 v1 파이프라인 (epochs=1)

| 단계          | 상태         | 소요 시간 |
| ------------- | ------------ | --------- |
| validate-data | ✅ Succeeded | ~5s       |
| train         | ✅ Succeeded | ~3m       |
| evaluate      | ✅ Succeeded | ~30s      |
| save-model    | ✅ Succeeded | ~5s       |

### 6.2 v2 파이프라인 (epochs=1)

동일 조건으로 재실행 → v2로 버전 저장 성공

### 6.3 NAS 모델 저장 확인

```bash
ssh ubuntu@(control-plane-public-ip) "ls -lh /data/datasets/models/"
```

```
total 12M
-rw-rw-rw- 1 root root 6.0M Apr  7 08:27 visdrone-v1.pt
-rw-rw-rw- 1 root root 6.0M Apr  7 09:28 visdrone-v2.pt
```

---

## 🛠️ 7. 트러블슈팅

### 문제 1: evaluate 단계 shared memory 부족

```
ERROR: Unexpected bus error encountered in worker.
RuntimeError: unable to write to file </torch_...>: No space left on device (28)
```

**원인:** evaluate 템플릿에 dshm 볼륨이 누락됨. train 템플릿에는 있었지만 evaluate에는 없었음.

**해결:** evaluate 템플릿에 dshm 볼륨 추가

```yaml
volumeMounts:
  - name: dshm
    mountPath: /dev/shm
volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 8Gi
```

> **핵심:** GPU를 사용하는 모든 템플릿에 dshm이 필요하다. train뿐 아니라 evaluate도 GPU를 쓰므로 동일하게 적용해야 한다.

### 문제 2: kubectl patch 중복 적용으로 Duplicate value 에러

```
spec.volumes[4].name: Duplicate value: "dshm"
```

**원인:** patch로 수정 시 templates 인덱스를 잘못 지정해 train 템플릿에 dshm이 중복 추가됨.

**해결:** WorkflowTemplate 삭제 후 yaml 파일로 새로 apply. patch 방식보다 yaml 파일 관리가 안전하다.

```bash
kubectl delete workflowtemplate yolov8-dag-pipeline -n ai-team
kubectl apply -f yolov8-dag-pipeline.yaml
```

> **교훈:** WorkflowTemplate 수정은 patch보다 yaml 파일로 관리하는 것이 실수를 줄인다. 파일을 내려받을 때 브라우저의 중복 파일명 처리(`(1)` 자동 추가)에 주의해야 한다.

---

## 💡 8. 핵심 인사이트

**DAG는 파이프라인의 실패 지점을 명확하게 만든다.** 단일 컨테이너 Job은 어느 단계에서 실패했는지 로그를 직접 파야 알 수 있다. DAG 구조에서는 UI에서 시각적으로 어느 단계가 실패했는지 즉시 파악 가능하고, 실패한 단계만 재실행할 수 있다.

**모델 버전 관리는 파라미터로 설계해야 한다.** `model-version` 파라미터 하나로 v1, v2, v3를 구분해 NAS에 저장하는 구조는 실험 추적의 기초다. Submit할 때마다 버전을 올려주면 모델 히스토리가 자동으로 쌓인다.

**GPU를 쓰는 모든 단계에 dshm이 필요하다.** 학습(train)뿐 아니라 평가(evaluate)도 DataLoader 워커를 사용하므로 shared memory가 필요하다. GPU 컨테이너를 새로 추가할 때마다 dshm 볼륨 포함 여부를 체크리스트로 확인해야 한다.

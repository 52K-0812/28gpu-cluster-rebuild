# YOLOv8 VisDrone 멀티GPU 학습 Job

## 🗂️ 1. 작업 개요

> **작업 일자:** 2026-04-03
> **작업 목적:** VisDrone2019 데이터셋을 NAS에 구축하고, V100 4장 멀티GPU로 YOLOv8 드론 Object Detection 모델을 학습한다.
> **대상 서버:** v100-gpu-01, NAS
> **작업 환경:** Kubernetes v1.29, NFS PVC, ultralytics:8.1.0
> **최종 결과:** V100 4장 DDP 멀티GPU 학습 정상 가동

---

## 🏗️ 2. 작업 흐름

```text
[VisDrone 데이터셋 다운로드 → NAS /data/datasets/visdrone/]
        │ YOLO 포맷 변환 (convert_visdrone.py)
        ▼
[K8s Job 제출 → ai-team 네임스페이스]
        │ nodeSelector: gpu-type=v100
        ▼
[V100 4장 DDP 멀티GPU 학습]
        │ NFS PVC 마운트 (/data)
        ▼
[NAS /data/yolov8/runs/visdrone_exp1/weights/best.pt 저장]
```

---

## 📦 3. Step 1 — VisDrone 데이터셋 준비

### 3.1 NAS에 디렉토리 생성 및 다운로드

```bash
mkdir -p /data/datasets/visdrone
cd /data/datasets/visdrone

# Train 세트 (1.4GB)
wget -c https://github.com/ultralytics/assets/releases/download/v0.0.0/VisDrone2019-DET-train.zip

# Val 세트
wget -c https://github.com/ultralytics/assets/releases/download/v0.0.0/VisDrone2019-DET-val.zip
```

> **10GbE 효과:** 1.4GB 다운로드 1분 42초 완료 (14.4MB/s)

### 3.2 압축 해제

```bash
sudo apt install -y unzip
unzip VisDrone2019-DET-train.zip && unzip VisDrone2019-DET-val.zip
```

**결과 구조:**

```text
/data/datasets/visdrone/
├── VisDrone2019-DET-train/
│   ├── images/   (6,471장)
│   └── annotations/
├── VisDrone2019-DET-val/
│   ├── images/   (548장)
│   └── annotations/
```

---

## 🔄 4. Step 2 — YOLO 포맷 변환 스크립트

VisDrone 어노테이션은 YOLO 포맷이 아니라 변환이 필요하다.

```bash
cat > /data/datasets/visdrone/convert_visdrone.py << 'EOF'
import os
from pathlib import Path

def convert_visdrone_to_yolo(data_root):
    for split in ['VisDrone2019-DET-train', 'VisDrone2019-DET-val']:
        ann_dir = Path(data_root) / split / 'annotations'
        img_dir = Path(data_root) / split / 'images'
        label_dir = Path(data_root) / split / 'labels'
        label_dir.mkdir(exist_ok=True)

        for ann_file in ann_dir.glob('*.txt'):
            img_file = img_dir / ann_file.name.replace('.txt', '.jpg')
            if not img_file.exists():
                continue

            from PIL import Image
            img = Image.open(img_file)
            w, h = img.size

            yolo_lines = []
            with open(ann_file) as f:
                for line in f:
                    parts = line.strip().split(',')
                    if len(parts) < 6: continue
                    bx, by, bw, bh = map(int, parts[:4])
                    cat = int(parts[5])
                    if cat == 0 or cat == 11: continue  # ignored regions, others
                    cls = cat - 1  # 1~10 → 0~9
                    cx = (bx + bw/2) / w
                    cy = (by + bh/2) / h
                    nw = bw / w
                    nh = bh / h
                    yolo_lines.append(f"{cls} {cx:.6f} {cy:.6f} {nw:.6f} {nh:.6f}")

            with open(label_dir / ann_file.name, 'w') as f:
                f.write('\n'.join(yolo_lines))

        print(f"{split} 변환 완료!")

convert_visdrone_to_yolo('/data/datasets/visdrone')
EOF
```

**클래스 매핑 (10개):**

| ID  | 클래스          |
| --- | --------------- |
| 0   | pedestrian      |
| 1   | people          |
| 2   | bicycle         |
| 3   | car             |
| 4   | van             |
| 5   | truck           |
| 6   | tricycle        |
| 7   | awning-tricycle |
| 8   | bus             |
| 9   | motor           |

---

## 🚀 5. Step 3 — K8s Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: yolov8-visdrone-training
  namespace: ai-team
spec:
  template:
    spec:
      restartPolicy: Never
      nodeSelector:
        gpu-type: v100
      containers:
        - name: yolov8-trainer
          image: ultralytics/ultralytics:8.1.0
          command: ['/bin/bash', '-c']
          args:
            - |
              pip install pillow -q
              python /data/datasets/visdrone/convert_visdrone.py
              cat > /data/datasets/visdrone/visdrone.yaml << 'DATASET'
              path: /data/datasets/visdrone
              train: VisDrone2019-DET-train/images
              val: VisDrone2019-DET-val/images
              nc: 10
              names: [pedestrian, people, bicycle, car, van, truck, tricycle, awning-tricycle, bus, motor]
              DATASET
              yolo detect train \
                data=/data/datasets/visdrone/visdrone.yaml \
                model=yolov8n.pt \
                epochs=100 \
                imgsz=640 \
                batch=16 \
                device=0,1,2,3 \
                project=/data/yolov8/runs \
                name=visdrone_exp1
          resources:
            limits:
              nvidia.com/gpu: 4
          volumeMounts:
            - name: data-volume
              mountPath: /data
            - name: dshm
              mountPath: /dev/shm
      volumes:
        - name: data-volume
          nfs:
            server: (control-plane-public-ip)
            path: /data
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 16Gi
```

### Job 적용

```bash
kubectl apply -f /data/datasets/visdrone/visdrone-train-job.yaml
```

---

## 📊 6. 모니터링

### Pod 상태 확인

```bash
kubectl get pods -n ai-team | grep visdrone
```

### 실시간 로그

```bash
kubectl logs -f $(kubectl get pods -n ai-team | grep visdrone | grep Running | awk '{print $1}') -n ai-team
```

### Grafana GPU 대시보드

Tailscale: `http://(vpn-endpoint)3000` → DCGM Dashboard → V100 GPU 사용률 실시간 확인

---

## 🛠️ 7. 트러블슈팅

### 문제 1: CUDA no kernel image error

```text
CUDA error: no kernel image is available for execution on the device
```

**원인:** `ultralytics:latest` 이미지의 PyTorch 2.11이 V100 Compute Capability 7.0 미지원 **해결:** `ultralytics:8.1.0` 이미지로 교체 (PyTorch 2.1.0 → CC 7.0 지원)

| 이미지             | PyTorch | V100 호환 |
| ------------------ | ------- | --------- |
| ultralytics:latest | 2.11.0  | ❌        |
| ultralytics:8.1.0  | 2.1.0   | ✅        |

### 문제 2: Shared Memory 부족

```text
RuntimeError: unable to write to file </torch_...>: No space left on device
```

**원인:** K8s Pod 기본 `/dev/shm` 크기(64MB)가 DDP 멀티GPU 데이터 로딩에 부족 **해결:** YAML에 `emptyDir(Memory)` 볼륨 16Gi 추가

```yaml
- name: dshm
  emptyDir:
    medium: Memory
    sizeLimit: 16Gi
```

---

## 💡 8. 핵심 인사이트

**PyTorch 버전과 GPU Compute Capability 호환성을 반드시 확인해야 한다.** V100은 CC 7.0으로, 최신 PyTorch(2.5+)는 CC 7.5부터 지원한다. CUDA 버전이 아니라 PyTorch 빌드 타겟이 문제였다. 이미지 버전을 낮춰 해결했다.

**DDP 멀티GPU Job은 `/dev/shm` 설정이 필수다.** K8s Pod의 기본 shared memory는 64MB로, 멀티GPU DDP 학습 시 워커 프로세스 간 데이터 공유에 부족하다. `emptyDir: medium: Memory`로 충분한 shm을 할당해야 한다.

# GitHub Actions CI/CD 트리거 방식 변경

**날짜:** 2026-05-06  
**작업자:** 52K-0812

---

## 변경 내용

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 트리거 방식 | `on: push` (main 브랜치 push 시 자동 실행) | `on: workflow_dispatch` (수동 실행만) |
| 파라미터 입력 | push 시 폴백값 고정 (`model_version: v3`) | Run workflow 버튼에서 직접 입력 |
| `required` | 없음 | `true` (빈 값으로 실행 불가) |
| epochs default | 10 | 50 |
| model_version default | v9 | v8 |

---

## 변경 이유

### 1. model_version 폴백 버그 발견

기존 코드:
```yaml
VERSION=${{ github.event.inputs.model_version || 'v3' }}
```

push 트리거 시 `github.event.inputs.model_version`이 null이므로 항상 `v3`로 폴백됨. v9 학습을 의도했으나 실제로는 v3로 실행되어 NAS에 `visdrone-v9.pt`가 아닌 `visdrone-v3.pt`로 저장되는 문제 발생.

### 2. 실수 push 시 GPU 낭비 위험

push마다 V100 4장이 학습에 투입되는 구조는 실수로 push할 때마다 불필요한 GPU 자원이 소모됨.

### 3. 버전 관리 명확성

수동 트리거로 전환하면 어떤 버전을 몇 epoch으로 실행했는지 GitHub Actions 실행 이력에 명시적으로 기록됨.

---

## 현재 사용법

```
GitHub 레포 → Actions → YOLOv8 Training Pipeline
→ Run workflow → 입력값 설정 후 실행

입력값:
  epochs:        50
  batch_size:    16
  model_version: v8  (또는 v9, v10 ...)
```

---

## 관련 사고 기록

이 변경의 직접적인 계기는 2026-05-06 운영 중 발생한 아래 사고임.

- v9 학습 실행 의도 → push 트리거로 실행됨
- model_version 폴백으로 v3가 전달됨
- MLflow에 v9로 등록됐지만 실제 artifact 경로는 v3 학습 결과
- champion alias 변경 후 `/reload-champion` 호출 시 파일 없음 오류 발생
- MLflow Registry 전체 삭제 후 v7 기준으로 수동 복구 진행
- 상세 복구 과정 → docs/incidents/ 참고

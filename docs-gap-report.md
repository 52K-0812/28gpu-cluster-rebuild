# 4단계 누락 점검 보고서

> **점검 범위:** 문서 구조 자체의 누락
> **점검 항목:** README 링크 유효성 / runbooks 인덱스 / journal Phase README / 라이선스 / .gitignore
> **점검 일자:** 2026-04-29

---

## 1. README 링크 유효성

### README 이미지 링크 (7개) — 전부 유효 ✅

| README 경로 | 실제 파일 존재 |
|---|---|
| `./docs/images/20260312_204425.jpg` | ✅ |
| `./docs/images/KakaoTalk_20260413_143557870.jpg` | ✅ |
| `./docs/images/20260323_182524.jpg` | ✅ |
| `./docs/images/20260324_140843.jpg` | ✅ |
| `./docs/images/20260316_130432.jpg` | ✅ |
| `./docs/images/20260316_184714.jpg` | ✅ |
| `./docs/images/20260316_191154.jpg` | ✅ |

### README 문서 내부 링크 — 대부분 유효 ✅ / 2건 누락

| 링크 | 상태 |
|---|---|
| `docs/overview/current-architecture.md` | ✅ 링크 존재, 파일 존재 |
| `docs/overview/cluster-diagram.md` | ⚠️ **문서 구조 텍스트 트리에만 언급. 마크다운 링크 없음** |
| `docs/overview/timeline.md` | ⚠️ **문서 구조 텍스트 트리에만 언급. 마크다운 링크 없음** |
| journal/ 34개 파일 링크 | ✅ docs-inventory.md 대조, 전부 존재 확인 |
| runbooks/ 5개 파일 링크 | ✅ 전부 존재 확인 |

> **관찰:** `cluster-diagram.md`와 `timeline.md`는 docs/overview/ 폴더에 실제로 존재하는 핵심 문서이나,
> README의 "📁 작업 문서" 섹션에서 클릭 가능한 링크가 없음.
> "📁 문서 구조" 코드 블록 트리에 파일명만 텍스트로 등장함.

---

## 2. runbooks/ 폴더 인덱스

| 항목 | 상태 |
|---|---|
| `docs/runbooks/README.md` | ❌ **없음** |

**현황:** runbook 5개 파일만 있고, 폴더 내 목차/인덱스 문서 없음.

**보완 방안:** README.md의 작업 문서 섹션에 runbook 링크가 이미 포함되어 있어 외부 진입점은 있음. 다만 GitHub에서 `docs/runbooks/` 폴더를 직접 탐색할 때 README가 없어 컨텍스트 없이 파일 목록만 보임.

**처리 방향:** `docs/runbooks/README.md` 인덱스 파일 추가 검토 (선택).

---

## 3. journal/ Phase 폴더 README

| 폴더 | README.md |
|---|---|
| `docs/journal/1.cheetah-복구/` | ❌ **없음** |
| `docs/journal/2.GPU_클러스터_재구축/` | ❌ **없음** |
| `docs/journal/3.AI_학습_팀_환경_구축/` | ❌ **없음** |
| `docs/journal/4.ML_파이프라인_구축/` | ❌ **없음** |
| `docs/journal/5.서비스_노출_및_운영_안정화/` | ❌ **없음** |

**현황:** 5개 Phase 폴더 모두 README.md 없음. 루트 README.md의 "작업 문서" 섹션이 각 Phase 설명 역할을 대신하고 있음.

**처리 방향:** 각 Phase 폴더 README는 선택적. 루트 README.md가 이미 Phase별 파일 목록과 설명을 포함하고 있으므로 현재 구조로도 충분히 탐색 가능.

---

## 4. 라이선스 파일

| 항목 | 상태 |
|---|---|
| `LICENSE` | ❌ **없음** |
| `LICENSE.md` | ❌ **없음** |

**현황:** 공개 레포이나 라이선스 파일 없음.

**처리 방향:** 포트폴리오 목적 레포이므로 저작권 보호를 위해 라이선스 지정이 권장됨. 단, 필수 아님. 지정 시 `CC BY-NC-SA 4.0` (문서 공개, 상업 이용 금지) 또는 `MIT` (코드 위주) 검토.

---

## 5. .gitignore 검토

**현재 .gitignore 내용:**

```
.obsidian/
.gitignore
.claude
nano yolov8-serving-code.live.yaml
```

| 항목 | 평가 |
|---|---|
| `.obsidian/` | ✅ 적절. Obsidian 메타데이터 폴더 제외 |
| `.claude` | ✅ 적절. Claude Code 작업 디렉터리 제외 |
| `.gitignore` | ⚠️ **이상. `.gitignore` 자체를 exclude 목록에 넣으면 git이 추적하지 않게 됨. 의도 불명확.** |
| `nano yolov8-serving-code.live.yaml` | ⚠️ **이상. `nano` 에디터로 파일을 편집하다 실수로 파일명이 추가된 것으로 보임. 현재는 `nano yolov8-serving-code.live.yaml`이라는 이름의 파일을 무시하는 패턴으로 해석됨.** |

**누락 항목 (선택적 추가):**

| 패턴 | 설명 |
|---|---|
| `*.bak` | 백업 파일 제외 (docs-inventory.md에서 "`.bak` 제외" 기준이나 gitignore에 미기재) |
| `.DS_Store` | macOS 부산물 |
| `Thumbs.db` | Windows 부산물 |
| `__pycache__/` | Python 캐시 (임시 스크립트 사용 시) |

---

## 누락 항목 우선순위 요약

| 우선순위 | ID | 항목 | 파일 / 위치 | 처리 방향 |
|---|---|---|---|---|
| 🟡 검토 | G-1 | `cluster-diagram.md` 링크 누락 | `Readme.md` 작업 문서 섹션 | 링크 추가 검토 |
| 🟡 검토 | G-2 | `timeline.md` 링크 누락 | `Readme.md` 작업 문서 섹션 | 링크 추가 검토 |
| 🟡 검토 | G-3 | `.gitignore` 이상 항목 2개 | `.gitignore` | `.gitignore` 자기 제외 행 + `nano ...` 행 제거 |
| 🟢 선택 | G-4 | `runbooks/README.md` 없음 | `docs/runbooks/` | 인덱스 추가 검토 |
| 🟢 선택 | G-5 | journal Phase README 없음 | `docs/journal/*/` | Phase별 README 추가 검토 |
| 🟢 선택 | G-6 | 라이선스 파일 없음 | 루트 | LICENSE 추가 검토 |
| 🟢 선택 | G-7 | `.gitignore` 누락 패턴 | `.gitignore` | `*.bak`, `.DS_Store` 등 추가 검토 |

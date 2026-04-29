# 3단계 정합성 점검 보고서

> **기준 문서:** `docs/overview/current-architecture.md` (2026-04-28 기준)
> **대조 대상:** `journal/`, `runbooks/`, `incidents/` 전체
> **점검 일자:** 2026-04-29
> **점검 범위:** 노드 역할/구성 일치, 구조 변경 반영 누락, runbook 절차 일치, incidents 재발 방지책 반영

---

## ⛔ CRITICAL — 즉시 수정 필요

### C-1. `current-architecture.md` L20 — IP 마스킹 미처리 및 플레이스홀더 오류

| 위치 | 현재 값 | 올바른 값 | 종류 |
|---|---|---|---|
| L20, NAS 행 IP 열 | `` `<MASTER-IP>` (1G) / `10.10.10.157` (10G) `` | `` `<NAS-1G-IP>` (1G) / `<NAS-10G-IP>` (10G) `` | IP 노출 + 잘못된 플레이스홀더 |

- `10.10.10.157` — 실제 NAS 10G IP. 공개 레포에 노출됨. `cluster-diagram.md`에서는 이미 수정했으나 `current-architecture.md`는 미처리.
- `<MASTER-IP>` — NAS의 1G IP 위치에 master-01 플레이스홀더가 잘못 사용됨. `<NAS-1G-IP>`가 맞음.
- **→ 수정 완료 (2026-04-29)**

---

## ⚠️ 보안 관찰 — 저널 파일 내 실제 IP 노출

### S-1. `docs/journal/5.서비스_노출_및_운영_안정화/4_27_JupyterHub_GitHub_OAuth_전환.md`

- 파일 내 `112-76-56-162` (LB-INGRESS-IP 대역)가 플레이스홀더 없이 다수 등장 (L9, L87, L88, L139, L140 등)
- 예: `https://hub.112-76-56-162.nip.io/hub/oauth_callback`
- 정합성 이슈는 아니나, 공개 레포에서 캠퍼스 내부 서비스 접속 IP가 노출됨.
- **처리 방향:** `112-76-56-162` → `<LB-INGRESS-IP-DASHED>` 치환 검토 필요 (승인 후 수정)

---

## ✅ 정합 확인 — 일치 항목 (7개)

### A-1. 2080ti-gpu-04 역할 표기

| 문서 | 내용 |
|---|---|
| `current-architecture.md` | `2080ti-gpu-04 \| Worker` + `nodeSelector: gpu-type=2080ti` (hostname 고정 없음) + 변경이력 "hostname nodeSelector 고정 해제 (2026-04-17)" |
| `4_16_YOLOv8_서빙_이미지화.md` | 로컬 이미지 시절 hostname=2080ti-gpu-04 고정 — **과거 상태**, 현재 운영 구성 아님 |
| `4_17_YOLOv8_서빙_이미지_DockerHub_등록.md` | hostname nodeSelector 제거 완료. `{"gpu-type":"2080ti"}` 최종 확인 |

**결론:** "서빙 전용" 고정 흔적 없음. current-architecture.md와 완전 일치. ✅

---

### A-2. JupyterHub 인증 방식 (DummyAuth 제거)

| 문서 | 내용 |
|---|---|
| `current-architecture.md` Section 5 | `~~DummyAuthenticator~~` 취소선, "2026-04-27 전환 (DummyAuthenticator 제거)" 명시 |
| `4_27_JupyterHub_GitHub_OAuth_전환.md` | full values 방식으로 DummyAuth 완전 제거 + dry-run에서 DummyAuthenticator 미출현 검증 |

**결론:** DummyAuth 흔적 없음. ✅

---

### A-3. Ingress + TLS 도입 후 NodePort 직접 노출 안내

| 문서 | 내용 |
|---|---|
| `runbook_model_serving.md` | 기본 접속 `https://serving.<LB-INGRESS-IP>.nip.io` (Ingress+TLS). NodePort `30600`은 "fallback: Ingress 장애 시"로 명시. Chrome flags 섹션에 "근본 해결: Ingress + TLS 완료. 기본 접속 주소 사용 시 불필요" 주석 포함 |
| `runbook_argo_dag.md` | Argo UI `http://TAILSCALE-IP:30500` — Ingress 대상 아님. current-architecture.md "관리자 서비스 NodePort" 분류와 일치 |

**결론:** NodePort가 구식 메인 접속 방법으로 남아 있는 부분 없음. ✅

---

### A-4. PriorityClass / LimitRange / ResourceQuota 커버리지

| 문서 | 내용 |
|---|---|
| `current-architecture.md` | Section 12 (PriorityClass 4계층), Section 13 (LimitRange ai-team), Section 14 (ResourceQuota ai-team) 모두 존재 |
| `runbook_model_serving.md` | 배포 매니페스트에 `priorityClassName: serving-critical` 포함 |

**결론:** 도입 사실 누락 없음. LimitRange/ResourceQuota는 네임스페이스 수준 설정으로 runbooks에 별도 기재 불필요(자동 주입). ✅

---

### A-5. INC-2026-04-28 Grafana chart 업그레이드 재발 방지책

| 문서 | 내용 |
|---|---|
| `current-architecture.md` Section 11 | `chart 82.15.1 고정` 운영 주의사항 명시 |
| `current-architecture.md` Section 16 | `Grafana datasource 이중 정의 잠재 위험` 미해결 과제로 기재 |
| `incidents/4_28_grafana_chart_upgrade.md` | 재발 방지 절차 (helm --version 고정, values 명시, 백업) 기록 |

**결론:** current-architecture.md에 반영됨. ✅

---

### A-6. journal 구조 변경 → current-architecture.md 반영 완료 목록

| journal 작업 | 반영 위치 | 상태 |
|---|---|---|
| 4_27 Ingress + TLS 완료 | Section 6 | ✅ |
| 4_27 GitHub OAuth 전환 | Section 5 | ✅ |
| 4_17 DockerHub 전환 + hostname nodeSelector 해제 | Section 9 + 변경이력 | ✅ |
| 4_28 PriorityClass Phase A~C | Section 12 | ✅ |
| 4_28 LimitRange (Phase D) | Section 13 | ✅ |
| 4_28 ResourceQuota (Phase E) | Section 14 | ✅ |
| 3_31 SoC 설계 (master-01 NoSchedule taint) | Section 1 설계 근거 | ✅ |

---

## ⬜ 마이너 관찰 — 수정 선택적 (승인 후 처리)

### M-1. `current-architecture.md` L43~L50 — 데이터망 범위 표기

```
[데이터망 — 10GbE, 10.10.10.x]
    GPU 노드(153~156) ↔ NAS(157)
```

- 특정 단일 IP가 아닌 대역 표기이나, `cluster-diagram.md`에서는 동일 내용을 플레이스홀더로 처리함.
- 처리 방향: `10.10.10.x` → `<DATA-NET-RANGE>`, `153~156` → `<GPU-NODE-10G-RANGE>` 검토.

### M-2. `runbook_model_serving.md` — LimitRange 기본값 주입 미명시

- `yolov8-serving` 배포 매니페스트에 CPU/Memory requests/limits 미선언 (GPU limit만 있음).
- ai-team LimitRange 자동 주입되므로 기능 문제 없음. 다만 매니페스트만 보면 리소스 제한이 없어 보임.
- 처리 방향: 배포 매니페스트에 LimitRange 기본값 주석 추가 (선택).

### M-3. monitoring-values.yaml NodePort/PVC 명시 권고 미절차화

- `INC-2026-04-28` Section 6-2에서 "kubectl patch 수정값을 values 파일에도 명시하라" 권고.
- 실제 values 파일 수정 절차를 담은 runbook 없음. current-architecture.md 미해결 과제에도 미기재.
- 처리 방향: `current-architecture.md` Section 16에 "monitoring-values.yaml NodePort/PVC 미반영" 행 추가 (선택).

---

## 수정 우선순위 요약

| 우선순위 | ID | 파일 | 작업 | 상태 |
|---|---|---|---|---|
| 🔴 즉시 | C-1 | `current-architecture.md` L20 | `<MASTER-IP>` → `<NAS-1G-IP>`, `10.10.10.157` → `<NAS-10G-IP>` | **완료** |
| 🟡 검토 | S-1 | `4_27_JupyterHub_GitHub_OAuth_전환.md` | `112-76-56-162` → `<LB-INGRESS-IP-DASHED>` | 승인 대기 |
| 🟢 선택 | M-1 | `current-architecture.md` L43~L50 | `10.10.10.x` 계열 → 플레이스홀더 | 승인 대기 |
| 🟢 선택 | M-2 | `runbook_model_serving.md` | LimitRange 기본값 주석 추가 | 승인 대기 |
| 🟢 선택 | M-3 | `current-architecture.md` Section 16 | monitoring-values.yaml 미결 과제 행 추가 | 승인 대기 |

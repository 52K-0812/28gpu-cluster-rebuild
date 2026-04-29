# 문서 인벤토리

> 생성일: 2026-04-29
> 기준: git log 마지막 커밋일 / 라인 수 wc -l 기준

---

## 전체 통계

| 구분 | 파일 수 | 비고 |
|---|---|---|
| **합계** | **58** | .bak 제외 |
| Readme.md | 1 | 루트 |
| docs/overview/ | 3 | |
| docs/journal/ | 35 | 5개 Phase 폴더 |
| docs/incidents/ | 14 | |
| docs/runbooks/ | 5 | |

---

## journal/ 폴더별 파일 수

| Phase 폴더 | 파일 수 |
|---|---|
| 1.cheetah-복구/ | 9 |
| 2.GPU_클러스터_재구축/ | 8 |
| 3.AI_학습_팀_환경_구축/ | 3 |
| 4.ML_파이프라인_구축/ | 11 |
| 5.서비스_노출_및_운영_안정화/ | 4 |

---

## 파일 전체 목록

| 파일 경로 | 라인 수 | 마지막 수정 (git) |
|---|---|---|
| Readme.md | 502 | 2026-04-29 |
| docs/overview/cluster-diagram.md | 246 | 2026-04-29 |
| docs/overview/current-architecture.md | 389 | 2026-04-28 |
| docs/overview/timeline.md | 219 | 2026-04-28 |
| **docs/runbooks/** | | |
| runbook_argo_dag.md | 400 | 2026-04-28 |
| runbook_etcd_restore.md | 227 | 2026-04-28 |
| runbook_ingress_tls.md | 481 | 2026-04-28 |
| runbook_letsencrypt_dns01.md | 509 | 2026-04-28 |
| runbook_model_serving.md | 311 | 2026-04-28 |
| **docs/journal/1.cheetah-복구/** | | |
| 3_12_AI_클러스터_복구_1.md | 84 | 2026-04-14 |
| 3_12_AI_클러스터_복구_2.md | 100 | 2026-04-14 |
| 3_12_AI_클러스터_복구_3.md | 101 | 2026-04-14 |
| 3_12_AI_클러스터_복구_4.md | 94 | 2026-04-14 |
| 3_12_AI_클러스터_복구_5.md | 83 | 2026-04-14 |
| 3_16_NAS_복구_1.md | 129 | 2026-04-14 |
| 3_16_NAS_복구_2.md | 144 | 2026-04-14 |
| 3_17_HPE_ACPI_장애.md | 107 | 2026-04-14 |
| 3_19_GPU_드라이버_업데이트.md | 372 | 2026-04-14 |
| **docs/journal/2.GPU_클러스터_재구축/** | | |
| 0_시스템_재구축_개요.md | 82 | 2026-04-14 |
| 1_백업.md | 67 | 2026-04-14 |
| 2_우분투_OS_재설치.md | 122 | 2026-04-14 |
| 3_K8s_엔진_설치.md | 115 | 2026-04-14 |
| 4_K8s_클러스터_초기화.md | 91 | 2026-04-14 |
| 5_Calico_CNI_설치.md | 65 | 2026-04-14 |
| 6_워커_노드_구성_및_클러스터_합류.md | 148 | 2026-04-14 |
| 7_GPU_클러스터_완전_구축_가이드.md | 790 | 2026-04-14 |
| **docs/journal/3.AI_학습_팀_환경_구축/** | | |
| 3_30_RBAC_Namespace_JupyterHub_구축.md | 341 | 2026-04-14 |
| 4_01_Grafana_ServiceAccount_분리_및_보안_강화.md | 365 | 2026-04-14 |
| 4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정.md | 333 | 2026-04-14 |
| **docs/journal/4.ML_파이프라인_구축/** | | |
| 4_02_YOLOv8_COCO_학습_및_웹캠_추론_테스트.md | 177 | 2026-04-14 |
| 4_03_YOLOv8_VisDrone_멀티GPU_학습_Job.md | 255 | 2026-04-14 |
| 4_04_YOLOv8_VisDrone_학습_결과_보고서.md | 121 | 2026-04-14 |
| 4_06_Argo_Workflows_설치_및_WorkflowTemplate_구성.md | 421 | 2026-04-14 |
| 4_07_Alertmanager_이메일_알람_구성.md | 248 | 2026-04-14 |
| 4_09_Argo_Workflows_Tailscale_접속_및_포트_변경.md | 175 | 2026-04-16 |
| 4_13_GitHub_Actions_CICD.md | 281 | 2026-04-14 |
| 4_13_MLflow_설치_및_Argo_DAG_연동.md | 424 | 2026-04-14 |
| 4_15_MLflow_alias_FastAPI_엔드포인트_분리.md | 313 | 2026-04-27 |
| 4_16_YOLOv8_서빙_이미지화.md | 234 | 2026-04-27 |
| 4_17_YOLOv8_서빙_이미지_DockerHub_등록.md | 194 | 2026-04-27 |
| **docs/journal/5.서비스_노출_및_운영_안정화/** | | |
| 4_27_Ingress_TLS_도입_및_host_기반_라우팅.md | 516 | 2026-04-28 |
| 4_27_JupyterHub_GitHub_OAuth_전환.md | 307 | 2026-04-27 |
| 4_28_PriorityClass_ResourceQuota_Phase_A_B_B1.md | 351 | 2026-04-28 |
| 4_28_PriorityClass_ResourceQuota_Phase_C_D_E.md | 424 | 2026-04-28 |
| **docs/incidents/** | | |
| 3_17_HPE_ACPI_장애.md | 113 | 2026-04-14 |
| 3_19_GPU_드라이버_업데이트.md | 400 | 2026-04-14 |
| 3_23_10G_NIC_이전.md | 259 | 2026-04-14 |
| 3_23_2080Ti_GPU_미인식.md | 154 | 2026-04-14 |
| 3_27_Grafana_DCGM_No_data.md | 171 | 2026-04-14 |
| 3_27_NAS_NIC_접촉불량.md | 235 | 2026-04-14 |
| 3_27_Portainer_Tailscale_접속불가.md | 97 | 2026-04-14 |
| 3_31_네트워크_장애_및_클러스터_설계_개선.md | 342 | 2026-04-14 |
| 4_01_Grafana_대시보드_미표시_및_RBAC_권한_문제.md | 287 | 2026-04-14 |
| 4_01_JupyterHub_다중_접속_장애_및_서비스_설계_개선.md | 380 | 2026-04-14 |
| 4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정.md | 333 | 2026-04-14 |
| 4_09_Alertmanager_Silence_DaemonSet_알람_억제.md | 105 | 2026-04-14 |
| 4_16_incidents_이미지화_장애_3건.md | 152 | 2026-04-27 |
| 4_28_grafana_chart_upgrade.md | 207 | 2026-04-28 |

---

## 특이사항 (육안 확인)

- `docs/incidents/4_02_GPU_노드_네트워크_최적화_10GbE_라우팅_설정.md` — 파일명이 incidents/에 있으나 내용은 journal/3번 폴더와 동일한 작업 (journal과 incidents에 같은 내용 중복 수록)
- `docs/journal/1.cheetah-복구/3_17_HPE_ACPI_장애.md` / `docs/incidents/3_17_HPE_ACPI_장애.md` — 동일 파일명 양쪽 존재 (내용 일치 여부 미확인)
- `docs/journal/1.cheetah-복구/3_19_GPU_드라이버_업데이트.md` / `docs/incidents/3_19_GPU_드라이버_업데이트.md` — 동일 파일명 양쪽 존재
- `4_13_GitHub_Actions_CICD.md` / `4_13_MLflow_설치_및_Argo_DAG_연동.md` — 날짜(4_13) 중복, 순서 구분 번호 없음

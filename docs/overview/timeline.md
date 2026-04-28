# 🗺️ 작업 타임라인 (상세)

> 전체 작업 흐름을 날짜 단위로 기록한 문서. README 타임라인의 풀버전입니다.
> 각 항목의 근거는 `docs/journal/`, `docs/incidents/`의 개별 문서를 참고하세요.

---

```
━━━ Phase 1 : 레거시 복구 (3/12 ~ 3/19) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━

3월 12일  ──  구형 CHEETAH 시스템 복구 시도
              └ GRUB 단일 사용자 모드로 패스워드 재설정 (인증 정보 전량 분실)
              └ K8s 인증서 5년치 만료 → 갱신 및 워커노드 재연결
              └ ES / Kafka / MetalLB 장애 복구 (8시간 소요)

3월 16일  ──  NAS RAID 복구
              └ 0번 SSD 빨간불 감지 → MegaRAID BIOS 직접 진입
              └ RAID 1 미러링 재구성 + RAID 5 27.3TB 재생성

3월 17일  ──  HPE ACPI 로그 스팸 장애 해결
              └ GRUB 커널 파라미터 튜닝으로 영구 해결

3월 19일  ──  GPU 드라이버 업그레이드 시도 → 재구축 결정 ⭐ 핵심 판단
              apt 방식 실패(의존성 지옥) → Runfile로 드라이버 418.87 → 535 성공
              toolkit 설치 성공, Docker GPU 인식 성공
              K8s Device Plugin nvidia.com/gpu: 0 → 3가지 경로(DaemonSet/수동 바이너리/토폴로지 매니페스트) 모두 실패
              Ubuntu 18.04 라이브러리 경로 체계 구조적 문제 → 이후에도 반복될 것으로 판단
              → 클러스터 전면 재구축 결정

━━━ Phase 2 : 클러스터 재구축 (3/20 ~ 3/27) ━━━━━━━━━━━━━━━━━━━━━━━━━━

3월 20일  ──  데이터 백업 및 재구축 준비
              └ 기존 CHEETAH 시스템 전체 데이터 NAS로 tar 아카이빙
              └ ubuntu 홈 디렉토리 권한/심볼릭링크 보존 백업

3월 20~21일  ──  Ubuntu 22.04 클린 설치 + K8s 클러스터 초기화
              └ Containerd + kubeadm v1.29 + Calico CNI
              └ 워커노드 6대 Join

3월 23일  ──  10G 네트워크 병목 발견 및 해결 ⭐
              └ NAS 네트워크 조회 중 10G NIC 부재 발견
              └ master-02 유휴 10G NIC → NAS로 직접 이전 (PCIe 카드 물리 이전)
              └ GPU 노드 ↔ NAS 전송속도 1G → 10G (약 10배)
              └ 2080ti-gpu-03 GPU 인식 상태 조회 → 8장 중 7장만 감지 확인

3월 24일  ──  GPU 미인식 물리 점검 → 27장 확정 → 서비스 구축 시작
              └ 서버 커버 오픈, 8핀 전원 케이블 위치 변경 시도 → 변화 없음
              └ PCIe 슬롯 교체까지 필요 → 일정 우선으로 스킵
              └ "27장으로 먼저 완성" 판단 → 즉시 서비스 구축 돌입

3월 24~27일  ──  GPU Operator · NFS Provisioner · 모니터링 · 외부접속 완성
              └ NVIDIA GPU Operator (V100 + 2080Ti 혼합 환경 자동 관리)
              └ NFS Provisioner, Prometheus + Grafana, MetalLB, Tailscale

3월 27일  ──  운영 중 장애 3건 연속 발생 및 해결
              └ NAS 10G NIC (BCM57810) PCIe 접촉불량 → rev ff 진단 → 물리 재삽입 → 9.41Gbps 복구
              └ Grafana DCGM Dashboard No data → ServiceMonitor 라벨 불일치 → json replace 패치
              └ Portainer Tailscale 접속 불가 → port-forward 좀비 상태 → 서비스 재시작

━━━ Phase 3 : 팀 환경 구축 및 서비스 안정화 (3/30 ~ 4/2) ━━━━━━━━━━━━━━

3월 30일  ──  AI 팀 환경 구축 (RBAC + JupyterHub + GPU 검증)
              └ ai-team 네임스페이스 + RBAC ((팀이름)-01~05) 구성
              └ JupyterHub Helm 배포 (chart 4.3.3 / JupyterHub 5.4.4)
              └ 이미지 호환성 문제 해결 → cschranz/gpu-jupyter:v1.6_cuda-12.0_ubuntu-22.04 채택
              └ Tailscale port-forward systemd 서비스 등록 → http://TAILSCALE-IP:8000
              └ K8s 스케줄러 GPU 자동 배정 확인 (V100 / 2080Ti 분산 배치)
              └ TensorFlow / PyTorch CUDA 인식 완료
              └ MNIST 소프트맥스 회귀 학습 성공 (GPU 연산 동작 검증)

4월 1일   ──  JupyterHub 다중 접속 장애 → 서비스 구조 전면 재설계 ⭐
              └ 팀원 동시 접속 시 세션 끊김 원인 규명
              └ kubectl port-forward 남용 → 1인용 터널의 한계 확인
              └ MetalLB IP Pool에 master-01 IP(151) 포함 → ARP 충돌 → speaker 89회 재시작
              └ Calico CrashLoopBackOff 연쇄 장애 복구
              └ 구 클러스터 허용 IP(MASTER-IP) MetalLB Pool 재활용
              └ 서비스 구조 재설계: 학과생용(158, 포트 없음) / 관리자용(151:303xx) 분리
              └ JupyterHub http://MASTER-IP 공개 접속 완성
              └ Grafana 전용 ServiceAccount(grafana-sa) 생성 → 최소 권한 원칙 적용

4월 2일   ──  10GbE 라우팅 최적화 — ML 학습을 위한 인프라 정비
              └ GPU 노드 기본 라우트가 1GbE → DCGM 스크래핑 타임아웃 근본 원인 확인
              └ 전 GPU 노드 192.168.0.0/16 경로를 10GbE로 우선 라우팅 (metric 50)
              └ v100-gpu-01 물리 케이블 Cat5 → Cat5e/6 교체 → 1000Mbps 복구

━━━ Phase 4 : ML 파이프라인 및 서빙 구축 (4/2 ~ 4/17) ━━━━━━━━━━━━━━━━━━━━━━━━━━

4월 2일   ──  YOLOv8 COCO 학습 파이프라인 첫 가동
              └ YOLOv8 COCO 학습 K8s Job 제출 → mAP@0.5 46.8% 달성
              └ 로컬 PC 웹캠 실시간 추론 테스트 성공

4월 3일   ──  VisDrone 데이터셋 구축 + V100 멀티GPU DDP 학습 시작
              └ VisDrone2019 데이터셋 NAS 다운로드 및 YOLO 포맷 변환
              └ V100 4장 멀티GPU DDP 학습 K8s Job 가동
              └ PyTorch CC 7.0 호환성 문제 → ultralytics:8.1.0 이미지로 교체
              └ /dev/shm 부족 문제 → emptyDir Memory 16Gi 추가

4월 4일   ──  VisDrone 학습 결과 분석 및 검증
              └ V100 4장 DDP 학습 완료 (100 epochs, 1.517시간)
              └ mAP@0.5 33.4% 달성 (car 75.4% 최고 / bicycle 7.2% 최저)
              └ Grafana DCGM 대시보드로 GPU 사용률 실시간 확인
              └ 로컬 PC 웹캠 추론 테스트 성공
              └ 소형 객체 한계 분석 → imgsz=1280 / YOLOv8s 개선 방향 도출

4월 6일   ──  Argo Workflows 도입 → MLOps 파이프라인 1단계 완성 ⭐
              └ Argo Workflows v4 Helm 설치 (install.yaml CRD 누락 문제 → Helm으로 해결)
              └ MetalLB IP Pool에 MASTER-IP 추가 → Argo UI 외부 노출
              └ ai-team 네임스페이스 연동 (controller.workflowNamespaces 설정)
              └ NAS /data/datasets 직접 마운트 Static PV/PVC 구성 (nfs-datasets-pvc)
              └ YOLOv8 VisDrone WorkflowTemplate 작성 (epochs/batch/gpu-type 파라미터화)
              └ 팀원이 SSH 없이 Argo UI로 학습 Job 제출 가능

4월 7일   ──  Alertmanager 이메일 알람 구성 + Argo DAG 파이프라인 ⭐
              └ Gmail SMTP 연동 (앱 비밀번호 발급, TLS 설정)
              └ GPU 특화 PrometheusRule 3개 등록
              └   · GPUHighTemperature: 온도 >85°C → critical (2분)
              └   · GPUMemoryHigh: 메모리 >95% → warning (5분)
              └   · GPUNodeDown: DCGM Exporter 응답 없음 → critical (1분)
              └ 실제 클러스터 알람 이메일 수신 확인 → 모니터링 스택 완성
              └ Argo DAG 4단계 파이프라인 구성 (validate-data → train → evaluate → save-model)
              └ 단계별 실패 시 파이프라인 중단 · UI에서 단계별 상태 시각화
              └ visdrone-v1.pt / visdrone-v2.pt NAS 버전별 저장 확인

4월 9일   ──  Alertmanager 오탐 억제 + Argo Tailscale 접속 구성
              └ continuous-image-puller DaemonSet 오탐 분석
              └   · master-01 NoSchedule taint로 인한 의도된 상태 → Silence 1년 등록
              └ Argo UI 포트 변경 (2746 → 30500) + Helm upgrade
              └ systemd port-forward 서비스로 Tailscale 접속 구성
              └ http://TAILSCALE-IP:30500 외부 접속 완료

4월 13일  ──  마무리 스프린트: 운영 기반 3종 동시 롤아웃 ⭐
              (etcd DR · MLflow 실험추적 · GitHub Actions CI/CD)
              ※ 각 컴포넌트 설계·매니페스트를 사전 준비한 상태에서 일괄 배포
              └ [etcd 백업] distroless 컨테이너 대응 → /proc/{pid}/root 스냅샷 복사
              └ [etcd 백업] 호스트 crontab 매일 02:00 자동 백업 · NAS 7일 보존 · 74MB DR 검증 ✅
              └ [MLflow] PostgreSQL 백엔드 K8s 배포 + Argo DAG 연동
              └ [MLflow] ultralytics autolog → params 105개 + mAP50/5095 자동 기록
              └ [MLflow] K8s 내부 DNS(cluster.local) 활용 → NodePort 의존성 제거
              └ [GitHub Actions] 프라이빗 레포(28gpu-cluster-cicd) + Self-hosted Runner 구성
              └ [GitHub Actions] git push → 16초 내 Argo cicd-pipeline 자동 트리거 확인
              └ [GitHub Actions] workflow_dispatch로 epochs/batch/version 수동 파라미터 입력 지원

4월 15일  ──  MLflow alias + FastAPI 운영 서빙 연결 + 웹 UI 구축 ⭐
              └ [MLflow] Model Registry alias "champion" 기반 버전 관리 적용
              └ [Argo DAG] evaluate 단계에 best.pt artifact 연동 로직 추가
              └ [FastAPI] /predict-demo(COCO) / /predict(champion) 엔드포인트 분리
              └ [FastAPI] NAS 저장 모델 파일 우선 로드 + artifact 경로 폴백 구조 적용
              └ [FastAPI] /reload-champion 으로 재시작 없는 수동 반영 가능
              └ [UI] 루트(/) 페이지에 파일 업로드 · 웹캠 캡처 · 결과 시각화 웹 UI 추가
              └ [검증] champion v4 NAS 로드 확인 · /health 정상 응답 · 웹 UI 추론 성공

4월 16일  ──  서빙 이미지화 — pip install 런타임 제거 ⭐
              └ nerdctl v1.7.6 + buildkit v0.14.1 master-01에 구성
              └ Dockerfile 작성: ultralytics:8.1.0 베이스 + fastapi/mlflow pip install 내장
              └ yolov8-serving:local-v1 빌드 (13.7 GiB)
              └ tar export → scp → 2080ti-gpu-04 nerdctl --namespace k8s.io load
              └ Deployment 교체: command/args 제거 + ConfigMap /app 마운트 제거 + hostname 고정
              └ 장애 3건 해소: containerd namespace 격리 · ConfigMap read-only 충돌 · nodeSelector 범위 불일치
              └ /health · /predict-demo · /predict 전 엔드포인트 정상 확인

4월 17일  ──  DockerHub 등록 + nodeSelector 고정 해제 ⭐
              └ nerdctl login https://index.docker.io/v1 + config.json auths 키 교정
              └ 1jkim/yolov8-serving:v1 DockerHub push 완료 (203s, 13.7 GiB)
              └ Deployment 이미지 교체: yolov8-serving:local-v1 → 1jkim/yolov8-serving:v1
              └ hostname nodeSelector 제거 → gpu-type: 2080ti 단독 유지
              └ 2080Ti 풀(02~04) 전체에서 자동 스케줄 가능한 구조로 전환
              └ 검증: Pod Running · 1jkim/yolov8-serving:v1 · champion_ready=true · v5 정상

━━━ Phase 5 : 서비스 노출 및 운영 안정화 (4/27 ~ ) ━━━━━━━━━━━━━━━━━━━━━━━━━━

4월 27일  ──  Ingress + TLS 도입 — 외부 HTTPS 진입점 일원화 ⭐
              └ NGINX Ingress Controller 설치 (master-02 고정 배치)
              └ cert-manager 설치 + cluster-ca self-signed ClusterIssuer 구성
              └ MetalLB Pool에 Ingress IP 추가 (신규 LB IP 할당)
              └ host 기반 라우팅 전환 (catch-all → nip.io wildcard DNS)
              └ serving.{IP}.nip.io / hub.{IP}.nip.io Certificate (DNS SAN 포함) 발급
              └ JupyterHub Ingress (WebSocket annotation 포함) · YOLOv8 Serving Ingress 생성
              └ Let's Encrypt HTTP-01 캠퍼스 환경 제약으로 중단 → self-signed 유지 결정
              └ HTTPS 적용으로 웹캠 getUserMedia() 제약 해소
              └ 검증: hub · serving 양쪽 HTTPS 302 + JupyterHub Notebook 셀 실행까지 통과

4월 27일  ──  JupyterHub GitHub OAuth 전환 — DummyAuthenticator 완전 제거 ⭐
              └ HTTPS 선결 조건 충족 확인 후 즉시 OAuth 전환 진행
              └ GitHub OAuth App 생성 (callback: https://hub.{IP}.nip.io/hub/oauth_callback)
              └ K8s Secret `jupyterhub-github-oauth` 등록 (Client ID/Secret 환경변수 주입)
              └ full values 방식으로 DummyAuthenticator 완전 제거 + GitHubOAuthenticator 적용
              └ 허용 사용자: 52K-0812, Yeeeho, Da-Woon-J / 관리자: 52K-0812
              └ 팀원 로그인 검증 · 허용 외 계정 403 차단 확인
              └ DummyAuth PVC(claim-member-01~05) 및 Hub DB 레코드 정리 완료

4월 28일  ──  PriorityClass 4계층 도입 — 자원 경합 대비 파드 보호 체계 확립 ⭐
              (Phase A: 정의 → Phase B: 시스템 파드 → Phase B-1: MLflow 보강 → Phase C: 서빙)
              └ serving-critical(1000000) / training-normal(100) PriorityClass 신규 정의
              └ cert-manager / argo-workflows / ingress-nginx → system-cluster-critical Helm upgrade
              └ jupyterhub hub/proxy/user-scheduler → extraPodSpec escape hatch로 적용 (schema 불일치 우회)
              └ monitoring → Helm upgrade 중 chart 82.15.1 → 84.1.2 의도치 않은 업그레이드 발생 (INC-2026-04-28)
              └   · Grafana CrashLoopBackOff, NodePort 변경, PVC 미마운트 → rollback + kubectl patch로 복구
              └   · 이후 monitoring은 kubectl patch로 개별 적용 (chart 재렌더링 없이)
              └ mlflow-postgres / mlflow-server → kubectl patch (Helm 비관리 구조)
              └ yolov8-serving → serving-critical 적용 완료 (Phase C)
              └ 재발 방지: monitoring Helm upgrade 시 --version 82.15.1 고정 의무화
              └ 검증: 전 컴포넌트 PriorityClass 적용 확인 · champion_ready=true · JupyterHub HTTPS 정상
```

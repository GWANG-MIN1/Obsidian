
# Reliability Labs

서비스를 안정적으로 운영하는 방법(SRE) 학습 실습 및 명령어 정리 저장소

인프라를 만들고(Terraform), 배포하고(GitOps), 관측하고(Prometheus), 지키고(Kyverno), 비용을 재고(FinOps) 나면
마지막 질문이 남는다 — **"고장 났을 때 어떻게 할 것인가."**

신뢰성은 장애를 없애는 일이 아니다. 분산 시스템에서 **부분 고장은 상시 발생한다.**
목표는 **고장이 사용자에게 도달하지 않게 만들고, 도달했을 때 빨리 회복하는 것**이다.

## 구조
- `commands.md` - PromQL(SLO·번레이트)·kubectl 진단·Velero·부하 테스트 레퍼런스
- `01-reliability-basics/` - 가용성 vs 신뢰성, 실패는 정상, MTBF·MTTR, 안전 여유, 복잡계
- `02-slo-operations/` - SLI 선정, SLO 설정, 에러 버짓 정책, 번레이트 알림
- `03-incident-response/` - 온콜, 인시던트 커맨더, 심각도 등급, 완화 우선, 커뮤니케이션
- `04-postmortem/` - 비난 없는 회고, 타임라인, 기여 요인 분석, 액션 아이템 추적
- `05-resilience-patterns/` - 타임아웃·재시도·백오프, 서킷 브레이커, 벌크헤드, 우아한 성능 저하
- `06-kubernetes-resilience/` - probe 정확도, PDB, 토폴로지 분산, 우선순위·선점, 노드 장애
- `07-backup-dr/` - RPO·RTO, Velero, etcd·DB 백업, 복구 리허설, 멀티 리전
- `08-capacity-planning/` - 수요 예측, 오토스케일링 계층, 부하 테스트, 헤드룸
- `09-chaos-engineering/` - 가설 기반 실험, 폭발 반경 제한, 게임데이, 도구
- `10-reliability-operations/` - 온콜 로테이션, 런북, 변경 관리, 신뢰성 리뷰


# Cost Labs

클라우드·Kubernetes 비용을 보이게 만들고 줄이는 학습 실습 및 명령어 정리 저장소

인프라를 만들고(Terraform), 배포하고(GitOps), 관측하고(Prometheus), 지키는(Kyverno) 다음에 오는 질문은 **"이게 얼마인가"** 다.
클라우드에서 비용은 재무 부서가 아니라 **아키텍처를 정하는 사람이 만든다** — 인스턴스 타입 한 줄, NAT 개수 한 줄이 곧 청구서다.

## 구조
- `commands.md` - AWS CLI(ce·ec2·s3)·kubectl 리소스·OpenCost 레퍼런스
- `01-finops-basics/` - 클라우드 비용 모델, FinOps 3단계, 단위 경제성, 비용은 누구 책임인가
- `02-cost-visibility/` - 태그 전략, 비용 할당 태그, Cost Explorer·CUR, 계정 분리
- `03-kubernetes-cost-allocation/` - 공유 클러스터 비용 쪼개기, OpenCost·Kubecost, 유휴 비용
- `04-compute-purchasing/` - 온디맨드·Savings Plans·RI·Spot, EKS 가격 구조, 커밋 전략
- `05-spot-instances/` - 중단 처리, 인스턴스 다양화, PDB, 워크로드 적합성 판단
- `06-karpenter/` - NodePool, 자동 인스턴스 선택, consolidation, Cluster Autoscaler 비교
- `07-rightsizing/` - requests/limits 실측 기반 조정, VPA, QoS 클래스, 오버커밋
- `08-storage-network-cost/` - EBS·S3 클래스, NAT Gateway, 크로스 AZ, 데이터 전송료
- `09-observability-cost/` - 메트릭 카디널리티, 로그 보존, CloudWatch, 샘플링 트레이드오프
- `10-cost-operations/` - 예산·알림, 이상 탐지, 낭비 정리 루틴, 비용을 코드로

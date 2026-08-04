
# Terraform Labs

Terraform으로 인프라를 코드화하는 학습 실습 및 명령어 정리 저장소

Docker·Kubernetes가 **애플리케이션을 선언형으로 다룬다면**, Terraform은 그 애플리케이션이 올라갈 **인프라 자체(VPC·서브넷·클러스터·IAM)를 선언형으로 다룬다.**
콘솔에서 손으로 만든 인프라는 재현할 수 없고 리뷰할 수 없다. 코드로 옮기는 순간 인프라도 Git 이력·PR 리뷰·롤백의 대상이 된다.

## 구조
- `commands.md` - 자주 쓰는 Terraform 명령어 레퍼런스
- `01-iac-basics/` - IaC 개념, 선언형 vs 명령형, 도구 비교(CloudFormation·Pulumi·Ansible), 핵심 워크플로
- `02-provider-resource/` - HCL 문법, terraform/provider 블록, resource 정의, 참조와 의존성
- `03-state/` - tfstate의 정체, 원격 백엔드(S3+잠금), state 명령, 드리프트
- `04-variables-outputs/` - variable·locals·output, 타입, 값 주입 우선순위, tfvars 환경 분리
- `05-module/` - 모듈화·재사용, 입출력 연결, 레지스트리 모듈, 버전 고정
- `06-meta-arguments/` - count vs for_each, depends_on, lifecycle, provider alias, dynamic 블록
- `07-data-import/` - data source, import 블록, moved·removed 블록(리팩터링)
- `08-workspace-environment/` - workspace vs 디렉터리 분리, 백엔드 key 전략, Terragrunt
- `09-aws-vpc-eks/` - 실습: VPC + EKS 프로비저닝, IRSA, kubeconfig 연결
- `10-cicd-policy/` - CI에서 plan/apply, OIDC 인증, tfsec·checkov, Atlantis, Policy as Code

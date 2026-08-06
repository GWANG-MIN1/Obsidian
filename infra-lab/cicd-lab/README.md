
# CI/CD Labs

GitHub Actions와 ArgoCD로 배포 파이프라인을 만드는 학습 실습 및 명령어 정리 저장소

Terraform으로 인프라를 만들고, Kubernetes에 앱을 올리고, 관측성으로 상태를 보게 됐다면 남은 것은 **"어떻게 바꿀 것인가"** 다.
배포가 무섭고 느린 조직은 배포를 적게 하고, 적게 배포하니 한 번의 변경이 커져서 더 무서워진다. CI/CD는 그 고리를 끊는 일이다.

## 구조
- `commands.md` - gh·argocd·kubectl argo rollouts·docker buildx 레퍼런스
- `01-cicd-basics/` - CI vs CD, 파이프라인 단계, DORA 4지표, Push 방식 vs Pull(GitOps)
- `02-github-actions-basics/` - workflow·job·step, 트리거, 러너, 컨텍스트·표현식, 아티팩트·캐시
- `03-github-actions-advanced/` - matrix, 재사용 워크플로, composite action, concurrency, OIDC, 게이팅 전략
- `04-container-image-pipeline/` - 멀티스테이지 빌드, buildx 캐시, 태그 전략, ECR, 스캔·SBOM·서명
- `05-argocd-advanced/` - Application 구조, sync options, sync wave·hook, ServerSideApply, 드리프트
- `06-app-of-apps-applicationset/` - 계층 배포, ApplicationSet 제너레이터, 멀티환경·멀티클러스터
- `07-gitops-repo-strategy/` - 앱 레포 vs 매니페스트 레포, 이미지 태그 자동 갱신, 환경 승격
- `08-deployment-strategies/` - Recreate·RollingUpdate·Blue-Green·Canary, 트래픽 전환, DB 호환성
- `09-argo-rollouts/` - Rollout CRD, 단계적 카나리, AnalysisTemplate, 자동 롤백
- `10-pipeline-operations/` - DORA 측정, 배포 알림, 롤백 절차, 자주 겪는 문제


# Security Labs

Kubernetes·클라우드 워크로드를 지키는 학습 실습 및 명령어 정리 저장소

보안은 배포 직전에 한 번 검토하는 절차가 아니라 **파이프라인과 클러스터에 상시로 돌아가는 가드레일**이어야 한다.
CI에서 먼저 잡고(shift-left), 놓친 것은 admission에서 막는다(runtime) — 두 겹으로 세우는 것이 이 트랙의 기본 전제다.

## 구조
- `commands.md` - kubectl auth·Kyverno·Trivy·cosign·ESO·kube-bench 레퍼런스
- `01-security-basics/` - 4C 모델, shift-left와 런타임 이중 방어, 위협 모델, 최소권한 원칙
- `02-pod-container-security/` - securityContext, 비-root, 읽기전용 루트FS, capabilities, seccomp
- `03-pod-security-standards/` - PSA(privileged·baseline·restricted), 네임스페이스 라벨, PSP 폐기 이후
- `04-kyverno/` - admission 웹훅 구조, validate·mutate·generate, Audit→Enforce 전환, 정책 작성
- `05-rbac-least-privilege/` - Role·ClusterRole·바인딩, ServiceAccount, 권한 감사, escalate·bind 위험
- `06-secrets-management/` - K8s Secret의 한계, External Secrets+IRSA+SSM, Sealed Secrets·SOPS 비교
- `07-image-security/` - Trivy 스캔, 베이스 이미지, SBOM, cosign 서명과 Kyverno verifyImages 강제
- `08-network-security/` - NetworkPolicy default-deny, 이그레스 제어, CNI 지원, mTLS
- `09-cluster-hardening/` - CIS 벤치마크와 kube-bench, EKS에서 조치 가능한 것, 감사 로그, IMDSv2
- `10-runtime-detection/` - Falco 런타임 탐지, 감사 로그 분석, 침해 지표, 사고 대응 절차

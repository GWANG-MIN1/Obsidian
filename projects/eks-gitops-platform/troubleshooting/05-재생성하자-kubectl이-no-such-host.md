# 05. 재생성하자 kubectl 이 no such host

- **발생**: 2026-07-23 · Phase 4 검증 시작, 클러스터 재생성 직후 (기록 커밋 19:48 `29b0f59`)
- **증상 한 줄**: `apply` · `rollout` · `get` 할 것 없이 모든 `kubectl` 명령이 같은 DNS 에러로 실패
- **원문**: 저장소 [`docs/troubleshooting/05-stale-kubeconfig-after-recreate.md`](https://github.com/GWANG-MIN1/eks-gitops-platform/blob/main/docs/troubleshooting/05-stale-kubeconfig-after-recreate.md)

---

## 현상

```
Unable to connect to the server: dial tcp: lookup
9F52761B....gr7.ap-northeast-2.eks.amazonaws.com: no such host
```

## 환경

- 07-22 클러스터는 destroy, 07-23 에 **같은 이름**(`eks-gitops-dev`)으로 재생성
- kubeconfig 는 전날 `update-kubeconfig` 한 상태 그대로
- Windows PowerShell 에서 작업 (검증 기록의 스크린샷 · 메모 기준)

## 진단

- **`no such host` 는 DNS 에 그 주소가 없다는 뜻이다.** 방화벽이나 자격증명 문제가 아니라, kubectl 이 보는 엔드포인트 자체가 세상에 없다.
- 엔드포인트의 긴 ID(`9F52761B...`)는 **클러스터마다 새로 발급된다.** 어제 클러스터의 주소였다.
- `aws eks list-clusters --region ap-northeast-2` → `["eks-gitops-dev"]` → 클러스터는 살아 있고 kubeconfig 만 낡았다.

## 원인

kubeconfig 는 클러스터 **이름**이 아니라 **API 엔드포인트 URL** 을 저장한다. 이름은 매일 같지만 URL 은 매일 바뀐다.
그래서 `update-kubeconfig` 는 "계정당 1회"가 아니라 **"apply 마다 1회"** 다.

## 해결

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name eks-gitops-dev
kubectl get nodes   # 노드가 보이면 끝
```

## 재발 방지

❌ **코드에 안 들어갔다.** 저장소 문서의 교훈은 *"재생성 루틴을 Makefile 로 묶는다면 apply 직후에 update-kubeconfig 를 넣어라 —
사람이 까먹는 단계는 자동화가 답"* 이다. 2026-09-14 기준 Makefile 의 `apply` 는 `terraform apply` 한 줄이고, kubeconfig 를 갱신하는 타깃이 없다.
Terraform output `configure_kubectl` 이 명령어를 **문자열로 알려 줄 뿐**이다.

이 단계는 재생성마다 손으로 치는 명령 8개 중 하나일 뿐이기도 하다 — [결정 01](../decisions/01-Terraform은-클러스터-경계에서-멈춘다.md) 의 인계점 표에서
*"Terraform → 내 PC"* 칸이 이것이다. 같은 증상은 이 사건 뒤에 쓴 [terraform-lab/09](../../../infra-lab/terraform-lab/09-aws-vpc-eks/README.md) 의
"자주 막히는 지점" 표에도 들어가 있다.

## 배운 점

- **`no such host` 는 네트워크 장애처럼 보이지만, destroy · 재생성 워크플로우에서는 거의 항상 "죽은 클러스터를 가리키는 kubeconfig" 다.**
  `aws eks list-clusters` 한 줄로 가른다.
- **교훈에 "자동화가 답"이라고 쓰고 자동화를 안 했다.** 저장소 문서가 스스로 처방했는데 반영되지 않은 두 건(01 · 05)이
  **둘 다 "사람이 기억해야 하는 단계"** 다. 코드로 고친 02 · 03 은 Git 이 대신 기억한다.
  → [재발 방지 현황](README.md#재발-방지-현황-2026-09-14-기준)

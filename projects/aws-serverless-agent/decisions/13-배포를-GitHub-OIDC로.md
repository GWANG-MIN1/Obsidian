# 13. 배포를 저장된 액세스 키 대신 GitHub OIDC 로

> 메모: GitHub Secrets 에 AWS 키를 넣는 대신 OIDC 로 역할을 잠깐 빌림. 배포 역할을 에이전트 스택에 두면 "역할이 없어서 배포를 못 하는" 순환이 생겨서 pipeline 스택을 분리

---

## 상황

Day 21 까지 배포는 전부 로컬에서 `npx cdk deploy` 였다.
CI 로 옮기려면 GitHub Actions 에 AWS 자격증명을 줘야 한다.

가장 흔한 방법은 IAM 유저를 만들고 액세스 키를 GitHub Secrets 에 넣는 것이다.
그러면 **장기 자격증명이 저장소 설정에 영구히 앉아 있게 된다.**
유출되면 회수하기 전까지 계속 유효하고, 로테이션은 손으로 해야 한다.

## 결정

**GitHub OIDC** 로 간다. 저장된 AWS 키는 **0개**.

```yaml
permissions:
  id-token: write        # ← OIDC 토큰 발급 권한. 없으면 credentials 를 못 읽는다
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      aws-region: us-east-1
      # ↑ 액세스 키 입력 자체가 없다
```

그리고 **배포 역할을 별도 `Day21PipelineStack` 으로 분리한다.**

트리거도 나눴다.

| 이벤트 | 하는 일 |
|---|---|
| push / PR | `cdk synth` — **검증만** |
| 수동 dispatch | `cdk deploy` |

## 왜

- **저장된 장기 키가 없다.** GitHub 이 워크플로 실행마다 OIDC 토큰을 발급하고,
  AWS 가 그걸 검증해서 역할을 **잠깐** 빌려준다. 유출될 영구 자격증명이 존재하지 않는다.
- **신뢰를 이 저장소로 한정한다.** 신뢰 정책의 `sub` 를 `repo:GWANG-MIN1/aws-serverless-agent:*` 로
  좁혔다. 안 좁히면 **아무 레포나 이 역할을 빌린다** (함정 #72).
  브랜치까지 좁히려면 `:ref:refs/heads/main`.
- **권한도 최소로.** CI 역할에는 `sts:AssumeRole on cdk-*` 만 준다.
  실제 배포 권한은 **CDK 부트스트랩 역할**이 갖고 있다. 관리자 정책을 그대로 붙이는 대신
  "부트스트랩 역할을 빌릴 권한"만 주는 구조다 (함정 #74).
- **push 로 배포가 안 되게 했다.** 자동화하되 사고는 막는다.
  push 는 `cdk synth` 로 템플릿이 합성되는지만 보고, 실제 배포는 수동 dispatch 다.
  학습 프로젝트라 배포가 곧 과금이고, 실수로 밀어 넣을 이유가 없다.

## 왜 배포 역할을 별도 스택으로 뺐나 (닭-달걀)

이게 이 day 의 진짜 설계 포인트다.

배포 역할을 에이전트 스택(`Day21CicdStack`) 안에 두면 이렇게 된다.

```
CI 가 배포하려면      → 배포 역할이 필요하고
배포 역할이 생기려면  → 스택이 배포돼야 하고
스택이 배포되려면    → CI 가 배포해야 하고 …
```

**순환이다** (함정 #73). 그래서 나눴다.

```
Day21PipelineStack   OIDC provider + 배포 역할     ← 1회 수동 배포 (로컬에서)
Day21CicdStack       에이전트 본체                  ← CI 가 배포
```

CI 의 **정체성**(누구인가)과 CI 가 **배포하는 대상**(무엇을 만드는가)을 갈랐다.
정체성은 한 번 손으로 만들고, 그다음부터 대상은 자동이다.

## 왜 다른 건 안 썼나

**IAM 유저 + 액세스 키를 GitHub Secrets 에**

가장 빠르다. 5분이면 된다. 그리고 **영구 자격증명이 저장소에 앉는다.**
유출 경로가 여럿이고(포크된 워크플로, 로그 출력, 액션 공급망), 회수는 사람이 해야 한다.
2026년에 이걸 고를 이유가 없다.

**셀프호스티드 러너에 인스턴스 역할**

EC2 러너에 IAM 역할을 붙이면 키가 없다. 그런데 **러너를 운영해야 한다** —
서버리스 프로젝트에서 상주 서버를 들이는 셈이다. [결정 11](11-Telegram-대신-Discord.md) 의
게이트웨이 봇을 안 쓴 것과 같은 이유다.

**CodePipeline / CodeBuild**

AWS 안에서 끝나서 자격증명 문제가 아예 없다. 안 쓴 이유:
코드가 GitHub 에 있는데 파이프라인만 AWS 로 가면 트리거 연결이 한 겹 더 생기고,
**GitHub Actions OIDC 가 DevOps 실무 표준**이라 그쪽을 익히는 게 낫다고 봤다.

**push 에 자동 배포 (승인 게이트 없이)**

CI/CD 의 교과서적 모습이긴 하다. 안 한 이유는 학습 프로젝트의 성격 —
배포가 곧 과금이고, 매 커밋마다 CloudFront 를 재배포할 이유가 없다.
Day 21 README 에 "PR `cdk diff` 코멘트 / main 머지 승인 게이트"를 **옵션**으로 남겼다.

## 결과

- GitHub Actions 가 **저장된 AWS 키 0개**로 배포하는 걸 확인했다.
  저장소 Secrets 화면에 AWS 키가 없는 스크린샷을 데모로 남겼다.
- push/PR 에서 `cdk synth` 가 자동으로 돈다 — 합성 에러를 로컬에서 안 돌려도 잡힌다.

## 확인하지 못한 것

**`cdk diff` 를 PR 코멘트로 안 올린다.** synth 는 "합성이 되나"만 보고,
"무엇이 바뀌나"는 안 보여준다. 실제 인프라 리뷰는 여전히 손으로 한다.

**롤백이 없다.** deploy 가 실패하면 CloudFormation 이 알아서 롤백하지만,
성공한 뒤에 문제가 발견되면 이전 커밋으로 되돌려 다시 dispatch 하는 수밖에 없다.

## 관련 함정

→ [트러블 13](../troubleshooting/13-키리스-배포가-자격증명을-못-읽는다.md) (#71~77)

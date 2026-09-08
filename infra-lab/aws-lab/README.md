# aws-lab

AWS를 직접 쓰면서 만난 개념들.
프로젝트에서 나왔지만 **다음 프로젝트에서도 다시 볼 내용**이라
프로젝트 폴더가 아니라 여기에 둔다.

각 노트는 `개념 정리 + 실행 가능한 명령 + 배운 점` 구성을 따르고,
끝에 **"처음 만난 곳"**으로 프로젝트의 결정·트러블슈팅 노트를 링크한다.

> 지금 아홉 노트는 전부 [aws-serverless-agent](../../projects/aws-serverless-agent/README.md)에서 나온
> **서버리스(Lambda 중심)** 주제다. EC2·VPC·RDS 같은 다른 영역이 쌓이면 이 표를 나눈다.

|     | 노트                                                                  | 한 줄                                         |
| --- | ------------------------------------------------------------------- | ------------------------------------------- |
| 01  | [Function URL vs API Gateway](Function-URL-vs-API-Gateway.md)       | HTTP로 노출하는 세 방법 — 응답 스트리밍이 선택을 강제한다         |
| 02  | [Lambda async invoke로 시간 분리](Lambda-async-invoke로-시간-분리.md)     | HTTP 시간과 작업 시간 떼어 놓기 — Lambda 자체가 큐다        |
| 03  | [DynamoDB 단일 vs 멀티 테이블](DynamoDB-단일-vs-멀티-테이블.md)             | PK가 가능한 질의를 결정한다 · 정렬키와 `ScanIndexForward`   |
| 04  | [Bedrock Converse 도구 호출 루프](Bedrock-Converse-도구-호출-루프.md)      | Agent Loop의 최소 형태 — `toolUse`/`toolResult` 짝 |
| 05  | [SigV4 presigned WebSocket URL](SigV4-presigned-WebSocket-URL.md)   | 키 없는 브라우저를 IAM만으로 — 보안 토큰을 서명에서 뺀다          |
| 06  | [Lambda@Edge 운영 제약](Lambda-Edge-운영-제약.md)                         | us-east-1 고정 · 환경변수 불가 · 삭제 지연              |
| 07  | [X-Ray로 Lambda 계측하기](X-Ray-Lambda-계측.md)                          | active tracing만으론 함수 상자만 보인다                 |
| 08  | [esbuild ESM 번들의 동적 require](esbuild-ESM-번들의-동적-require.md)      | ESM엔 require가 없다 — 모듈 로드 시점에 터진다             |
| 09  | [GitHub OIDC로 CDK 배포하기](GitHub-OIDC로-CDK-배포.md)                   | 닭-달걀 분리와 권한 위임 — 저장된 키 0개                   |

**요청 하나가 지나가는 순서**로 늘어놓았다 —
노출(01) → 실행 분리(02) → 저장(03) → 모델(04) → 실시간(05) → 엣지(06),
그리고 그 위에 관측(07) · 빌드(08) · 배포(09)가 얹힌다.

## 관통하는 주제

**01·06은 "이 서비스로 무엇을 할 수 없는가"** — 비교표에서 결정을 내리는 칸은 대개 하나고,
그건 기능이 많은 쪽이 아니라 **탈락 조건**이다.
API Gateway는 스트리밍이 안 되고, CloudFront Function은 origin을 못 바꾼다.

**03·04는 "구조가 인터페이스를 바꾼다"** — PK를 `user_id`로 두면 `userId`가 모든 경로에 따라다니고,
도구를 하나로 두면 확장이 도구 추가가 아니라 샌드박스 주입이 된다.
안쪽 설계가 바깥쪽 시그니처를 정한다.

**05·09는 "권한을 어떻게 좁혀서 넘기는가"** — 세션정책으로 교집합을 만들고,
배포 권한을 직접 주는 대신 위임한다. 둘 다 **상한선을 먼저 긋고 런타임에 더 좁힌다**.

**02·07·08은 "켰다고 동작하는 게 아니다"** — 쪼갰는데 동기로 붙어 있고,
계측을 켜도 호출은 안 보이고, 배포가 성공해도 모듈은 로드되지 않을 수 있다.
**설정과 실제 동작 사이에 조용한 단절이 있다.**

## 이 트랙의 함정에는 공통점이 있다

아홉 노트가 다루는 실패 대부분이 **"배포는 되는데 실행 시점에 깨지는"** 종류다.
`cdk synth`도 CloudFormation도 아무 불평을 하지 않는다.

원본 프로젝트가 함정 78개를 누적하고 남긴 관찰이 여기에도 그대로 적용된다.

> 12개 중 9개가 "코드는 컴파일·deploy 되는데 실행 시점에 깨지는" 종류.
> 실배포 + 실호출까지 가야 발견.

→ [aws-serverless-agent / troubleshooting](../../projects/aws-serverless-agent/troubleshooting/README.md)

**그래서 검증은 "됐는가"가 아니라 다음 셋으로 한다.**

| 확인 방법 | 무엇이 잡히나 |
|---|---|
| **시간으로 잰다** (`-w "%{time_total}"`) | 스트리밍이 버퍼링되는 것(01), 비동기가 동기인 것(02) |
| **수신 쪽에서 본다** | publish가 아무 데도 안 가는 것(05) |
| **번들을 로드해 본다** | INIT 크래시(08) |

## 다른 트랙과의 관계

| 이 트랙 | 이어지는 곳 |
|---|---|
| [03 DynamoDB](DynamoDB-단일-vs-멀티-테이블.md) | 관계형 스키마 개념 → [`../db-lab/`](../db-lab/README.md) |
| [07 X-Ray](X-Ray-Lambda-계측.md) | 분산추적 일반 개념 → [`../observability-lab/07-tracing/`](../observability-lab/07-tracing/) |
| [09 OIDC](GitHub-OIDC로-CDK-배포.md) | GitHub Actions OIDC 기본 → [`../cicd-lab/03-github-actions-advanced/`](../cicd-lab/03-github-actions-advanced/) |
| [09 OIDC](GitHub-OIDC로-CDK-배포.md) | Terraform 쪽 같은 패턴 → [`../terraform-lab/10-cicd-policy/`](../terraform-lab/10-cicd-policy/) |
| 비용 감각 | Cost Explorer·태그 전략 → [`../cost-lab/02-cost-visibility/`](../cost-lab/02-cost-visibility/) |

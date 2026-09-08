# aws-serverless-agent

AWS 서버리스 위에서 도는 AI 에이전트를, 원본 레퍼런스를 분해해서 하루에 한 조각씩 다시 쌓은 학습 프로젝트.

- 저장소: https://github.com/GWANG-MIN1/aws-serverless-agent
- 원본: [breath103/serverless-agent](https://github.com/breath103/serverless-agent) (AWS Summit Korea 2026 DEV308)
- 기간: 2026-05-23 ~ 2026-06-25 (커밋 151개 · 작업일 23일 · day 폴더 21개)
- 한 줄 요약: **완성된 레퍼런스를 한 번에 복제하지 않고 부품 → 조립 → 차용 → 확장 순으로
  다시 만들면서, 실배포에서만 드러나는 함정 78개를 밟아 본 프로젝트**

> 저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 **"내가 무엇을 정했고 무엇에 막혔나"** 다.
> 다른 프로젝트는 [`projects/`](../README.md) 에 있다.
> 같은 내용을 옮겨 적지 않는다.

> **기록의 성격** — 이 프로젝트는 각 `day-XX-*/README.md` 에 그날의 함정을 번호로 누적해 두었다.
> 그래서 [gym-management-db](../gym-management-db/README.md) 와 달리 커밋 로그에서 되짚을 필요가 없었다.
> 이 노트는 그 기록을 **날짜 순이 아니라 "판단"과 "함정" 축으로 다시 접은 것**이다.

---

## [타임라인](타임라인.md)

커밋 151개 · 2026-05-23 ~ 2026-06-25 · 4 Phase + 캡스톤, 그리고 **마침표를 찍은 뒤 8일 만에 발견한 치명적 버그**

---

## 결정 — 왜 그렇게 만들었나

| | 노트 |
|---|---|
| 01 | [원본을 한 번에 복제하지 않고 하루에 한 조각씩](decisions/01-원본을-한-번에-복제하지-않는다.md) |
| 02 | [API Gateway 대신 Lambda Function URL](decisions/02-API-Gateway-대신-Function-URL.md) |
| 03 | [메시지 정렬키를 타임스탬프 단독에서 합성으로](decisions/03-정렬키를-타임스탬프-합성으로.md) |
| 04 | [API와 Worker를 SQS 없이 async invoke로 분리](decisions/04-API와-Worker를-async-invoke로-분리.md) |
| 05 | [단일 테이블 대신 도메인별 멀티 테이블](decisions/05-단일-테이블-대신-멀티-테이블.md) |
| 06 | [도구를 여러 개 대신 executeCode 하나로](decisions/06-도구를-executeCode-하나로.md) |
| 07 | [실시간을 폴링 대신 IoT MQTT push로](decisions/07-실시간을-IoT-MQTT-push로.md) |
| 08 | [브라우저 권한을 AssumeRole + 세션정책으로](decisions/08-브라우저-권한을-세션정책으로.md) |
| 09 | [CloudFront Function 대신 Lambda@Edge + SSM](decisions/09-CF-Function-대신-Lambda-Edge와-SSM.md) |
| 10 | [skill은 샌드박스에 주입하고 호출기록은 LLM에 숨긴다](decisions/10-skill을-샌드박스에-주입.md) |
| 11 | [Telegram 대신 Discord Interactions](decisions/11-Telegram-대신-Discord.md) |
| 12 | [캘린더를 OAuth 대신 공개 ICS 읽기 전용으로](decisions/12-캘린더를-공개-ICS-읽기-전용으로.md) |
| 13 | [배포를 액세스 키 대신 GitHub OIDC로](decisions/13-배포를-GitHub-OIDC로.md) |

## 트러블슈팅 — 무엇에 막혔나

저장소에 누적한 함정은 78개다. 그중 **원인과 해결이 한 줄로 안 끝나는 것** 13개를 풀어 적었다.

| | 노트 | 원본 # |
|---|---|---|
| 01 | [S3 버킷은 만들어졌는데 브라우저는 403](troubleshooting/01-S3는-만들어졌는데-403.md) | #9 |
| 02 | [CloudFront를 끼우자 API가 403](troubleshooting/02-CloudFront를-끼우자-API가-403.md) | #11 · #12 |
| 03 | [분리했는데 API가 여전히 Worker를 기다린다](troubleshooting/03-분리했는데-API가-Worker를-기다린다.md) | #13 · #15 |
| 04 | [toolResult를 넣으면 ValidationException](troubleshooting/04-toolResult를-넣으면-ValidationException.md) | #21 · #22 · #23 |
| 05 | [IoT publish가 아무 데도 안 간다](troubleshooting/05-IoT-publish가-아무-데도-안-간다.md) | #27 · #28 · #29 |
| 06 | [브라우저 WSS 연결이 서명 불일치로 거부](troubleshooting/06-WSS-서명이-계속-거부된다.md) | #32 · #33 · #36 · #38 |
| 07 | [Lambda@Edge는 환경변수를 못 쓴다](troubleshooting/07-Lambda-Edge는-환경변수를-못-쓴다.md) | #39 · #40 · #41 |
| 08 | ["오늘 비용"이 0으로 나온다](troubleshooting/08-오늘-비용이-0으로-나온다.md) | #47 · #48 · #52 |
| 09 | [Discord가 Endpoint URL을 저장하지 못한다](troubleshooting/09-Discord가-Endpoint-URL을-저장-못-함.md) | #53 · #54 · #58 |
| 10 | [봇이 "응답 실패"만 띄운다 — 3초의 벽](troubleshooting/10-봇이-응답-실패만-띄운다.md) | #55 · #56 · #57 |
| 11 | [캘린더는 호출됐는데 일정이 0건](troubleshooting/11-캘린더가-0건을-돌려준다.md) | #59~64 |
| 12 | [완성 8일 뒤, 재배포하니 전 요청이 500](troubleshooting/12-재배포하니-전-요청이-500.md) | **#78 ⭐** |
| 13 | [키리스 배포가 자격증명을 못 읽는다](troubleshooting/13-키리스-배포가-자격증명을-못-읽는다.md) | #71~77 |

## 개념 노트

이 프로젝트에서 나왔지만 다음에도 쓸 지식은
[`infra-lab/aws-lab/`](../../infra-lab/aws-lab/README.md) 에 두고 링크한다.

- [x] [Function URL vs API Gateway](../../infra-lab/aws-lab/Function-URL-vs-API-Gateway.md) — 응답 스트리밍이 선택을 강제한다
- [x] [Lambda async invoke로 시간 분리](../../infra-lab/aws-lab/Lambda-async-invoke로-시간-분리.md) — Lambda 자체가 큐다
- [x] [DynamoDB 단일 vs 멀티 테이블](../../infra-lab/aws-lab/DynamoDB-단일-vs-멀티-테이블.md) — PK가 가능한 질의를 결정한다
- [x] [Bedrock Converse 도구 호출 루프](../../infra-lab/aws-lab/Bedrock-Converse-도구-호출-루프.md) — `toolUse`/`toolResult` 짝과 교대 규칙
- [x] [SigV4 presigned WebSocket URL](../../infra-lab/aws-lab/SigV4-presigned-WebSocket-URL.md) — IoT `iotdevicegateway` 특례
- [x] [Lambda@Edge 운영 제약](../../infra-lab/aws-lab/Lambda-Edge-운영-제약.md) — us-east-1 · 환경변수 불가 · 삭제 지연
- [x] [X-Ray로 Lambda 계측하기](../../infra-lab/aws-lab/X-Ray-Lambda-계측.md) — `captureAWSv3Client` 와 async invoke trace 연결
- [x] [esbuild ESM 번들의 동적 `require`](../../infra-lab/aws-lab/esbuild-ESM-번들의-동적-require.md) — `createRequire` banner
- [x] [GitHub OIDC로 CDK 배포하기](../../infra-lab/aws-lab/GitHub-OIDC로-CDK-배포.md) — 닭-달걀 분리와 권한 위임

일반 개념은 기존 트랙에 있다 —
분산추적 [`observability-lab/07-tracing/`](../../infra-lab/observability-lab/07-tracing/) ·
GitHub Actions OIDC [`cicd-lab/03-github-actions-advanced/`](../../infra-lab/cicd-lab/03-github-actions-advanced/)

---

## 숫자로 말할 것

| | |
|---|---:|
| 커밋 | 151개 (2026-05-23 ~ 2026-06-25, 작업일 23일) |
| day 폴더 | 21개 (각각 독립 실행 가능한 CDK 프로젝트) |
| PR | 12개 (Day 12부터 브랜치 → PR 방식으로 전환) |
| 검증 스크린샷 | 50장 |
| 누적 트러블슈팅 | 78개 |
| Phase 2 실비용 | ~$0.07 (Day 5~9, 약 일주일) — 이 중 99%가 Bedrock 호출 |

**주의해서 말할 것**

- 트러블슈팅 78개는 **함정 목록**이지 장애 기록이 아니다. 대부분 "배포는 되는데 호출하면 깨지는" 종류고,
  실제로 밟은 것과 문서를 읽다가 미리 피한 것이 섞여 있다.
- 비용 $0.07은 **Phase 2(Day 5~9)만** 집계한 값이다. Phase 3~4의 CloudFront·IoT·Lambda@Edge·
  Cost Explorer 구간은 따로 집계하지 않았다.
- 부하 테스트, 동시성 테스트, 자동 테스트가 **없다.** 검증은 전부 배포 후 손으로 호출한 것이다.
  → [아직 안 한 것](#아직-안-한-것과-이유)
- "22일 로드맵"은 Day 번호이지 달력 날짜가 아니다. 실제 작업일은 23일이고 Day 1~4는 나흘에 몰려 있다.

---

## 아직 안 한 것과 이유

**자동 테스트가 없다**

검증이 전부 "배포하고 curl 쳐 보기"다. 그래서 [트러블 12](troubleshooting/12-재배포하니-전-요청이-500.md)처럼
**환경이 바뀌면 무너지는 종류**를 못 잡았다. 이 프로젝트에서 가장 크게 비어 있는 칸이다.
gym-management-db는 반대로 리뷰를 받고 테스트 56개를 넣어서 이 부분이 채워졌다 —
같은 사람이 만든 두 프로젝트인데 검증 체계가 정반대다.

**`node:vm` 은 진짜 격리가 아니다**

`while(true){}` 같은 동기 무한루프는 `Promise.race` 타임아웃으로도 못 막는다. 이벤트 루프 자체가 멈춘다.
원본도 같은 한계다. 진짜 격리는 `worker_threads` 나 isolate 가 필요하고,
지금은 **학습용 경계**로만 쓴다고 못박아 두었다. → [결정 06](decisions/06-도구를-executeCode-하나로.md)

**인증이 없다**

`userId` 를 요청에 그대로 실어 보낸다. 남의 `userId` 를 넣으면 남의 세션이 보인다.
원본은 scrypt + HTTP-only 쿠키로 로그인을 붙였지만, 이 프로젝트의 주제(에이전트 구조)에서
비중이 뒤집힌다고 보고 넣지 않았다. **없는 것은 없다고 적어 두는 쪽**을 택했다.

**Phase 3~4 비용을 집계하지 않았다**

Phase 2까지는 실제 청구서로 $0.07을 확인했는데, CloudFront·IoT·Lambda@Edge·Cost Explorer가
붙는 구간은 감각적인 추정만 있다. 아이러니하게도 [`awsCost` skill](decisions/10-skill을-샌드박스에-주입.md)을
만들어 놓고 정작 그걸로 이 프로젝트 비용을 정리하지 않았다.

**재연결과 만료 처리**

브라우저 WSS URL은 1시간짜리다. 만료되거나 끊기면 재발급 로직이 없어서 새로고침해야 한다.
원본은 RxJS 백오프로 재연결하는데, 데모 범위에서는 단발 연결로 끝냈다.

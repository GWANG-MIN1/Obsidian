# Decisions

이 프로젝트에서 **무엇을 왜 그렇게 정했는가**의 기록.
`../troubleshooting/` 이 "막힌 것"이라면 여기는 "고른 것"이다.
형식과 원칙은 [`gym-management-db/decisions/`](../../gym-management-db/decisions/README.md) 와 같다.

```
## 상황              무엇을 정해야 했는가
## 결정              무엇을 택했는가
## 왜                근거 — 확인한 사실 기준으로
## 왜 다른 건 안 썼나   버린 선택지와 그 이유
```

## 이 프로젝트에서 결정이 갖는 특수성

**따라 만들 원본이 있었다.** 그래서 결정이 세 종류로 갈린다.

| 종류 | 예 |
|---|---|
| **원본을 확인하고 따라간 것** | [02](02-API-Gateway-대신-Function-URL.md) Function URL · [05](05-단일-테이블-대신-멀티-테이블.md) 멀티 테이블 · [09](09-CF-Function-대신-Lambda-Edge와-SSM.md) Lambda@Edge |
| **원본과 일부러 다르게 간 것** | [11](11-Telegram-대신-Discord.md) Discord · [12](12-캘린더를-공개-ICS-읽기-전용으로.md) 공개 ICS · [13](13-배포를-GitHub-OIDC로.md) OIDC |
| **원본에 없는 나만의 규칙** | [01](01-원본을-한-번에-복제하지-않는다.md) 하루 한 조각 |

"원본이 그렇게 해서"는 근거가 아니다. **원본이 왜 그렇게 했는지 확인한 것**이 근거다.
[02](02-API-Gateway-대신-Function-URL.md) 가 그 예다 — 원본이 Function URL 을 쓴다는 사실보다,
API Gateway 가 응답 스트리밍을 지원하지 않는다는 사실이 결정의 근거다.

## 원칙

- **버린 이유가 핵심이다.** "Lambda@Edge 를 썼다"는 누구나 말한다.
  "CloudFront Function 은 URI 문자열만 만질 수 있어서 origin 을 못 바꾼다"는 해본 사람만 말한다.
- **근거는 확인한 것만 적는다.** 트러블슈팅 노트와 같은 규율.
- **한계를 같이 적는다.** [06](06-도구를-executeCode-하나로.md) 의 `node:vm` 처럼,
  "이건 진짜 격리가 아니다"를 결정 안에 남겨 둬야 나중에 그대로 쓰지 않는다.
- 재사용 가능한 지식은 여기 말고 [`infra-lab/`](../../../infra-lab/) 에 두고 링크한다.

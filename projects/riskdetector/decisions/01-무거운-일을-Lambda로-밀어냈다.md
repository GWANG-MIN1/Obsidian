# 01. 무거운 일을 Spring 에서 Lambda 로 밀어냈다

> 메모: OCR 10~20초 + 분석 30~60초를 Spring 요청 스레드가 붙잡고 있으면
> Render 인스턴스 하나로 동시 사용자를 받을 수 없다. 2026-04-06 ~ 04-29

---

## 상황

계약서 한 건을 처리하는 데 두 번의 외부 호출이 필요했다.

| 작업 | 외부 의존 | 체감 소요 |
|---|---|---|
| OCR | Upstage Document Parse API | 10~20초 |
| 독소조항 분석 | Bedrock Claude 3.5 Sonnet + KB 검색 | 30~60초 |

둘을 Spring 안에서 하면 **요청 하나가 최대 1분 넘게 스레드를 점유**한다.
배포 대상은 Render 무료/저가 인스턴스 한 대였다. 게다가 두 작업의 실패 양상이 다르다 —
OCR 은 외부 API 의 429(rate limit)와 이미지 품질, 분석은 모델 쿼터와 JSON 파싱 실패다.

## 결정

**Spring 은 오케스트레이터로만 쓰고, 실제 일은 Lambda 3개에 나눠 밀어냈다.**

```
Spring Boot (backend_core)
  ├─ S3 업로드는 직접 한다 (S3Util)
  ├─ ocr_lambda            ← 동기 호출 (결과가 바로 필요)
  ├─ bedrock_lambda        ← 비동기 호출 (fire-and-forget)
  └─ analysis_result_loader ← bedrock_lambda 가 호출, Spring 은 관여 안 함
```

Lambda 를 **3개로 쪼갠 기준은 "실패 도메인"** 이다.

| Lambda | 외부 의존 | 실패하면 |
|---|---|---|
| `ocr_lambda` | Upstage API | 그 페이지만 빠진다 (`partial_success`) |
| `bedrock_lambda` | Bedrock / Gemini | 분석만 실패, OCR 결과는 남는다 |
| `analysis_result_loader` | PostgreSQL | 분석은 끝났는데 저장만 실패 |

## 왜

- **실패를 부분 재시도할 수 있다.** OCR 은 성공하고 분석만 실패한 상태가 **정상적인 중간 상태**로
  존재한다. `Contract` + `OcrContent` 는 남아 있고 `ContractAnalysis.processStatus` 만 `FAILED` 다.
  사용자는 재업로드 없이 분석만 다시 누를 수 있다.
- **런타임이 다르다.** Spring 은 Java 17, Lambda 는 Python 3.13 이다. OCR·RAG 쪽은
  팀의 AI 담당이 Python 으로 쓰고 있었고, 그걸 Java 로 옮기는 비용이 없었다.
- **Spring 의 응답시간 SLA 가 다르다.** `/api/ocr/upload` 는 결과를 돌려줘야 하지만
  `/api/analysis` 는 `analysisId` 만 주고 끝나면 된다. 프론트가 폴링한다.

## 왜 다른 건 안 썼나

**Spring 안에서 `@Async` 스레드로**

`AsyncConfig` 에 `ocrExecutor` · `analysisExecutor` 를 실제로 만들어 뒀고 지금도 쓰고 있다.
하지만 **스레드를 늘려도 Render 인스턴스 한 대의 메모리·CPU 는 그대로**다.
분석 요청 5개가 동시에 오면 Bedrock 응답을 기다리는 스레드 5개가 인스턴스에 앉아 있게 된다.
Lambda 로 던지면 그 대기가 **AWS 쪽 동시성**으로 옮겨 간다.

**SQS 큐를 앞에 두고 워커를 따로**

[serverless-uptime-monitor 결정 01](../../serverless-uptime-monitor/decisions/01-헬스체크를-enumerator와-worker로-분리.md)
에서 내가 나중에 택한 구조다. 여기선 안 썼다 — 큐·DLQ·가시성 타임아웃을 설계할 시간이 없었고,
`InvocationType=Event` 가 이미 **AWS 내부 큐 + 2회 자동 재시도**를 공짜로 준다.
대신 그 재시도가 로더 중복 호출로 이어질 수 있어서 로더를 멱등하게 만들어야 했다.
→ [결정 03](03-DB-쓰기를-워커에게-넘겼다.md)

**EC2 에 올려서 그냥 오래 붙잡기**

시연 하루 이틀이라면 실제로 이게 가장 단순하다. 안 쓴 이유는 팀이 Render 를 이미 쓰고 있었고,
AWS 계정은 Lambda·Bedrock·S3 용으로만 열려 있었기 때문이다. 판단이라기보다 **주어진 조건**이었다.

## 결과

동시 사용자 부하를 실제로 측정하지 않았다. **"설계상 그럴 것"이라는 근거뿐**이다.
시연 중 문제는 없었지만, 그건 동시 사용자가 없었기 때문일 수도 있다.
같은 종류의 미검증을 [serverless-uptime-monitor](../../serverless-uptime-monitor/README.md) 에도 적어 뒀다.

## 배운 점

**분리 기준은 "무거움"이 아니라 "실패 양상"이다.**
처음엔 "느린 걸 밖으로 뺀다"로 생각했는데, 3개로 쪼갠 실제 이유는 속도가 아니라
*OCR 실패와 분석 실패와 저장 실패가 사용자에게 서로 다른 의미*라는 점이었다.
느리다는 이유로만 쪼개면 경계가 엉뚱한 데 생긴다.

# 02. OCR 은 동기 호출, 분석은 비동기 호출

> 메모: 같은 `LambdaUtil` 에 `invokeAndWait` 와 `invokeAsync` 가 같이 있다.
> 기준은 "결과가 이 응답에 필요한가". 2026-04-06 / 04-29

---

## 상황

[결정 01](01-무거운-일을-Lambda로-밀어냈다.md) 로 Lambda 3개에 일을 밀어냈지만,
**호출 방식을 통일할 수 없었다.**

- `POST /api/ocr/upload` 는 **OCR 결과를 그 응답에 담아야** 한다. 사용자가 다음 화면에서
  추출된 텍스트를 보고 **직접 수정·마스킹**하기 때문이다. 결과 없이는 다음 화면이 없다.
- `POST /api/analysis` 는 **`analysisId` 만 주면 끝**이다. 프론트가 그 id 로 폴링한다.

## 결정

같은 유틸에 두 경로를 두고, **"결과가 이 응답에 필요한가"로 갈랐다.**

```java
// 동기 — RequestResponse. 6MB 한도가 있어 5.5MB 에서 미리 끊는다.
private static final int SYNC_PAYLOAD_LIMIT_BYTES = 5_500_000;

public InvokeResponse invokeAndWait(String functionName, Object payload) throws Exception {
    String payloadJson = objectMapper.writeValueAsString(payload);
    int sizeBytes = payloadJson.getBytes(StandardCharsets.UTF_8).length;
    if (sizeBytes > SYNC_PAYLOAD_LIMIT_BYTES) {
        throw new IllegalStateException("Lambda payload too large: " + sizeBytes + " bytes");
    }
    return lambdaClient.invoke(InvokeRequest.builder()
            .functionName(functionName).payload(SdkBytes.fromUtf8String(payloadJson)).build());
}

// 비동기 — Event. AWS 가 즉시 202 를 주고 Lambda 는 백그라운드에서 돈다.
public void invokeAsync(String functionName, Object payload) throws Exception {
    lambdaClient.invoke(InvokeRequest.builder()
            .functionName(functionName)
            .invocationType(InvocationType.EVENT)
            .payload(SdkBytes.fromUtf8String(objectMapper.writeValueAsString(payload)))
            .build());
}
```

- OCR: `invokeAndWait` + **재시도 3회** (429 감지 시 1.5초 × 시도횟수 백오프)
- 분석: `invokeAsync` 후 즉시 `analysisId` 반환, 실패는 `analysis.fail()` 로만 기록
- 챗봇 KB 검색: `invokeAndWait` — 사용자가 답을 기다리는 중이라 동기
  ([결정 10](10-챗봇만-OpenAI로-분석은-Bedrock으로.md))

## 왜

- **동기 호출에는 6MB 페이로드 한도가 있다.** OCR 은 S3 키만 넘기니 문제없지만,
  분석은 **계약서 전문을 페이지별 문자열 배열**로 넘긴다. 페이지가 많으면 한도에 닿는다.
  `invokeAsync` 쪽(Event, 256KB)이 한도가 **더 작다**는 게 함정인데, 이 프로젝트는
  OCR 텍스트가 그만큼 크지 않아 실제로 안 걸렸다 — **확인은 안 했다.**
- **동기 호출은 Lambda 실행시간만큼 Spring 스레드를 붙잡는다.** OCR 은 페이지당 10~20초라
  받아들였고, 분석 30~60초는 받아들일 수 없었다.
- **OCR 은 실패를 사용자에게 바로 말할 수 있다.** 응답에 `ocrStatus` 를
  `success` / `partial_success` / `fail` 로 담는다. 비동기면 이걸 못 준다.

## 왜 다른 건 안 썼나

**OCR 도 비동기로 던지고 폴링**

일관성은 좋아지지만, **업로드 직후 화면이 할 일이 없어진다.** 이 서비스의 UX 는
OCR 결과를 사용자가 손으로 고치고 마스킹하는 단계가 핵심이라, 그 화면이 빈 채로 폴링을 돌면
"업로드가 된 건지" 자체가 불확실해진다. 페이지 1~2장이면 20초 안에 끝나니 기다리게 했다.

**분석도 동기로 두고 타임아웃을 늘리기**

Render 의 프록시 타임아웃과 Spring 스레드 점유를 둘 다 건드려야 한다.
그리고 60초를 기다리는 HTTP 요청은 **모바일 네트워크에서 자주 끊긴다.**

**API Gateway 를 Lambda 앞에 두고 HTTP 로 호출**

Spring 이 이미 AWS SDK 로 IAM 자격증명을 들고 있어서 `lambda:InvokeFunction` 으로 직접 부르는 게
배선이 하나 적다. API Gateway 를 끼우면 엔드포인트 보호를 또 설계해야 한다.
나중에 [aws-serverless-agent 결정 02](../../aws-serverless-agent/decisions/02-API-Gateway-대신-Function-URL.md)
에서 같은 종류의 판단을 다시 하게 된다.

## 결과

**비동기로 던진 대가가 바로 나타났다.** 분석 결과를 DB 에 쓸 주체가 사라져서
세 번째 Lambda 를 만들어야 했다. → [결정 03](03-DB-쓰기를-워커에게-넘겼다.md)

## 배운 점

**비동기는 "기다리지 않는다"가 아니라 "결과를 다른 경로로 받는다"다.**
`InvocationType=Event` 한 줄로 응답은 빨라졌지만, 그 대가로 **결과 수신 경로를 새로 설계**해야 했다.
줄어든 코드보다 늘어난 코드가 많았다. 같은 교훈을 나중에
[Lambda async invoke로 시간 분리](../../../infra-lab/aws-lab/Lambda-async-invoke로-시간-분리.md) 에 개념 노트로 적었다.

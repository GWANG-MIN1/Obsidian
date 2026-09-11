# 04. OCR 페이지를 병렬 대신 순차로

> 메모: `CompletableFuture` 와 스레드풀 5개를 만들어 놓고 바로 `.join()` 한다.
> 병렬로 설계했다가 rate limit 때문에 되돌린 흔적. 2026-04-06 ~

---

## 상황

계약서가 여러 장이면 페이지마다 OCR Lambda 를 호출해야 한다.
처음 설계는 당연히 **병렬**이었다 — 페이지 5장이면 5개를 동시에 던져 20초에 끝내는 것.

그런데 Upstage Document Parse API 가 **429(too many requests)** 를 돌려주기 시작했다.
동시에 던진 요청 중 일부가 실패하고, 그 페이지는 결과에서 그냥 빠진다.

## 결정

**페이지를 순차로 처리하고, 페이지 사이에 1.5초를 쉰다.**

```java
private static final int OCR_MAX_ATTEMPTS = 3;
private static final long OCR_PAGE_DELAY_MS = 1500L;

// Upstage OCR rate limit 방지를 위해 페이지는 순차 처리한다.
List<OcrPageResult> results = new ArrayList<>();
for (int i = 0; i < files.size(); i++) {
    OcrPageResult result = processOcrPageAsync(files.get(i), contractId, s3KeyPrefix, i).join();
    if (result != null && result.isSuccess()) results.add(result);
    if (i + 1 < files.size()) sleepQuietly(OCR_PAGE_DELAY_MS);
}
results.sort(Comparator.comparingInt(OcrPageResult::getPageIdx));
```

그리고 429 를 만나면 **같은 페이지를 최대 3번** 다시 던진다. 대기는 시도 횟수에 비례한다.

```java
if (attempt < OCR_MAX_ATTEMPTS && isRateLimited(rawPayload)) {
    sleepQuietly(OCR_PAGE_DELAY_MS * attempt);   // 1.5s → 3.0s
    continue;
}
```

```java
private boolean isRateLimited(String payload) {
    return payload != null && (payload.contains("429") || payload.contains("too_many_requests"));
}
```

## 왜

- **일부 페이지가 빠진 계약서는 분석이 무의미하다.** 5장 중 3장만 읽힌 계약서를 분석하면
  빠진 장에 있던 독소조항을 **"없다"고 보고**한다. 느린 것보다 나쁘다.
- **속도는 페이지 수에 비례하지만 계약서는 보통 1~3장이다.** 3장이면 순차로도
  30~60초 + 대기 3초다. 시연 범위에서 감당 가능했다.
- **응답 본문 문자열에서 429 를 찾는 방식**은 조잡하지만, OCR Lambda 가 예외를
  `{"success": false, "error": "Upstage API error (429): ..."}` 로 평평하게 내려서
  구조화된 에러 코드가 없었다. Lambda 를 고치는 게 정답인데 **그러지 않고 문자열을 봤다.**

## 왜 다른 건 안 썼나

**동시 요청 수를 2~3개로 제한 (세마포어)**

Upstage 의 정확한 rate limit 수치를 **확인하지 못했다.** 문서에서 못 찾았고 실측도 안 했다.
한도를 모르는 상태에서 "2개는 괜찮을 것"이라고 가정하는 것보다 1개로 두는 게 안전했다.
**한도를 알아낼 시간을 안 쓴 것이 실제 이유다.**

**Lambda 안에서 여러 페이지를 한 번에 처리**

Lambda 한 번 호출로 N페이지를 돌리면 Spring 쪽 루프가 사라진다.
하지만 Lambda 타임아웃(기본 설정 기준) 안에 N장이 다 끝나야 하고, **중간 실패 시 전부 잃는다.**
한 페이지 단위가 재시도 단위로 더 낫다.

**SQS 로 페이지를 fan-out**

나중에 [serverless-uptime-monitor](../../serverless-uptime-monitor/decisions/01-헬스체크를-enumerator와-worker로-분리.md)
에서 실제로 택한 구조다. 여기선 **rate limit 이 병목**이라 fan-out 이 문제를 악화시킨다.
일감을 쪼개도 외부 API 한도는 그대로다.

## 결과

**병렬 설계의 잔재가 코드에 남았다.**

```java
@Bean(name = "ocrExecutor")
public Executor ocrExecutor() {
    return Executors.newFixedThreadPool(5);   // 5개짜리 풀인데
}

// 실제로는 한 번에 하나만 제출하고 바로 join
processOcrPageAsync(files.get(i), ...).join();
```

`CompletableFuture` + 전용 스레드풀 5개가 있는데 **동시성이 1**이다.
스레드 풀은 사실상 "요청 스레드 밖에서 실행한다"는 역할만 한다.

**읽는 사람이 병렬이라고 오해할 코드**를 남긴 게 이 결정의 실제 비용이다.
지금이라면 `.join()` 옆에 이유를 주석으로 붙이거나 풀 크기를 1로 줄였을 것이다.
(주석은 루프 위에 한 줄 있지만, `Executor` 선언부에는 없다.)

## 배운 점

**설계를 되돌릴 때 배선까지 되돌려야 한다.**
병렬 → 순차로 판단을 바꿨는데 **구조는 병렬인 채로 실행만 순차**가 됐다.
코드를 읽는 다음 사람(혹은 3주 뒤의 나)이 "왜 병렬인데 안 빠르지"를 다시 조사하게 된다.

같은 종류의 잔재를 [결정 03](03-DB-쓰기를-워커에게-넘겼다.md) 의 죽은 메서드 5개에도 남겼다.
**판단을 바꾼 기록은 커밋에만 있고 코드에는 이전 판단이 남아 있는** 패턴이 이 프로젝트에 반복된다.

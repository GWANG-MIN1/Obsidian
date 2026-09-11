# 10. 챗봇만 OpenAI 로, 분석은 Bedrock 으로

> 메모: 같은 서비스에 LLM 제공자가 둘이다. 기준은 "정확성 우선"과 "즉응성 우선".
> 그리고 **챗봇은 백엔드를 안 지나간다.** 2026-05-18 ~ 05-21

---

## 상황

계약서 분석은 이미 Bedrock Claude 3.5 Sonnet 으로 돌고 있었다.
여기에 챗봇("아르디")을 붙여야 했는데, 두 작업의 성질이 달랐다.

| | 분석 | 챗봇 |
|---|---|---|
| 한 번 | 계약서 전문 1회 | 대화 턴마다 |
| 요구 | **정확성** (틀리면 해롭다) | **즉응성** (기다리면 답답하다) |
| 출력 | 구조화 JSON | 자연어 스트림 |
| 실패 시 | 재분석 | 다시 물어보면 됨 |

## 결정

**분석은 Bedrock, 챗봇은 OpenAI `gpt-4o-mini`.** 그리고 챗봇은 **Next.js Route Handler 안에** 뒀다.

```
분석   Spring → bedrock_lambda → Claude 3.5 Sonnet (+ KB 하이브리드 검색 + RRF)
챗봇   브라우저 → Next.js /api/chatbot (route.ts)
              ├─ Spring POST /api/chatbot/retrieve  → bedrock_lambda (mode=retrieve) → KB
              └─ OpenAI gpt-4o-mini  (stream)
```

즉 **KB 검색은 Bedrock 을 쓰고, 답변 생성만 OpenAI** 다.
검색을 위해 `bedrock_lambda` 에 `mode` 분기를 추가했다.

```python
def lambda_handler(event, context):
    if str(event.get("mode") or "").strip().lower() == "retrieve":
        return handle_retrieve_only(event)   # LLM 호출 없이 KB 검색만
```

Spring 쪽은 얇은 통로 하나다 (`ChatbotRetrievalService`) — **기존 분석 Lambda 를 재사용**해서
새 Lambda 를 배포하지 않았다.

```java
InvokeResponse response = lambdaUtil.invokeAndWait(
        awsConfig.getLambda().getAnalysisFunctionName(), payload);   // ← 같은 함수
```

## 왜

- **스트리밍 UX.** 답변이 한 번에 뜨는 것과 타이핑처럼 흐르는 것의 체감 차이가 컸다.
  OpenAI SDK 의 스트리밍이 그때 손에 익어 있었고, Next.js `ReadableStream` 으로
  바로 흘려보낼 수 있었다.
- **비용.** 대화는 호출 횟수가 분석의 수십 배다. `gpt-4o-mini` 는 Claude 3.5 Sonnet 보다
  훨씬 싸다. *"대화라는 일반 도메인에 굳이 비싼 모델을 안 쓴다."*
- **역할 분리가 실패 격리이기도 하다.** 챗봇이 죽어도 분석은 돌고, 반대도 성립한다.
  실제로 제공자가 달라서 **OpenAI 장애가 분석에 영향을 주지 않는다.**
- **챗봇을 프론트에 둔 이유**: 스트리밍을 Spring 을 한 번 더 거치게 하면 **버퍼링**이 생긴다.
  그리고 Render 백엔드는 콜드스타트로 30초를 잘 자는데, 챗봇까지 거기 걸면
  첫 질문이 매번 타임아웃이었다.
- **KB 검색만 백엔드를 지나가게 한 이유**: AWS 자격증명을 **프론트에 두지 않기 위해서**다.
  브라우저에서 Bedrock 을 직접 부르려면 키가 노출된다.

## 왜 다른 건 안 썼나

**챗봇도 Bedrock Claude 로**

같은 제공자로 통일하면 IAM·로깅·비용이 한 곳에 모인다.
안 쓴 이유는 위의 스트리밍·비용이고, 추가로 **Bedrock `converse` 스트리밍을 써 본 적이 없었다.**
`converse` 는 분석에서 비스트리밍으로만 쓰고 있었다.
(나중에 [Bedrock Converse 도구 호출 루프](../../../infra-lab/aws-lab/Bedrock-Converse-도구-호출-루프.md) 를
따로 공부한 게 이 공백 때문이다.)

**챗봇 라우트를 Spring 에 만들기 (`POST /api/chatbot/chat`)**

백엔드 담당인 내 입장에서 자연스러운 선택이었다. 안 한 이유가 셋이다.
(가) Spring 에서 SSE/스트리밍을 프론트까지 끊김 없이 흘리려면 배선이 하나 더 늘고,
(나) Render 백엔드 콜드스타트가 챗봇 첫 응답을 30초로 만들고,
(다) OpenAI Java SDK 를 새로 익혀야 했다.

**결과적으로 백엔드 담당이 만든 챗봇이 백엔드 밖에 있다.** 이게 나중에 대가를 치른다 —
Supabase 이전 때 **Spring 은 통째로 교체됐지만 챗봇 라우트는 그대로 살아남았다.**
→ [트러블 01](../troubleshooting/01-Spring이-전시-5일-전에-교체됐다.md)

## 결과 — 비대칭 하나

**같은 KB 를 쓰는데 검색 품질이 다르다.**

| | 분석 경로 | 챗봇 경로 (`handle_retrieve_only`) |
|---|---|---|
| 검색 타입 | `HYBRID` (BM25+벡터), 미지원 시 `SEMANTIC` 폴백 | **기본값** (설정 안 함) |
| 리랭킹 | 선택적 (`BEDROCK_RERANK_MODEL_ARN`) | **없음** |
| 다중 쿼리 RRF | 있음 (LLM 이 쿼리 N개 생성 → RRF 병합) | **없음** |
| top-K | 5 | 4 |
| 계약유형 필터 | 있음 | 있음 |

챗봇 경로는 **단일 쿼리 · 리랭킹 없음**이다. 같은 질문을 분석 파이프라인에 넣으면
더 좋은 근거가 나올 수 있는데, 챗봇은 그 경로를 안 쓴다.
의도한 타협이라기보다 **`handle_retrieve_only` 를 빨리 만들려고 최소 구성으로 짠 것**이다.

**그리고 Gemini 경로는 "런타임 폴백"이 아니라 "제공자 선택"이다.**
모델을 하나 더 붙여 둔 것은 시연 중 Bedrock 이 막히는 상황을 대비한 것인데,
전환 지점이 **런타임이 아니라 환경변수**다.

```python
def get_llm_provider() -> str:
    return os.getenv("LLM_PROVIDER", "").strip().lower() or "bedrock"

# analyze_contract_with_bedrock
if provider == "gemini":
    ...   # Gemini 경로 (모델 2개 · 쿼터 에러 시 보조 모델로, 재시도 3회)
response = runtime_client.converse(...)   # Bedrock 경로 — try/except 없음
```

즉 Bedrock 이 throttling 되면 **그대로 예외가 나고 `lambda_handler` 가 `success: false`** 를 돌려준다.
Gemini 로 자동으로 넘어가지는 않고, **`LLM_PROVIDER=gemini` 로 바꿔 다시 띄워야** 한다.
런타임 재시도가 실제로 걸려 있는 곳은 *Gemini → Gemini 보조 모델*(쿼터 에러 시)뿐이다.

**대비 수단으로서는 이게 충분했다** — 전시 중에 환경변수 하나만 바꾸면 되고,
Lambda 는 재배포 없이 설정 변경만으로 다음 콜드스타트부터 반영된다.
다만 *"장애가 나면 코드가 알아서 넘어간다"* 와는 다른 물건이고, 그 차이를 기록해 둔다.

## 배운 점

**"역할별로 다른 모델"은 좋은 결정인데, 관측이 두 배가 된다.**
제공자가 둘이면 장애도 두 종류, 비용도 두 곳, 프롬프트도 두 곳이다.
이 프로젝트는 **어느 쪽도 관측하지 않았다** — 토큰 사용량을 집계하지 않았고
비용은 청구서로 확인하지 않았다. 같은 누락을
[serverless-uptime-monitor](../../serverless-uptime-monitor/README.md) 에도 적어 뒀다.

**그리고 "빠르게 붙이려고 최소 구성으로" 만든 경로는 그대로 굳는다.**
챗봇 검색에 하이브리드·리랭킹·RRF 를 안 넣은 건 임시였는데, 프로젝트가 끝날 때까지 그대로였다.

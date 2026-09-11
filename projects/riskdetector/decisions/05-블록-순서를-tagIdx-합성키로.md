# 05. OCR 블록 순서를 tagIdx 합성키로

> 메모: `tagIdx = pageIdx * 100 + elIdx`. 한 정수로 페이지와 순서를 같이 담았다.
> **100블록에서 깨진다.** 2026-04-02

---

## 상황

OCR 결과는 페이지마다 **HTML 블록 배열**로 온다 (`<h1>`, `<p>`, `<table>` …).
이걸 `ocr_content` 테이블에 한 행씩 저장해야 하는데, 두 가지 순서가 동시에 필요했다.

1. **화면 표시 순서** — 페이지 1의 블록들 → 페이지 2의 블록들
2. **분석 입력 순서** — 분석 Lambda 에는 **페이지 단위 문자열 배열**로 넘겨야 한다

그리고 독소조항 결과가 `sourceContractTagIdx` 로 **원문 블록을 역참조**한다.
즉 이 키는 저장·조회·분석·하이라이트 네 곳에서 쓰인다.

## 결정

**페이지 번호와 요소 번호를 하나의 정수로 합성했다.**

```java
// OcrProcessService — tagIdx: 페이지 순서 * 100 + 요소 순서
.tagIdx(result.getPageIdx() * 100 + elIdx)
```

되돌릴 때는 나누기로 페이지를 복원한다.

```java
// AnalysisProcessService
private List<String> groupByPage(List<OcrContent> ocrContents) {
    return ocrContents.stream()
            .collect(Collectors.groupingBy(
                    c -> c.getTagIdx() / 100,          // ← 페이지 복원
                    TreeMap::new,                      // ← 페이지 순서 보장
                    Collectors.mapping(OcrContent::getContent, Collectors.joining("\n"))
            ))
            .values().stream().collect(Collectors.toList());
}
```

조회는 `findByContractIdOrderByTagIdx` 하나로 끝난다 — 정렬 컬럼이 하나니까.

## 왜

- **컬럼 하나로 전역 정렬이 된다.** `ORDER BY tag_idx` 만으로 "1페이지 3번째 블록"이
  "2페이지 1번째 블록"보다 앞에 온다. `(page, element)` 두 컬럼이면 복합 정렬이 필요하다.
- **프론트에 정수 하나만 넘기면 된다.** 독소조항이 `sourceContractTagIdx: 102` 를 주면
  프론트는 그 값으로 블록을 찾아 하이라이트한다. 튜플을 주고받을 필요가 없다.
- **100 은 "한 페이지에 블록이 100개 넘지 않는다"는 가정**이다. 계약서 한 장의
  문단·표·제목을 합쳐 100개를 넘는 걸 못 봤다.

## 왜 다른 건 안 썼나

**`page_idx` 와 `element_idx` 를 별도 컬럼으로**

**이게 맞는 선택이었다.** 한도가 없고, 의미가 스키마에 드러나고, 정렬은
`ORDER BY page_idx, element_idx` 다. 안 쓴 이유는 그때 `ocr_content` 테이블에
컬럼을 하나만 더 두고 싶었다는 것 — **근거라고 부를 만한 게 아니다.**

**전역 시퀀스(0, 1, 2, …)로 매기고 페이지는 별도 컬럼**

정렬은 되지만 분석 입력을 만들 때 페이지를 다시 묶어야 하고, 그러면 어차피 페이지 컬럼이 필요하다.

**소수점이나 문자열 키 (`"01-003"`)**

정렬은 되는데 `sourceContractTagIdx` 가 프론트·Lambda·DB 세 곳을 지나야 해서
**정수가 아니면 직렬화가 번거로워진다.**

## 결과 — 아직 안 고친 버그

**한 페이지에 블록이 100개를 넘으면 다음 페이지 그룹으로 새어 들어간다.**

```
1페이지 블록 102개 →  tagIdx = 0..101
                      tagIdx 100, 101 은 100 으로 나누면 → 페이지 "1"
2페이지 첫 블록     →  tagIdx = 100  ← 충돌
```

두 가지가 동시에 깨진다.
- **`tagIdx` 가 중복**된다 (1페이지 100번째 블록과 2페이지 0번째 블록이 둘 다 `100`)
- `groupByPage` 가 그 블록들을 **틀린 페이지로 묶는다**

`sourceContractTagIdx` 로 원문을 찾는 하이라이트도 **엉뚱한 블록을 가리킨다.**

지금까지 티가 안 난 이유는 **계약서 한 장이 블록 100개를 넘은 적이 없기 때문**이다.
고치려면 `100` 을 `1000` 으로 올리는 임시 조치보다 **페이지·요소를 두 컬럼으로 분리**하는 게 맞다.
그러려면 DB, JPA 엔티티, 로더의 `CREATE TABLE`, 프론트의 역참조를 **네 곳 다** 고쳐야 한다.
→ [결정 03](03-DB-쓰기를-워커에게-넘겼다.md) 의 "스키마 정의가 두 곳"

## 배운 점

**"이 값이 N을 안 넘는다"는 가정은 코드에 안 적히면 사라진다.**
`* 100` 은 그냥 숫자라 읽는 사람이 한도를 알 수 없다. 최소한
`MAX_ELEMENTS_PER_PAGE = 100` 같은 상수로 이름을 붙였으면 다음 사람이 물어봤을 것이다.

**그리고 한도를 넘겼을 때 에러가 아니라 조용히 틀린다.** 이게 더 나쁘다 —
예외가 나면 알 수 있지만 이건 **하이라이트가 한 칸 밀린 채로 정상 동작**한다.
[serverless-uptime-monitor 트러블 02](../../serverless-uptime-monitor/troubleshooting/02-같은-페이지네이션-버그를-세-번.md)
의 *"에러 없이 잘리는 종류의 버그"* 와 같은 부류다.

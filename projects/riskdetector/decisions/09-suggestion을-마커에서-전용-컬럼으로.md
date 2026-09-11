# 09. suggestion 을 마커에서 전용 컬럼으로

> 메모: `reason_reference` 안에 `[RD_SUGGESTION]` 을 붙여 두 값을 한 컬럼에 넣고 있었다.
> 전용 컬럼을 만들었는데 **마커 경로를 지우지 않았다.** 2026-05-16, 세 커밋 38초

---

## 상황

분석 결과의 독소조항 하나는 이런 필드를 갖는다.

| 필드 | 내용 |
|---|---|
| `clause` | 문제가 되는 원문 |
| `reason` | 왜 위험한가 |
| `reason_reference` | 근거 법령·판례 id |
| **`suggestion`** | **어떻게 고치면 되는가** |

`suggestion` 이 나중에 추가된 요구였고, **테이블에 컬럼이 없었다.**
그래서 로더가 `reason_reference` 하나에 두 값을 마커로 구분해 밀어 넣고 있었다.

```python
SUGGESTION_MARKER = "[RD_SUGGESTION]"

def build_reason_reference(item):
    source_text = normalize_source_ids(item.get("sourceIds") or ...)
    suggestion = str(item.get("suggestion", "") or item.get("after", "") or "").strip()
    if not suggestion:
        return source_text
    if not source_text:
        return f"{SUGGESTION_MARKER}{suggestion}"
    return f"{source_text}\n{SUGGESTION_MARKER}{suggestion}"
```

프론트는 이걸 문자열에서 다시 쪼갰다. **DB 를 텍스트 봉투로 쓰고 있었던 것이다.**

## 결정

**전용 컬럼 `suggestion` 을 만들고, 마커 경로는 폴백으로 남겼다.**

2026-05-16 15:50~15:51, 세 커밋으로 관통시켰다.

| 커밋 | 계층 | 한 것 |
|---|---|---|
| `43ffef3` | JPA 엔티티 | `ToxicClause.suggestion` 추가 (`@Column(columnDefinition = "TEXT")`) |
| `5b4fded` | DTO | `ToxicDto.suggestion` 추가 + API 응답에 포함 |
| `2e1fe8f` | Lambda 로더 | 컬럼 생성 + `INSERT` 에 값 추가 |

로더 쪽이 중요하다. **`CREATE TABLE IF NOT EXISTS` 만으로는 컬럼이 안 생겼다.**

```python
cur.execute(f"""
    CREATE TABLE IF NOT EXISTS {TOXICS_TABLE} (
        ..., suggestion TEXT, ...
    )
""")
# 이미 존재하는 테이블에는 위 구문이 아무것도 하지 않으므로
cur.execute(f"""
    ALTER TABLE {TOXICS_TABLE}
    ADD COLUMN IF NOT EXISTS suggestion TEXT
""")
```

프론트는 **전용 컬럼을 우선**하고 마커를 폴백으로 쓴다.

```ts
function normalizeToxic(toxic?: ToxicSlim): ToxicSlim | undefined {
  if (!toxic) return undefined;
  const parsed = splitReasonReference(toxic.reasonReference);
  return {
    ...toxic,
    reasonReference: parsed.reference || toxic.reasonReference,
    suggestion: hasField(toxic.suggestion) ? toxic.suggestion : parsed.suggestion || toxic.suggestion,
  };
}
```

## 왜

- **`ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 를 로더 안에 넣은 이유**: 마이그레이션 도구가
  없었다. Flyway·Liquibase·Alembic 어느 것도 안 썼고, 운영 DB 는 이미 데이터가 들어 있었다.
  **로더가 매 호출마다 스키마를 보정하는** 형태가 된다. 매번 도는 DDL 이라 비용이 붙지만
  (`IF NOT EXISTS` 는 존재 확인만 하고 끝난다) 시연 규모에서 무시할 수 있었다.
- **마커 폴백을 남긴 이유**: 컬럼을 만들기 **전에 저장된 행**들은 `suggestion` 이 `NULL` 이고
  값은 `reason_reference` 안에 들어 있다. 그 행들을 백필하지 않았으므로 읽는 쪽이 둘 다 봐야 한다.
- **세 계층을 38초 안에 관통한 이유**: 엔티티만 바꾸면 조회가 `NULL` 을 주고,
  로더만 바꾸면 API 가 그 값을 안 내려준다. **어느 하나만 배포되면 기능이 조용히 반쪽**이 된다.

## 왜 다른 건 안 썼나

**마커 방식을 그대로 유지**

동작은 한다. 안 유지한 이유는 (가) `reason_reference` 를 그냥 화면에 뿌리면
`[RD_SUGGESTION]` 이 사용자에게 보인다, (나) 근거 법령으로 **검색·필터**를 걸 수 없다,
(다) 마커 문자열이 원문에 우연히 들어오면 파싱이 깨진다.

**마이그레이션 도구 도입 (Flyway)**

맞는 선택이지만 두 가지가 걸렸다. 쓰기 주체가 **Lambda(Python)** 인데
Flyway 는 Spring 쪽에 붙는다 → [결정 03](03-DB-쓰기를-워커에게-넘겼다.md).
스키마를 Spring 이 관리하고 쓰기는 Lambda 가 하면 **버전 관리와 실제 쓰기가 갈라진다.**
그리고 시연까지 3주였다.

같은 문제를 나중에 [gym-management-db 결정 03](../../gym-management-db/decisions/03-Alembic-도입과-기존-DB-편입.md)
에서는 **Alembic 을 도입하고 기존 DB 를 편입**하는 쪽으로 풀었다. 여기선 안 했다.

**기존 행을 백필하고 마커 코드를 제거**

`UPDATE ... SET suggestion = substring(reason_reference from ...)` 한 번이면 됐을 것이다.
**안 한 이유가 없다. 그냥 안 했다.**

## 결과

**같은 값이 지금 두 곳에 쓰인다.**

```python
# replace_toxic_clauses — 같은 값을 두 번 넣는다
str(item.get("reason", "")),
build_reason_reference(item),                                       # ← reason_reference 에 마커로
str(item.get("suggestion", "") or item.get("after", "") or ""),     # ← suggestion 컬럼에
```

전용 컬럼이 생긴 뒤에도 `build_reason_reference` 가 마커를 계속 붙인다.
마커 파싱 코드도 프론트 **두 파일**(`api/chatbot/route.ts`, `analysis/result/page.tsx`)에 살아 있다.

즉 **하위호환 경로를 만들어 두고, 호환이 필요 없어진 뒤에도 안 걷었다.**
지금 이걸 지우려면 (가) 옛 행 백필, (나) 로더의 마커 부착 제거, (다) 프론트 두 곳 파싱 제거 —
세 단계가 필요하고, **어느 행이 옛 행인지 알 방법이 없다** (플래그를 안 남겼다).

## 배운 점

**폴백에는 유효기간을 같이 적어야 한다.**
"옛 행을 위해 남긴다"는 결정은 옳았는데, *언제 지울 수 있는가* 를 안 적어서 영구화됐다.
백필을 같이 했으면 폴백은 그날 지울 수 있었다.

**그리고 마이그레이션 도구가 없으면 DDL 이 애플리케이션 코드로 스며든다.**
`ADD COLUMN IF NOT EXISTS` 가 로더 실행 경로에 들어간 순간,
**스키마 변경 이력이 git 커밋에만 존재**하고 DB 에는 버전 개념이 없어졌다.
어떤 컬럼이 언제 생겼는지 알려면 코드 히스토리를 읽어야 한다.

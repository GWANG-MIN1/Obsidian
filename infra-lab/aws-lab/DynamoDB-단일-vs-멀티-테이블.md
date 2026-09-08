# DynamoDB 단일 vs 멀티 테이블 · 키 설계

> 메모: single-table design이 정설인데 접근 패턴이 도메인별로 깔끔하면 멀티가 낫다. 그리고 PK가 무엇이냐가 가능한 질의를 결정한다

---

## 개념

DynamoDB 커뮤니티의 정설은 **single-table design** — 모든 엔티티를 한 테이블에
PK/SK 오버로딩으로 욱여넣는 방식이다. 조인 흉내, 트랜잭션, 핫파티션 분산에 유리하다.

그런데 **접근 패턴이 테이블 경계와 1:1로 맞으면 멀티가 더 단순하다.**

| | 단일 테이블 | 멀티 테이블 |
|---|---|---|
| 여러 엔티티 한 번에 조회 | ✅ 한 Query | ❌ 앱에서 조합 |
| 크로스 도메인 트랜잭션 | ✅ 같은 테이블 | 여러 테이블에 걸침 |
| 핫파티션 분산 | ✅ 유리 | 테이블별 |
| **접근 패턴이 단순할 때 가독성** | PK/SK 규약을 계속 들고 있어야 | ✅ 테이블 이름이 곧 도메인 |
| **IAM 최소권한** | 조건으로 표현해야 (까다로움) | ✅ **테이블 단위로 분리** |
| TTL·용량·백업 설정 | 테이블 전체에 일괄 | ✅ 도메인별로 다르게 |

**IAM이 결정적인 경우가 많다.** 멀티면 "이 Lambda는 Users를 아예 못 본다"가
정책 한 줄로 강제된다. 단일 테이블에서 같은 걸 하려면 조건 키로 행을 걸러야 하는데 훨씬 어렵다.

**대가**: 크로스 도메인 질의를 앱에서 조합해야 하고, `TransactWriteItems`가 여러 테이블에 걸친다.
접근 패턴이 단방향(예: 세션 → 메시지)이면 문제가 안 된다.

> **"single-table이 항상 정답"이 아니다.** 접근 패턴이 도메인별로 깔끔하면 멀티가 읽기 쉽다.
> 판단 기준은 유행이 아니라 **"내 질의가 테이블 경계를 넘나드는가"** 다.

## PK가 가능한 질의를 결정한다

이게 스키마 설계의 핵심이다. **PK를 무엇으로 두느냐가 무엇을 물어볼 수 있는지를 정한다.**

```
PK = session_id   →  "이 세션의 메시지"           ✅ Query
                     "이 유저의 세션 목록"          ❌ Scan 밖에 없다

PK = user_id      →  "이 유저의 세션 목록"         ✅ Query
                     "세션 단건"                   ❌ user_id 를 같이 알아야 한다
```

두 번째를 고르면 **세션을 다루는 모든 경로에 `userId`가 따라다닌다.**
API 입력에도, 내부 호출 payload에도 들어가야 한다. 스키마 결정이 API 시그니처를 바꾼다.

`Scan`은 전체 테이블을 읽는다. 1만 건이면 1만 건을 다 가져온다.
**학습용 외에는 쓰지 않는다.** Scan이 필요해졌다는 건 대개 키 설계가 접근 패턴과 안 맞는다는 신호다.

## 정렬키 — 타임스탬프 단독은 덮어쓴다

`PK + SK`가 같으면 `PutItem`은 **덮어쓴다.** SK를 ISO 타임스탬프로만 두면
같은 밀리초에 들어온 두 번째 항목이 첫 번째를 지운다.

```js
const makeSk = () => `${new Date().toISOString()}#${randomUUID()}`;
// 2026-05-30T07:46:53.761Z#f2d4fb16-6bad-42fe-a60d-60b1bc596879
```

- ISO가 앞이라 **사전순 = 시간순**이 유지된다
- uuid가 뒤에서 고유성을 보장한다
- 타입은 그대로 STRING이라 **테이블 정의를 안 건드린다**
- 콘솔에서 눈으로 읽힌다 (ULID 대비 디버깅 이점)

> 정밀도를 올리는 방식(마이크로초)은 확률만 낮출 뿐 0으로 못 만들고,
> Lambda 인스턴스가 여럿이면 다시 깨진다. **실패를 감지해 고치는 것보다 실패가 불가능한 키를 쓴다.**

## `ScanIndexForward` — 기본값이 "가장 오래된 N개"다

| 값 | 정렬 | `Limit N` 의 의미 |
|---|---|---|
| `true` (**기본값**) | SK 오름차순 | 가장 **오래된** N개 |
| `false` | SK 내림차순 | 가장 **최근** N개 |

채팅 이력처럼 "최근 N개"가 필요한데 기본값을 그대로 쓰면,
**대화가 길어질수록 첫 대화만 컨텍스트로 들어간다.** 이력이 짧으면 티가 안 난다.

```js
const res = await ddb.send(new QueryCommand({
  KeyConditionExpression: "session_id = :s",
  ScanIndexForward: false,        // 최근 N개를 가져오고
  Limit: HISTORY_LIMIT,
}));
const history = res.Items.slice().reverse();   // 쓸 땐 시간순으로 뒤집는다
```

## 실행

```ts
// 도메인별 테이블 + 테이블 단위 최소권한
const users    = new ddb.Table(this, 'Users',    { partitionKey: { name: 'id', type: S } });
const sessions = new ddb.Table(this, 'Sessions', { partitionKey: { name: 'user_id', type: S },
                                                   sortKey:      { name: 'id', type: S } });
const messages = new ddb.Table(this, 'Messages', { partitionKey: { name: 'session_id', type: S },
                                                   sortKey:      { name: 'created_at_id', type: S } });

users.grantReadWriteData(apiFn);          // API 만 유저를 만진다
sessions.grantReadWriteData(apiFn);
messages.grantReadWriteData(apiFn);

messages.grantReadWriteData(workerFn);    // Worker 는 Users 를 못 본다
sessions.grantReadWriteData(workerFn);
```

모든 테이블에 공통으로 둘 것.

```ts
billingMode: ddb.BillingMode.PAY_PER_REQUEST,     // 트래픽 예측이 안 될 때
removalPolicy: cdk.RemovalPolicy.DESTROY,         // 학습용. 실서비스는 RETAIN(기본값)
```

### 자주 밟는 것 두 개

**`undefined` 필드가 있으면 Put이 실패한다.** 한 테이블에 여러 모양의 행을 넣으면
행 종류마다 있는 필드가 다르다. 저장 직전에 걸러낸다.

```js
const clean = Object.fromEntries(Object.entries(row).filter(([, v]) => v !== undefined));
```

**조건부 갱신은 best-effort로 감싼다.** 부모 행이 아직 없을 수 있다.

```js
try {
  await ddb.send(new UpdateCommand({
    Key: { user_id, id },
    ConditionExpression: "attribute_exists(user_id)",   // 없으면 만들지 않는다
    UpdateExpression: "SET updated_at = :t",
  }));
} catch (e) { console.warn("bump skipped:", e.name); }   // 본 작업은 이미 성공했다
```

## 배운 점

**스키마 결정이 API 시그니처까지 바꾼다.** `SessionsTable`의 PK를 `user_id`로 두는 순간
"유저별 목록"이 열리는 대신 **`userId`가 모든 경로에 따라다닌다.**
키 설계는 DB 안의 문제가 아니라 시스템 전체의 문제다.

**"동작한다"와 "맞다"는 다르다.** `ScanIndexForward` 기본값은 두 턴만 테스트하면
잘못된 걸 알 수 없다. **경계에서 확인해야** 드러나는 종류가 있다.

**키 이름을 바꿀 땐 그 키를 참조하는 곳을 전부 같이 본다.** 컬럼명을 바꿨다가
커서 페이지네이션이 조용히 깨진 적이 있다. 정렬키는 응답 커서로도 나가기 때문이다.

**유행하는 설계보다 내 접근 패턴을 먼저 적는다.** single-table을 고르든 멀티를 고르든,
근거는 "커뮤니티가 뭘 권하는가"가 아니라 **"내가 무엇을 물어보는가"** 다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 05](../../projects/aws-serverless-agent/decisions/05-단일-테이블-대신-멀티-테이블.md) ·
  [결정 03](../../projects/aws-serverless-agent/decisions/03-정렬키를-타임스탬프-합성으로.md)
- 관계형 쪽 스키마 개념: [`../db-lab/`](../db-lab/README.md)

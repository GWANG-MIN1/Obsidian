# 09. Discord 개발자 포털이 Endpoint URL 을 저장하지 못한다

- **발생**: 2026-06-09 (Day 18) · 저장소 함정 **#53 · #54 · #58**
- **증상 한 줄**: Interactions Endpoint URL 을 넣고 저장하면 검증 실패로 거부

---

## 현상

Day 18 에서 Discord Interactions 를 붙였다
([결정 11](../decisions/11-Telegram-대신-Discord.md)).
개발자 포털에 Lambda Function URL 을 Interactions Endpoint URL 로 넣고 저장을 누르면 거부된다.

Discord 는 저장 전에 **검증 PING** 을 보낸다. 그 PING 에 제대로 응답해야 URL 이 등록된다.
검증이 실패하면 저장 자체가 안 된다.

## 환경

- Discord Interactions (웹훅 방식, 게이트웨이 봇 아님)
- Lambda Function URL, Node.js 20, `node:crypto` — **외부 의존성 0**
- Ed25519 서명 검증

## 진단

### 1) 서명 검증이 뭘 요구하는지 확인한다

Discord 는 모든 요청에 두 헤더를 붙인다.

```
X-Signature-Ed25519:   <서명>
X-Signature-Timestamp: <타임스탬프>
```

검증 대상은 **`timestamp + raw body`** 를 이어붙인 문자열이다.
애플리케이션의 Public Key(hex)로 Ed25519 검증한다.

### 2) `node:crypto` 가 공개키를 못 읽는다 (#54)

Discord 가 주는 Public Key 는 **raw ed25519 32바이트**(hex 문자열)다.
`crypto.createPublicKey()` 는 이 형식을 그대로 못 받는다. SPKI DER 를 원한다.

접두어를 붙여 감싸야 한다.

```js
const SPKI_PREFIX = Buffer.from("302a300506032b6570032100", "hex");
const key = crypto.createPublicKey({
  key: Buffer.concat([SPKI_PREFIX, Buffer.from(publicKeyHex, "hex")]),
  format: "der",
  type: "spki",
});
```

`302a300506032b6570032100` 은 "이건 ed25519 공개키다"를 뜻하는 ASN.1 헤더다.
라이브러리(`tweetnacl` 등)를 쓰면 이 과정이 감춰지는데,
[의존성 0 을 택했으므로](../decisions/11-Telegram-대신-Discord.md) 직접 감싸야 했다.

### 3) 검증 대상 body 가 원본이 아니었다 (#58)

키 문제를 고쳤는데 여전히 실패한다.

Lambda Function URL 은 요청 본문을 **base64 로 인코딩해서 줄 수 있다** (`isBase64Encoded: true`).
그리고 핸들러에서 습관적으로 이렇게 했다.

```js
const body = JSON.parse(event.body);
// ... 검증할 때
verify(timestamp + JSON.stringify(body))    // ← 여기서 깨진다
```

**파싱했다가 다시 직렬화하면 원본과 바이트가 달라진다.**
키 순서, 공백, 유니코드 이스케이프, 숫자 표기 — 하나만 달라도 서명이 안 맞는다.

서명은 **바이트에 대한** 것이지 의미에 대한 것이 아니다.

## 원인

세 겹이었다.

**#54 — raw ed25519 키를 SPKI DER 로 안 감쌈.** `node:crypto` 가 못 읽는다.

**#58 — 검증 대상이 raw body 가 아니었다.** base64 디코딩 처리 누락 +
파싱 후 재직렬화로 바이트가 변형됐다.

**#53 — 그 결과 PING 검증이 실패했다.** 포털이 URL 을 저장하지 못한 겉 증상.

## 해결

```js
// 1) raw 문자열을 먼저 확보한다 — 파싱 전에
const raw = event.isBase64Encoded
  ? Buffer.from(event.body, "base64").toString("utf8")
  : event.body;

// 2) raw 로 검증한다
const sig = event.headers["x-signature-ed25519"];
const ts  = event.headers["x-signature-timestamp"];
const ok  = crypto.verify(null, Buffer.from(ts + raw), key, Buffer.from(sig, "hex"));
if (!ok) return { statusCode: 401, body: "invalid signature" };

// 3) 검증 통과 후에 파싱한다
const body = JSON.parse(raw);
if (body.type === 1) return json({ type: 1 });     // PING → PONG
```

**순서가 핵심이다** — raw 확보 → 검증 → 파싱.
파싱을 먼저 하면 원본을 잃는다.

그리고 PING 응답은 `{ type: 1 }` 이다. 이게 PONG 이다.

## 재발 방지

- **서명 검증이 있는 웹훅은 raw body 를 먼저 확보한다.**
  파싱은 검증 뒤다. 프레임워크가 body 를 자동 파싱하면 raw 를 따로 보존하는 설정이 필요하다.
  (Express 의 `express.raw()`, Hono 의 `c.req.text()` 등)
- **Function URL / API GW 의 `isBase64Encoded` 를 항상 확인한다.**
  content-type 에 따라 켜지고 꺼져서, 테스트로는 재현이 안 되는 경우가 있다.
- **의존성을 안 쓰기로 했으면 그 라이브러리가 감춰 주던 걸 알아야 한다.**
  SPKI DER 래핑이 그것이었다. 의존성 0 은 공짜가 아니다.
- **Public Key 를 hex 그대로 정확히 옮긴다.** 앞뒤 공백이나 잘린 문자로도 실패한다.

## 배운 점

**서명 검증에서 "같은 데이터"는 "같은 바이트"다.**
`JSON.parse` → `JSON.stringify` 는 의미를 보존하지만 바이트를 보존하지 않는다.
암호학적 검증에서는 그게 다르다는 걸 몸으로 배웠다.

**저장이 안 되는 것도 증상이다.** 포털 UI 가 "저장 실패"만 보여줘서
처음엔 Discord 쪽 문제인 줄 알았다. 실제로는 **내 Lambda 가 PING 에 답을 못 한 것**이고,
CloudWatch 로그를 보니 요청은 오고 있었다. **호출 로그를 먼저 봤으면 빨랐다.**

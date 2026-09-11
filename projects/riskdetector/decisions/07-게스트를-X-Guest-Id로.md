# 07. 게스트를 X-Guest-Id 헤더로 — 로그인 없이 쓰게

> 메모: 전시장에서 심사위원에게 "먼저 로그인하세요"라고 할 수 없었다. 2026-03-31 ~ 05-28

---

## 상황

이 서비스의 첫 화면은 **계약서 업로드**다. 거기서 로그인을 요구하면
*"내 계약서를 남한테 올려도 되나"* 를 고민하기 전에 이미 이탈한다.
그리고 시연에서 심사위원이 직접 써 보게 하려면 **Google 로그인 화면이 중간에 끼면 안 됐다.**

그런데 업로드한 계약서는 분석이 끝난 뒤 **다시 조회**해야 하고,
"내 계약서 목록"도 보여 줘야 한다. 로그인이 없으면 **무엇을 기준으로 소유자를 구분**하는가.

## 결정

**브라우저가 만든 게스트 id 를 `X-Guest-Id` 헤더로 받고, 계약 행에 같이 저장했다.**

```
Contract
  ├─ user              (로그인 사용자면 FK, 게스트면 null)
  └─ guestSessionId    (게스트면 값, 로그인 사용자면 null)
```

```java
// OcrProcessService.processUpload
User user = isGuest(email) ? null : userRepository.findByEmail(email)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));
String effectiveGuestSessionId = user == null && StringUtils.hasText(guestSessionId)
        ? guestSessionId.trim()
        : null;
```

CORS 에 헤더를 허용 목록으로 넣어야 했다.

```java
config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With", "X-Guest-Id"));
```

조회할 때는 **둘 중 하나로만** 일치를 본다 → [결정 06](06-인가를-서비스-계층에서.md) 의 `hasAccess()`.

## 왜

- **쿠키 대신 헤더**를 쓴 이유: 프론트와 백엔드가 다른 도메인이라 쿠키는
  `SameSite=None; Secure` 가 필요하다([결정 08](08-JWT를-쿠키와-URL-양쪽으로.md)).
  게스트 식별자에까지 그 조건을 걸고 싶지 않았고, 헤더는 **프론트가 명시적으로 붙이는 것**이라
  어디서 오는 값인지 코드에서 보인다.
- **`user` 와 `guestSessionId` 를 한 테이블에 nullable 로 둔 이유**: 게스트 계약과
  회원 계약을 별 테이블로 나누면 조회·분석·삭제 경로가 전부 두 배가 된다.
  "소유자 컬럼이 둘 중 하나"가 쿼리에서는 `WHERE user_id = ? OR guest_session_id = ?` 하나다.
- **로그인 전환을 안 만들었다.** 게스트로 올린 계약서를 로그인 후 가져오는 기능은 없다.
  전시 시나리오에 없었고, 만들면 *"이 게스트 id 가 정말 이 사용자 것인가"* 를 증명할 방법이
  없기 때문이다 — 헤더는 아무나 보낼 수 있다.

## 왜 다른 건 안 썼나

**게스트에게도 JWT 발급 (익명 토큰)**

서버가 서명한 토큰이니 위조가 안 되고, 필터에서 인가를 잠글 수 있다.
안 쓴 이유는 **수명 관리**다. 게스트 토큰이 만료되면 그 사람의 계약서에 영구히 접근 불가가 된다.
localStorage 의 문자열은 브라우저가 지울 때까지 남는다.
(그 대가로 **위조 가능**하다 — 아래 참고.)

**세션 쿠키 (`JSESSIONID`)**

`SessionCreationPolicy.STATELESS` 를 이미 택했고, 인스턴스가 재시작하면 세션이 날아간다.
Render 무료 인스턴스는 **자주 잠들고 깨어난다.**

**IP + User-Agent 해시**

모바일 네트워크에서 IP 가 바뀌고, 같은 카페 Wi-Fi 에서 두 사람이 같은 소유자가 된다.

## 결과 — 남은 구멍

**`X-Guest-Id` 는 서버가 검증하지 않는 값이다.**
다른 사람의 게스트 id 를 알아내면 그 사람의 계약서를 볼 수 있다.
프론트가 UUID 를 만들어 쓰므로 추측은 어렵지만, **서버는 이 값의 출처를 모른다.**

그리고 하위호환 구멍이 하나 더 있다.

```java
// 이전 게스트 계약은 session id가 없을 수 있어 contractId 접근을 유지한다.
if (!StringUtils.hasText(contract.getGuestSessionId())) return true;
```

`guestSessionId` 가 비어 있는 옛 게스트 계약은 **`contractId` 만 알면 누구나** 볼 수 있다.
일부러 남긴 것이고 주석도 있지만, **그 데이터를 정리하거나 마이그레이션하지 않았다.**

**업로드된 계약서의 자동 삭제 정책도 없다.** 계획으로만 두고 안 만들었다 —
S3 라이프사이클도, DB 정리 배치도 없다. 게스트가 올린 계약서는 지워질 방법이 없이 남아 있다.

## 배운 점

**"로그인 없이 쓰게 한다"는 UX 결정이 보안 모델을 바꾼다.**
헤더 하나를 추가한 것처럼 보이지만, 실제로는
*서버가 검증할 수 없는 식별자를 소유권 판단에 쓰기로* 정한 것이다.

이 프로젝트에서 개인정보에 가장 신경 쓴 부분은 **OCR 마스킹**이었는데
(계약서 내용을 LLM 에 보내기 전에 사용자가 직접 가린다),
정작 **저장된 계약서의 접근 통제는 헤더 문자열 하나**에 걸려 있었다.
보호의 강도가 층마다 다른 걸 그때는 눈치채지 못했다.

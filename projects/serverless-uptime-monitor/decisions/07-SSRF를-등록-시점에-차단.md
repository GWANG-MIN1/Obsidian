# 07. SSRF 를 등록 시점에 차단 (worker 재검증은 보류)

> 메모: `POST /endpoints` 가 완전 공개다. 아무나 URL 을 넣으면 worker 가 그 URL 로 GET 을 쏜다.
> upgrade-05, 스켈레톤 2026-06-30 → 구현 2026-07-01 커밋 `664c3de`

---

## 상황

이 서비스의 핵심 동작이 그대로 취약점이다.

```
누구나 POST /endpoints 로 URL 등록
  → worker 가 AWS VPC 안에서 그 URL 로 HTTP GET
    → 응답 코드와 지연시간을 저장
      → GET /endpoints/{id}/history 로 누구나 조회
```

**AWS 내부에서 임의 주소로 요청을 보내고 결과를 돌려받는 장치**다. 전형적인 SSRF.
가장 나쁜 대상은 EC2 인스턴스 메타데이터(`169.254.169.254`)지만,
Lambda 는 IMDS 가 없으니 실제 위험은 **사설망 스캔**과 **내부 서비스 호출** 쪽이다.

기존 검증은 스킴과 호스트 존재만 봤다.

```python
def is_valid_url(url):
    parsed = urlparse(url.strip())
    return parsed.scheme in ("http", "https") and bool(parsed.netloc)
```

`http://10.0.0.5`, `http://localhost`, `http://169.254.169.254` 전부 통과한다.

## 결정

**등록 시점에** 호스트를 IP 로 해석해 내부/예약 대역이면 `400` 으로 거부한다.

```python
def _ip_is_blocked(ip_str):
    ip = ipaddress.ip_address(ip_str)
    return (
        ip.is_private
        or ip.is_loopback
        or ip.is_link_local     # 169.254.0.0/16 (IMDS 169.254.169.254 포함)
        or ip.is_reserved
        or ip.is_multicast
        or ip.is_unspecified
    )
```

```python
def is_blocked_host(url):
    host = urlparse(url.strip()).hostname
    if not host:
        return True                         # 호스트가 없으면 막는다
    if _ip_is_blocked(host):                # IP 리터럴이면 바로 검사
        return True
    try:
        infos = socket.getaddrinfo(host, None)
    except socket.gaierror:
        return False                        # 해석 실패는 여기서 판단 안 함
    return any(_ip_is_blocked(info[4][0]) for info in infos)
```

차단하면 `register_blocked` 구조적 로그를 남기고 400 을 돌려준다.

## 왜

- **`ipaddress` 표준 라이브러리의 분류를 쓴다.** 정규식으로 `10.`, `192.168.` 을 걸러내는
  방식은 IPv6, 8진수 표기(`0177.0.0.1`), 예약 대역을 다 놓친다.
  `is_private`/`is_loopback`/`is_link_local`/`is_reserved` 는 표준이 정의한 대역이다.
- **해석되는 IP 를 전부 검사한다** (`any`). 도메인 하나가 여러 A/AAAA 레코드를 가질 수 있고,
  **하나라도 내부면 차단**이다.
- **DNS 해석 실패는 차단하지 않는다.** 존재하지 않는 도메인은 SSRF 가 아니라 그냥 오타다.
  등록은 되고, worker 가 `DNS resolution failed` 로 DOWN 처리한다
  → [트러블 03](../troubleshooting/03-DNS-실패와-타임아웃이-같은-메시지.md)
- **호스트가 없으면(`None`) 차단한다.** 판단 불가는 통과가 아니다 —
  [결정 06](06-살아있음을-status-대신-신선도로.md) 의 "모르는 것을 정상으로 치지 않는다"와 같다.

## 왜 다른 건 안 썼나

**allowlist (등록 가능한 도메인 목록)**

가장 안전하다. 그리고 **이 서비스의 목적을 없앤다** — 임의의 사이트를 감시하는 게 기능이다.

**worker 쪽에서만 검사**

등록은 되는데 체크가 안 되는 상태가 된다. 사용자는 왜 DOWN 인지 모른다.
**거부는 입력 지점에서 즉시** 알려 주는 쪽이 낫다.

**프록시/NAT 로 내부망 접근 자체를 끊기**

네트워크 계층에서 푸는 정공법이다. VPC 에 Lambda 를 넣고 NAT 게이트웨이로만 나가게 하면
사설망 접근이 원천 차단된다. 그런데 **NAT 게이트웨이가 시간당 과금**이고,
이 프로젝트는 "EC2 없이 프리티어로" 가 전제였다. 비용 때문에 안 썼다 —
**보안 결정을 비용으로 타협한 자리**라는 걸 적어 둔다.

## 안 한 것 — worker 요청 직전 재검증

`docs/upgrades/05` 의 체크박스에 **빈칸으로 남아 있다.**

```
- [ ] lambda/health_check_worker/handler.py — do_request 전 재검증(선택)
```

등록 시점에 공인 IP 로 해석되던 도메인이 **나중에 사설 IP 로 바뀌면**(DNS rebinding)
worker 는 그대로 요청한다. 등록 검사는 **그 순간의 DNS 응답**을 본 것이지 계약이 아니다.

지금 이게 구멍인 이유가 하나 더 있다 — **이 서비스는 같은 URL 을 1분마다 영원히 친다.**
한 번 등록해 두면 공격자는 DNS 레코드만 바꾸면 된다. 등록 시점 검사가 특히 약한 형태다.

막으려면 `do_request` 직전에 같은 검사를 하고, 해석한 IP 로 직접 연결해야 한다
(검사한 IP 와 실제 연결 IP 가 달라지는 TOCTOU 를 없애야 하므로).
`urllib` 로는 번거롭고, 그래서 **선택 항목으로 미뤄 뒀다.**

## 배운 점

**입력 검증이 "그 순간에 참"인 것과 "계속 참"인 것을 구분해야 한다.**
스킴 검사는 계속 참이고, DNS 해석 결과는 그 순간에만 참이다.
같은 함수 안에 있다고 같은 강도의 보증이 아니다.

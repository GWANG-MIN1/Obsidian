# 03. DNS 실패와 타임아웃이 같은 메시지로 나왔다

- **발생**: 2026-05-16 · 커밋 `bb48277`
- **증상 한 줄**: Slack 알림의 Error 필드에 `<urlopen error [Errno -2] Name or service not known>` 같은
  파이썬 내부 문자열이 그대로 찍힌다

---

## 현상

1단계에서 만든 `do_request` 는 예외 처리가 두 갈래뿐이었다.

```python
except urllib.error.HTTPError as e:
    return "DOWN", e.code, f"HTTP {e.code}"
except Exception as e:
    return "DOWN", None, str(e)          # ← 나머지 전부 여기로
```

`urllib` 가 던지는 것 중 HTTP 응답이 온 경우만 `HTTPError` 다.
**응답 자체가 없는 실패는 전부 마지막 줄로 떨어진다.**

- 도메인이 없다 (DNS 해석 실패)
- 응답이 10초 안에 안 온다 (타임아웃)
- 연결이 거부됐다

셋 다 `str(e)` 로 나가서 알림을 받는 사람이 **무엇이 잘못됐는지 구분할 수 없다.**
그리고 `HTTPError` 쪽도 `HTTP 500` 까지만 있어서 500 인지 503 인지의 **의미**가 안 보인다.

## 환경

- Lambda(Python 3.12), `urllib.request` (외부 의존성 없음)
- 타임아웃 10초, `User-Agent: uptime-monitor/1.0`

## 진단

`urllib.error.URLError` 는 **원인을 `reason` 에 감싸서** 던진다.
그래서 `URLError` 하나만 잡아서는 부족하고, `reason` 의 타입을 봐야 한다.

| `reason` 타입 | 실제 의미 |
|---|---|
| `socket.timeout` | 응답 시간 초과 |
| `socket.gaierror` | **DNS 해석 실패** (getaddrinfo) |
| `OSError` errno 16 / 111 | 연결 거부 등 |

Python 3.10+ 에서는 `socket.timeout` 이 `TimeoutError` 의 별칭이라
**`URLError` 로 안 감싸이고 그대로 올라오는 경로**도 있다. 그래서 둘 다 잡아야 한다.

## 원인

예외를 **타입이 아니라 뭉뚱그려** 잡았다. `except Exception` 이 서로 다른 실패를
하나의 문자열로 만들어 버렸다.

## 해결

`reason` 을 풀어서 사람이 읽는 문장으로 바꿨다.

```python
except urllib.error.HTTPError as e:
    return "DOWN", e.code, f"HTTP {e.code} {e.reason}"      # 500 → "HTTP 500 Internal Server Error"
except urllib.error.URLError as e:
    reason = e.reason
    if isinstance(reason, socket.timeout):
        return "DOWN", None, "Connection timed out"
    if isinstance(reason, socket.gaierror):
        return "DOWN", None, "DNS resolution failed"
    if isinstance(reason, OSError) and getattr(reason, "errno", None) in (16, 111):
        return "DOWN", None, "DNS resolution failed"
    return "DOWN", None, str(reason)
except TimeoutError:                                        # URLError 로 안 감싸이는 경로
    return "DOWN", None, "Connection timed out"
except Exception as e:
    return "DOWN", None, str(e)                             # 최후의 그물은 남긴다
```

같은 날 저장소 README 에 **Error 필드 해설표**를 넣었다.

| 값 | 뜻 |
|---|---|
| `HTTP 500 Internal Server Error` | 서버 내부 오류 |
| `HTTP 503 Service Unavailable` | 서버 과부하/점검 |
| `Connection timed out` | 응답 시간 초과 (10s) |
| `DNS resolution failed` | 도메인 자체가 존재하지 않음 |

## 재발 방지

- **최후의 `except Exception` 은 남기되, 그 위에 구체적인 분기를 쌓는다.**
  없애면 예상 못 한 예외에서 worker 가 죽는다.
- 이 함수는 나중에 fan-out 재설계 때 `health_check` 에서 `health_check_worker` 로
  **그대로 옮겨졌다.** 옮길 때 분기가 살아 있는지 확인해야 하는 자리다.

## 아직 안 한 것

**TLS 인증서 오류가 구분되지 않는다.** `ssl.SSLCertVerificationError` 는 `URLError.reason` 에
담겨 오는데 위 분기에 없어서 `str(reason)` 으로 나간다. 만료된 인증서는 업타임 모니터가
잡아야 할 대표적인 사건인데 지금은 **원문 문자열**로만 보인다.

**errno 16 을 DNS 실패로 분류한 근거가 약하다.** 16은 `EBUSY`, 111은 `ECONNREFUSED` 다.
둘 다 "DNS resolution failed" 로 보내고 있는데, **연결 거부는 DNS 문제가 아니다.**
Lambda 환경에서 실제로 그렇게 관측된 것을 그대로 옮긴 것으로 보이지만,
**메시지가 원인을 잘못 지목하고 있다.**

## 배운 점

**알림 메시지는 사용자 인터페이스다.** 스택 트레이스나 `str(e)` 를 그대로 흘리면
받는 사람이 매번 번역해야 한다. 새벽에 오는 알림일수록 그렇다.

**예외를 잡는 단위가 곧 구분할 수 있는 실패의 단위다.** `except Exception` 하나로 받으면
그 아래의 모든 실패가 시스템에게는 같은 사건이 된다.

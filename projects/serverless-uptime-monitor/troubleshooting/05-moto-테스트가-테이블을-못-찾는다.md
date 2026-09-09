# 05. moto 테스트가 테이블을 못 찾는다 — import 시점 문제

- **발생**: 2026-06-30 (설계 단계에서 예상) → 2026-07-01 구현 · 커밋 `2c27028` → `331a70c`
- **증상 한 줄**: 핸들러가 `import` 되는 순간 boto3 리소스를 만들어서, moto 를 켜기 전에 배선이 끝나 있다
- **성격**: **밟기 전에 문서에 먼저 적어 둔 함정.** 다른 노트들과 성격이 다르다

---

## 현상

핸들러 파일들이 전부 이렇게 생겼다.

```python
dynamodb = boto3.resource("dynamodb")                      # ← 모듈 최상단
endpoints_table = dynamodb.Table(os.environ["ENDPOINTS_TABLE"])
history_table = dynamodb.Table(os.environ["HISTORY_TABLE"])
ALERT_QUEUE_URL = os.environ["ALERT_QUEUE_URL"]
```

Lambda 에서는 이게 옳다 — **콜드스타트에 한 번만** 만들고 이후 호출에서 재사용한다.

테스트에서는 반대로 걸린다. `import` 하는 순간

1. 환경변수를 읽고 (없으면 `KeyError`)
2. boto3 리소스를 만든다 (moto 밖이면 **실제 AWS 를 가리킨다**)

`@mock_aws` 데코레이터 안에서 테이블을 만들어도, 핸들러가 **그 전에 import 됐다면**
이미 다른 곳을 보고 있다.

`docs/upgrades/04` 가 이걸 **구현 전날 미리 적어 뒀다.**

> 핸들러는 import 시점에 boto3 리소스를 만드므로(conftest 참고),
> 테이블 생성 fixture 를 import 전/후 순서에 맞게 구성해야 한다.

## 환경

- pytest, `moto` (`requirements-dev.txt`)
- 핸들러 6개가 **전부 `lambda/<name>/handler.py`** — 파일 이름이 같다

## 진단

### 1) 환경변수 문제는 이미 풀려 있었다

`tests/conftest.py` 가 **import 되기 전에** 더미 값을 채운다.

```python
os.environ.setdefault("AWS_DEFAULT_REGION", "ap-northeast-2")
os.environ.setdefault("ENDPOINTS_TABLE", "test-endpoints")
os.environ.setdefault("HISTORY_TABLE", "test-history")
os.environ.setdefault("ALERT_QUEUE_URL", "https://sqs.test/alert-queue")
...
```

### 2) 파일 이름 충돌도 이미 풀려 있었다

핸들러가 전부 `handler.py` 라 `import handler` 로는 하나밖에 못 쓴다.
6/20에 테스트를 처음 넣을 때 이미 **경로로 직접 로드**하는 함수를 만들어 뒀다.

```python
def load_handler(name):
    """lambda/<name>/handler.py 를 고유한 모듈 이름으로 불러온다."""
    path = LAMBDA_DIR / name / "handler.py"
    spec = importlib.util.spec_from_file_location(f"{name}_handler", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module
```

### 3) 그 함수가 moto 문제도 같이 푼다

`load_handler` 는 **호출될 때 모듈을 실행한다.** 파일 상단의 `import` 문이 아니다.
즉 **로드 시점을 테스트가 고를 수 있다.**

```python
@mock_aws
def test_create_endpoint_writes_to_dynamodb():
    _create_tables()                  # ① moto 안에서 테이블 먼저 만들고
    register = load_handler("register")  # ② 그다음에 핸들러를 로드한다
```

이 순서라면 핸들러가 `boto3.resource()` 를 부를 때 이미 moto 가 가로채고 있고,
`dynamodb.Table("test-endpoints")` 가 방금 만든 가짜 테이블을 가리킨다.

## 원인

**모듈 최상단에서 외부 리소스를 만드는 코드는 import 시점이 곧 초기화 시점**이다.
테스트가 그 시점을 통제하지 못하면 mock 을 켤 자리가 없다.

## 해결

두 가지를 같이 썼다.

```python
mock_aws = pytest.importorskip("moto").mock_aws   # moto 없으면 모듈 통째로 skip
```

- **테스트마다 `@mock_aws` → `_create_tables()` → `load_handler()`** 순서를 지킨다.
- `pytest.importorskip` 으로 moto 미설치 환경에서 **에러가 아니라 skip** 이 되게 했다.
  스켈레톤 단계(6/30)에서도 `@pytest.mark.skip` 으로 CI 를 안 막았고, 구현하면서 그걸 걷어냈다.
  → [결정 10](../decisions/10-스켈레톤을-먼저-커밋하고-구현은-다음-날.md)

**두 번째 함정 하나가 더 있었다** — moto 는 DNS 를 가로채지 않는다.

```python
with patch.object(register, "is_blocked_host", return_value=False):
    resp = register.create_endpoint(...)
```

`is_blocked_host` 가 `socket.getaddrinfo` 로 **실제 DNS 조회**를 한다
([결정 07](../decisions/07-SSRF를-등록-시점에-차단.md)). moto 는 AWS API 만 가로채므로
이건 그대로 나간다 — 네트워크 없는 CI 에서 느려지거나 실패한다. 그래서 따로 패치했다.

## 재발 방지

- **핸들러를 테스트에서 쓸 때는 항상 `load_handler`.** 파일 상단 `import` 를 쓰지 않는다.
- **mock 컨텍스트 → 리소스 생성 → 핸들러 로드** 순서를 테스트마다 반복한다.
  픽스처로 빼면 순서가 숨어서, 새 테스트를 쓸 때 틀리기 쉽다.
- **AWS 밖으로 나가는 호출은 따로 패치한다.** moto 가 덮는 범위는 AWS API 까지다.

## 대안 — 왜 코드를 안 고쳤나

핸들러가 리소스를 **지연 생성**하게 바꾸면(`get_table()` 같은 함수) 테스트가 편해진다.

안 했다. **Lambda 에서는 최상단 초기화가 맞는 코드**이기 때문이다.
콜드스타트에 한 번 만들고 재사용하는 것이 지연 생성보다 빠르다.
**테스트 편의를 위해 프로덕션 코드를 느리게 만들 이유가 없다** —
테스트 쪽에서 로드 시점을 통제할 수 있으면 그걸로 충분하다.

## 배운 점

**import 는 부작용이 있는 실행이다.** 파이썬에서 모듈 최상단은 "선언"이 아니라 "코드"고,
그 안에서 외부 연결을 만들면 **테스트가 개입할 지점이 사라진다.**

**mock 의 경계를 알아야 한다.** `@mock_aws` 는 AWS 를 가짜로 만들지 DNS 나 HTTP 전체를
가짜로 만들지 않는다. 경계를 모르면 "왜 이 테스트만 CI 에서 느리지"로 시간을 쓴다.

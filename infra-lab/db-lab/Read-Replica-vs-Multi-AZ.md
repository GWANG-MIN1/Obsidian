# RDS Read Replica vs Multi-AZ Standby

> 메모: Replica는 비동기 복제라 직접 쿼리 가능(읽기 분산), Standby는 동기 복제이고 쿼리 불가(장애 대비). 목적이 다르다

---

## 개념

이름이 비슷해서 헷갈리지만 **목적이 다르다.**

| | Multi-AZ Standby | Read Replica |
|---|---|---|
| 목적 | 장애 대비 (가용성) | 읽기 부하 분산 |
| 복제 | 동기 | 비동기 |
| 직접 쿼리 | **불가** | 가능 |
| 엔드포인트 | 없음 (Primary 주소 그대로) | 별도 엔드포인트 |
| 장애 시 | 자동 failover (60~120초) | 수동 승격 |
| 데이터 지연 | 없음 | 있음 (복제 지연) |

**Standby는 보이지 않는다.** 평소엔 아무것도 안 하고 있다가 Primary가 죽으면 DNS가 그쪽을 가리킨다.
"읽기라도 시키면 아깝다"고 생각하기 쉬운데 그럴 수 없다.

**Replica는 별도 주소가 있다.** 애플리케이션이 그 주소로 연결해야 의미가 있다.

## 실행

### 켜는 것만으로는 아무 일도 안 일어난다

```hcl
create_read_replica = true
```

이러면 Replica 인스턴스가 생긴다. 그런데 앱이 Primary 엔진 하나만 쓰고 있으면
**요청은 전부 Primary로 간다.** 비용만 늘어난다.

읽기용 연결을 따로 만들어야 한다.

```python
engine = create_engine(DATABASE_URL, **opts)                       # 쓰기
read_engine = engine if not REPLICA else create_engine(READ_URL)   # 읽기

SessionLocal     = sessionmaker(bind=engine)
ReadSessionLocal = sessionmaker(bind=read_engine)
```

라우터는 무엇을 하는지만 선언한다.

```python
def list_members(db: Session = Depends(get_read_db)): ...   # 조회
def create_member(db: Session = Depends(get_db)):    ...   # 쓰기
```

### Replica 주소를 앱에 알리는 법

인프라 쪽에서 시크릿에 넣어 주면 앱은 그것만 보면 된다.

```hcl
secret_string = jsonencode(merge(
  { username = ..., host = aws_db_instance.primary.address, ... },
  var.create_read_replica ? { replica_host = aws_db_instance.replica[0].address } : {}
))
```

```python
replica_host = secret.get("replica_host")
read_url = _url_from_secret(secret, replica_host) if replica_host else write_url
```

없으면 Primary로 폴백한다. **기본값이 안전한 쪽**이라 Replica가 없는 환경에서도 그대로 돈다.

### 복제 지연을 의식해야 한다

비동기라서 **방금 쓴 것을 바로 읽으면 없을 수 있다.**

```
POST /members   → Primary 에 INSERT
GET  /members/1 → Replica 조회 → 아직 없음 (404)
```

그래서 "쓰고 나서 바로 확인하는" 경로는 Primary로 읽어야 한다.
목록 조회나 통계처럼 조금 늦어도 되는 것만 Replica로 보낸다.

### 헬스체크

`/health`가 Primary만 확인하면 **Replica가 끊겨도 200이 나온다.**
그 상태에서 조회만 실패한다. 실제로 운영한다면 읽기 커넥션도 확인하고,
Replica 장애 시 Primary로 넘어가는 처리가 필요하다.

## 배운 점

**인프라 옵션을 켜는 것과 애플리케이션이 그것을 쓰는 것은 별개다.**
Terraform 변수 하나로 리소스는 생기지만, 앱이 모르면 아무 효과가 없다.
"Read Replica 적용"이라고 적기 전에 요청이 실제로 그쪽으로 가는지 확인해야 한다.

**둘을 같이 쓸 수도 있다.** Multi-AZ로 가용성을 확보하고 Replica로 읽기를 분산한다.
목적이 다르니 배타적이지 않다.

---

- 처음 만난 곳: [gym-management-db](../../projects/gym-management-db/README.md) —
  [결정 05](../../projects/gym-management-db/decisions/05-읽기-커넥션-분리.md)

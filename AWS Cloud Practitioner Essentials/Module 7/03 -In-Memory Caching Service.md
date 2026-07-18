# Amazon ElastiCache

## 캐싱(Cache)이 필요한 이유

데이터베이스(RDS 등)에 **읽기(Read) 요청이 많아질수록** 다음과 같은 문제가 발생할 수 있다.

- 응답 지연(Latency) 증가
- 데이터베이스 부하 증가
- 애플리케이션 성능 저하

이러한 문제를 해결하는 가장 효과적인 방법 중 하나가 **캐시 계층(Cache Layer)** 을 도입하는 것이다.

---

# 캐싱(Cache)

캐싱은 **자주 사용하는 데이터를 메모리(RAM)에 저장**하여 빠르게 조회하는 기술이다.

## 특징

- 디스크(DB) 대신 **시스템 메모리(RAM)** 에 저장
- 매우 빠른 읽기 속도
- 자주 사용하는 데이터를 즉시 반환
- 데이터베이스 접근 횟수 감소

> 메모리(RAM)는 디스크보다 훨씬 빠르므로, 응답 시간을 크게 줄일 수 있다.

---

# 대표적인 인메모리 데이터 스토어

- Redis OSS
- Valkey
- Memcached

---

# Amazon ElastiCache

AWS에서 제공하는 **완전 관리형 인메모리(In-Memory) 캐시 서비스**

Redis OSS, Valkey, Memcached를 쉽게 배포하고 운영할 수 있도록 지원한다.

## 지원 엔진

- Redis OSS
- Valkey
- Memcached

---

# ElastiCache의 주요 특징

## 높은 성능

- 데이터를 메모리에 저장
- 매우 빠른 읽기/쓰기 성능 제공
- 낮은 지연 시간(Low Latency)

---

## 고가용성(High Availability)

- 여러 가용 영역(Multi-AZ)에 복제 가능
- 장애 발생 시 높은 가용성 제공

---

## 관리 자동화

AWS가 다음 작업을 자동으로 수행한다.

- 배포(Deployment)
- 운영(Operation)
- 확장(Scaling)
- 패치(Patching)
- 유지관리(Maintenance)

---

## 유연한 확장성

- 캐시 용량 확장 가능
- 트래픽 증가에 유연하게 대응

---

## 보안

- 저장 데이터 암호화
- 전송 데이터 암호화
- IAM 및 VPC 연동

---

# ElastiCache의 장점

- 데이터베이스 부하 감소
- 응답 속도 향상
- 애플리케이션 성능 향상
- 운영 오버헤드 감소
- 더 작은 DB 인스턴스 사용 가능 → 비용 절감

---

# ElastiCache 활용 사례

다음과 같이 빠른 응답이 필요한 서비스에 적합하다.

- 실시간 애플리케이션
- 게임 순위표(Leaderboard)
- 세션(Session) 저장
- 자주 조회되는 콘텐츠
- 인기 상품 조회
- 사용자 프로필
- 쇼핑몰 상품 정보

---

# ElastiCache 동작 과정

```
사용자
    │
    ▼
EC2(Application)
    │
    ▼
ElastiCache 확인
    │
 ┌──┴───────────────┐
 │                  │
 │ Cache Hit        │ Cache Miss
 │                  │
 ▼                  ▼
캐시 데이터 반환      Amazon RDS 조회
 │                  │
 │                  ▼
 │            데이터 조회
 │                  │
 │                  ▼
 │          ElastiCache 저장
 │                  │
 └──────────┬───────┘
            ▼
        사용자에게 반환
```

### Cache Hit

1. 사용자가 데이터를 요청
2. EC2가 ElastiCache 확인
3. 캐시에 데이터가 존재
4. 즉시 사용자에게 반환

→ 가장 빠른 응답

---

### Cache Miss

1. 사용자가 데이터를 요청
2. EC2가 ElastiCache 확인
3. 캐시에 데이터가 없음
4. Amazon RDS에서 데이터 조회
5. 조회한 데이터를 ElastiCache에 저장
6. 사용자에게 반환

→ 다음 요청부터는 캐시에서 바로 조회 가능

---

# Amazon RDS + ElastiCache 구조

```
          사용자
             │
             ▼
      Amazon EC2
             │
             ▼
     Amazon ElastiCache
        │          │
        │ Hit      │ Miss
        ▼          ▼
   데이터 반환   Amazon RDS
                    │
                    ▼
            데이터 조회
                    │
                    ▼
          ElastiCache 저장
                    │
                    ▼
              사용자에게 반환
```

---

# Amazon RDS vs Amazon ElastiCache

| 구분 | Amazon RDS | Amazon ElastiCache |
|------|------------|--------------------|
| 저장 위치 | 디스크 | 메모리(RAM) |
| 목적 | 영구 데이터 저장 | 자주 사용하는 데이터 캐싱 |
| 속도 | 일반적인 DB 속도 | 매우 빠름 |
| 데이터 유지 | 영구 저장 | 메모리 기반(휘발성) |
| 사용 목적 | 데이터 저장 | 성능 향상 및 DB 부하 감소 |

---

# 핵심 정리

- **ElastiCache**는 AWS의 **관리형 인메모리 캐시 서비스**이다.
- **Redis OSS, Valkey, Memcached**를 지원한다.
- 자주 사용하는 데이터를 메모리에 저장하여 매우 빠르게 조회할 수 있다.
- **Cache Hit**는 캐시에서 즉시 데이터를 반환한다.
- **Cache Miss**는 RDS에서 데이터를 조회한 뒤 캐시에 저장하고 사용자에게 반환한다.
- 데이터베이스 부하를 줄이고 응답 속도를 크게 향상시킬 수 있다.
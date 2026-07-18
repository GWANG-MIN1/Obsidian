# Amazon DynamoDB

## NoSQL(비관계형 데이터베이스)

NoSQL은 **비관계형(Non-Relational) 데이터베이스**로, 데이터를 테이블 간의 관계가 아닌 다양한 형태로 저장한다.

관계형 데이터베이스(RDBMS)는 미리 정의된 스키마와 SQL을 사용하여 데이터를 관리하지만, NoSQL은 유연한 데이터 구조를 제공하여 빠른 읽기/쓰기와 대규모 확장에 적합하다.

### RDBMS vs NoSQL

| RDBMS | NoSQL |
|--------|--------|
| 정해진 스키마(Schema) | 유연한 스키마 |
| SQL 사용 | Key-Value, Document 등 다양한 방식 |
| 테이블 간 관계(Relation) 중심 | 관계 없이 데이터 저장 |
| 복잡한 JOIN 가능 | JOIN 미지원 또는 제한적 |
| 트랜잭션 중심 | 대규모 분산 처리에 적합 |

> RDBMS는 잘 정의된 SQL을 사용하지만, 데이터 구조가 자주 변경되거나 대규모 트래픽 환경에서는 성능 및 확장성에 한계가 있을 수 있다.

---

# Amazon DynamoDB

AWS에서 제공하는 **완전 관리형(Managed) NoSQL 데이터베이스 서비스**

Key-Value 방식과 Document 방식의 데이터를 저장하며, 매우 빠른 응답 속도와 뛰어난 확장성을 제공한다.

---

# 데이터 구조

DynamoDB에서는 다음과 같은 구조를 사용한다.

```
테이블(Table)
    └── 항목(Item)
            └── 속성(Attribute)
                    ├── 이름(Name)
                    └── 값(Value)
```

### 구성 요소

- **Table** : 데이터를 저장하는 공간
- **Item** : 하나의 데이터(행과 유사)
- **Attribute** : Item을 구성하는 속성(열과 유사)

예시

| UserID | Name | Age | Hobby |
|--------|------|-----|--------|
| 1001 | Kim | 24 | Soccer |

- Item = 한 명의 사용자 정보
- Attribute = UserID, Name, Age, Hobby

---

# DynamoDB의 특징

- 비관계형(NoSQL) 데이터베이스
- Key-Value 및 Document 모델 지원
- 데이터 = **Item(속성들의 모음)**
- 속성(Attribute)은 **이름(Name) + 값(Value)** 으로 구성
- 언제든지 속성 추가 및 삭제 가능
- 같은 테이블의 모든 Item이 동일한 속성을 가질 필요가 없음

예시

```
Item 1
- Name
- Age
- Hobby

Item 2
- Name
- Phone
- Address
```

→ 두 Item의 속성이 서로 달라도 저장 가능

---

# DynamoDB의 장점

## 매우 빠른 성능

- 한 자릿수 밀리초(ms)의 지연 시간
- 대규모 읽기/쓰기 처리 가능

---

## 자동 확장(Auto Scaling)

- 프로비저닝된 용량(Provisioned Capacity) 기반 자동 확장
- 필요에 따라 읽기/쓰기 처리량 자동 증가 및 감소

---

## 서버 관리 불필요

- 서버 프로비저닝 필요 없음
- 유지보수 기간(Maintenance Window) 없음
- 패치 및 운영을 AWS가 관리

---

## 높은 가용성

- 여러 가용 영역(AZ)에 자동 복제
- 장애 발생 시에도 높은 가용성 제공

---

## 높은 내구성

- 데이터 자동 복제
- 장애 발생 시 데이터 보호

---

## 보안

- 저장 데이터 암호화(Encryption at Rest)
- 전송 데이터 암호화(In Transit)
- IAM과 연동한 접근 제어

---

# DynamoDB 글로벌 테이블(Global Tables)

전 세계 여러 AWS 리전에 데이터를 자동으로 복제하는 기능

## 특징

- 여러 리전에서 동시에 읽기/쓰기 가능
- 다중 리전(Multi-Region) 복제
- 낮은 지연 시간(Low Latency)
- 글로벌 서비스 구축에 적합

---

# 실제 사용 사례

## Amazon Prime Day

DynamoDB는 Amazon Prime Day와 같은 초대형 이벤트에서 사용된다.

- 수십조 건의 API 호출 처리
- 초당 최대 **1억 4,600만 건(Requests per Second)** 처리
- 자동으로 트래픽 증가에 대응

---

# DynamoDB를 사용하는 경우

다음과 같은 상황에서 적합하다.

- 빠른 읽기/쓰기 성능이 필요한 경우
- 사용자 수가 급격히 증가하는 서비스
- 서버 관리 부담을 줄이고 싶은 경우
- 데이터 구조가 자주 변경되는 경우
- 글로벌 서비스를 운영하는 경우

예시
- 쇼핑몰 장바구니
- 게임 랭킹
- 세션 저장소
- IoT 데이터
- 실시간 로그 저장

---

# Amazon RDS vs Amazon DynamoDB

| 구분 | Amazon RDS | Amazon DynamoDB |
|------|------------|-----------------|
| 데이터베이스 종류 | 관계형(RDBMS) | 비관계형(NoSQL) |
| 데이터 구조 | 테이블(행·열) | Key-Value / Document |
| 스키마 | 고정 | 유연 |
| JOIN | 지원 | 미지원 |
| 확장 방식 | 주로 수직 확장(Scale Up) | 수평 확장(Scale Out) |
| 성능 | 일반적인 DB 성능 | 매우 빠른 응답 속도 |
| 관리 | AWS 관리 | AWS 완전 관리 |

---

# 핵심 정리

- **DynamoDB** = AWS의 **완전 관리형 NoSQL 데이터베이스**
- **Key-Value** 및 **Document** 모델을 지원한다.
- 같은 테이블의 Item들이 서로 다른 Attribute를 가져도 된다.
- 매우 빠른 응답 속도와 자동 확장(Auto Scaling)을 제공한다.
- **Global Tables**를 통해 여러 리전에서 동시에 데이터를 사용할 수 있다.
- 대규모 트래픽과 실시간 서비스에 적합하다.
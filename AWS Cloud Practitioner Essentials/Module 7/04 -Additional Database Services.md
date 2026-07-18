# AWS 목적별 데이터베이스(Purpose-Built Databases)

## 올바른 데이터베이스 선택

AWS의 핵심 철학은 **"모든 용도에 맞는 만능 데이터베이스는 없다."** 이다.

애플리케이션의 데이터 구조와 요구사항에 따라 가장 적합한 데이터베이스를 선택하는 것이 중요하다.

> **Purpose-Built Database** = 특정 목적에 최적화된 데이터베이스

---

# AWS에서 제공하는 목적별 데이터베이스

| 데이터 형태 | 추천 서비스 |
|-------------|------------|
| 관계형 데이터 | Amazon RDS, Amazon Aurora |
| Key-Value 데이터 | Amazon DynamoDB |
| 문서(Document) 데이터 | Amazon DocumentDB |
| 그래프(Graph) 데이터 | Amazon Neptune |
| 블록체인(Blockchain) | Amazon Managed Blockchain |

---

# Amazon DynamoDB

완전 관리형 **NoSQL(Key-Value / Document) 데이터베이스**

## 적합한 데이터

- Key-Value 데이터
- Document 데이터
- 반정형(Semi-Structured) 데이터

예시
- 사용자 프로필
- 장바구니
- 게임 랭킹
- IoT 데이터

---

# 반정형 데이터(Semi-Structured Data)

반정형 데이터는 **고정된 행(Row)과 열(Column) 구조에 맞지 않는 데이터**이다.

특징

- 모든 데이터가 동일한 속성을 가질 필요 없음
- 데이터마다 구조가 달라도 저장 가능
- JSON, XML 등이 대표적인 형태

예시(JSON)

```json
{
  "name": "Kim",
  "age": 24,
  "hobby": ["Soccer", "Music"]
}
```

---

# Amazon DocumentDB (MongoDB 호환)

MongoDB와 호환되는 **완전 관리형 Document 데이터베이스 서비스**

JSON 형태의 문서를 저장하며, 반정형 데이터를 처리하도록 설계되었다.

## 특징

- MongoDB 호환
- Document 기반 저장
- 동적 스키마(Dynamic Schema)
- JSON 스타일 문서 저장
- 완전 관리형 서비스

---

## 장점

- MongoDB 애플리케이션과 높은 호환성
- 뛰어난 성능과 확장성
- 높은 읽기 처리량(Read Throughput)
- 다양한 구조의 데이터 저장 가능

---

## 활용 사례

- 고객 정보
- 제품 카탈로그
- CMS(콘텐츠 관리 시스템)
- 웹 애플리케이션 데이터
- 스프레드시트처럼 구조가 일정하지 않은 데이터

---

# Amazon Neptune

완전 관리형 **그래프(Graph) 데이터베이스 서비스**

복잡하게 연결된 데이터 간의 **관계(Relationship)** 를 저장하고 분석하는 데 특화되어 있다.

---

## 특징

- 그래프 데이터베이스
- 데이터 간 관계 분석에 최적화
- 높은 성능과 확장성
- 복잡한 연결 관계 탐색 가능

---

## 활용 사례

- SNS 친구 관계
- 추천 시스템
- 네트워크 분석
- 사기(Fraud) 탐지
- 지식 그래프(Knowledge Graph)

예시

```
철수 ── 친구 ── 영희
 │               │
 │               │
구매          구매
 │               │
 ▼               ▼
상품 A      상품 B
```

→ 데이터 간 관계와 패턴을 쉽게 찾을 수 있다.

---

# Amazon Managed Blockchain

블록체인 네트워크를 쉽게 구축하고 운영할 수 있도록 지원하는 **완전 관리형 블록체인 서비스**

## 특징

- 블록체인 네트워크 구축
- 노드 관리 자동화
- 높은 보안성과 신뢰성

---

## 활용 사례

- 공급망 관리(Supply Chain)
- 식자재 유통 이력 관리
- 금융 거래 기록
- 계약 추적

예시

```
농장
   │
   ▼
도매업체
   │
   ▼
물류회사
   │
   ▼
식료품점
```

→ 식품의 이동 경로를 투명하게 추적 가능

---

# 데이터베이스 성능 향상

## Amazon DynamoDB Accelerator (DAX)

DAX는 DynamoDB 전용 **인메모리 캐시 서비스**이다.

자주 조회되는 데이터를 메모리에 저장하여 읽기 성능을 크게 향상시킨다.

---

## 특징

- DynamoDB 전용 캐시
- 읽기 지연 시간 감소
- 마이크로초(μs) 수준 응답
- 읽기 처리량 향상

> DAX는 **DynamoDB 전용 ElastiCache**와 비슷한 역할을 한다.

---

# AWS Backup

AWS의 **중앙 집중식 백업 관리 서비스**

여러 AWS 스토리지 및 데이터베이스의 백업을 하나의 서비스에서 관리할 수 있다.

---

## 지원 대상

- Amazon EBS
- Amazon EFS
- Amazon RDS
- Amazon Aurora
- Amazon DynamoDB
- Amazon DocumentDB
- Amazon Neptune
- Amazon FSx
- 온프레미스(On-Premises) 환경

---

## 장점

- 중앙 집중식 백업 관리
- 자동 백업 정책
- 리전 간 백업 복제(Cross-Region Backup)
- 규정 준수(Compliance) 간소화
- 백업 및 복구 작업 자동화

---

# AWS Backup이 필요한 이유

AWS에는 다양한 데이터베이스와 스토리지 서비스가 존재한다.

각 서비스마다 백업 방식을 따로 관리하면 복잡해질 수 있다.

AWS Backup은 이를 하나의 서비스에서 통합 관리하여 백업 작업을 단순화한다.

---

# 목적별 데이터베이스 선택 예시

| 상황 | 추천 서비스 |
|------|-------------|
| 온라인 쇼핑몰 주문 관리 | Amazon RDS |
| 게임 랭킹 | Amazon DynamoDB |
| JSON 문서 저장 | Amazon DocumentDB |
| SNS 친구 관계 분석 | Amazon Neptune |
| 공급망 추적 | Amazon Managed Blockchain |

---

# 핵심 정리

- **모든 용도에 맞는 만능 데이터베이스는 없다.**
- 데이터 특성에 맞는 **Purpose-Built Database**를 선택하는 것이 중요하다.
- **DynamoDB** → Key-Value 및 Document 데이터
- **DocumentDB** → JSON 기반 반정형(Document) 데이터
- **Neptune** → 그래프(Graph) 데이터와 관계 분석
- **Managed Blockchain** → 블록체인 네트워크 구축
- **DAX** → DynamoDB 전용 인메모리 캐시로 읽기 성능 향상
- **AWS Backup** → 다양한 AWS 스토리지와 데이터베이스를 중앙에서 통합 백업 관리
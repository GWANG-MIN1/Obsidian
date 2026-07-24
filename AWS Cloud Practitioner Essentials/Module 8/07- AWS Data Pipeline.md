# 07. AWS 데이터 파이프라인(Data Pipeline)

> **AWS 데이터 파이프라인(Data Pipeline)** 은 다양한 데이터 소스에서 데이터를 **수집(Ingest), 저장(Store), 카탈로그화(Catalog), 변환(Transform), 분석(Analyze)** 하는 과정을 자동화하는 데이터 처리 아키텍처이다.

데이터 파이프라인을 사용하면 반복적인 수작업을 줄이고 오류를 최소화하며, 데이터 분석과 AI/ML에 필요한 데이터를 효율적으로 준비할 수 있다.

---

# AWS 데이터 파이프라인 구성

```
데이터 소스
      │
      ▼
① 데이터 수집(Ingestion)
      │
      ▼
② 데이터 저장(Storage)
      │
      ▼
③ 데이터 카탈로그(Catalog)
      │
      ▼
④ 데이터 변환(ETL)
      │
      ▼
⑤ 데이터 분석(Analytics)
      │
      ▼
⑥ 시각화 및 검색
```

---

# 1. 데이터 수집 및 저장 (Ingestion & Storage)

데이터 파이프라인의 첫 번째 단계는 데이터를 다양한 소스에서 수집하여 저장하는 것이다.

---

## 데이터 레이크(Data Lake)

> **원시(Raw) 데이터를 다양한 형태 그대로 저장하는 저장소**

### 특징

- 구조화/반정형/비정형 데이터 저장
- 높은 확장성
- 매우 유연한 데이터 활용

### AWS 서비스

### Amazon S3

- 객체 스토리지(Object Storage)
- 데이터 레이크 구축
- AI/ML 및 분석의 데이터 저장소

---

## 데이터 웨어하우스(Data Warehouse)

> **분석 및 비즈니스 인텔리전스(BI)에 최적화된 구조화된 데이터 저장소**

### 특징

- 분석용 데이터 저장
- 빠른 SQL 분석
- 복잡한 집계 및 보고서 작성

### AWS 서비스

### Amazon Redshift

- 완전 관리형 데이터 웨어하우스
- 대규모 분석 처리
- BI 및 데이터 분석 최적화

---

# 데이터 수집(Ingestion)

데이터 수집은 다양한 소스의 데이터를 원하는 스토리지(S3, Redshift 등)로 이동시키는 과정이다.

---

## Amazon Kinesis Data Streams

> 실시간 스트리밍 데이터 수집 서비스

### 특징

- 실시간 데이터 처리
- 여러 애플리케이션이 동일한 데이터를 소비 가능
- 높은 처리량 지원

### 사용 사례

- IoT 데이터
- 로그 수집
- 실시간 분석

---

## Amazon Data Firehose

> 데이터를 거의 실시간(Near Real-Time)으로 수집하여 목적지로 전달하는 서비스

### 특징

- 자동 데이터 적재
- 데이터 압축
- 데이터 암호화
- 데이터 일괄 처리(Buffering)

### 사용 사례

- Amazon S3 적재
- Amazon Redshift 적재
- OpenSearch 적재

---

# 2. 데이터 카탈로그(Data Catalog)

데이터를 저장한 후에는 **어떤 데이터가 어디에 있는지 관리**해야 한다.

이를 위해 데이터 카탈로그를 사용한다.

---

## AWS Glue Data Catalog

> 데이터를 검색하고 관리하는 **완전 관리형 메타데이터 저장소(Metadata Repository)**

### 주요 기능

- 데이터 검색
- 메타데이터 관리
- 데이터 자동 카탈로그화
- 데이터 검색 자동화
- 데이터 거버넌스 지원
- 규정 준수 지원

---

# AWS Glue

> 데이터 카탈로그와 ETL 기능을 제공하는 **완전 관리형 서비스**

AWS Glue는 Data Catalog와 ETL 기능을 함께 제공한다.

---

# 3. 데이터 변환 (ETL)

분석을 시작하기 전에 데이터를 정제하고 원하는 형식으로 변환해야 한다.

---

## AWS Glue ETL

> 완전 관리형 ETL 서비스

### 주요 기능

- ETL 작업 생성
- 데이터 정제
- 데이터 변환
- 시각적(Visual) ETL 작업 생성
- 스케줄 기반 자동 실행

### 장점

- 코드 없이(No-code) ETL 가능
- 서버 관리 불필요
- 자동 확장

---

# 4. 데이터 처리 및 분석

---

## Amazon EMR

> Apache Spark, Hadoop 등의 오픈소스 프레임워크를 이용한 대규모 데이터 처리 서비스

### 특징

- Spark 지원
- Hadoop 지원
- 대용량 데이터 처리
- 높은 유연성

### 적합한 환경

- 빅데이터 전문가
- 복잡한 데이터 처리
- 사용자 지정 데이터 처리

---

## Amazon Athena

> Amazon S3에 저장된 데이터를 SQL로 분석하는 **완전 관리형 서버리스 쿼리 서비스**

### 특징

- 서버 관리 불필요
- 표준 SQL 사용
- Amazon S3 직접 분석
- AWS 외부 데이터 분석 가능

### 적합한 환경

- 간단한 SQL 분석
- 빈번하지 않은 쿼리
- 빠른 데이터 탐색

---

## Amazon Redshift

> 고성능 분석을 위한 **완전 관리형 데이터 웨어하우스**

### 특징

- 복잡한 SQL 처리
- 대규모 데이터 분석
- BI 최적화
- 세밀한 쿼리 제어
- 고급 분석 기능

### 적합한 환경

- 대규모 데이터 분석
- 데이터 웨어하우스 구축
- BI 시스템

---

# 5. 데이터 시각화

## Amazon QuickSight

> 대화형 BI(Business Intelligence) 대시보드 및 보고서를 생성하는 서비스

### 주요 기능

- 대시보드 생성
- 보고서 작성
- 데이터 시각화
- 조직 전체 공유
- 대규모 사용자 지원

### Amazon Q in QuickSight

QuickSight에는 **Amazon Q**가 통합되어 있어 자연어를 사용해 데이터를 분석할 수 있다.

예시

> "지난달 가장 많이 판매된 상품은?"

↓

자동으로 차트와 인사이트 생성

### 사용 대상

- 데이터 분석가
- 경영진
- 일반 비즈니스 사용자

---

# 6. 검색 및 운영 분석

## Amazon OpenSearch Service

> 실시간 검색, 로그 분석 및 운영 데이터를 분석하는 서비스

### 주요 기능

- 로그 분석
- 실시간 검색
- 운영 데이터 분석
- 애플리케이션 모니터링

### 사용 사례

- 시스템 로그 분석
- 웹사이트 검색
- 보안 분석
- 애플리케이션 성능 모니터링

---

# AWS 데이터 파이프라인 예시

```text
데이터 생성
      │
      ▼
Amazon Kinesis Data Streams
또는
Amazon Data Firehose
      │
      ▼
Amazon S3 (Data Lake)
또는
Amazon Redshift (Data Warehouse)
      │
      ▼
AWS Glue Data Catalog
      │
      ▼
AWS Glue ETL
      │
      ▼
Amazon Athena
Amazon EMR
Amazon Redshift
      │
      ▼
Amazon QuickSight
또는
Amazon OpenSearch Service
```

---

# 서비스별 역할 정리

| 단계 | AWS 서비스 | 역할 |
|------|------------|------|
| 데이터 수집 | Amazon Kinesis Data Streams | 실시간 데이터 스트리밍 |
| 데이터 수집 | Amazon Data Firehose | 거의 실시간 데이터 적재 |
| 데이터 저장 | Amazon S3 | 데이터 레이크 |
| 데이터 저장 | Amazon Redshift | 데이터 웨어하우스 |
| 데이터 카탈로그 | AWS Glue Data Catalog | 메타데이터 관리 및 데이터 검색 |
| 데이터 변환 | AWS Glue | ETL 및 데이터 정제 |
| 데이터 처리 | Amazon EMR | Spark/Hadoop 기반 빅데이터 처리 |
| 데이터 분석 | Amazon Athena | 서버리스 SQL 분석 |
| 데이터 분석 | Amazon Redshift | 고성능 데이터 분석 |
| 데이터 시각화 | Amazon QuickSight | BI 대시보드 및 시각화 |
| 검색 및 분석 | Amazon OpenSearch Service | 로그 분석 및 실시간 검색 |

---

# 핵심 정리

- AWS 데이터 파이프라인은 **데이터 수집 → 저장 → 카탈로그화 → 변환 → 분석 → 시각화**의 전 과정을 자동화한다.
- **Amazon S3**는 데이터 레이크, **Amazon Redshift**는 데이터 웨어하우스로 주로 사용된다.
- **Amazon Kinesis Data Streams**와 **Amazon Data Firehose**는 데이터 수집을 담당한다.
- **AWS Glue Data Catalog**는 메타데이터를 관리하고, **AWS Glue**는 ETL 작업을 수행한다.
- **Amazon EMR**, **Amazon Athena**, **Amazon Redshift**는 데이터 처리 및 분석을 담당하며, **Amazon QuickSight**는 분석 결과를 시각화한다.
- **Amazon OpenSearch Service**는 로그 분석과 실시간 검색 등 운영 데이터 분석에 적합하다.
- 데이터 파이프라인은 AI/ML뿐 아니라 BI, 로그 분석, 실시간 모니터링 등 다양한 데이터 활용 시나리오에 사용된다.
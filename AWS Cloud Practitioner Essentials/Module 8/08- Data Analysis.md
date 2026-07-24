# 08. AWS 데이터 파이프라인 예제 - 전자상거래 추천 시스템

> 전자상거래(E-Commerce) 애플리케이션에서 고객 행동 데이터를 실시간으로 수집하고 분석하여 **추천 모델을 지속적으로 개선하는 데이터 파이프라인** 예시이다.

이 파이프라인은 고객의 행동 데이터를 자동으로 수집하고, 저장·분석한 뒤 AI 모델을 재학습하여 더 정확한 추천 서비스를 제공한다.

---

# 전체 아키텍처

```text
고객(전자상거래 앱)
        │
        ▼
Amazon DynamoDB
(고객 데이터 저장)
        │
        ▼
Amazon Kinesis Data Streams
(실시간 데이터 수집)
        │
        ▼
Amazon Data Firehose
(데이터 전송 및 Lambda 호출)
        │
        ▼
AWS Lambda
(데이터 변환)
        │
        ▼
Amazon S3
(데이터 레이크)
        │
        ▼
AWS Glue Data Catalog
(메타데이터 관리)
        │
        ├──────────────┐
        ▼              ▼
Amazon Athena     Amazon SageMaker AI
(SQL 분석)         (모델 재학습)
```

---

# 데이터 파이프라인 흐름

## ① 고객 활동 발생

전자상거래 애플리케이션에서 고객이

- 상품 조회
- 상품 클릭
- 장바구니 추가
- 구매

등의 행동을 수행한다.

이 데이터는 추천 시스템의 중요한 학습 데이터가 된다.

---

## ② Amazon DynamoDB

고객 활동 데이터는 **Amazon DynamoDB**에 저장된다.

### 특징

- NoSQL 데이터베이스
- JSON 형태의 데이터 저장
- 빠른 읽기/쓰기
- 실시간 애플리케이션에 적합

예시 데이터

```json
{
  "userId": "1001",
  "product": "Keyboard",
  "action": "Click",
  "time": "2026-07-24"
}
```

---

## ③ Amazon Kinesis Data Streams

DynamoDB의 변경 데이터를 실시간으로 수집한다.

### 역할

- 스트리밍 데이터 수집
- 여러 소비자에게 데이터 전달
- 실시간 처리

---

## ④ AWS Lambda

Amazon Data Firehose는 데이터를 전달하기 전에 **AWS Lambda**를 호출할 수 있다.

Lambda에서는

- 데이터 정제
- 형식 변환
- 필요한 필드 추가
- 데이터 필터링

등의 작업을 수행한다.

---

## ⑤ Amazon Data Firehose → Amazon S3

Firehose는 데이터를 버퍼링한 뒤 Amazon S3로 저장한다.

### Firehose 기능

- 자동 데이터 전송
- 압축
- 암호화
- 일괄(Buffering) 저장

### Amazon S3

- 데이터 레이크(Data Lake)
- CSV, JSON 등 다양한 형식 저장
- 여러 서비스가 동시에 접근 가능

---

## ⑥ AWS Glue Data Catalog

S3에 저장된 데이터를 자동으로 분석하여 메타데이터를 생성한다.

### 역할

- 데이터 스키마 관리
- 테이블 생성
- 데이터 위치 관리
- 메타데이터 저장

이를 통해 다른 AWS 서비스는 S3의 데이터를 **데이터베이스 테이블처럼** 사용할 수 있다.

---

## ⑦ Amazon Athena

데이터 과학자(Data Scientist)는 Amazon Athena를 사용하여 S3 데이터를 SQL로 바로 분석할 수 있다.

### 특징

- 서버리스(Serverless)
- 표준 SQL 사용
- 임시(Ad-hoc) 쿼리 지원
- S3 데이터를 직접 조회

### 활용 예시

```sql
SELECT *
FROM customer_clicks
WHERE action='Purchase';
```

Athena는 Glue Data Catalog의 **스키마와 테이블 정보를 이용해** 데이터를 데이터베이스처럼 조회한다.

---

## ⑧ Amazon SageMaker AI

분석된 최신 데이터를 활용하여 추천 모델을 다시 학습한다.

### 역할

- 모델 재학습(Retraining)
- 새 모델 생성
- 모델 배포

학습된 최신 모델은 다시 전자상거래 서비스에서 사용되어 더 정확한 상품 추천을 제공한다.

---

# 데이터 파이프라인의 장점

## 자동화

한 번 구축하면 데이터 흐름이 자동으로 수행된다.

- 데이터 수집
- 데이터 저장
- 데이터 분석
- AI 모델 학습

모든 과정이 자동으로 반복된다.

---

## 시간 절약

수작업 없이 데이터를 자동 처리하므로

- 업무 시간 단축
- 운영 비용 절감
- 개발 생산성 향상

효과를 얻을 수 있다.

---

## 동일한 데이터의 재사용

하나의 데이터를 여러 서비스에서 동시에 활용할 수 있다.

예를 들어,

- Amazon Athena → 데이터 분석
- Amazon SageMaker → AI 모델 학습
- Amazon QuickSight → 대시보드 생성
- Amazon Bedrock → 생성형 AI 활용

처럼 하나의 데이터가 다양한 목적으로 활용된다.

---

## 확장성

AWS의 관리형 서비스를 사용하므로

- 데이터 증가
- 사용자 증가
- AI 모델 증가

에도 별도의 인프라 관리 없이 확장할 수 있다.

---

# 자동화의 중요성

만약 이 과정을 모두 수작업으로 수행한다면

- 데이터를 직접 복사
- 데이터 형식 변환
- 데이터 업로드
- 분석 실행
- 모델 재학습

등을 반복해야 하므로 매우 비효율적이다.

반면 데이터 파이프라인을 구축하면 이러한 과정이 자동으로 수행되어 **효율성과 생산성을 크게 향상**시킬 수 있다.

---

# 핵심 정리

- 전자상거래 서비스에서는 고객 행동 데이터를 지속적으로 수집하여 추천 시스템을 개선한다.
- **Amazon DynamoDB**는 고객 데이터를 저장하고, **Amazon Kinesis Data Streams**가 이를 실시간으로 수집한다.
- **Amazon Data Firehose**는 필요 시 **AWS Lambda**를 호출하여 데이터를 변환한 후 **Amazon S3**에 저장한다.
- **AWS Glue Data Catalog**는 S3 데이터의 스키마와 메타데이터를 관리하여 다른 서비스가 쉽게 활용할 수 있도록 한다.
- **Amazon Athena**는 Glue Catalog를 기반으로 S3 데이터를 SQL로 분석하고, **Amazon SageMaker AI**는 최신 데이터를 이용해 추천 모델을 재학습한다.
- 자동화된 데이터 파이프라인은 반복 작업을 줄이고, 동일한 데이터를 다양한 분석 및 AI 서비스에서 재사용할 수 있어 시간 절약과 운영 효율성 향상에 큰 도움이 된다.
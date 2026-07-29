# AWS Serverless 아키텍처 예시

## Serverless 아키텍처란?

**Serverless Architecture(서버리스 아키텍처)** 는 개발자가 서버를 직접 구축하거나 관리하지 않고 애플리케이션을 개발하고 실행하는 방식이다.

AWS가 인프라를 관리하므로 개발자는 **비즈니스 로직(코드)** 작성에만 집중할 수 있다.

대표적인 서버리스 서비스

- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- Amazon S3
- Amazon SES
- AWS X-Ray

---

# 예시 1. 서버리스 백엔드 API

## 아키텍처

```
클라이언트

      │

      ▼

Amazon API Gateway

      │

      ▼

AWS Lambda

      │

      ▼

Amazon DynamoDB

      │

      ▼

응답(Response)

        ▲

AWS X-Ray (요청 추적)
```

---

## 동작 과정

### 1. 클라이언트 요청

사용자가 HTTP 요청을 보낸다.

↓

### 2. Amazon API Gateway

API Gateway는

- HTTP 요청 수신
- 요청 유효성 검사
- Lambda 함수 호출
- 응답 반환

을 수행한다.

↓

### 3. AWS Lambda

Lambda 함수는

- 비즈니스 로직 실행
- DynamoDB 조회 또는 저장

을 수행한다.

↓

### 4. Amazon DynamoDB

데이터를 저장하거나 조회한다.

↓

### 5. API Gateway

Lambda의 결과를 사용자에게 반환한다.

---

## AWS X-Ray

AWS X-Ray는 요청(Request)이 시스템을 통과하는 전체 과정을 추적한다.

예를 들어

```
Client

↓

API Gateway

↓

Lambda

↓

DynamoDB
```

까지의 요청 흐름을 추적하여

- 성능 분석
- 병목 현상 확인
- 오류 발생 위치 확인

등을 지원한다.

---

# 예시 2. 서버리스 문의하기(Contact Form)

정적 웹사이트에서 문의 양식을 제출하는 예시이다.

## 아키텍처

```
Amazon S3
(정적 웹사이트)

      │

      ▼

Amazon API Gateway

      │

      ▼

AWS Lambda

      │

      ▼

Amazon Simple Email Service (SES)

      │

      ▼

이메일 발송
```

---

## 동작 과정

### Amazon S3

HTML, CSS, JavaScript로 구성된 정적 웹사이트를 호스팅한다.

↓

### 사용자가 문의하기(Form)를 제출

↓

### API Gateway

요청을 받아 Lambda를 호출한다.

↓

### Lambda

입력 내용을 처리한 후 Amazon SES를 호출한다.

↓

### Amazon SES

이메일을 발송한다.

---

## 특징

이 아키텍처에서는 데이터를 데이터베이스에 저장하지 않는다.

Lambda가 직접 **Amazon SES API**를 호출하여 이메일을 전송한다.

따라서

- 서버 없음
- 데이터베이스 없음
- 이메일만 전송

이라는 매우 간단한 서버리스 구조가 된다.

---

# 예시 1과 예시 2 비교

| 서버리스 백엔드 | 문의하기(Form) |
|----------------|---------------|
| DynamoDB 사용 | SES 사용 |
| 데이터 저장 | 이메일 전송 |
| CRUD API | Contact Form |
| Lambda + DB | Lambda + Email |

같은 **API Gateway + Lambda** 구조를 사용하지만 **사용 사례(Use Case)** 는 완전히 다를 수 있다.

---

# Lambda의 역할

Lambda 함수에는 애플리케이션의 **비즈니스 로직(코드)** 이 작성된다.

예를 들어

- 데이터 저장
- 데이터 조회
- 이메일 발송
- 인증 처리
- 파일 처리

등 다양한 작업을 수행할 수 있다.

---

# 예시 3. Amazon Connect 기반 고객 지원

Amazon Connect를 활용하면 서버리스 방식으로 고객 지원 시스템을 구축할 수 있다.

## 아키텍처

```
고객

│

▼

Amazon Connect
(IVR)

├── 라이브 상담원

├── 채팅

└── AWS Lambda

        │

        ▼

SNS 또는 Email

        │

        ▼

고객
```

---

## 동작 과정

### 1. 고객이 전화

↓

### 2. Amazon Connect

IVR(자동 응답 시스템)이 실행된다.

↓

### 3. 고객 선택

- 상담원 연결
- 채팅
- 콜백 요청

↓

### 4. Lambda

필요한 로직을 수행한다.

예를 들어

- 고객 조회
- 예약 확인
- 콜백 등록

↓

### 5. SNS 또는 Email

고객에게 알림을 전송한다.

---

# Callback(콜백) 기능

대기 시간이 길어질 경우

Amazon Connect는

- 콜백 예약
- 채팅 전환
- 이메일 안내

등의 대체 채널을 제공할 수 있다.

이를 통해 고객은 오랜 시간 전화 대기를 하지 않아도 된다.

---

# Amazon CloudFront 활용

Amazon CloudFront를 함께 사용하면

- 웹사이트
- 고객 포털
- 상담 페이지

등을 빠르게 제공할 수 있다.

CloudFront는 정적 콘텐츠를 전 세계 엣지 로케이션에 캐싱하여 사용자에게 더 빠른 응답을 제공한다.

---

# Amazon Connect 기반 아키텍처

```text
고객
├─ 전화(IVR) ───────────────▶ Amazon Connect
│                              │
│                              ├── 전화 ▶ 라이브 에이전트
│                              ├── 채팅 ▶ 라이브 에이전트
│                              │
│                              └── 대기 시간 증가
│                                      │
│                                      ▼
│                               콜백 또는 채팅 선택
│                                      │
│                                      ▼
│                                 AWS Lambda
│                                      │
│                                      ▼
│                              Amazon SNS/Email
│                                      │
│                                      ▼
│                           SMS 또는 이메일 알림
│
└─ 웹사이트(URL) ▶ Amazon CloudFront
                    │
                    └── 채팅 시작 ▶ Amazon Connect
```
---

# 서버리스 아키텍처의 장점

- 서버 관리 불필요
- 자동 확장(Auto Scaling)
- 사용한 만큼만 비용 지불(Pay as You Go)
- 높은 가용성
- 빠른 개발 및 배포
- AWS 서비스와 손쉬운 통합

---

# 핵심 정리

- **Serverless Architecture**는 개발자가 서버를 관리하지 않고 애플리케이션을 개발하는 방식이다.
- **Amazon API Gateway**는 HTTP 요청을 수신하고 유효성을 검사한 후 **AWS Lambda**를 호출하며, 결과를 클라이언트에 반환한다.
- **AWS Lambda**는 비즈니스 로직을 실행하며 **DynamoDB**에 데이터를 저장하거나 **Amazon SES**를 호출해 이메일을 전송하는 등 다양한 작업을 수행할 수 있다.
- **AWS X-Ray**는 API Gateway, Lambda, DynamoDB 등 여러 서비스 간의 요청 흐름을 추적하여 성능 분석과 문제 해결을 지원한다.
- 정적 웹사이트는 **Amazon S3**에서 호스팅하고, 문의 양식은 **API Gateway → Lambda → Amazon SES**를 통해 이메일을 전송하는 서버리스 아키텍처를 구성할 수 있다.
- **Amazon Connect**는 **AWS Lambda**, **Amazon SNS**, **Amazon CloudFront**와 연동하여 콜백, 채팅, 이메일 알림 등을 제공하는 스마트한 고객 지원 시스템을 구축할 수 있다.
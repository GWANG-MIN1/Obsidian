# AWS 결제 및 비용 관리 도구

## AWS 계정의 결제 방식

AWS에서는 **단일 계정 결제**와 **통합 결제(Consolidated Billing)** 방식을 사용할 수 있다.

---

# 1. 단일 계정 결제 (Single Account Billing)

하나의 AWS 계정에서 모든 AWS 리소스를 관리하고 비용을 지불하는 방식이다.

### 특징

- 하나의 AWS 계정 사용
- 하나의 청구서(Bill)
- 하나의 결제 수단
- 소규모 프로젝트나 개인 사용자에게 적합

```
AWS Account
│
├── EC2
├── S3
├── RDS
└── Lambda

↓

하나의 청구서
```

---

# 2. 통합 결제 (Consolidated Billing)

여러 AWS 계정을 하나의 **Management Account(관리 계정)** 아래에서 관리하고, 청구서를 하나로 통합하는 방식이다.

주로 **AWS Organizations**를 사용하여 구성한다.

구성 예시

```
Management Account
│
├── 개발 계정
├── 운영 계정
├── 인프라 계정
└── 보안 계정

↓

청구서는 Management Account로 통합
```

### 특징

- 여러 AWS 계정 관리
- 청구서 통합
- 결제 관리 단순화
- 조직 전체 비용 관리 가능

또한 통합 결제를 사용하면 조직 전체의 사용량을 합산하여 **볼륨 기반 할인** 등의 혜택을 받을 수 있다.

---

# AWS Organizations와 통합 결제

AWS Organizations를 사용하면 다음과 같은 장점을 얻을 수 있다.

- 중앙 집중식 계정 관리
- 통합 결제(Consolidated Billing)
- 볼륨 기반 할인 혜택
- 보안 정책 중앙 관리
- 규정 준수 정책 적용

---

# AWS Billing Dashboard

## AWS Billing Dashboard란?

**AWS Billing Dashboard**는 AWS 비용 및 과금 정보를 확인하는 기본 대시보드이다.

비용을 관리할 때 가장 먼저 확인하는 화면이다.

확인 가능한 정보

- 현재까지 발생한 비용
- 월별 청구 금액
- 서비스별 비용
- 결제 내역
- 예상 청구 금액

---

# AWS Budgets

## AWS Budgets란?

**AWS Budgets**는 사용자가 설정한 예산(Budget)을 기준으로 **비용과 사용량을 추적**하는 서비스이다.

예산을 초과하거나 초과할 것으로 예상될 경우 알림을 받을 수 있다.

---

# AWS Budgets의 주요 기능

### 1. 사용자 지정 예산 생성

원하는 기준으로 예산을 설정할 수 있다.

예시

- 월 예산
- 서비스별 예산
- 프로젝트별 예산

---

### 2. 비용 추적

현재 비용이 예산 대비 얼마나 사용되었는지 확인할 수 있다.

---

### 3. 사용량 추적

비용뿐 아니라 AWS 서비스의 사용량도 추적할 수 있다.

---

### 4. 다양한 기준으로 예산 설정

다음과 같은 기준으로 예산을 생성할 수 있다.

- AWS 서비스
- 비용 범주(Cost Category)
- AWS 계정
- 태그(Tag)

---

# 태그(Tag)

태그(Tag)는 AWS 리소스에 추가하는 **메타데이터(Metadata)** 이다.

리소스를 쉽게 식별하고 그룹화할 수 있도록 도와준다.

예시

```
Environment = Production

Project = ShoppingMall

Team = Backend
```

### 태그 활용

- 리소스 검색
- 프로젝트별 관리
- 비용 분석
- 비용 할당

---

# AWS Cost Explorer

## AWS Cost Explorer란?

**AWS Cost Explorer**는 AWS 비용을 분석하고 시각화하며 미래 비용을 예측하는 서비스이다.

---

# Cost Explorer의 주요 기능

- 서비스별 비용 분석
- 계정별 비용 분석
- 태그(Tag)별 비용 분석
- 사용량 분석
- 비용 추세 확인
- 미래 비용 예측(Forecast)

---

# 태그 기반 비용 할당 (Tag-based Cost Allocation)

Cost Explorer에서는 리소스에 지정한 태그를 이용하여 비용을 분석할 수 있다.

예시

```
Project = AI

↓

AI 프로젝트 비용 확인
```

```
Team = Backend

↓

Backend 팀 비용 확인
```

이를 통해

- 프로젝트별 비용
- 부서별 비용
- 팀별 비용

을 쉽게 확인할 수 있다.

---

# AWS Pricing Calculator

## AWS Pricing Calculator란?

**AWS Pricing Calculator**는 AWS 서비스를 실제로 배포하기 전에 **예상 비용을 계산**할 수 있는 웹 기반 도구이다.

즉,

> **AWS 인프라를 구축하기 전에 예상 요금을 미리 계산하여 예산을 계획하는 데 사용하는 서비스**이다.

---

# AWS Pricing Calculator의 주요 기능

- 예상 비용 계산
- 서비스별 비용 비교
- 다양한 구성 시뮬레이션
- 월 예상 비용 산출
- 인프라 설계 단계에서 비용 예측

예시

- EC2 인스턴스 종류 변경 시 비용 비교
- S3 스토리지 용량 변경에 따른 비용 계산
- RDS 인스턴스 크기별 비용 비교

---

# 서비스 비교

| 서비스 | 목적 |
|---------|------|
| **AWS Billing Dashboard** | 현재 발생한 비용과 청구 정보 확인 |
| **AWS Budgets** | 예산 설정 및 비용·사용량 추적 |
| **AWS Cost Explorer** | 비용 분석, 시각화 및 미래 비용 예측 |
| **AWS Pricing Calculator** | 배포 전에 예상 비용 계산 |

---

# 핵심 정리

- AWS는 **단일 계정 결제**와 **통합 결제(Consolidated Billing)** 를 지원한다.
- **통합 결제**는 AWS Organizations를 통해 여러 계정의 청구서를 하나의 Management Account로 통합할 수 있다.
- **AWS Billing Dashboard**는 현재 비용과 청구 정보를 확인하는 기본 대시보드이다.
- **AWS Budgets**는 예산을 설정하고 비용 및 사용량을 추적하며, 예산 초과 시 알림을 받을 수 있다.
- **태그(Tag)** 는 리소스를 식별하고 그룹화하는 메타데이터로, 비용 분석과 비용 할당에 활용된다.
- **AWS Cost Explorer**는 서비스, 계정, 태그별 비용을 분석하고 미래 비용을 예측하는 서비스이다.
- **AWS Pricing Calculator**는 AWS 서비스를 배포하기 전에 예상 비용을 계산하는 웹 기반 도구이다.
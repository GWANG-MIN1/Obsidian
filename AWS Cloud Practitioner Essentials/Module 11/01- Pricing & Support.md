# AWS 요금 및 지원 (Billing & Support)

## AWS 요금 및 지원이란?

AWS에서는 사용한 만큼 비용을 지불하는 **종량제(Pay-as-you-go)** 방식으로 서비스를 이용한다.

따라서 AWS 서비스를 효율적으로 사용하려면

- 서비스별 과금 방식
- 비용 관리
- 비용 예측
- 비용 최적화
- 지원(Support) 플랜

을 이해하는 것이 중요하다.

---

# AWS 비용 관리가 필요한 이유

AWS에서는 사용하는 서비스에 따라 비용이 발생한다.

예를 들어

- EC2 인스턴스 실행 시간
- S3 스토리지 사용량
- 데이터 전송량
- RDS 데이터베이스 사용량

등에 따라 비용이 청구된다.

따라서 비용을 지속적으로 확인하고 예측하는 것이 중요하다.

---

# 학습할 주요 내용

이번 단원에서는 다음 내용을 학습한다.

- AWS 서비스의 과금 방식 이해
- AWS Billing Dashboard
- AWS Cost Explorer
- 비용 예측(Forecasting)
- AWS Support 플랜
- 비용 최적화(Best Practices)
- 비용 절감 사례

---

# AWS Billing Dashboard

## AWS Billing Dashboard란?

**AWS Billing Dashboard**는 현재까지 발생한 AWS 사용 비용과 청구 정보를 확인할 수 있는 서비스이다.

확인 가능한 정보

- 현재 누적 비용
- 월별 청구 금액
- 서비스별 비용
- 계정별 비용
- 예상 청구 금액

---

# AWS Billing Dashboard의 주요 기능

- 현재 사용 비용 확인
- 월별 청구 내역 확인
- 서비스별 비용 분석
- 결제 정보 관리
- 청구서 다운로드

---

# AWS Cost Explorer

## AWS Cost Explorer란?

**AWS Cost Explorer**는 AWS 비용을 시각적으로 분석하고 향후 비용을 예측할 수 있는 서비스이다.

단순히 비용을 확인하는 것뿐만 아니라

- 비용 추세 분석
- 서비스별 비용 비교
- 사용량 분석
- 비용 예측(Forecast)

까지 가능하다.

---

# AWS Cost Explorer의 주요 기능

### 1. 비용 분석

서비스별 비용을 분석할 수 있다.

예시

- EC2
- S3
- Lambda
- RDS

---

### 2. 비용 추세 확인

일별, 주별, 월별 비용 변화를 그래프로 확인할 수 있다.

---

### 3. 비용 예측(Forecast)

과거 사용량을 기반으로 앞으로 발생할 비용을 예측한다.

예를 들어

- 이번 달 예상 비용
- 다음 달 예상 비용

등을 확인할 수 있다.

---

### 4. 필터링

다양한 기준으로 비용을 분석할 수 있다.

예시

- 서비스별
- 계정별
- 리전별
- 태그(Tag)별

---

# AWS Support Plan

AWS는 다양한 수준의 기술 지원(Support)을 제공한다.

대표적인 Support Plan

- Basic
- Developer
- Business
- Enterprise

각 플랜은

- 기술 지원 수준
- 응답 시간
- 추가 서비스

등이 다르므로 조직 규모와 요구사항에 맞는 플랜을 선택해야 한다.

---

# 비용 최적화(Cost Optimization)

비용 최적화란 필요한 성능을 유지하면서 AWS 비용을 최소화하는 것을 의미한다.

대표적인 방법

- 사용하지 않는 리소스 삭제
- 적절한 인스턴스 타입 선택
- Auto Scaling 활용
- Savings Plans 및 Reserved Instances 활용
- 스토리지 수명 주기(Lifecycle) 정책 적용
- Cost Explorer와 Trusted Advisor를 통한 비용 분석

---

# 비용 최적화 사용 사례

### 사례 1

사용하지 않는 EC2 인스턴스를 종료하여 비용 절감

---

### 사례 2

S3 Lifecycle을 사용하여 오래된 데이터를 저렴한 스토리지 클래스로 이동

---

### 사례 3

Auto Scaling을 사용하여 필요한 만큼만 EC2 인스턴스를 실행

---

### 사례 4

AWS Cost Explorer를 통해 비용 증가 원인을 분석하고 향후 비용을 예측

---

# Billing Dashboard vs Cost Explorer

| 서비스 | 목적 |
|---------|------|
| **AWS Billing Dashboard** | 현재 및 과거의 청구 정보와 비용 확인 |
| **AWS Cost Explorer** | 비용 분석, 시각화 및 향후 비용 예측 |

---

# 핵심 정리

- AWS는 **종량제(Pay-as-you-go)** 방식으로 과금된다.
- **AWS Billing Dashboard**는 현재까지의 비용과 청구 정보를 확인하는 서비스이다.
- **AWS Cost Explorer**는 비용을 분석하고 미래 비용을 예측하는 서비스이다.
- AWS는 **Basic, Developer, Business, Enterprise** 등 다양한 Support Plan을 제공한다.
- 비용 최적화는 불필요한 리소스를 제거하고 적절한 서비스를 선택하여 AWS 비용을 절감하는 것이다.
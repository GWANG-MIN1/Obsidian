# AWS Trusted Advisor & IAM Access Analyzer

# AWS Trusted Advisor

## AWS Trusted Advisor란?

**AWS Trusted Advisor**는 AWS 환경을 실시간으로 점검하여 **비용, 성능, 보안, 내결함성 및 서비스 한도** 측면에서 개선 사항을 추천해 주는 서비스이다.

쉽게 말하면,

> **AWS 계정을 자동으로 점검하여 모범 사례(Best Practices)를 따르고 있는지 확인하고 개선 사항을 제안하는 서비스**이다.

점검 결과는 **AWS Management Console**에서 확인할 수 있다.

---

# AWS Trusted Advisor의 점검 항목

Trusted Advisor는 다음 5가지 영역을 중심으로 AWS 환경을 평가한다.

### 1. 비용 최적화 (Cost Optimization)

불필요한 비용이 발생하는 리소스를 찾아 비용 절감을 지원한다.

예시

- 사용하지 않는 EC2 인스턴스
- 유휴(Idle) Load Balancer
- 사용하지 않는 EBS 볼륨
- 오래된 스냅샷

---

### 2. 성능 (Performance)

서비스 성능을 향상시킬 수 있는 권장 사항을 제공한다.

예시

- 높은 사용률의 EC2 인스턴스
- 성능 개선 가능한 구성 추천

---

### 3. 보안 (Security)

보안 모범 사례를 준수하는지 확인한다.

예시

- 루트(Root) 사용자 MFA 미설정
- 공개 접근 가능한 S3 버킷
- 보안 그룹의 과도한 접근 허용
- IAM 보안 설정 점검

> 예를 들어 **루트 사용자에 대해 MFA(다중 인증)를 설정하지 않은 경우 경고를 제공**한다.

---

### 4. 내결함성 (Fault Tolerance)

장애 발생 시 서비스가 계속 동작할 수 있도록 구성되었는지 확인한다.

예시

- Multi-AZ 미구성
- 백업 미설정
- 고가용성 구성 여부 확인

---

### 5. 서비스 한도 (Service Limits)

AWS 서비스의 사용량이 한도(Service Quotas)에 가까운지 확인한다.

예시

- EC2 인스턴스 한도
- VPC 개수 제한
- Elastic IP 사용량

---

# AWS Trusted Advisor의 장점

- AWS 계정 상태를 실시간으로 점검
- 비용 절감 기회 제공
- 보안 취약점 탐지
- 성능 향상 권장 사항 제공
- 장애 대비(내결함성) 개선
- 서비스 한도 초과 방지

---

# AWS IAM Access Analyzer

## IAM Access Analyzer란?

**IAM Access Analyzer**는 AWS 리소스에 대한 **외부 액세스를 분석**하고, IAM 정책이 조직의 보안 정책과 일치하는지 검증하는 서비스이다.

또한 권한을 분석하여 **최소 권한 원칙(Principle of Least Privilege)** 을 적용할 수 있도록 지원한다.

쉽게 말하면,

> **누가 어떤 리소스에 접근할 수 있는지 분석하고, 과도한 권한을 찾아 최소 권한으로 개선하도록 도와주는 서비스**이다.

---

# IAM Access Analyzer의 주요 기능

### 1. 외부 액세스 분석

외부 계정이나 조직에서 접근 가능한 리소스를 찾아준다.

분석 대상 예시

- S3 버킷
- KMS 키
- IAM 역할
- SQS 큐
- Secrets Manager 비밀 정보

---

### 2. IAM 정책 검증

IAM 정책을 분석하여

- 잘못된 정책
- 위험한 정책
- 과도한 권한

등을 찾아준다.

---

### 3. 권한 세분화

필요 이상의 권한을 제거하여 최소 권한 원칙을 적용할 수 있도록 지원한다.

예시

```
기존

AdministratorAccess

↓

개선

S3:GetObject
S3:PutObject
```

---

### 4. 사용하지 않는 권한 확인

실제로 사용하지 않는 권한을 분석하여 제거할 수 있도록 도와준다.

이를 통해 불필요한 권한을 줄이고 보안을 강화할 수 있다.

---

### 5. 정책 검토 자동화

IAM 정책을 자동으로 분석하여 보안 위험을 줄인다.

---

# IAM Access Analyzer의 장점

- 권한 세분화 지원
- IAM 정책 검증
- 최소 권한 원칙 적용 지원
- 외부 액세스 분석
- IAM 정책 검토 자동화
- 보안 강화

---

# IAM Access Analyzer 사용 사례

- 누가 어떤 리소스에 접근 가능한지 확인
- 외부 계정에 공유된 리소스 분석
- 사용하지 않는 권한 제거
- 과도한 IAM 권한 최소화
- 최소 권한 정책 생성 및 적용

---

# AWS Trusted Advisor vs IAM Access Analyzer

| 서비스 | 목적 |
|---------|------|
| **AWS Trusted Advisor** | AWS 환경을 점검하여 비용, 성능, 보안, 내결함성 및 서비스 한도 측면의 개선 사항을 추천 |
| **IAM Access Analyzer** | IAM 정책과 리소스의 외부 액세스를 분석하여 최소 권한 원칙을 적용하고 보안을 강화 |

---

# 핵심 정리

- **AWS Trusted Advisor**는 AWS 환경을 실시간으로 점검하여 비용, 성능, 보안, 내결함성 및 서비스 한도에 대한 개선 사항을 추천하는 서비스이다.
- Trusted Advisor는 **비용 최적화, 성능, 보안, 내결함성, 서비스 한도**의 5개 영역을 점검한다.
- **IAM Access Analyzer**는 AWS 리소스의 외부 액세스를 분석하고 IAM 정책을 검증하여 최소 권한 원칙을 적용하도록 지원하는 서비스이다.
- IAM Access Analyzer를 활용하면 **누가 무엇에 접근할 수 있는지 확인하고, 사용하지 않는 권한과 과도한 권한을 줄여 보안을 강화**할 수 있다.
# AWS 거버넌스(Governance)

## 거버넌스(Governance)란?

**거버넌스(Governance)** 란 조직의 IT 목표를 달성하기 위해 **정책(Policies), 프로세스(Processes), 구조(Structure)** 를 관리하고, 조직이 규칙과 표준을 준수하는지 확인하는 프레임워크이다.

쉽게 말하면,

> **조직 전체가 정해진 규칙과 보안 정책을 올바르게 따르도록 관리하는 체계**이다.

---

# 거버넌스가 필요한 이유

회사가 AWS를 처음 사용할 때는 계정이 하나뿐이라 관리가 쉽다.

하지만 조직이 성장하면

- 여러 AWS 계정
- 여러 부서
- 다양한 프로젝트
- 서로 다른 보안 정책

을 관리해야 하므로 거버넌스가 매우 중요해진다.

---

# AWS Control Tower

## AWS Control Tower란?

**AWS Control Tower**는 여러 AWS 계정을 **중앙에서 쉽고 안전하게 구성하고 관리**할 수 있도록 도와주는 거버넌스 서비스이다.

AWS Organizations를 기반으로 동작하며, AWS의 모범 사례(Best Practices)에 따라 멀티 계정 환경을 자동으로 구성한다.

---

# AWS Control Tower가 필요한 이유

사용자가 많아지고 AWS 계정이 늘어날수록

- 초기 계정 설정
- 보안 정책 적용
- 규정 준수 관리
- 계정 운영

이 복잡해진다.

AWS Control Tower는 이러한 작업을 자동화하여 일관된 환경을 유지할 수 있도록 지원한다.

---

# AWS Control Tower의 주요 기능

### 1. 계정 설정 표준화

새로운 AWS 계정을 생성할 때

- 네트워크
- 보안 설정
- IAM 구성

등을 AWS 모범 사례에 맞게 자동으로 구성한다.

---

### 2. 계정 프로비저닝(Account Provisioning)

새로운 AWS 계정을 자동으로 생성하고 조직에 추가할 수 있다.

---

### 3. 가드레일(Guardrails)

**가드레일(Guardrails)** 은 조직 전체에 적용되는 보안 및 규정 준수 규칙이다.

예시

- 루트 계정 사용 제한
- CloudTrail 활성화 강제
- S3 공개 접근 차단

가드레일을 통해 모든 계정이 동일한 정책을 준수하도록 할 수 있다.

---

### 4. 규정 준수 모니터링

Control Tower 대시보드에서

- 계정 상태
- 가드레일 적용 여부
- 규정 준수 상태

를 한눈에 확인할 수 있다.

---

### 5. 계정 및 리소스 관리

팀에서 새로운 계정이나 리소스를 생성해도

- 지속적으로 감시(Monitoring)
- 규정 위반 여부 확인
- 필요한 경우 수정(Remediation)

할 수 있다.

---

# AWS Control Tower 동작 과정

```
AWS Organizations
        ↓
Control Tower 설정
        ↓
계정 자동 생성
        ↓
가드레일 적용
        ↓
보안 및 규정 준수 정책 적용
        ↓
지속적인 모니터링 및 관리
```

---

# AWS Control Tower의 장점

- 멀티 계정 환경 자동 구성
- AWS 모범 사례 기반 초기 설정
- 보안 정책 자동 적용
- 가드레일을 통한 규정 준수 관리
- 중앙에서 모든 계정 관리
- 워크로드 운영 효율 향상

---

# AWS Service Catalog

## AWS Service Catalog란?

**AWS Service Catalog**는 조직에서 **승인된 AWS 리소스(Products)** 를 카탈로그 형태로 관리하고 사용자에게 제공하는 서비스이다.

관리자는 사용할 수 있는 리소스를 미리 등록하고, 사용자는 승인된 리소스만 셀프서비스(Self-Service) 방식으로 배포할 수 있다.

---

# AWS Service Catalog의 주요 기능

- 승인된 AWS 리소스 카탈로그 생성
- 카탈로그 공유
- 표준화된 리소스 배포
- 리소스 구성 관리
- 액세스 제어

---

# 활용 예시

새로운 AWS 계정을 생성하면

자동으로

- VPC
- 서브넷
- 보안 그룹
- IAM 역할
- 보안 도구

등의 표준 리소스를 동일한 방식으로 배포할 수 있다.

이를 통해 조직 전체의 일관성을 유지할 수 있다.

---

# AWS Service Catalog의 장점

- 승인된 리소스를 빠르게 검색 및 배포
- 셀프서비스(Self-Service) 지원
- 표준화된 리소스 사용
- 거버넌스 유지
- 여러 계정에서도 일관된 환경 유지
- 배포 시간 단축

---

# AWS Service Catalog 사용 사례

- 조직 전체 리소스 프로비저닝
- 액세스 제어 적용
- 표준 인프라 배포
- CI/CD 파이프라인의 리소스 프로비저닝 자동화

---

# AWS License Manager

## AWS License Manager란?

**AWS License Manager**는 소프트웨어 라이선스를 중앙에서 추적하고 관리하는 서비스이다.

이를 통해 라이선스 사용량을 관리하고 라이선스 비용을 최적화할 수 있다.

---

# AWS License Manager의 주요 기능

- 라이선스 추적
- 라이선스 사용량 관리
- 라이선스 규정 준수 확인
- 라이선스 비용 최적화
- 라이선스 사용 현황 가시성 제공

---

# AWS License Manager의 장점

- 라이선스 사용 현황 가시성 확보
- 라이선스 관리 자동화
- 라이선스 미준수 위험 감소
- 비용 최적화
- 중앙 집중식 라이선스 관리

---

# AWS License Manager 사용 사례

- Microsoft 라이선스 관리
- Oracle 라이선스 관리
- Software Assurance 관리
- Microsoft License Mobility 활용
- 라이선스 사용량 최적화

---

# 서비스 비교

| 서비스 | 목적 |
|---------|------|
| **AWS Control Tower** | 멀티 계정 환경의 거버넌스 및 초기 구성 자동화 |
| **AWS Service Catalog** | 승인된 AWS 리소스를 카탈로그 형태로 제공하고 표준화된 배포 지원 |
| **AWS License Manager** | 소프트웨어 라이선스 추적, 관리 및 비용 최적화 |

---

# 핵심 정리

- **거버넌스(Governance)** : 정책, 프로세스 및 구조를 통해 조직의 IT 목표와 규정 준수를 관리하는 체계
- **AWS Control Tower** : 멀티 계정 환경을 AWS 모범 사례에 따라 자동 구성하고 가드레일을 통해 거버넌스를 적용하는 서비스
- **Guardrails** : 보안 및 규정 준수를 위한 조직 전체의 정책과 규칙
- **AWS Service Catalog** : 승인된 AWS 리소스를 카탈로그 형태로 제공하여 표준화된 리소스를 배포하는 서비스
- **AWS License Manager** : 소프트웨어 라이선스를 중앙에서 관리하고 비용을 최적화하는 서비스
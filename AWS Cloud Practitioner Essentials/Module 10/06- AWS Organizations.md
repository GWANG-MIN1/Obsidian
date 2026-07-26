# AWS Organizations

## AWS Organizations란?

**AWS Organizations**는 여러 AWS 계정을 **중앙에서 통합 관리**할 수 있는 계정 관리 서비스이다.

처음 AWS를 사용할 때는 일반적으로 **하나의 AWS 계정**으로 시작한다.

하지만 기업에서 AWS 사용 규모가 커지면 용도에 따라 여러 개의 AWS 계정을 운영하게 된다.

예시

- 프로덕션(Production) 계정
- 비프로덕션(Non-Production) 계정
- 개발(Development) 계정
- 인프라 팀 전용 계정
- 개발자 전용 계정

각 계정은 서로 다른 사용자, 권한, AWS 서비스 접근 권한 및 사용자 지정 설정을 가질 수 있다.

---

# AWS Organizations가 필요한 이유

여러 AWS 계정을 각각 관리하면

- 결제 관리가 복잡해지고
- 보안 정책을 일관되게 적용하기 어려우며
- 권한 관리가 번거로워진다.

AWS Organizations를 사용하면 여러 계정을 하나의 조직(Organization)으로 묶어 중앙에서 관리할 수 있다.

---

# AWS Organizations의 주요 기능

### 1. 여러 AWS 계정 중앙 관리

조직 내 여러 AWS 계정을 하나의 콘솔에서 관리할 수 있다.

관리 항목

- 계정 생성 및 관리
- 통합 결제(Billing)
- 액세스 관리
- 보안 관리
- 규정 준수(Compliance)
- 계정 간 리소스 공유

---

### 2. 계층적 계정 그룹화 (Organizational Unit, OU)

AWS Organizations는 계정을 **OU(Organizational Unit)** 라는 그룹 단위로 관리할 수 있다.

예시

```
Organization
│
├── Production OU
│      ├── Web Account
│      └── Database Account
│
├── Development OU
│      ├── Dev Account
│      └── Test Account
│
└── Security OU
       └── Audit Account
```

OU를 사용하면

- 부서별
- 프로젝트별
- 개발/운영 환경별
- 규정 준수 요구사항별

로 계정을 쉽게 그룹화할 수 있다.

---

# 규정 준수를 위한 OU 활용

특정 규제나 보안 요구사항을 충족해야 하는 계정을 하나의 OU에 배치할 수 있다.

예시

- 의료 서비스 계정
- 금융 서비스 계정
- 개인정보 처리 계정

이처럼 동일한 규정 준수 요구사항을 가진 계정을 하나의 OU로 관리하면 정책을 일괄 적용하기 쉽다.

---

# 관리 계정 (Management Account)

조직에는 하나의 **Management Account(관리 계정)** 가 존재한다.

관리 계정의 관리자는

- 조직 전체 관리
- 계정 생성
- OU 생성
- SCP 적용
- 조직 정책 관리

등을 수행할 수 있다.

---

# SCP (Service Control Policy)

**SCP(Service Control Policy)** 는 AWS Organizations에서 사용하는 **조직 수준의 권한 제어 정책**이다.

SCP는 IAM 정책보다 상위에서 동작하며, **계정이 사용할 수 있는 AWS 서비스와 API의 최대 권한을 제한**한다.

> **SCP는 권한을 부여하는 정책이 아니라, 사용할 수 있는 최대 권한을 제한하는 정책이다.**

즉, IAM 정책에서 권한을 허용했더라도 SCP에서 차단하면 해당 작업은 수행할 수 없다.

---

# SCP 적용 대상

SCP는 다음 대상에 적용할 수 있다.

- Organization(조직 전체)
- Organizational Unit(OU)
- 개별 Member Account(멤버 계정)

※ **SCP는 IAM 사용자(User), 그룹(Group), 역할(Role)과 같은 자격 증명(Credentials)이나 개별 AWS 리소스에 직접 적용하는 것이 아니다.**  
SCP는 **조직, OU 또는 AWS 계정 단위**에 적용되어 그 계정 안의 IAM 사용자와 역할이 가질 수 있는 최대 권한을 제한한다.

---

# SCP 활용 예시

### 예시 1. 개발 계정에서 RDS 생성 금지

```
Development OU

허용
- EC2

차단
- RDS
```

---

### 예시 2. 모든 계정에서 IAM 사용자 삭제 금지

```
SCP

Deny
iam:DeleteUser
```

---

### 예시 3. 특정 리전만 사용 허용

```
허용
- ap-northeast-2 (서울)

차단
- us-east-1
- eu-west-1
```

---

# AWS Organizations의 장점

- 여러 AWS 계정을 중앙에서 관리
- 통합 결제(Consolidated Billing) 지원
- 보안 및 규정 준수 정책 일괄 적용
- OU를 이용한 계층적 계정 관리
- SCP를 통한 조직 전체 권한 제어
- 계정 간 일관된 보안 정책 적용

---

# AWS Organizations 핵심 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| Organization | 여러 AWS 계정을 관리하는 최상위 조직 |
| Management Account | 조직 전체를 관리하는 기본 계정 |
| Member Account | 조직에 속한 일반 AWS 계정 |
| Organizational Unit (OU) | 여러 계정을 그룹화하는 단위 |
| Service Control Policy (SCP) | 조직, OU, 계정 단위의 최대 권한을 제한하는 정책 |

---

# 핵심 정리

- **AWS Organizations**는 여러 AWS 계정을 중앙에서 통합 관리하는 서비스이다.
- 결제, 보안, 액세스 제어 및 규정 준수를 중앙에서 관리할 수 있다.
- **OU(Organizational Unit)** 를 이용해 계정을 부서별, 프로젝트별, 환경별로 그룹화할 수 있다.
- **Management Account**가 조직 전체를 관리한다.
- **SCP(Service Control Policy)** 는 조직, OU 또는 멤버 계정에 적용되며 AWS 서비스와 API의 **최대 권한**을 제한한다.
- SCP는 IAM 사용자나 개별 리소스에 직접 적용되는 정책이 아니라 **계정 단위의 권한 제한 정책**이다.
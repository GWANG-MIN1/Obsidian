# AWS IAM (Identity and Access Management)

## 사용자 권한 및 액세스 제어

AWS 계정을 생성하면 **루트 사용자(Root User)** 자격 증명이 생성됩니다.

루트 사용자는 계정의 **소유자**로 생각하면 됩니다.

> ☕ 예시: 커피숍의 사장

루트 사용자는 AWS 계정 내 **모든 리소스**를 제어할 수 있습니다.

- Amazon EC2
- Amazon S3
- Amazon RDS
- AI/ML 서비스
- IAM
- 결제(Billing)
- 기타 모든 AWS 서비스

---

# 루트 사용자 보안

루트 사용자는 가장 강력한 권한을 가지므로 반드시 보호해야 합니다.

## 권장 사항

- 강력한 비밀번호 사용
- MFA(다중 인증) 활성화

> **루트 사용자는 계정 생성 및 긴급 상황(Emergency)에서만 사용하는 것이 권장됩니다.**

일상적인 업무(Task)에는 루트 사용자를 사용하지 않습니다.

---

# IAM(User) 사용

일반적인 업무는 **IAM 사용자(IAM User)** 를 생성하여 수행합니다.

IAM 사용자는 생성 직후에는 **아무 권한도 없습니다.**

즉,

> 기본적으로 모든 작업이 거부(Deny)됩니다.

필요한 권한을 명시적으로 부여해야 합니다.

---

# 최소 권한 원칙 (Principle of Least Privilege)

사용자에게 **업무 수행에 필요한 최소한의 권한만 부여**하는 보안 원칙입니다.

예를 들어

- 개발자 → EC2 관리 가능
- 데이터 분석가 → S3 읽기만 가능
- 회계팀 → Billing 조회만 가능

불필요한 권한은 부여하지 않습니다.

---

# IAM 정책 (IAM Policy)

IAM 정책은 **사용자가 수행할 수 있는 권한을 정의하는 JSON 문서**입니다.

## 예시

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "s3:ListBucket",
    "Resource": "arn:aws:s3:::coffee_shop_reports"
  }
}
```

이 정책을 사용자에게 연결하면

- `coffee_shop_reports` S3 버킷의 목록 조회(ListBucket)가 허용됩니다.

---

## 주요 요소

### Version

정책 언어 버전

```json
"Version": "2012-10-17"
```

---

### Effect

권한 적용 방식

- `Allow` → 허용
- `Deny` → 거부

---

### Action

허용 또는 거부할 AWS API 작업

예시

- `s3:ListBucket`
- `ec2:StartInstances`
- `lambda:InvokeFunction`

---

### Resource

권한이 적용될 AWS 리소스

예시

```
arn:aws:s3:::coffee_shop_reports
```

---

# IAM 그룹 (IAM Group)

사용자마다 정책을 하나씩 연결하는 것은 관리가 어렵습니다.

이럴 때 **IAM 그룹(Group)** 을 사용합니다.

예시

```
Developers
 ├─ Kim
 ├─ Lee
 └─ Park
```

그룹에 정책을 연결하면

→ 그룹의 모든 사용자가 동일한 권한을 상속받습니다.

예시

- Developers
- Managers
- Finance
- Interns

---

# IAM 구성 요소

AWS IAM은 다음 요소들로 구성됩니다.

- **Root User** : 계정 소유자
- **IAM User** : 개별 사용자
- **IAM Group** : 사용자 그룹
- **IAM Policy** : 권한을 정의하는 JSON 문서
- **IAM Role** : 임시 권한 제공

---

# IAM 역할 (IAM Role)

IAM 역할(Role)은 **임시로 권한을 부여**하기 위한 기능입니다.

특징

- 권한(Policy)을 연결할 수 있음
- Allow / Deny 정책 사용
- 임시로 맡는 권한
- 사용자 이름(User Name) 없음
- 비밀번호 없음
- Access Key 같은 정적 자격 증명 없음

즉,

필요한 순간에만 역할을 맡아 권한을 얻습니다.

---

## IAM Role을 사용하는 이유

모든 사람에게 IAM 사용자를 생성할 필요가 없습니다.

예를 들어

회사 직원은

- 회사 계정(사내 계정)으로 로그인
- 필요한 경우 AWS Role을 맡아 AWS 리소스 접근

처럼 사용할 수 있습니다.

---

# AWS IAM Identity Center

AWS IAM Identity Center는

AWS 계정과 애플리케이션의 **사용자 및 권한을 중앙에서 관리**하는 서비스입니다.

주요 기능

- 사용자 및 그룹 중앙 관리
- 여러 AWS 계정 관리
- 여러 애플리케이션 접근 관리
- Single Sign-On(SSO) 지원
- 페더레이션(Federation) 지원

---

## Single Sign-On (SSO)

한 번 로그인하면

하나의 자격 증명으로 여러 서비스에 로그인할 수 있습니다.

예시

```
회사 계정 로그인

        ↓

AWS Console
Slack
Salesforce
GitHub
```

비밀번호를 각각 기억할 필요가 없습니다.

---

## 페더레이션(Federation)

기업의 기존 로그인 시스템(예: Microsoft Entra ID, Okta 등)과 연동하여,

**하나의 자격 증명 세트**로 여러 애플리케이션과 AWS 서비스에 접근할 수 있도록 지원합니다.

---

# AWS Secrets Manager

Secrets Manager는

민감한 정보(Secrets)를 **안전하게 저장, 관리, 교체(Rotation)** 하는 서비스입니다.

관리 가능한 정보

- 데이터베이스 자격 증명
- API Key
- 비밀번호
- 인증 토큰
- 기타 보안 암호(Secret)

---

## 주요 기능

- 안전한 저장
- 자동 암호 교체(Rotation)
- 애플리케이션에서 안전하게 조회
- 접근 권한 제어(IAM)

---

# AWS Systems Manager (SSM)

AWS Systems Manager는

AWS 및 하이브리드 환경의 서버(노드)를 **중앙에서 관리**하는 서비스입니다.

---

## 주요 기능

- 노드(Node) 상태 확인
- 운영체제(OS) 정보 조회
- 사용자 관리
- 레지스트리 편집
- 보안 패치 자동화
- 명령 실행 자동화
- 여러 계정 및 리전 통합 관리

---

## 노드(Node)

노드는 네트워크에서 관리 대상이 되는 장치입니다.

예시

- EC2 인스턴스
- 온프레미스 서버
- 가상 머신(VM)
- 하이브리드 서버

---

# 핵심 정리

- **Root User**는 계정 전체를 관리하는 최고 권한 사용자이다.
- 루트 사용자는 **MFA를 활성화**하고 일상적인 작업에는 사용하지 않는다.
- 일반 업무는 **IAM User**를 사용한다.
- IAM 사용자는 기본적으로 아무 권한도 없다.
- **최소 권한 원칙(Least Privilege)** 에 따라 필요한 권한만 부여한다.
- **IAM Policy**는 권한을 정의하는 JSON 문서이다.
- **IAM Group**을 사용하면 여러 사용자에게 동일한 권한을 쉽게 부여할 수 있다.
- **IAM Role**은 사용자나 서비스가 **임시로 권한을 획득**하는 방식이다.
- **IAM Identity Center**는 SSO와 중앙 집중식 사용자 관리를 제공한다.
- **Secrets Manager**는 비밀번호, API Key 등 민감한 정보를 안전하게 관리한다.
- **Systems Manager(SSM)** 는 서버 및 노드를 중앙에서 관리하고 운영을 자동화하는 서비스이다.
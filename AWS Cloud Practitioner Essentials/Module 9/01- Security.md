# AWS 계정 보안 (AWS Account Security)

## AWS 계정 보안의 중요성

AWS 계정은 클라우드 리소스에 대한 모든 권한을 관리하는 핵심 요소입니다.

계정이 탈취되면 다음과 같은 문제가 발생할 수 있습니다.

- 데이터 유출
- 서비스 중단
- 과도한 비용 발생
- 리소스 오용 및 사기(Fraud)

따라서 AWS 계정 보안은 가장 먼저 고려해야 하는 요소입니다.

---

# 반드시 알아야 할 보안 구성 요소

## 1. 인증 (Authentication)

**인증(Authentication)** 은 자격 증명(Credentials)을 통해 사용자 또는 엔터티(Entity)의 신원을 확인하는 과정입니다.

### 예시
- 사용자 이름 + 비밀번호
- MFA(다중 인증)
- Access Key
- IAM Role

> "당신이 누구인지 확인하는 과정"

---

## 2. 권한 부여 (Authorization)

**권한 부여(Authorization)** 는 인증이 완료된 사용자에게 수행 가능한 작업을 허용하는 과정입니다.

예를 들어,

- S3 읽기만 가능
- EC2 생성 가능
- RDS 삭제 불가

등의 권한을 IAM 정책으로 제어합니다.

> "무엇을 할 수 있는지 결정하는 과정"

---

# 인증(Authentication) vs 권한 부여(Authorization)

|구분|인증 (Authentication)|권한 부여 (Authorization)|
|---|---|---|
|목적|사용자의 신원 확인|허용된 작업 결정|
|질문|"누구인가?"|"무엇을 할 수 있는가?"|
|예시|로그인, MFA|IAM Policy|

---

# 인증과 권한 부여가 중요한 이유

AWS에서는 인증(Authentication)과 권한 부여(Authorization)가 다음과 같은 역할을 수행합니다.

- 고객 신뢰 유지
- 사기(Fraud) 방지
- 무단 접근 방지
- 리소스 오용 방지

궁극적으로 조직은 **무단 액세스(Unauthorized Access)** 와 **오용(Misuse)** 을 방지해야 합니다.

---

# AWS 공동 책임 모델 (Shared Responsibility Model)

AWS는 **공동 책임 모델(Shared Responsibility Model)** 을 제공합니다.

즉,

- AWS는 **클라우드 자체의 보안(Security of the Cloud)** 을 책임지고,
- 고객은 **클라우드 내 보안(Security in the Cloud)** 을 책임집니다.

따라서 IAM 설정, 권한 관리, 데이터 보호 등은 고객의 책임입니다.

---

# AWS 보안 제어(Security Controls)

AWS는 다음과 같은 보안 제어(Security Controls)를 제공합니다.

## 1. 사용자 권한 및 액세스 관리
- IAM(User, Group, Role)
- IAM Policy
- MFA
- AWS Organizations

---

## 2. 네트워크 및 애플리케이션 보호
- Security Group
- Network ACL
- AWS WAF
- AWS Shield

---

## 3. 데이터 보호
- 암호화(Encryption)
- AWS KMS
- AWS Secrets Manager
- AWS Certificate Manager(ACM)

---

## 4. 탐지 및 사고 대응
- AWS CloudTrail
- Amazon GuardDuty
- Amazon Inspector
- AWS Security Hub
- Amazon Detective

---

# 핵심 정리

- AWS 계정 보안은 가장 중요한 보안 요소이다.
- 보안의 핵심은 **인증(Authentication)** 과 **권한 부여(Authorization)** 이다.
- 인증은 **신원을 확인**하는 과정이다.
- 권한 부여는 **허용된 작업을 결정**하는 과정이다.
- AWS는 공동 책임 모델을 기반으로 보안을 제공한다.
- AWS 보안 제어는 크게 다음 4가지 영역으로 구성된다.
  1. 사용자 권한 및 액세스 관리
  2. 네트워크 및 애플리케이션 보호
  3. 데이터 보호
  4. 탐지 및 사고 대응
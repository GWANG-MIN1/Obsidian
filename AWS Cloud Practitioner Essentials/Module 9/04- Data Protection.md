# 데이터 보호 (Data Protection)

데이터 보호(Data Protection)는 AWS 보안의 핵심 요소 중 하나입니다.

데이터는 **저장될 때(At Rest)** 와 **전송될 때(In Transit)** 모두 보호되어야 합니다.

가장 대표적인 보호 방법은 **암호화(Encryption)** 입니다.

---

# 암호화 (Encryption)

암호화는

**권한이 있는 사용자만 데이터를 읽을 수 있도록 데이터를 보호하는 기술**입니다.

암호화된 데이터는 승인된 사용자만 **암호화 키(Encryption Key)** 를 사용하여 복호화(Decrypt)할 수 있습니다.

즉,

- 올바른 키가 있어야 데이터 접근 가능
- 키가 없으면 데이터 내용을 확인할 수 없음

---

# 암호화 과정

```
원본 데이터

      ↓

암호화(Encryption)

      ↓

암호문(Ciphertext)

      ↓

암호화 키(Key)

      ↓

복호화(Decryption)

      ↓

원본 데이터
```

---

# 데이터 암호화 방식

AWS에서는 데이터를 다음 두 가지 방식으로 보호합니다.

## 1. 저장 시 암호화 (Encryption at Rest)

데이터가 저장 장치에 저장되어 있을 때 암호화하는 방식입니다.

즉,

데이터가 이동하지 않고 저장된 상태를 보호합니다.

예시

- Amazon S3 객체
- Amazon EBS 볼륨
- Amazon DynamoDB 테이블
- Amazon RDS 데이터베이스

---

## Amazon S3

Amazon S3는

새로 생성한 버킷에서 **기본적으로 저장 시 암호화(Server-Side Encryption)** 를 지원합니다.

따라서

버킷에 업로드되는 모든 새로운 객체는 자동으로 암호화됩니다.

---

## Amazon EBS

Amazon EBS는

- EBS 볼륨
- EBS 스냅샷

모두 저장 시 암호화할 수 있습니다.

---

## Amazon DynamoDB

Amazon DynamoDB는

모든 테이블에 대해 **서버 측 저장 시 암호화(Server-Side Encryption)** 를 기본적으로 제공합니다.

필요한 경우 추가적인 암호화 제어도 가능합니다.

---

# AWS Key Management Service (AWS KMS)

AWS KMS는

암호화 키(Encryption Key)를 생성하고 관리하는 서비스입니다.

암호화 자체보다 **키 관리(Key Management)** 에 중점을 둔 서비스입니다.

---

## 주요 기능

- 암호화 키 생성
- 암호화 키 저장
- 암호화 키 관리
- 데이터 암호화 및 복호화 지원
- 키 접근 권한 제어

---

## IAM과 연동

KMS는 IAM과 연동되어

특정 IAM 사용자 또는 IAM 역할(Role)만

암호화 키를 사용할 수 있도록 설정할 수 있습니다.

---

## 중요한 특징

> **암호화 키는 KMS를 벗어나지 않습니다.**

즉,

키를 안전하게 보호하면서 암호화와 복호화를 수행할 수 있습니다.

---

## 암호화 키(Encryption Key)

암호화 키는

데이터를 잠그고(암호화) 다시 여는(복호화)

**임의의 숫자와 문자열로 구성된 값**입니다.

비유하면

> 집의 열쇠처럼, 올바른 키가 있어야만 문을 열 수 있습니다.

---

# Amazon Macie

Amazon Macie는

AWS에 저장된 **민감한 데이터(Sensitive Data)** 를 자동으로 탐지하고 보호하는 서비스입니다.

---

## 탐지 가능한 데이터

- 개인정보(PII)
- 주민등록번호
- 신용카드 번호
- 이메일 주소
- 기타 민감한 정보

---

## 주요 기능

- 민감한 데이터 자동 탐지
- 데이터 분류(Classification)
- 보안 위험 모니터링
- 데이터 노출 여부 확인

---

# 전송 중 암호화 (Encryption in Transit)

데이터가

한 시스템에서 다른 시스템으로 이동하는 동안 암호화하는 방식입니다.

예시

```
사용자

      ↓

인터넷

      ↓

AWS 서버
```

이 과정에서 데이터가 암호화되어 제3자가 내용을 볼 수 없도록 보호합니다.

---

# SSL / TLS

전송 중 암호화는 일반적으로

SSL 또는 TLS 프로토콜을 사용합니다.

## SSL (Secure Sockets Layer)

초기의 보안 통신 프로토콜입니다.

현재는 대부분 사용되지 않습니다.

---

## TLS (Transport Layer Security)

TLS는 SSL의 개선된 버전으로,

현재 대부분의 HTTPS 통신에서 사용됩니다.

---

# SSL/TLS 인증서

SSL/TLS 인증서는

두 시스템 간 **암호화된 네트워크 연결**을 설정하는 데 사용됩니다.

이를 통해

- 데이터 도청 방지
- 데이터 변조 방지
- 서버 신원 확인

이 가능합니다.

---

# HTTPS

웹사이트 주소 앞의

```
https://
```

는

**Hypertext Transfer Protocol Secure**의 약자입니다.

브라우저 주소창의 🔒 자물쇠 아이콘은

해당 웹사이트가 SSL/TLS 인증서를 사용하여

안전하게 보호되고 있음을 의미합니다.

---

# AWS Certificate Manager (ACM)

AWS Certificate Manager(ACM)는

SSL/TLS 인증서를 중앙에서 관리하는 서비스입니다.

---

## 주요 기능

- SSL/TLS 인증서 발급(Provisioning)
- 인증서 관리
- 인증서 자동 갱신
- 인증서 배포(Deployment)

---

## 보호 대상

ACM은 다음과 같은 AWS 서비스의 통신을 보호할 수 있습니다.

- Elastic Load Balancer (ELB)
- Amazon CloudFront
- Amazon API Gateway
- 기타 AWS 서비스 및 내부 리소스

---

# 저장 시 암호화 vs 전송 중 암호화

|구분|저장 시 암호화 (At Rest)|전송 중 암호화 (In Transit)|
|---|---|---|
|보호 대상|저장된 데이터|이동 중인 데이터|
|사용 기술|AWS KMS, Server-Side Encryption|SSL/TLS|
|대표 서비스|S3, EBS, DynamoDB|HTTPS, ACM|

---

# 핵심 정리

- 데이터는 **저장 시(At Rest)** 와 **전송 중(In Transit)** 모두 암호화해야 한다.
- **암호화(Encryption)** 는 승인된 사용자만 데이터를 읽을 수 있도록 보호하는 기술이다.
- **Amazon S3**는 업로드되는 객체를 기본적으로 저장 시 암호화한다.
- **Amazon EBS**는 볼륨과 스냅샷을 암호화할 수 있다.
- **Amazon DynamoDB**는 서버 측 저장 시 암호화를 기본 제공한다.
- **AWS KMS**는 암호화 키를 생성·관리하며, 키 접근 권한을 IAM으로 제어할 수 있다.
- **Amazon Macie**는 저장된 민감한 데이터를 자동으로 탐지하고 보호한다.
- **SSL/TLS**는 전송 중 데이터를 암호화하는 프로토콜이며, TLS는 SSL의 개선된 버전이다.
- **HTTPS**는 SSL/TLS를 사용하는 안전한 웹 통신 방식이다.
- **AWS Certificate Manager(ACM)** 는 SSL/TLS 인증서를 발급·관리·배포하는 서비스이다.
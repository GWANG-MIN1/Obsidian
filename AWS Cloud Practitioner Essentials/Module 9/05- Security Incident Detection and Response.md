# 탐지 및 사고 대응 (Detection & Incident Response)

보안은 **예방(Prevention)** 만으로 충분하지 않습니다.

공격은 언제든 발생할 수 있으므로,

보안 이벤트를 **지속적으로 탐지(Detection)** 하고, 신속하게 **대응(Response)** 하는 것이 중요합니다.

AWS는 이를 위해 다양한 보안 서비스를 제공합니다.

---

# Amazon Inspector

Amazon Inspector는

AWS 리소스의 **자동화된 보안 평가(Security Assessment)** 를 수행하는 서비스입니다.

리소스의 취약점을 지속적으로 검사하여 보안 위험을 찾아냅니다.

---

## 주요 기능

- 자동화된 보안 평가 수행
- AWS 보안 모범 사례(Best Practices)와 비교
- Amazon EC2의 네트워크 노출 여부 탐지
- 취약한 소프트웨어 및 패키지 발견
- 알려진 보안 취약점(CVE) 탐지

---

## 평가 결과

평가가 완료되면

AWS 콘솔에서 보안 결과(Security Findings)를 제공합니다.

결과에는

- 심각도(Severity)
- 취약점 설명
- 영향 범위
- 권장 해결 방법(Remediation)

등이 함께 표시됩니다.

---

# Amazon GuardDuty

Amazon GuardDuty는

AWS 계정을 **지속적으로 모니터링** 하여 위협을 탐지하는 서비스입니다.

AI 및 머신러닝을 활용하여 이상 행동과 보안 위협을 자동으로 분석합니다.

---

## 주요 기능

- 지속적인 보안 모니터링
- AI/ML 기반 위협 탐지
- 악성 활동 탐지
- 계정 이상 행동 탐지
- 비정상적인 API 호출 탐지

---

## 예시

- 의심스러운 로그인
- 비정상적인 네트워크 트래픽
- 암호화폐 채굴 활동
- 계정 탈취 시도
- 악성 IP 접근

---

# Amazon Detective

Amazon Detective는

GuardDuty 등에서 탐지한 보안 이벤트를 **조사(Investigation)** 하는 서비스입니다.

단순히 위협을 발견하는 것이 아니라,

왜 발생했는지 원인을 분석하도록 도와줍니다.

---

## 주요 기능

- 자동화된 보안 조사
- 대화형 위협 시각화
- 리소스 간 관계 분석
- 생성형 AI 기반 인사이트 제공

---

## 활용 예시

```
GuardDuty

      ↓

의심스러운 활동 탐지

      ↓

Amazon Detective

      ↓

공격 원인 분석

      ↓

영향 범위 파악
```

이를 통해 보안 위협을 더 빠르고 정확하게 이해할 수 있습니다.

---

# Amazon Security Hub

AWS에는 다양한 보안 서비스가 존재합니다.

각 서비스의 결과를 개별적으로 확인하면 관리가 어렵습니다.

Amazon Security Hub는

여러 AWS 보안 서비스의 결과를 **한 곳에서 통합 관리**하는 서비스입니다.

---

## 주요 기능

- 하나의 통합 보안 대시보드 제공
- 보안 결과(Security Findings) 통합
- 자동화된 보안 모니터링
- 실행 가능한 인사이트 제공
- 보안 규정 준수(Compliance) 확인

---

## 연동 가능한 서비스

- Amazon Inspector
- Amazon GuardDuty
- Amazon Macie
- AWS IAM Access Analyzer
- AWS Firewall Manager
- 기타 AWS 보안 서비스

---

## Security Hub 활용 흐름

```
Amazon Inspector

Amazon GuardDuty

Amazon Macie

        ↓

Amazon Security Hub

        ↓

통합 보안 대시보드

        ↓

위협 분석 및 대응
```

---

# 탐지 및 사고 대응 서비스 비교

|서비스|주요 역할|
|---|---|
|Amazon Inspector|취약점 및 보안 설정 평가|
|Amazon GuardDuty|AI/ML 기반 위협 탐지 및 지속적인 모니터링|
|Amazon Detective|보안 사고 원인 조사 및 분석|
|Amazon Security Hub|보안 결과 통합 및 중앙 관리|

---

# 핵심 정리

- 보안은 **예방뿐 아니라 탐지와 사고 대응**도 중요하다.
- **Amazon Inspector**는 EC2와 AWS 리소스의 취약점을 자동으로 평가하고 보안 권장 사항을 제공한다.
- **Amazon GuardDuty**는 AI/ML 기반으로 지속적인 모니터링을 수행하여 보안 위협을 탐지한다.
- **Amazon Detective**는 탐지된 보안 이벤트를 조사하고 원인과 영향 범위를 분석한다.
- **Amazon Security Hub**는 여러 AWS 보안 서비스의 결과를 한곳에 통합하여 중앙에서 관리할 수 있도록 지원한다.
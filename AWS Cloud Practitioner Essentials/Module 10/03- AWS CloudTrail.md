# AWS CloudTrail

## AWS CloudTrail이란?

**AWS CloudTrail**은 AWS 계정에서 발생하는 **모든 API 호출과 사용자 활동을 기록(Audit Logging)** 하는 서비스이다.

CloudTrail을 사용하면 누가, 언제, 어디서, 어떤 작업을 수행했는지 중앙에서 추적하고 감사(Audit)할 수 있다.

---

# 감사(Audit)가 중요한 이유

IT 시스템에서는 **트랜잭션(Transaction)** 과 **변경 사항(Change)** 을 기록하는 것이 매우 중요하다.

온프레미스(물리적 데이터센터) 환경에서는

- 작업자가 변경 사항을 기록하지 않을 수도 있고
- 실수로 기록이 누락될 수도 있다.

반면 AWS에서는 모든 작업이 **API 호출(API Call)** 을 통해 수행되므로, 작업 내역을 자동으로 기록할 수 있다.

즉,

> **모든 AWS 작업은 API 호출이며, CloudTrail이 이를 자동으로 기록한다.**

---

# CloudTrail 이벤트 (CloudTrail Events)

**CloudTrail 이벤트(Event)** 는 AWS에서 발생하는 **개별 API 호출 또는 리소스 작업 하나**를 의미한다.

이벤트에는 다음과 같은 정보가 포함된다.

- 누가(User) 요청했는가
- 언제(Time) 요청했는가
- 어떤 작업(Action)을 수행했는가
- 어떤 AWS 리소스를 사용했는가
- 어느 IP 주소에서 요청했는가
- 요청 결과(Success / Failure)
- 요청이 거부되었는지 여부

**예시**

```
Event
----------------------------
사용자 : admin
서비스 : IAM
작업 : CreateUser
시간 : 2026-07-26 14:10
IP : 203.xxx.xxx.xxx
결과 : Success
```

> 하나의 API 호출 = 하나의 CloudTrail 이벤트

---

# CloudTrail 로그 (CloudTrail Logs)

CloudTrail은 여러 이벤트를 모아 **로그(Log)** 로 저장한다.

로그는 기본적으로 **Amazon S3** 버킷에 저장되며,

- 장기간 또는 무기한 보관 가능
- 중앙에서 관리 가능
- 감사(Audit) 및 규정 준수(Compliance)에 활용 가능
- 삭제 및 수정이 어렵도록 관리하여 **변조 방지(Tamper Resistance)** 지원

즉,

```
API 호출
      ↓
CloudTrail Event 생성
      ↓
여러 Event 저장
      ↓
CloudTrail Log
      ↓
Amazon S3 저장
```

---

# CloudTrail Insights

**CloudTrail Insights**는 평소와 다른 **비정상적인 API 활동(Anomalous API Activity)** 을 자동으로 분석하고 감지하는 기능이다.

다음과 같은 이상 징후를 탐지할 수 있다.

- 갑자기 API 호출 수가 급격히 증가
- 평소보다 오류(Failure)가 급격히 증가
- 비정상적인 사용자 활동 발생

**예시**

```
평소 IAM CreateUser
하루 5회
        ↓

오늘
200회 발생
        ↓

CloudTrail Insights
이상 활동 감지
```

이를 통해 보안 사고나 잘못된 설정을 빠르게 발견할 수 있다.

---

# CloudTrail의 활용

CloudTrail을 이용하면 다음과 같은 상황을 쉽게 확인할 수 있다.

- 누가 EC2 인스턴스를 삭제했는가?
- 누가 IAM 권한을 변경했는가?
- 어느 IP에서 로그인했는가?
- API 요청이 실패한 이유는 무엇인가?
- 특정 리소스가 언제 생성 또는 삭제되었는가?

이를 통해 보안 사고 조사 및 감사 업무를 효율적으로 수행할 수 있다.

---

# CloudTrail의 장점

- AWS API 호출을 중앙에서 기록 및 감사
- 모든 사용자 활동 추적 가능
- 사용자 권한(IAM) 변경 내역 확인
- 요청 시간, IP 주소, 응답 결과 등 상세 정보 제공
- Amazon S3에 안전하게 장기 보관
- 로그 변조 방지 지원
- CloudTrail Insights를 통해 이상 활동 자동 감지
- 보안 감사(Audit), 규정 준수 및 사고 분석에 활용

---

# CloudWatch vs CloudTrail

| CloudWatch | CloudTrail |
|------------|------------|
| 시스템 상태 모니터링 | 사용자 활동 및 API 호출 기록 |
| Metrics, Logs, Alarm 제공 | Audit Log 제공 |
| 성능 및 장애 감지 | 변경 이력 및 보안 감사 |
| "시스템이 정상인가?" 확인 | "누가 무엇을 했는가?" 확인 |

---

# 핵심 정리

- **CloudTrail** : AWS API 호출 및 사용자 활동을 기록하는 감사(Audit) 서비스
- **CloudTrail Event** : 개별 API 호출 또는 리소스 작업 하나를 의미
- **CloudTrail Log** : 여러 Event를 모아 Amazon S3에 저장한 감사 로그
- **CloudTrail Insights** : 비정상적인 API 활동을 자동으로 분석하고 탐지하는 기능
- CloudTrail은 보안 감사, 규정 준수, 변경 이력 추적 및 사고 분석에 활용된다.
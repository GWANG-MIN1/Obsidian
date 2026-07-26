# Amazon CloudWatch

## Amazon CloudWatch란?

**Amazon CloudWatch**는 AWS에서 제공하는 **모니터링 서비스**로, AWS 인프라와 애플리케이션의 상태를 실시간으로 모니터링할 수 있다.

주요 기능
- 리소스의 **지표(Metrics)** 수집 및 추적
- 로그(Logs) 수집 및 분석
- 경보(Alarm) 생성
- 대시보드(Dashboard)를 통한 시각화

---

# 지표(Metrics)

**지표(Metric)** 란 리소스의 상태를 나타내는 수치 데이터이다.

예시
- CPU 사용률(CPU Utilization)
- 메모리 사용량
- 네트워크 트래픽
- 디스크 읽기/쓰기 속도
- EC2 인스턴스 상태

CloudWatch는 이러한 지표를 지속적으로 수집하여 시스템 상태를 모니터링한다.

---

# Amazon CloudWatch Alarm

CloudWatch에서는 **경보(Alarm)** 를 설정할 수 있다.

동작 방식

```
지표(Metric)
      ↓
임계값(Threshold) 설정
      ↓
조건 충족 시 Alarm 발생
      ↓
자동 작업 또는 알림 수행
```

예시
- CPU 사용률이 **80% 이상**이면 Alarm 발생
- 네트워크 오류가 일정 횟수 이상 발생하면 Alarm 발생
- 디스크 사용량이 90% 이상이면 관리자에게 알림

---

# Amazon SNS와 연동

CloudWatch Alarm은 **Amazon SNS(Simple Notification Service)** 와 연동할 수 있다.

Alarm이 발생하면 SNS를 통해

- 문자(SMS)
- 이메일
- 애플리케이션 알림

등을 자동으로 전송할 수 있다.

**예시**

```
CPU 사용률 85%
        ↓
CloudWatch Alarm 발생
        ↓
Amazon SNS
        ↓
운영자에게 문자 및 이메일 전송
```

---

# CloudWatch Dashboard

CloudWatch Dashboard를 사용하면 여러 지표를 한 화면에서 시각적으로 확인할 수 있다.

특징
- 여러 AWS 서비스의 지표를 한 곳에서 확인
- 그래프 및 차트 제공
- 자동 새로고침(Auto Refresh) 지원
- 별도의 브라우저 새로고침 없이 최신 정보 확인 가능

---

# CloudWatch Logs

**CloudWatch Logs**는 애플리케이션과 AWS 서비스에서 생성되는 로그를 중앙에서 수집하고 관리하는 서비스이다.

주요 기능
- 로그 중앙 관리
- 로그 검색(Search)
- 로그 필터링(Filter)
- 로그 분석

예시
- 특정 오류 코드(HTTP 500) 검색
- Exception 로그만 필터링
- 특정 시간대의 오류 로그 분석

---

# CloudWatch의 장점

- 모든 지표(Metrics)를 한 곳에서 확인 가능
- 애플리케이션, 인프라 및 AWS 서비스의 가시성(Visibility) 확보
- 장애를 빠르게 감지하여 **평균 수리 시간(MTTR, Mean Time To Repair)** 단축
- 운영 효율 향상 및 **총 소유 비용(TCO, Total Cost of Ownership)** 절감
- 수집된 데이터를 분석하여 운영 및 리소스 최적화를 위한 인사이트 도출

---

# 활용 예시

EC2 인스턴스 풀(Pool)의 CPU 사용률을 지속적으로 분석하여

- 평균 사용률이 매우 낮다면 → 인스턴스 수를 줄여 비용 절감
- 평균 사용률이 매우 높다면 → Auto Scaling 정책을 조정하여 성능 향상

즉, CloudWatch는 단순히 모니터링하는 것을 넘어 **운영 효율과 비용을 최적화하기 위한 인사이트를 제공하는 서비스**이다.

---

# 핵심 정리

- **CloudWatch** : AWS 리소스와 애플리케이션을 모니터링하는 서비스
- **Metrics** : 리소스 상태를 나타내는 수치 데이터
- **CloudWatch Alarm** : 임계값을 설정하여 조건 충족 시 경보 발생
- **Amazon SNS** : Alarm 발생 시 문자, 이메일 등으로 알림 전송
- **Dashboard** : 지표를 실시간으로 시각화하여 확인
- **CloudWatch Logs** : 로그를 중앙에서 수집, 검색 및 분석
- **CloudWatch의 목표** : 시스템 가시성 확보, 장애 대응 시간 단축, 운영 효율 및 비용 최적화
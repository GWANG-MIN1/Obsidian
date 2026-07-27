# AWS 비용 최적화 (Cost Optimization Best Practices)

## 비용 최적화란?

AWS 비용 최적화(Cost Optimization)는 **필요한 성능과 가용성을 유지하면서 불필요한 비용을 줄이는 것**을 의미한다.

비용 절감만을 목표로 하는 것이 아니라,

> **비용 효율성(Cost Efficiency), 성능(Performance), 운영 효율성(Operational Excellence)** 사이의 최적의 균형을 찾는 것이 핵심이다.

---

# 1. Amazon EC2 적정 규모 조정 (Right Sizing)

EC2 인스턴스가 실제 워크로드보다 너무 크거나 작은 경우 비용이 낭비될 수 있다.

적정 규모 조정(Right Sizing)은 **워크로드 요구사항에 맞게 EC2 리소스를 분석하여 적절한 인스턴스 크기를 선택**하는 것이다.

### 장점

- 과도한 비용 절감
- 성능 유지
- 리소스 활용률 향상

---

# 2. AWS Compute Optimizer

## Compute Optimizer란?

**AWS Compute Optimizer**는 EC2, Auto Scaling 그룹, EBS 등의 사용 패턴을 분석하여 가장 적합한 리소스를 추천하는 서비스이다.

쉽게 말하면,

> **비용 최적화를 위한 '전문 컨설턴트'처럼 현재 리소스를 분석하고 적절한 크기를 추천해 주는 서비스**이다.

### 주요 기능

- EC2 적정 규모 추천
- EBS 볼륨 최적화
- Auto Scaling 그룹 최적화
- 과도한 프로비저닝(Over-Provisioning) 방지
- 부족한 프로비저닝(Under-Provisioning) 방지

---

# 3. Amazon EC2 Spot Instances

## Spot Instance란?

Spot Instance는 AWS의 **남는 EC2 용량**을 매우 저렴한 가격에 사용할 수 있는 인스턴스이다.

최대 **90%까지 할인**된 가격으로 사용할 수 있다.

### 적합한 워크로드

- 배치(Batch) 작업
- 데이터 분석
- 머신러닝 학습
- CI/CD
- 장애 발생 시 재시작이 가능한 애플리케이션

> ⚠️ AWS에서 용량이 필요하면 Spot Instance가 회수될 수 있으므로 중단에 유연한(Interruptible) 워크로드에 적합하다.

---

# 4. AWS Auto Scaling

## Auto Scaling이란?

Auto Scaling은 애플리케이션의 수요에 따라 EC2 인스턴스 수를 자동으로 늘리거나 줄이는 서비스이다.

### 장점

- 트래픽 증가 시 자동 확장(Scale Out)
- 트래픽 감소 시 자동 축소(Scale In)
- 불필요한 리소스 자동 제거
- 초과 지출 방지

---

# 5. Elastic Load Balancing (ELB)

ELB는 여러 EC2 인스턴스에 트래픽을 자동으로 분산하는 서비스이다.

### 장점

- 부하 분산
- 고가용성
- Auto Scaling과 함께 사용 가능
- 리소스 활용률 향상

---

# 6. 사용하지 않는 리소스 정리

사용하지 않는 리소스는 지속적으로 비용이 발생할 수 있으므로 정기적으로 정리해야 한다.

대표적인 예

- 사용하지 않는 EC2
- 미연결 EBS 볼륨
- 오래된 EBS 스냅샷
- 사용하지 않는 Elastic IP
- 불필요한 RDS 인스턴스

---

# 7. Amazon RDS 비용 최적화

## 읽기 전용(Read Replica) 활용

읽기 요청이 많은 애플리케이션에서는 **Read Replica(읽기 전용 복제본)** 를 사용하여 읽기 부하를 분산할 수 있다.

이를 통해

- 읽기 성능 향상
- 수평 확장(Scale Out)
- 비싼 DB 인스턴스로 업그레이드할 필요 감소

---

## RDS Storage Auto Scaling

스토리지 사용량이 증가하면 자동으로 스토리지를 확장하는 기능이다.

### 장점

- 스토리지 부족 방지
- 과도한 스토리지 할당 방지
- 비용 최적화

---

# 8. Amazon ElastiCache 활용

자주 조회되는 데이터를 메모리에 저장하여 데이터베이스 접근을 줄이는 서비스이다.

### 장점

- 응답 속도 향상
- RDS 부하 감소
- 비용 절감
- 읽기 성능 향상

---

# 9. Amazon S3 비용 최적화

## 적절한 스토리지 클래스 선택

데이터 접근 빈도에 따라 적절한 S3 Storage Class를 선택하면 비용을 절감할 수 있다.

예시

- S3 Standard
- S3 Standard-IA
- S3 One Zone-IA
- S3 Glacier Instant Retrieval
- S3 Glacier Flexible Retrieval
- S3 Glacier Deep Archive

---

## 데이터 압축

텍스트 기반 데이터는 압축하여 저장하면 저장 공간과 비용을 줄일 수 있다.

---

## Lifecycle 정책 활용

S3 Lifecycle 정책을 사용하면 객체를 자동으로

- 저렴한 스토리지 클래스로 이동
- 오래된 객체 삭제

할 수 있다.

예를 들어

> 30일만 보관하면 되는 백업 데이터를 삭제하지 않으면 수년간 불필요한 스토리지 비용이 발생할 수 있다.

---

# 10. 네트워크 비용 최적화

데이터 전송 비용도 AWS 비용의 중요한 요소이다.

### 비용 절감 방법

- 가용 영역(AZ) 간 불필요한 트래픽 최소화
- 인터넷 데이터 전송 최소화
- 동일 리전 내 통신 활용
- CloudFront 활용

---

## Amazon VPC Endpoint

VPC Endpoint는 인터넷을 거치지 않고 **VPC에서 AWS 서비스로 프라이빗하게 연결**하는 기능이다.

### 장점

- 인터넷 경유 트래픽 감소
- 보안 향상
- 데이터 전송 비용 절감 가능

---

# 비용 최적화 서비스 정리

| 서비스 | 비용 최적화 방법 |
|---------|----------------|
| **Compute Optimizer** | EC2, EBS 등의 적정 규모 추천 |
| **Spot Instances** | 최대 90% 할인된 EC2 사용 |
| **Auto Scaling** | 수요에 따라 EC2 자동 증감 |
| **Elastic Load Balancing** | 트래픽 분산으로 리소스 효율 향상 |
| **Amazon RDS Read Replica** | 읽기 부하 분산 및 수평 확장 |
| **RDS Storage Auto Scaling** | 스토리지 자동 확장으로 과소·과대 프로비저닝 방지 |
| **Amazon ElastiCache** | 자주 조회되는 데이터를 메모리에 캐싱 |
| **Amazon S3 Lifecycle** | 데이터 자동 이동 및 삭제 |
| **Amazon VPC Endpoint** | 프라이빗 연결을 통해 데이터 전송 비용 절감 |

---

# 핵심 정리

- 비용 최적화는 **비용, 성능, 운영 효율성의 균형을 맞추는 것**이 핵심이다.
- **Compute Optimizer**는 EC2와 EBS 등의 사용 패턴을 분석하여 적정 규모를 추천한다.
- **Spot Instances**는 남는 EC2 용량을 최대 **90% 할인**된 가격으로 제공하며, 중단 가능한 워크로드에 적합하다.
- **Auto Scaling**은 수요에 따라 EC2 인스턴스를 자동으로 늘리거나 줄여 비용을 절감한다.
- **Read Replica**와 **ElastiCache**는 읽기 성능을 향상시키고 데이터베이스 비용을 줄이는 데 효과적이다.
- **S3 Lifecycle 정책**과 적절한 스토리지 클래스 선택은 스토리지 비용을 크게 절감할 수 있다.
- **VPC Endpoint**를 활용하면 인터넷을 거치지 않고 AWS 서비스에 연결하여 보안과 비용 효율성을 높일 수 있다.
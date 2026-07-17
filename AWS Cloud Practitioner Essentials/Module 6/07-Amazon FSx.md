# Amazon FSx

> **다양한 파일 시스템을 AWS에서 완전관리형(Managed)으로 제공하는 서비스**

## 특징

- 완전관리형 파일 스토리지
- 다양한 파일 시스템 지원
- 높은 성능과 확장성
- 운영 및 유지보수 자동화

### 장점

- 파일 시스템 구축 및 관리 간소화
- 관리형 인프라 제공
- 확장 가능한 스토리지
- 비용 효율성
- 높은 가용성 및 안정성

---

# Amazon FSx 종류

## 1. Amazon FSx for Windows File Server

> **Windows Server 기반 공유 파일 시스템**

### 특징
- SMB(Server Message Block) 프로토콜 지원
- Windows 환경에 최적화
- Active Directory 연동 가능

### 사용 사례
- Windows 파일 서버
- 홈 디렉터리
- Microsoft 애플리케이션

---

## 2. Amazon FSx for NetApp ONTAP

> **NetApp ONTAP 파일 시스템을 AWS에서 완전관리형으로 제공**

### 특징
- ONTAP 기능 제공
- 스냅샷 및 복제 지원
- 멀티 프로토콜(NFS, SMB, iSCSI) 지원

### 사용 사례
- 엔터프라이즈 파일 시스템
- 데이터베이스
- VMware 환경

---

## 3. Amazon FSx for OpenZFS

> **OpenZFS 기반 완전관리형 파일 시스템**

### 특징
- 높은 성능
- 낮은 지연 시간
- 스냅샷 및 복제 기능 지원

### 사용 사례
- 개발 및 테스트
- 분석 워크로드
- 일반 Linux 파일 시스템

---

## 4. Amazon FSx for Lustre

> **고성능 Lustre 파일 시스템을 AWS에서 완전관리형으로 제공**

### 특징
- 매우 높은 처리량
- 낮은 지연 시간
- 병렬 파일 시스템 제공

### 사용 사례
- 머신러닝
- HPC(고성능 컴퓨팅)
- 빅데이터 분석
- 미디어 렌더링

# ⭐EFS와 FSx 차이 
| Amazon EFS                     | Amazon FSx                         |
| ------------------------------ | ---------------------------------- |
| AWS에서 제공하는 **공용 Linux 파일 시스템** | **특정 파일 시스템**을 선택해서 사용             |
| Linux(NFS) 전용                  | Windows, ONTAP, OpenZFS, Lustre 지원 |
| 여러 EC2가 공유                     | 목적에 맞는 전문 파일 시스템 제공                |
| 자동 확장                          | 파일 시스템마다 기능과 성능이 다름                |

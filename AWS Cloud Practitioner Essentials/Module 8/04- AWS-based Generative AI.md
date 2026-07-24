# 04. 딥러닝(Deep Learning)과 생성형 AI(Generative AI)

---

# 딥러닝(Deep Learning)

> **기계학습(Machine Learning)의 하위 분야**로, 인간의 뇌를 모방한 **인공 신경망(Artificial Neural Network, ANN)** 을 이용하여 데이터를 학습하는 기술

기존 ML보다 훨씬 복잡한 데이터와 패턴을 학습할 수 있으며, 이미지 인식, 음성 인식, 자연어 처리 등 다양한 AI 기술의 기반이 된다.

---

## 딥러닝의 동작 방식

딥러닝은 여러 개의 **인공 뉴런(Artificial Neuron)** 으로 구성된 신경망을 사용한다.

각 인공 뉴런은 **수학적 함수(Mathematical Function)** 를 수행하며, 여러 계층(Layer)으로 연결되어 있다.

```
입력 데이터
      ↓
입력층(Input Layer)
      ↓
은닉층(Hidden Layer)
      ↓
은닉층(Hidden Layer)
      ↓
출력층(Output Layer)
      ↓
결과
```

### 특징

- 각 계층(Layer)은 입력 정보를 요약하고 중요한 특징을 추출한다.
- 추출된 정보는 다음 계층으로 전달된다.
- 계층이 깊어질수록 더욱 복잡한 패턴을 학습할 수 있다.

---

## 딥러닝 활용 분야

- 이미지 인식
- 음성 인식
- 자연어 처리(NLP)
- 자율주행
- 추천 시스템
- 생성형 AI

---

# 파운데이션 모델 (Foundation Model, FM)

> 방대한 양의 데이터를 이용해 **사전 학습(Pre-trained)** 된 대규모 AI 모델

하나의 모델이 다양한 작업(Task)을 수행할 수 있도록 학습되어 있으며, 특정 작업에 맞게 추가 학습(Fine-tuning)하여 사용할 수 있다.

### 특징

- 매우 큰 규모의 데이터로 학습
- 다양한 작업 수행 가능
- 여러 AI 애플리케이션의 기반 모델

---

# 대규모 언어 모델 (Large Language Model, LLM)

> **텍스트 데이터를 중심으로 학습한 파운데이션 모델(Foundation Model)**

LLM은 문장을 이해하고 생성할 수 있으며, 다양한 자연어 처리 작업을 수행한다.

### 가능한 작업

- 질문 답변(Q&A)
- 문서 요약
- 번역
- 코드 생성
- 이메일 작성
- 대화(Chatbot)

> **LLM은 Foundation Model의 한 종류이다.**  
> 즉, 모든 LLM은 FM이지만, 모든 FM이 LLM인 것은 아니다.

---

# 사전 학습된 FM 활용

사전 학습된 Foundation Model은 다양한 작업에 맞게 조정(Customization)하여 사용할 수 있다.

예를 들어,

- 고객 상담 챗봇
- 문서 요약
- 코드 생성
- 상품 추천

등의 용도로 추가 학습하거나 프롬프트만 변경하여 활용할 수 있다.

---

# Amazon SageMaker JumpStart

> 사전 학습된 Foundation Model과 ML 솔루션을 빠르게 배포할 수 있는 기능

### 주요 기능

- 다양한 사전 학습된 FM 선택
- 몇 번의 클릭만으로 모델 배포
- ML 솔루션 빠른 시작
- 사용 사례에 맞게 모델 사용자 지정(Customization)

### 장점

- 모델을 처음부터 학습할 필요 없음
- 빠른 프로토타입 제작
- 개발 시간 단축

---

# Amazon Bedrock

> 다양한 Foundation Model(FM)을 사용할 수 있는 **완전 관리형 생성형 AI 서비스**

Amazon뿐 아니라 여러 AI 기업이 제공하는 FM을 하나의 API로 사용할 수 있다.

### 주요 특징

- 완전 관리형 서비스
- 인프라 관리 불필요
- 다양한 Foundation Model 제공
- 공통 API를 통해 모델 사용
- 기업 데이터를 이용한 맞춤형(Customization) 가능
- 데이터는 안전하게 보호되며 비공개로 활용

### 장점

- 다양한 FM을 쉽게 비교 및 선택
- 자체 인프라 구축 없이 생성형 AI 개발 가능
- 기업 데이터 기반 생성형 AI 구축 가능

---

# Amazon Q

> 기업 업무를 지원하는 **생성형 AI 기반 대화형 어시스턴트**

기업의 다양한 정보 저장소(Repository)와 연동하여 업무 생산성을 향상시킨다.

### 주요 기능

- 질문 답변
- 문서 검색
- 코드 생성
- 업무 자동화
- 조직 내부 정보 활용

---

## Amazon Q Business

기업 내부 데이터를 기반으로 질문에 답변하는 AI 어시스턴트

### 활용 예시

- 사내 문서 검색
- 업무 매뉴얼 검색
- 정책 및 규정 조회
- 사내 지식 기반(Q&A)

---

## Amazon Q Developer

개발자를 위한 생성형 AI 어시스턴트

### 활용 예시

- 코드 생성
- 코드 설명
- 코드 리팩토링
- 디버깅 지원
- AWS 개발 지원

---

# SageMaker JumpStart vs Bedrock vs Amazon Q

| 서비스 | 목적 | 주요 기능 |
|---------|------|-----------|
| **Amazon SageMaker JumpStart** | FM 및 ML 모델을 빠르게 시작 | 사전 학습된 모델 선택, 배포 및 사용자 지정 |
| **Amazon Bedrock** | 생성형 AI 애플리케이션 개발 | 다양한 Foundation Model을 공통 API로 제공 |
| **Amazon Q Business** | 기업 업무 지원 | 사내 데이터 기반 질의응답 및 문서 검색 |
| **Amazon Q Developer** | 개발자 지원 | 코드 생성, 디버깅, AWS 개발 지원 |

---

# 핵심 정리

- **딥러닝(Deep Learning)** 은 인간의 뇌를 모방한 인공 신경망을 이용하는 기계학습의 한 분야이다.
- **Foundation Model(FM)** 은 방대한 데이터로 사전 학습된 대규모 AI 모델이며, 다양한 작업에 활용할 수 있다.
- **LLM(Large Language Model)** 은 텍스트 데이터를 중심으로 학습한 Foundation Model이다.
- **Amazon SageMaker JumpStart** 는 사전 학습된 FM과 ML 솔루션을 빠르게 배포하고 사용자 지정할 수 있도록 지원한다.
- **Amazon Bedrock** 은 다양한 AI 기업의 Foundation Model을 하나의 API로 제공하는 완전 관리형 생성형 AI 서비스이다.
- **Amazon Q** 는 생성형 AI 기반 업무 지원 서비스로, **Amazon Q Business** 는 기업용 AI 어시스턴트, **Amazon Q Developer** 는 개발자를 위한 AI 코딩 어시스턴트이다.
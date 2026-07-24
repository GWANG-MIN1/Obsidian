# 05. AWS 생성형 AI 서비스

AWS는 생성형 AI 애플리케이션을 빠르게 개발할 수 있도록 다양한 서비스를 제공한다.

```
AWS 생성형 AI 서비스

├── Amazon SageMaker JumpStart
├── Amazon Bedrock
└── Amazon Q
      ├── Amazon Q Business
      └── Amazon Q Developer
```

---

# Amazon SageMaker JumpStart

> **Amazon SageMaker AI** 내에서 ML 모델의 구축(Build), 학습(Train), 배포(Deploy)를 빠르게 시작할 수 있도록 지원하는 머신러닝 허브

다양한 사전 구축된 ML 솔루션과 Foundation Model(FM)을 제공하여 처음부터 모델을 개발하지 않아도 된다.

### 주요 기능

- 사전 구축된 ML 솔루션 제공
- Foundation Model(FM) 제공
- 클릭 몇 번으로 모델 배포
- 모델 미세 조정(Fine-tuning)
- 빠른 프로토타이핑

### 지원 분야

- 컴퓨터 비전(Computer Vision)
- 자연어 처리(NLP)
- 테이블 형식 데이터(Tabular Data)
- 추천 시스템
- 예측 모델

### 사용 사례

- 신속한 ML 모델 배포
- 사용자 지정(Fine-tuned) 모델 개발
- ML 실험 및 프로토타입 제작

---

# Amazon Bedrock

> 대규모 **Foundation Model(FM)** 을 활용하여 생성형 AI 애플리케이션을 구축하기 위한 **완전 관리형 서비스**

AWS가 인프라를 관리하므로 개발자는 생성형 AI 애플리케이션 개발에만 집중할 수 있다.

### 주요 기능

- 다양한 Foundation Model 제공
- 단일 통합 API 제공
- 모델 실험 및 비교
- 기업 데이터 기반 모델 사용자 지정
- 생성형 AI 애플리케이션 구축

### 지원 모델 예시

- Amazon Foundation Models
- Claude (Anthropic)
- Stable Diffusion

### 장점

- 여러 AI 기업의 FM을 하나의 API로 사용
- 자체 데이터로 미세 조정(Fine-tuning)
- AWS 서비스와 손쉬운 통합
- 인프라 관리 불필요

### 사용 사례

- 엔터프라이즈 생성형 AI
- 텍스트·이미지 등 **멀티모달(Multimodal) 콘텐츠 생성**
- 고급 대화형 AI(Chatbot)

---

# Amazon Q

> 기업의 생산성과 업무 효율을 높이기 위한 **생성형 AI 어시스턴트**

기업의 데이터를 활용하여 질문에 답하거나 업무를 지원한다.

### 주요 기능

- 질문 답변(Q&A)
- 업무 자동화
- 문서 검색
- 코드 생성
- 업무 생산성 향상

---

# Amazon Q Business

> 회사의 내부 데이터와 문서를 기반으로 답변을 제공하는 기업용 생성형 AI 어시스턴트

기업의 정보 저장소(Repository)와 연결하여 필요한 정보를 검색하고 업무를 지원한다.

### 주요 기능

- 기업 문서 검색
- 질문 답변
- 문제 해결 지원
- 업무 수행 지원
- 다양한 기업 시스템과 보안 연결

### 사용 사례

- 정보 검색
- 자동화된 워크플로
- 데이터 기반 인사이트 추출

---

# Amazon Q Developer

> 개발자의 코딩 생산성을 높여주는 생성형 AI 코딩 어시스턴트

다양한 프로그래밍 언어와 IDE를 지원하여 코드 작성부터 검토까지 도와준다.

### 지원 언어

- C#
- Java
- JavaScript
- Python
- TypeScript

### 주요 기능

- 코드 자동 완성
- 함수 생성
- 코드 블록 생성
- 코드 설명
- 코드 리팩토링
- 코드 검토 지원

또한 여러 IDE와 통합되어 개발자가 더 빠르고 효율적으로 코드를 작성할 수 있도록 지원한다.

### 사용 사례

- 코드 생성 속도 향상
- 코드 품질 향상
- 보안 취약점 개선
- 코드 검토 자동화

---

# 서비스 비교

| 서비스 | 목적 | 주요 기능 | 대표 사용 사례 |
|---------|------|-----------|----------------|
| **Amazon SageMaker JumpStart** | ML 모델 개발 시작 | 사전 구축된 ML 솔루션 및 FM 제공 | ML 모델 배포, Fine-tuning, 프로토타입 |
| **Amazon Bedrock** | 생성형 AI 개발 | 다양한 Foundation Model을 단일 API로 제공 | 생성형 AI, 멀티모달 콘텐츠 생성, 챗봇 |
| **Amazon Q Business** | 기업 업무 지원 | 사내 데이터 기반 질의응답 및 문서 검색 | 정보 검색, 워크플로 자동화, 인사이트 추출 |
| **Amazon Q Developer** | 개발 생산성 향상 | 코드 생성, 자동 완성, 코드 리뷰 | 코드 작성, 리팩토링, 보안 개선 |

---

# 핵심 정리

- **Amazon SageMaker JumpStart**는 SageMaker AI 내에서 사전 구축된 ML 솔루션과 Foundation Model을 활용하여 ML 모델을 빠르게 구축·학습·배포할 수 있는 머신러닝 허브이다.
- **Amazon Bedrock**은 Amazon과 Anthropic(Claude), Stability AI(Stable Diffusion) 등의 Foundation Model을 단일 API로 제공하는 완전 관리형 생성형 AI 서비스이다.
- **Amazon Q**는 기업의 업무 생산성을 향상시키는 생성형 AI 어시스턴트이다.
- **Amazon Q Business**는 기업 내부 데이터를 활용하여 질문에 답하고 업무를 지원하는 서비스이다.
- **Amazon Q Developer**는 다양한 프로그래밍 언어와 IDE를 지원하여 코드 생성, 코드 리뷰, 리팩토링 등을 수행하는 개발자용 AI 어시스턴트이다.
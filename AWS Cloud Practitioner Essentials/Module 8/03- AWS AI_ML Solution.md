# 03. AWS AI/ML 서비스 계층(Tiers)

AWS는 AI와 ML을 활용하는 수준에 따라 크게 **3개의 계층(Tier)** 으로 서비스를 제공한다.

```
Tier 1 : 사전 구축된 AI 서비스
        ↓
Tier 2 : ML 서비스
        ↓
Tier 3 : ML 프레임워크 및 인프라
```

---

# Tier 1. 사전 구축된 AWS AI 서비스 (Pre-trained AI Services)

> 이미 학습된 AI 모델을 API 형태로 제공하는 완전 관리형 서비스

- 모델을 직접 학습할 필요 없음
- API 호출만으로 AI 기능 사용 가능
- AI 개발 경험이 많지 않아도 쉽게 활용 가능

---

# 1. 언어 서비스 (Language Services)

텍스트 또는 음성을 해석하여 의미 있는 정보를 추출하거나 변환하는 서비스

## Amazon Comprehend

> 자연어 처리(NLP)를 사용하여 문서에서 핵심 정보를 추출하는 서비스

### 주요 기능

- 핵심 문구(Key Phrases) 추출
- 언어(Language) 감지
- 감정 분석(Sentiment Analysis)
- 개체(Entity) 인식
- 문서 분석

### 사용 사례

- 콘텐츠 분류
- 고객 감정 분석
- 규정 준수(Compliance) 모니터링

---

## Amazon Polly

> 텍스트(Text)를 자연스러운 음성(Speech)으로 변환하는 서비스 (Text-to-Speech)

### 특징

- 다양한 언어 지원
- 남성/여성 음성 지원
- 다양한 억양 지원

### 사용 사례

- 가상 어시스턴트
- e-러닝 애플리케이션
- 시각 장애인을 위한 접근성 개선

---

## Amazon Transcribe

> 음성을 텍스트로 변환하는 서비스 (Speech-to-Text)

### 특징

- 다양한 언어 지원
- 화자 식별(Speaker Identification)
- 사용자 지정 어휘(Custom Vocabulary)
- 실시간 음성 변환(Real-time Transcription)

### 사용 사례

- 고객 통화 기록
- 자동 자막 생성
- 미디어 콘텐츠 메타데이터 생성

---

## Amazon Translate

> 텍스트를 다양한 언어로 번역하는 서비스

### 특징

- 실시간 번역
- 배치(Batch) 번역
- 다국어 지원

### 사용 사례

- 문서 번역
- 다국어 웹사이트
- 글로벌 애플리케이션

---

# 2. 컴퓨터 비전 및 검색 서비스

문서, 이미지, 동영상 등의 콘텐츠를 분석하거나 필요한 정보를 검색하는 서비스

---

## Amazon Kendra

> 자연어 처리(NLP)를 이용한 지능형 엔터프라이즈 검색 서비스

### 특징

- 문서의 의미(Context)를 이해
- 단순 키워드 검색보다 정확한 답변 제공
- 기업 내부 문서 검색 최적화

### 사용 사례

- 기업 문서 검색
- 챗봇
- 애플리케이션 검색 기능

---

## Amazon Rekognition

> 이미지 및 동영상 분석 서비스

### 주요 기능

- 객체(Object) 인식
- 사람(Face) 인식
- 텍스트 인식
- 장면(Scene) 분석
- 활동(Activity) 분석

※ Amazon S3에 저장된 이미지와 동영상을 분석할 수 있다.

### 사용 사례

- 콘텐츠 검열(Content Moderation)
- 신원 확인(Identity Verification)
- 미디어 분석
- 스마트 홈 자동화

---

## Amazon Textract

> 문서에서 텍스트와 데이터를 자동으로 추출하는 서비스

### 주요 기능

- 인쇄된 텍스트 추출
- 손글씨 추출
- 양식(Form) 분석
- 표(Table) 데이터 추출

### 사용 사례

- 금융 문서 처리
- 의료 문서 처리
- 정부 기관 문서 처리
- OCR 기반 자동화

---

# 3. 대화형 AI 및 개인화 서비스

사용자와 문자 또는 음성으로 상호작용하거나 개인 맞춤형 추천을 제공하는 서비스

---

## Amazon Lex

> 애플리케이션에 음성 및 텍스트 기반 대화형 인터페이스를 추가하는 서비스

### 사용 기술

- **NLU (Natural Language Understanding)** : 자연어 이해
- **ASR (Automatic Speech Recognition)** : 자동 음성 인식

### 사용 사례

- 가상 어시스턴트
- FAQ 챗봇
- 고객 상담 봇
- 음성 기반 애플리케이션

---

## Amazon Personalize

> 고객 데이터를 기반으로 개인 맞춤형 추천을 제공하는 서비스

### 특징

- 과거 행동 데이터 학습
- 실시간 추천
- 개인화된 사용자 경험 제공

### 사용 사례

- 상품 추천
- 영화 및 음악 추천
- 스트리밍 콘텐츠 추천
- 인기 상승 중인 콘텐츠 추천

---

# Tier 2. ML 서비스 (Machine Learning Services)

> 인프라를 직접 관리하지 않으면서도 자체 ML 모델을 구축·학습·배포할 수 있는 서비스

Tier 1보다 자유도가 높으며, 모델을 직접 개발할 수 있다.

---

## Amazon SageMaker AI

완전 관리형 머신러닝 플랫폼

### 주요 기능

- ML 모델 구축(Build)
- 모델 학습(Train)
- 모델 배포(Deploy)
- 데이터 준비
- 데이터 시각화
- 하이퍼파라미터 튜닝
- 디버깅(Debugging)
- 모니터링(Monitoring)

### 장점

- 원하는 ML 도구 사용 가능
- 완전 관리형 인프라 제공
- 반복 가능한 ML 워크플로 구축
- 개발부터 운영까지 하나의 환경에서 수행

---

# Tier 3. ML 프레임워크 및 인프라

> 가장 자유도가 높은 계층으로, ML 환경을 직접 구성하여 모델을 개발하는 방식

---

## ML 프레임워크 (Machine Learning Framework)

ML 모델을 구축하기 위한 **사전 구축된 라이브러리 및 개발 도구**

### AWS에서 지원하는 프레임워크

- PyTorch
- TensorFlow
- Apache MXNet

### 특징

- 자유로운 모델 설계
- 원하는 알고리즘 구현 가능
- 고급 ML 개발에 적합

---

## AWS ML 인프라

사용자 지정(Custom) ML 환경을 구축할 수 있도록 고성능 인프라를 제공한다.

### 대표 서비스

- Amazon EC2 (ML 최적화 인스턴스)
- Amazon EMR
- Amazon ECS (Elastic Container Service)

### 특징

- GPU 기반 고성능 연산 지원
- 대규모 ML 학습 가능
- 높은 유연성
- 고급 ML 워크로드에 적합

---

# AWS AI/ML 서비스 계층 비교

| 구분     | Tier 1 (AI 서비스)                     | Tier 2 (ML 서비스) | Tier 3 (ML 프레임워크 및 인프라)             |
| ------ | ----------------------------------- | --------------- | ----------------------------------- |
| 모델 학습  | ❌ 불필요                               | ✅ 가능            | ✅ 직접 수행                             |
| 사용 난이도 | 쉬움                                  | 보통              | 어려움                                 |
| 자유도    | 낮음                                  | 높음              | 매우 높음                               |
| 인프라 관리 | AWS                                 | AWS             | 사용자가 직접 구성                          |
| 대표 서비스 | Comprehend, Polly, Lex, Rekognition | SageMaker AI    | EC2, EMR, ECS + PyTorch, TensorFlow |

---

# 핵심 정리

- **Tier 1**: 이미 학습된 AI 모델을 API 형태로 제공하는 서비스로, 별도의 모델 학습 없이 AI 기능을 사용할 수 있다.
- **Tier 2**: Amazon SageMaker AI를 이용해 자체 ML 모델을 구축, 학습, 배포할 수 있는 완전 관리형 서비스이다.
- **Tier 3**: PyTorch, TensorFlow 등의 ML 프레임워크와 EC2, EMR, ECS 같은 인프라를 활용하여 완전히 사용자 정의된 ML 환경을 구축하는 방식이다.
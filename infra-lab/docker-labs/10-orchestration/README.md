# 10 컨테이너 오케스트레이션

단일 호스트에서 컨테이너 몇 개를 다루는 것을 넘어, **여러 호스트에 걸쳐 수십~수천 개의 컨테이너를 배포·확장·복구·연결**하는 일을 자동화하는 것이 오케스트레이션이다.  
Docker 학습의 종착점이자 Kubernetes로 넘어가는 다리.

---

## 왜 오케스트레이션이 필요한가

`docker run`이나 Compose는 **한 대의 호스트** 안에서만 동작한다. 실제 서비스 운영에는 다음이 필요하다.

| 요구사항 | 오케스트레이터가 하는 일 |
|----------|--------------------------|
| **스케줄링** | 어떤 호스트에 컨테이너를 배치할지 자동 결정 |
| **스케일링** | 부하에 따라 인스턴스 수 자동/수동 조절 |
| **셀프 힐링** | 죽은 컨테이너·노드를 감지해 자동 재생성 |
| **서비스 디스커버리** | 컨테이너 IP가 바뀌어도 이름으로 찾아 통신 |
| **로드 밸런싱** | 여러 인스턴스에 트래픽 분산 |
| **롤링 업데이트** | 무중단으로 새 버전 순차 교체, 실패 시 롤백 |
| **선언형 관리** | "원하는 상태"를 선언하면 시스템이 그 상태를 유지 |

> 핵심 개념: **선언형(Desired State)**. "컨테이너 3개를 항상 유지"라고 선언하면, 하나가 죽어도 오케스트레이터가 다시 3개로 맞춘다.

---

## Docker Swarm

Docker에 내장된 오케스트레이션 도구. 별도 설치 없이 `docker swarm`으로 시작할 수 있어 개념 학습에 적합하다.

### 구조

```
┌──────────────────────────────────────────┐
│              Swarm Cluster                │
│                                           │
│   Manager Node          Worker Nodes      │
│  ┌───────────┐      ┌──────┐ ┌──────┐    │
│  │ 스케줄링   │─────▶│ Task │ │ Task │    │
│  │ 상태 관리  │      └──────┘ └──────┘    │
│  │ (raft 합의)│      ┌──────┐ ┌──────┐    │
│  └───────────┘─────▶│ Task │ │ Task │    │
│                     └──────┘ └──────┘    │
└──────────────────────────────────────────┘
```

| 개념 | 설명 |
|------|------|
| **Node** | Swarm에 참여한 도커 호스트(Manager 또는 Worker) |
| **Manager** | 클러스터 상태 관리·스케줄링(raft로 합의), 홀수 개 권장(3, 5) |
| **Worker** | 실제 컨테이너(Task)를 실행 |
| **Service** | 실행하려는 애플리케이션의 선언(이미지·복제 수) |
| **Task** | 서비스의 단일 실행 단위 = 컨테이너 1개 |
| **Stack** | 여러 서비스를 묶은 배포 단위(Compose 파일로 정의) |

### 클러스터 구성

```bash
docker swarm init --advertise-addr <매니저IP>   # 매니저 초기화
docker swarm join-token worker                   # 워커 조인 토큰 확인
docker swarm join --token <토큰> <매니저IP>:2377  # 워커 참여
docker node ls                                   # 노드 목록
```

### 서비스 배포 & 스케일링

```bash
docker service create --name web --replicas 3 -p 80:80 nginx
docker service ls                        # 서비스 목록
docker service ps web                    # 태스크 배치 상태
docker service scale web=5               # 5개로 확장
docker service update --image nginx:1.27 web   # 롤링 업데이트
docker service rm web                    # 서비스 제거
```

### Stack 배포 (Compose 파일 재활용)

```bash
docker stack deploy -c compose.yaml myapp   # 스택 배포
docker stack services myapp                 # 스택의 서비스 확인
docker stack rm myapp                       # 스택 제거
```

---

## Swarm vs Kubernetes

| 항목 | Docker Swarm | Kubernetes |
|------|--------------|------------|
| 학습 난이도 | 낮음(도커 명령 그대로) | 높음 |
| 설치 | 내장, 즉시 사용 | 별도 구성(또는 관리형) |
| 스케일·기능 | 중소 규모에 충분 | 대규모·복잡한 요구 대응 |
| 생태계 | 제한적 | 방대(Helm, Operator, CRD 등) |
| 현업 채택 | 감소 추세 | 사실상 표준 |

> Swarm은 **오케스트레이션 개념을 가장 쉽게 익히는 도구**다. 개념(선언형·서비스·복제·셀프힐링)을 Swarm으로 이해하면 Kubernetes로 자연스럽게 넘어간다.

---

## Kubernetes로 가는 다리

Swarm에서 익힌 개념은 Kubernetes에서 이렇게 대응된다.

| Swarm | Kubernetes | 역할 |
|-------|------------|------|
| Service | Deployment | 원하는 복제 수 유지 |
| Task | Pod | 실행 단위(Pod는 컨테이너 1+개) |
| Manager | Control Plane | 클러스터 상태 관리 |
| Worker | Node | 워크로드 실행 |
| Stack | Namespace + 매니페스트 | 애플리케이션 묶음 |
| 내장 라우팅 메시 | Service / Ingress | 서비스 디스커버리·LB |
| `docker service scale` | `kubectl scale` | 스케일링 |

### 다음 학습 방향

1. **로컬 K8s** — `minikube`, `kind`, Docker Desktop 내장 Kubernetes로 실습
2. **핵심 오브젝트** — Pod, Deployment, Service, ConfigMap, Secret, Ingress
3. **패키징** — Helm 차트
4. **GitOps** — ArgoCD로 선언형 배포 자동화
5. **관리형 클러스터** — AWS EKS, GCP GKE 등

> 컨테이너(Docker) → 오케스트레이션(Swarm 개념) → Kubernetes → GitOps/관리형 K8s(EKS) 순으로 확장하면 DevOps 스택 전체가 연결된다.

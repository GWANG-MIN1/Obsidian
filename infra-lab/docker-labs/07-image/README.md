# 07 도커 이미지

도커 이미지의 생성, 구조, 추출, 배포 전반을 다룬다.  
컨테이너는 이미지로부터 실행되므로, 이미지를 직접 만들고 관리하는 능력은 Docker 운영의 핵심이다.

---

## 도커 이미지 생성

도커 이미지는 **Dockerfile**을 작성한 뒤 `docker build` 명령으로 생성한다.  
Dockerfile의 각 명령어는 이미지 레이어로 쌓이며, 빌드 캐시를 활용해 재빌드 속도를 높일 수 있다.

### Dockerfile 기본 구조

```dockerfile
FROM ubuntu:22.04

LABEL maintainer="your@email.com"

ENV APP_HOME=/app

WORKDIR $APP_HOME

COPY . .

RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8080

CMD ["node", "app.js"]
```

### 주요 Dockerfile 명령어

| 명령어 | 설명 |
|--------|------|
| `FROM` | 베이스 이미지 지정, 모든 Dockerfile의 시작점 |
| `RUN` | 빌드 시 실행할 쉘 명령 (레이어 생성) |
| `COPY` | 호스트 파일을 이미지 안으로 복사 |
| `ADD` | COPY + URL 다운로드 + tar 자동 압축 해제 (주로 COPY 권장) |
| `ENV` | 환경변수 설정 (빌드 + 런타임 모두 적용) |
| `ARG` | 빌드 시에만 사용하는 변수 (`--build-arg`로 주입) |
| `WORKDIR` | 이후 명령의 작업 디렉터리 지정 |
| `EXPOSE` | 컨테이너가 수신할 포트 문서화 |
| `CMD` | 컨테이너 기본 실행 명령 (docker run 시 덮어쓸 수 있음) |
| `ENTRYPOINT` | 컨테이너 진입점 고정 (CMD는 인자로 동작) |
| `USER` | 이후 명령을 실행할 사용자 지정 |

### docker build

```bash
docker build -t 이미지명:태그 .
docker build -f /경로/Dockerfile -t 이미지명:태그 .
docker build --build-arg VERSION=1.2 -t 이미지명:태그 .
docker build --no-cache -t 이미지명:태그 .
```

### CMD vs ENTRYPOINT

```dockerfile
# CMD만 사용: docker run 이미지명 bash 로 덮어쓰기 가능
CMD ["nginx", "-g", "daemon off;"]

# ENTRYPOINT 고정: 항상 node가 실행됨, CMD는 기본 인자
ENTRYPOINT ["node"]
CMD ["app.js"]
# → docker run 이미지명           → node app.js
# → docker run 이미지명 server.js → node server.js
```

### 멀티 스테이지 빌드

빌드 환경과 실행 환경을 분리해 최종 이미지 크기를 줄인다.

```dockerfile
# 1단계: 빌드
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 2단계: 실행 (빌드 결과물만 복사)
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 빌드 최적화 팁

1. **자주 변경되는 레이어를 뒤로** — `COPY package.json` → `RUN npm install` → `COPY . .` 순서로 소스 변경 시 npm install 캐시 재활용
2. **RUN 명령 합치기** — `&&`로 연결해 레이어 수 줄이기
3. **.dockerignore 활용** — `node_modules`, `.git`, 빌드 산출물 제외
4. **경량 베이스 이미지 사용** — `alpine`, `slim`, `distroless` 등

---

## 이미지 구조 이해

도커 이미지는 **읽기 전용 레이어의 스택**으로 구성된다.  
컨테이너를 실행하면 이 스택 위에 쓰기 가능한 레이어(Container Layer)가 하나 추가된다.

### 레이어 구조

```
┌─────────────────────────────┐
│  Container Layer (읽기/쓰기)  │  ← 컨테이너 실행 시 추가, 삭제 시 사라짐
├─────────────────────────────┤
│  Layer 4: CMD               │  ← 읽기 전용 이미지 레이어
│  Layer 3: COPY . .          │
│  Layer 2: RUN npm install   │
│  Layer 1: FROM node:20      │
└─────────────────────────────┘
```

- Dockerfile 명령어 1개 = 레이어 1개
- 각 레이어는 이전 레이어와의 **차이(diff)** 만 저장
- 같은 레이어를 여러 이미지가 **공유** → 디스크 절약

### Union Filesystem

여러 읽기 전용 레이어와 쓰기 레이어를 합쳐 하나의 파일 시스템처럼 보여주는 기술.  
Docker는 기본적으로 **overlay2** 드라이버를 사용한다.

**Copy-on-Write (CoW)**: 컨테이너가 기존 파일을 수정하면 해당 파일을 upperdir로 복사한 뒤 수정한다. 원본 레이어는 변경되지 않는다.

### 이미지 정보 확인

```bash
docker images
docker images -a                          # 중간 레이어 포함 전체
docker inspect 이미지명:태그
docker history 이미지명:태그
docker history --no-trunc 이미지명:태그   # 명령어 전체 출력
docker system df
docker system df -v                       # 이미지별 상세
```

### 댕글링 이미지

태그 없는 `<none>:<none>` 이미지. 같은 태그로 재빌드 시 이전 이미지가 태그를 잃으면 발생한다.

```bash
docker images -f dangling=true   # 댕글링 이미지 확인
docker image prune               # 댕글링 이미지 삭제
docker image prune -a            # 미사용 이미지 전체 삭제 (주의)
```

---

## 이미지 추출

도커 이미지나 컨테이너 파일 시스템을 파일로 저장하거나 다른 호스트로 옮길 때 사용한다.  
인터넷이 없는 환경(폐쇄망)에 이미지를 전달하거나 백업할 때 유용하다.

### docker save / load — 이미지 저장 & 로드

이미지의 **모든 레이어와 메타데이터**를 tar 파일로 저장한다.

```bash
docker save -o myimage.tar 이미지명:태그
docker save -o images.tar 이미지1:태그 이미지2:태그   # 여러 이미지 한 번에
docker save 이미지명:태그 | gzip > myimage.tar.gz     # gzip 압축

docker load -i myimage.tar
docker load < myimage.tar.gz
```

### docker export / import — 컨테이너 파일 시스템 저장

실행 중이거나 중지된 컨테이너의 파일 시스템을 tar로 저장한다.  
레이어 정보 없이 단순 파일 시스템 스냅샷이므로 `save`보다 용량이 작다.

```bash
docker export 컨테이너명 -o container.tar
docker import container.tar 이미지명:태그
```

### save vs export 비교

| 항목 | `docker save` | `docker export` |
|------|--------------|-----------------|
| 대상 | 이미지 | 컨테이너 |
| 레이어 보존 | O (전체 레이어 구조 유지) | X (단일 레이어로 평탄화) |
| 메타데이터 | O (히스토리, 환경변수 등 포함) | X (히스토리 삭제됨) |
| 복원 명령 | `docker load` | `docker import` |
| 주요 용도 | 이미지 백업, 폐쇄망 전달 | 컨테이너 스냅샷, 이미지 경량화 |

### docker commit

실행 중인 컨테이너의 현재 상태를 이미지로 만든다.

```bash
docker commit 컨테이너명 이미지명:태그
docker commit -m "nginx 설정 수정" -a "이름" 컨테이너명 이미지명:태그
```

> 재현 불가능하고 레이어가 불투명해지므로, 프로덕션에서는 Dockerfile 빌드를 권장한다.

---

## 이미지 배포

빌드한 이미지를 레지스트리에 푸시하여 다른 환경이나 팀원이 사용할 수 있게 한다.

### 이미지 태그

레지스트리에 푸시하려면 이미지명이 `레지스트리주소/유저명/이미지명:태그` 형식이어야 한다.

```bash
docker tag 원본이미지명:태그 새이미지명:태그

# Docker Hub 용
docker tag myapp:1.0 username/myapp:1.0
docker tag myapp:1.0 username/myapp:latest

# 프라이빗 레지스트리 용
docker tag myapp:1.0 registry.example.com/team/myapp:1.0
```

### Docker Hub push / pull

```bash
docker login
docker push username/이미지명:태그
docker pull username/이미지명:태그
docker logout
```

### 프라이빗 레지스트리

```bash
# 로컬 레지스트리 컨테이너 실행
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
docker pull localhost:5000/myapp:1.0

# 저장된 이미지 목록 확인 (API)
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/myapp/tags/list
```

### 태그 전략

| 전략 | 예시 | 설명 |
|------|------|------|
| SemVer | `myapp:1.2.3` | 명시적 버전 관리, 불변 |
| latest | `myapp:latest` | 항상 최신 이미지, CI/CD에서 자동 갱신 |
| Git SHA | `myapp:a1b2c3d` | 커밋 단위 추적, 디버깅에 유리 |
| 환경 | `myapp:prod` | 배포 환경 구분 |

> `latest`는 재현성이 없으므로 프로덕션에서는 SemVer 또는 Git SHA 태그를 권장한다.

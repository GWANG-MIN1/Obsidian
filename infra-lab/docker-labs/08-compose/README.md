# 08 Docker Compose

여러 컨테이너로 이루어진 애플리케이션을 하나의 YAML 파일로 정의하고 한 번에 실행·관리하는 도구.  
`docker run` 옵션을 일일이 입력하는 대신 선언형(declarative) 파일로 서비스·네트워크·볼륨을 함께 관리한다.

---

## 왜 Compose를 쓰는가

`docker run`으로 웹 + DB + 캐시를 각각 띄우려면 옵션이 길어지고 실행 순서·네트워크 연결을 수동으로 챙겨야 한다.  
Compose는 이 구성을 `compose.yaml` 한 파일에 기록하고 `docker compose up` 한 줄로 전체 스택을 재현한다.

- **선언형 구성** — 원하는 최종 상태를 파일에 기록, 명령형 스크립트 불필요
- **자동 네트워크 생성** — 같은 파일의 서비스끼리 서비스명으로 통신 가능
- **버전 관리** — YAML을 Git에 커밋해 인프라를 코드로 관리(IaC)
- **환경 재현** — 팀원 누구나 동일 환경을 한 번에 구성

> 최신 Docker에서는 `docker-compose`(하이픈, v1) 대신 `docker compose`(플러그인, v2)를 사용한다.

---

## compose.yaml 기본 구조

```yaml
services:
  web:
    build: ./web              # Dockerfile로 빌드
    ports:
      - "8080:80"             # 호스트:컨테이너
    environment:
      - APP_ENV=production
    depends_on:
      - db
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:16        # 레지스트리 이미지 사용
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  db-data:

networks:
  backend:
```

- **services** — 실행할 컨테이너 정의(핵심)
- **volumes** — 명명 볼륨 선언(데이터 영속화)
- **networks** — 사용자 정의 네트워크 선언

> `version:` 키는 Compose v2에서 더 이상 필요 없다(무시됨).

---

## 주요 서비스 옵션

| 옵션 | 설명 | `docker run` 대응 |
|------|------|-------------------|
| `image` | 사용할 이미지 | 이미지 인자 |
| `build` | Dockerfile 경로로 빌드 | `docker build` |
| `ports` | 포트 매핑 | `-p` |
| `environment` | 환경변수 | `-e` |
| `env_file` | .env 파일에서 환경변수 로드 | `--env-file` |
| `volumes` | 볼륨/바인드 마운트 | `-v` |
| `networks` | 연결할 네트워크 | `--network` |
| `depends_on` | 시작 순서 의존성 | (없음) |
| `restart` | 재시작 정책 | `--restart` |
| `command` | 기본 실행 명령 덮어쓰기 | CMD 덮어쓰기 |
| `deploy.resources` | CPU/메모리 제한 | `--cpus`, `--memory` |

---

## depends_on과 시작 순서

`depends_on`은 **시작 순서**만 보장하고, 의존 서비스가 실제로 "준비 완료"됐는지는 보장하지 않는다.  
DB 컨테이너가 시작됐어도 내부 DB 프로세스가 아직 연결을 못 받을 수 있다.  
→ **healthcheck**와 조합해 진짜 준비 상태를 기다린다.

```yaml
services:
  web:
    build: ./web
    depends_on:
      db:
        condition: service_healthy   # db가 healthy가 될 때까지 대기

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

---

## Healthcheck (컨테이너 상태 점검)

컨테이너 내부 애플리케이션이 정상 동작 중인지 주기적으로 검사한다.  
`docker ps`의 STATUS에 `(healthy)` / `(unhealthy)`로 표시된다.

```dockerfile
# Dockerfile에서 정의
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

```yaml
# compose에서 정의
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/"]
  interval: 30s      # 검사 주기
  timeout: 3s        # 응답 대기 시간
  retries: 3         # 실패 허용 횟수
  start_period: 40s  # 시작 후 유예 시간(실패로 안 침)
```

| 상태 | 의미 |
|------|------|
| `starting` | start_period 동안 초기화 중 |
| `healthy` | 검사 통과 |
| `unhealthy` | retries 연속 실패 |

---

## 재시작 정책 (restart)

컨테이너가 종료됐을 때 자동 재시작 여부를 결정한다. 운영 환경 안정성의 핵심.

| 정책 | 동작 |
|------|------|
| `no` | 재시작 안 함(기본값) |
| `on-failure` | 비정상 종료(exit code ≠ 0) 시에만 재시작 |
| `always` | 항상 재시작, 도커 데몬 재시작 시에도 복구 |
| `unless-stopped` | `always`와 같으나 사용자가 수동 중지한 경우 제외 |

```bash
docker run -d --restart unless-stopped nginx
docker run -d --restart on-failure:5 myapp   # 최대 5회 재시도
```

---

## Compose 주요 명령어

```bash
docker compose up                 # 포그라운드 실행
docker compose up -d              # 백그라운드 실행
docker compose up -d --build      # 이미지 재빌드 후 실행
docker compose down               # 컨테이너+네트워크 제거
docker compose down -v            # 볼륨까지 함께 제거
docker compose ps                 # 서비스 상태 확인
docker compose logs -f            # 전체 로그 스트리밍
docker compose logs -f web        # 특정 서비스 로그
docker compose exec web bash      # 실행 중 서비스에 접속
docker compose build              # 이미지만 빌드
docker compose pull               # 이미지만 미리 받기
docker compose restart web        # 특정 서비스 재시작
docker compose config             # 최종 병합된 설정 검증/출력
```

---

## 스케일링 & 환경 분리

### 서비스 복제

```bash
docker compose up -d --scale web=3   # web 서비스 3개 인스턴스
```

> 포트를 고정(`8080:80`)하면 충돌하므로 스케일링 시엔 포트 범위나 로드밸런서(리버스 프록시)를 앞단에 둔다.

### 환경별 오버라이드

```bash
# base + 환경별 파일 병합
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

- `compose.yaml` — 공통 설정
- `compose.override.yaml` — 로컬 개발용(자동 병합)
- `compose.prod.yaml` — 운영용(명시적으로 지정)

### .env 파일

```bash
# .env
TAG=1.2.0
DB_PASSWORD=secret
```

```yaml
services:
  web:
    image: myapp:${TAG}          # .env의 값 치환
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
```

---

## 실습 예시: 웹 + DB 스택

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb   # 서비스명 db로 접속
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up -d
docker compose ps        # app, db 상태 확인
docker compose logs -f app
docker compose down      # 정리 (볼륨은 유지)
```

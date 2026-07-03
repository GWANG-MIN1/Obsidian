# 09 컨테이너 보안

컨테이너는 호스트 커널을 공유하므로, 잘못 구성하면 컨테이너 탈출(container escape)이나 권한 상승으로 이어질 수 있다.  
"기본값은 편리하지만 안전하지 않다"는 전제로, 최소 권한 원칙을 적용하는 것이 핵심이다.

---

## 컨테이너 격리의 원리

컨테이너는 VM처럼 별도 커널을 갖는 게 아니라, 리눅스 커널 기능으로 프로세스를 격리한다.

| 기술 | 역할 |
|------|------|
| **Namespace** | 프로세스·네트워크·마운트·PID 등을 격리해 "따로 있는 것처럼" 보이게 함 |
| **cgroups** | CPU·메모리 등 자원 사용량 제한(06장) |
| **Capabilities** | root 권한을 잘게 쪼개 필요한 것만 부여 |
| **Seccomp** | 컨테이너가 호출할 수 있는 시스템콜을 필터링 |
| **AppArmor / SELinux** | 파일·자원 접근을 강제 접근 제어(MAC)로 제한 |

> 커널을 공유하기 때문에, 커널 취약점은 곧 모든 컨테이너의 위험이 된다. 호스트 커널 패치가 중요하다.

---

## 1. 비-root 사용자로 실행 (USER)

컨테이너 내부 프로세스가 root(UID 0)로 돌면, 탈출 시 호스트에도 영향이 크다.  
Dockerfile에서 전용 사용자를 만들어 실행한다.

```dockerfile
FROM node:20-alpine

RUN addgroup -S app && adduser -S app -G app

WORKDIR /app
COPY --chown=app:app . .

USER app                      # 이후 프로세스는 app 사용자로 실행
CMD ["node", "index.js"]
```

```bash
# 런타임에서 강제 지정도 가능
docker run --user 1000:1000 myapp
```

---

## 2. Capabilities 최소화

컨테이너 root는 이미 일부 capability만 갖지만, 대부분 애플리케이션은 그마저도 불필요하다.  
**모두 제거한 뒤 필요한 것만 추가**하는 것이 안전하다.

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

| Capability | 용도 |
|------------|------|
| `NET_BIND_SERVICE` | 1024 미만 포트 바인딩 |
| `CHOWN` | 파일 소유자 변경 |
| `SETUID` / `SETGID` | UID/GID 변경 |

```bash
docker run --security-opt no-new-privileges myapp   # 권한 상승 차단
```

---

## 3. 읽기 전용 파일 시스템

컨테이너 파일 시스템을 읽기 전용으로 만들면 침입자가 파일을 변조하기 어렵다.  
쓰기가 필요한 경로만 `tmpfs`나 볼륨으로 예외 처리한다.

```bash
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  myapp
```

---

## 4. 시크릿 관리

**환경변수에 비밀번호를 넣지 말 것.** `docker inspect`, 로그, 자식 프로세스에 노출된다.

```yaml
# Compose secrets — 파일로 마운트되어 /run/secrets/에 위치
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./db_password.txt
```

| 방식 | 안전도 |
|------|--------|
| `ENV`에 평문 | ❌ inspect·로그 노출 |
| `--env-file` | △ 파일 관리 필요, 여전히 환경변수 |
| Docker/Compose secrets | ✅ 파일 마운트, 이미지에 안 남음 |
| 외부 Vault (HashiCorp Vault, AWS Secrets Manager) | ✅✅ 중앙 관리·감사·회전 |

> 빌드 시 시크릿은 `docker build --secret`(BuildKit)으로 주입해 레이어에 남기지 않는다.

---

## 5. 이미지 보안

### 신뢰할 수 있는 베이스 이미지

- 공식(Official) 또는 검증된(Verified Publisher) 이미지 사용
- 태그를 `latest` 대신 **버전 고정**, 가능하면 **digest 고정**

```bash
docker pull nginx@sha256:abc123...   # 다이제스트로 고정 = 불변 보장
```

### 최소 베이스 이미지

공격 표면을 줄이기 위해 불필요한 패키지가 없는 경량 이미지를 쓴다.

| 이미지 | 특징 |
|--------|------|
| `alpine` | 5MB 수준, musl libc |
| `-slim` | 데비안 최소 구성 |
| `distroless` | 쉘·패키지 매니저 없음, 앱+런타임만 |
| `scratch` | 완전 빈 이미지(정적 바이너리용) |

### 취약점 스캔

```bash
docker scout cves 이미지명:태그        # Docker Scout
trivy image 이미지명:태그              # Trivy (CI에서 널리 사용)
```

> CI 파이프라인에 스캔을 넣어, 알려진 CVE가 있는 이미지는 배포를 막는다(Shift-Left 보안).

---

## 6. Rootless Docker

도커 데몬 자체를 root가 아닌 일반 사용자로 실행해, 데몬 탈취 시 피해를 줄인다.

```bash
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

- 데몬·컨테이너가 호스트에서 일반 사용자 권한만 가짐
- 일부 기능(특정 네트워크·포트) 제약이 있으나 보안상 권장

---

## 보안 체크리스트

- [ ] `USER`로 비-root 실행
- [ ] `--cap-drop ALL` 후 필요 capability만 추가
- [ ] `--security-opt no-new-privileges`
- [ ] `--read-only` + 필요한 경로만 tmpfs
- [ ] 시크릿은 환경변수 금지, secrets/Vault 사용
- [ ] 베이스 이미지 버전/digest 고정, 경량 이미지 선택
- [ ] CI에서 이미지 취약점 스캔(Trivy/Scout)
- [ ] 호스트 커널·도커 엔진 정기 패치
- [ ] `docker.sock`을 컨테이너에 마운트하지 않기(=호스트 장악과 동일)

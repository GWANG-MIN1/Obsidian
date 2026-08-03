# 01 컨테이너 기초

컨테이너는 가상머신이 아니라 **호스트 커널을 공유하는, 격리된 하나의 프로세스**다.  
별도 OS를 부팅하지 않으므로 시작이 즉각적이고 오버헤드가 작다. 이 장에서는 그 전제 위에서 컨테이너를 만들고·확인하고·드나들고·지우는 기본 생명주기를 다룬다.

---

## 컨테이너 vs 가상머신

```
      가상머신                        컨테이너
┌──────────────────┐        ┌──────────────────┐
│ App A  │  App B  │        │ App A  │  App B  │
├────────┼─────────┤        ├────────┴─────────┤
│ Guest  │  Guest  │        │  Docker Engine   │
│  OS    │   OS    │        ├──────────────────┤
├────────┴─────────┤        │   Host Kernel    │ ← 커널을 공유
│    Hypervisor    │        ├──────────────────┤
├──────────────────┤        │     Hardware     │
│   Host OS / HW   │        └──────────────────┘
└──────────────────┘
```

| 항목 | 가상머신 | 컨테이너 |
|---|---|---|
| 격리 단위 | 하드웨어 가상화 (Guest OS) | 프로세스 격리 (커널 공유) |
| 부팅 시간 | 수십 초 ~ 분 | 밀리초 ~ 초 |
| 이미지 크기 | GB 단위 | MB 단위 |
| 커널 | 각자 보유 | 호스트 것을 공유 |
| 격리 수준 | 강함 (커널 분리) | 상대적으로 약함 |

> 커널을 공유한다는 점이 컨테이너의 장점(가볍다)이자 보안 한계(커널 취약점이 곧 전체의 위험)다. → `09-security/`

---

## 격리를 만드는 커널 기능

컨테이너는 도커가 발명한 개념이 아니라, 리눅스 커널이 이미 갖고 있던 기능들의 조합이다.

| 기능 | 역할 |
|---|---|
| **Namespace** | PID·네트워크·마운트·호스트명 등을 격리해 "혼자 있는 것처럼" 보이게 함 |
| **cgroups** | CPU·메모리 등 자원 사용량을 제한 (→ `06-resource-limit/`) |
| **Capabilities** | root 권한을 잘게 쪼개 필요한 것만 부여 (→ `09-security/`) |
| **UnionFS** | 읽기 전용 이미지 레이어 위에 쓰기 레이어를 겹침 (→ `07-image/`) |

```bash
# 컨테이너가 실제로는 '호스트의 한 프로세스'일 뿐임을 확인
docker run -d --name demo nginx
docker inspect -f '{{.State.Pid}}' demo    # 호스트 기준 PID 출력
ps -ef | grep <위에서 나온 PID>              # 호스트 ps 에 그대로 보인다
```

---

## 이미지와 컨테이너의 관계

**이미지 = 읽기 전용 템플릿, 컨테이너 = 이미지 + 쓰기 가능 레이어.**  
같은 이미지로 컨테이너를 100개 만들어도 이미지 본체는 한 벌만 저장된다.

```
이미지 (읽기 전용)
├── layer 3
├── layer 2
└── layer 1
        │  docker run
        ▼
컨테이너 = 위 레이어들 + [ 쓰기 가능 레이어 ]
                          └─ 여기 쓴 내용은 컨테이너 삭제 시 함께 사라짐
```

> 컨테이너 내부에 만든 파일은 컨테이너를 지우면 같이 없어진다. 데이터를 살리려면 볼륨을 쓴다. → `03-volumes/`

---

## 컨테이너 생성 & 실행

```bash
# 대화형 실행 (-i 표준입력 유지, -t TTY 할당)
docker run -i -t --name mycontainer ubuntu:24.04

# 백그라운드 실행
docker run -d --name web nginx

# 생성만 하고 시작하지 않음
docker create --name web2 nginx

# 중지된 컨테이너를 다시 시작하며 붙기
docker start -ai mycontainer

# 종료되면 자동으로 삭제 (일회성 테스트에 유용)
docker run --rm -it ubuntu:24.04 bash

# 이미지의 기본 CMD 대신 다른 명령 실행
docker run --rm ubuntu:24.04 cat /etc/os-release
```

`docker run` = `docker create` + `docker start`를 한 번에 하는 명령이다.

| 옵션 | 의미 |
|---|---|
| `-i` | 표준입력(stdin)을 열어둔다 |
| `-t` | 가상 터미널(TTY)을 할당한다 |
| `-d` | 백그라운드(detached)로 실행 |
| `--name` | 컨테이너 이름 지정 (생략 시 랜덤 이름) |
| `--rm` | 컨테이너 종료 시 자동 삭제 |

> 이름은 중복될 수 없다. 이미 같은 이름이 있으면 `Conflict. The container name ... is already in use` 에러가 난다.  
> 태그를 생략하면 `:latest`가 자동으로 붙는다 — 재현성이 필요하면 태그를 명시하는 습관을 들이자.

---

## 컨테이너 라이프사이클

```
                    docker run = create + start
                  ┌──────────────────────────────┐
                  │                              │
   (없음) ──create──▶ created ──start──▶ running ──pause──▶ paused
                                    │      ▲                  │
                                    │      └────unpause───────┘
                        stop / kill │
                      프로세스 종료   ▼
                                  exited ──rm──▶ (삭제)
                                    │  ▲
                                    └──┘ start (재시작 가능)
```

```bash
docker stop 컨테이너명      # SIGTERM 전송 → 10초 후에도 안 죽으면 SIGKILL
docker stop -t 30 컨테이너명 # 유예 시간 30초로 변경
docker kill 컨테이너명      # SIGKILL 즉시 전송 (정리 작업 없이 강제 종료)
docker restart 컨테이너명
docker pause 컨테이너명 / docker unpause 컨테이너명
```

> `stop`은 애플리케이션에 종료 신호를 주고 기다리지만 `kill`은 곧바로 죽인다.  
> 운영에서는 데이터 유실을 막기 위해 `stop`을 쓰고, 앱이 SIGTERM을 제대로 처리하도록 만드는 것이 중요하다.

---

## 컨테이너 목록 확인

```bash
docker ps                      # 실행 중
docker ps -a                   # 전체 (중지된 것 포함)
docker ps -q                   # ID만 출력 (다른 명령에 넘길 때)
docker ps -a --filter "status=exited"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"

docker inspect 컨테이너명       # 설정 전체를 JSON 으로
docker logs -f 컨테이너명       # 표준출력 로그 실시간 확인
docker top 컨테이너명           # 컨테이너 안에서 도는 프로세스
docker stats                   # 실시간 CPU/메모리 사용량
```

---

## 컨테이너 안으로 들어가기 — attach vs exec

이 둘의 차이가 초보자가 컨테이너를 자꾸 죽이는 원인이다.

| 명령 | 동작 | 나올 때 |
|---|---|---|
| `docker attach` | 메인 프로세스(PID 1)의 입출력에 **연결** | `exit` 하면 PID 1이 죽어 **컨테이너도 종료** |
| `docker exec -it` | 컨테이너 안에서 **새 프로세스** 실행 | `exit` 해도 컨테이너는 계속 실행 |

```bash
# 실행 중인 컨테이너에 새 셸을 띄운다 (실무 기본)
docker exec -it 컨테이너명 bash

# 셸 없이 명령 하나만 실행
docker exec 컨테이너명 ls /etc

# 메인 프로세스에 직접 붙기
docker attach 컨테이너명
```

> `attach`나 대화형 `run` 상태에서 컨테이너를 **살려두고 빠져나오려면 `Ctrl+P`, `Ctrl+Q`**를 누른다.  
> 그냥 `exit`을 치면 PID 1이 종료되어 컨테이너가 함께 멈춘다.

---

## 컨테이너 삭제

```bash
docker rm 컨테이너명            # 중지된 컨테이너 삭제
docker rm -f 컨테이너명         # 실행 중이어도 강제 삭제
docker rm $(docker ps -aq)     # 전체 삭제
docker container prune         # 중지된 컨테이너 일괄 정리
```

> 실행 중인 컨테이너는 그냥 `rm` 하면 `You cannot remove a running container` 에러가 난다.  
> `-f`로 강제하거나 `stop` 후 삭제한다.

---

## 배운 점

- 컨테이너는 VM이 아니라 **호스트 커널을 공유하는 격리된 프로세스** — 그래서 가볍고, 그래서 격리가 상대적으로 약하다
- 격리는 도커가 아니라 **리눅스 커널(namespace·cgroups·capabilities)** 이 제공한다
- 이미지는 읽기 전용, 컨테이너는 그 위의 **쓰기 레이어** — 컨테이너를 지우면 쓴 데이터도 사라진다
- `docker run` = `create` + `start`
- 컨테이너 이름은 중복 불가 (이미 있으면 Conflict 에러)
- 태그를 생략하면 자동으로 `:latest`가 붙는다
- `exit` → 컨테이너 종료 / `Ctrl+P`, `Ctrl+Q` → 컨테이너 살려두고 나오기
- 컨테이너 안을 들여다볼 땐 `attach`가 아니라 **`exec -it`** 을 쓴다 (나가도 안 죽는다)
- `stop`은 SIGTERM 후 유예, `kill`은 즉시 SIGKILL

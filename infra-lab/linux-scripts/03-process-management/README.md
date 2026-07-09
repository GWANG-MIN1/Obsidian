# 03 프로세스 관리

실행 중인 모든 프로그램은 **프로세스**이며 고유한 PID를 갖는다. 컨테이너 안의 애플리케이션도 결국 호스트에서 하나의 프로세스로 보인다 — 프로세스를 관찰·제어하는 능력이 트러블슈팅의 기본기다.

---

## 프로세스 조회

```bash
ps aux               # 전체 프로세스 (BSD 스타일, 사용자·CPU·MEM 포함)
ps -ef               # 전체 프로세스 (표준 스타일, PPID 포함)
ps aux | grep nginx  # 특정 프로세스 필터
pgrep -a nginx       # 이름으로 PID 찾기
pstree -p            # 부모-자식 트리 구조
```

`ps aux` 주요 열:

| 열 | 의미 |
|----|------|
| `USER` | 프로세스 소유자 |
| `PID` | 프로세스 ID |
| `%CPU` / `%MEM` | CPU·메모리 사용률 |
| `STAT` | 상태 (R 실행, S 대기, Z 좀비, D 무중단 대기) |
| `START` | 시작 시각 |
| `COMMAND` | 실행 명령 |

---

## 실시간 모니터링

```bash
top                  # 실시간 자원 사용 (P: CPU정렬, M: MEM정렬, k: kill)
htop                 # 컬러·마우스 지원 향상판 (별도 설치)
```

`top` 상단 지표:

- **load average**: 1·5·15분 평균 부하. CPU 코어 수보다 크면 과부하 신호
- **%Cpu(s)**: us(사용자)·sy(시스템)·wa(IO 대기)·id(유휴)
- 프로세스 목록: PID·사용자·PR(우선순위)·%CPU·%MEM

---

## 시그널과 종료

`kill`은 프로세스에 **시그널을 보내는** 명령이다 (단순히 죽이는 게 아님).

| 시그널 | 번호 | 의미 |
|--------|------|------|
| `SIGTERM` | 15 | **정상 종료 요청** (정리 후 종료, 기본값) |
| `SIGKILL` | 9 | 강제 종료 (무시 불가, 정리 없음 — 최후 수단) |
| `SIGHUP` | 1 | 터미널 종료 / 설정 리로드 관례 |
| `SIGINT` | 2 | 인터럽트 (Ctrl+C) |
| `SIGSTOP` / `SIGCONT` | 19/18 | 일시 정지 / 재개 |

```bash
kill 1234            # SIGTERM (기본)
kill -15 1234        # 명시적 SIGTERM
kill -9 1234         # SIGKILL (강제)
kill -HUP 1234       # 설정 리로드 (nginx 등)
pkill -f "python app.py"   # 명령줄 패턴으로 종료
killall nginx        # 이름으로 전부 종료
```

> **먼저 SIGTERM, 안 되면 SIGKILL.** SIGKILL은 프로세스가 정리(파일 flush·연결 종료)할 기회를 주지 않아 데이터 손상 위험이 있다. Docker의 `docker stop`도 SIGTERM 후 유예시간 뒤 SIGKILL이다.

---

## 잡 제어 (포그라운드 / 백그라운드)

```bash
long_task            # 포그라운드 실행 (터미널 점유)
long_task &          # 백그라운드 실행
Ctrl+Z               # 현재 잡 일시 정지
jobs                 # 잡 목록
bg %1                # 1번 잡을 백그라운드로 재개
fg %1                # 1번 잡을 포그라운드로
```

### 로그아웃 후에도 유지

```bash
nohup long_task &                    # HUP 무시, nohup.out에 로그
setsid long_task                     # 새 세션에서 분리 실행
disown -h %1                         # 실행 중 잡을 셸에서 분리
```

> 장시간 작업은 `tmux`·`screen` 세션에서 돌리는 것이 가장 안전하다 — 접속이 끊겨도 세션이 살아있다.

---

## 우선순위 (nice)

CPU 스케줄링 우선순위를 `-20`(최고) ~ `19`(최저)로 조절한다. 기본은 0.

```bash
nice -n 10 backup.sh         # 낮은 우선순위로 시작 (양보)
nice -n -5 critical.sh       # 높은 우선순위 (root 필요)
renice -n 5 -p 1234          # 실행 중 프로세스 조정
```

> 숫자가 **클수록 "친절하게 양보"** = 낮은 우선순위. 배치·백업 작업에 `nice`를 걸면 대화형 작업이 느려지지 않는다.

---

## 프로세스 상세 들여다보기

```bash
lsof -p 1234                 # 프로세스가 연 파일·소켓
lsof -i :80                  # 80포트를 쓰는 프로세스
cat /proc/1234/status        # 상태·메모리·UID
ls -l /proc/1234/cwd         # 프로세스의 작업 디렉터리
strace -p 1234               # 시스템콜 추적 (디버깅)
```

---

## 배운 점
- 모든 프로그램은 PID를 가진 프로세스, `ps aux`·`top`이 관찰의 기본
- `kill`은 **시그널 전송** — SIGTERM(정상)을 먼저, SIGKILL(강제)은 최후
- `&`·`fg`·`bg`·`jobs`로 잡을 제어하고, 장시간 작업은 `nohup`/`tmux`로 보호
- `nice`/`renice`로 우선순위를 조절해 중요한 작업을 보호
- `/proc/<PID>`와 `lsof`로 프로세스 내부를 들여다볼 수 있다

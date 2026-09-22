# 04 systemd

systemd는 리눅스가 부팅될 때 **가장 먼저 실행되는 프로세스(PID 1)** 이자, 서버의 모든 서비스를 **켜고, 끄고, 죽으면 다시 살리고, 로그를 모아주는 관리인**이다.
"서비스를 부팅할 때 자동으로 뜨게 하려면?" "서비스가 자꾸 죽는데 원인은?" 같은 질문이 전부 여기서 나온다.

---

## 왜 필요한가 — init의 역할

컴퓨터를 켜면 커널이 올라온 다음, **누군가는 나머지를 다 띄워야** 한다. 네트워크 설정, 디스크 마운트, sshd, nginx, DB…
그 "누군가"가 **init 프로세스(PID 1)** 이고, 요즘 대부분의 배포판(Ubuntu, RHEL, Amazon Linux)에서는 그게 **systemd**다.

예전 방식(SysV init)과 비교하면 좋아진 점:

| 예전 (SysV init) | systemd |
|---|---|
| 셸 스크립트를 **순서대로 하나씩** 실행 | **동시에(병렬)** 띄워서 부팅이 빠름 |
| 서비스 간 의존성을 순서 번호로 대충 관리 | "A는 B 다음에" 같은 **의존성을 명시적으로** 선언 |
| 서비스가 죽으면 그냥 죽은 채로 | **죽으면 자동 재시작** 설정 가능 |
| 로그가 서비스마다 제각각 | **journald로 로그를 한곳에** 모음 |

---

## 유닛(Unit) — systemd가 관리하는 단위

systemd는 관리 대상을 전부 **유닛**이라고 부르고, 파일 확장자로 종류를 나눈다.

| 종류 | 뭘 관리하나 | 예 |
|---|---|---|
| `.service` | **프로세스(데몬)** — 가장 많이 씀 | `nginx.service`, `sshd.service` |
| `.timer` | **정해진 시간에 서비스 실행** (cron 대체) | `backup.timer` |
| `.socket` | 소켓에 요청이 오면 그때 서비스 실행 | `docker.socket` |
| `.mount` | 파일시스템 마운트 | `data.mount` |
| `.target` | **유닛 묶음** (부팅 단계 같은 것) | `multi-user.target` |

> **target**은 "이 단계까지 오면 이것들이 다 떠 있어야 한다"는 **묶음**이다.
> - `multi-user.target` ≈ 예전 runlevel 3 (GUI 없는 서버 모드) — **서버는 보통 여기**
> - `graphical.target` ≈ runlevel 5 (GUI 포함)

---

## 유닛 파일 읽기

내가 만든 Python 앱을 서비스로 등록한다고 해보자. `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Python App           # 설명 (status에 표시됨)
After=network-online.target         # 네트워크가 준비된 "다음에" 시작
Wants=network-online.target         # 네트워크 준비 유닛도 같이 켜달라

[Service]
Type=simple                         # 실행한 프로세스가 곧 서비스 본체
User=app                            # root가 아니라 app 사용자로 실행 (최소 권한)
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 app.py   # 시작 명령 (절대경로로!)
Restart=on-failure                  # 비정상 종료 시 자동 재시작
RestartSec=5                        # 재시작 전 5초 대기
Environment=ENV=prod

[Install]
WantedBy=multi-user.target          # enable하면 multi-user 단계에서 자동 시작
```

세 구역의 역할:

- **[Unit]** — 이 유닛이 **무엇이고, 무엇과 관계있는지** (설명, 순서, 의존성)
- **[Service]** — **어떻게 실행하는지** (명령, 사용자, 재시작 정책)
- **[Install]** — **enable 했을 때 어디에 매달릴지** (부팅 시 자동 시작 연결)

> **`After` vs `Wants`/`Requires` 헷갈리기 쉬움**
> - `After=` 는 **순서만** 정한다. "B가 켜진다면 그다음에 켜라" — B를 켜주진 않는다.
> - `Wants=` / `Requires=` 는 **같이 켜달라는 의존성**이다. `Requires`는 B가 실패하면 나도 실패, `Wants`는 B가 실패해도 나는 켠다.
> - 그래서 보통 **둘을 짝으로** 쓴다.

### 유닛 파일 위치와 우선순위

| 위치 | 누가 쓰나 | 우선순위 |
|---|---|---|
| `/etc/systemd/system/` | **관리자(나)** | 높음 |
| `/run/systemd/system/` | 실행 중 임시 생성 | 중간 |
| `/usr/lib/systemd/system/` (또는 `/lib/...`) | **패키지(apt, yum)** 가 설치 | 낮음 |

> 패키지가 설치한 `/usr/lib/...` 파일을 직접 고치면 **업데이트 때 덮어써진다.**
> 설정을 바꾸고 싶으면 `sudo systemctl edit nginx`로 **덮어쓰기용 조각(drop-in)** 을 만든다 → `/etc/systemd/system/nginx.service.d/override.conf`

---

## 자주 쓰는 명령어

```bash
sudo systemctl start nginx      # 지금 켜기
sudo systemctl stop nginx       # 지금 끄기 (SIGTERM → 유예 → SIGKILL, 01 참고)
sudo systemctl restart nginx    # 껐다 켜기
sudo systemctl reload nginx     # 끄지 않고 설정만 다시 읽기 (지원하는 서비스만)

sudo systemctl enable nginx     # 부팅 시 자동 시작 "예약"
sudo systemctl disable nginx    # 자동 시작 해제
sudo systemctl enable --now nginx   # 예약 + 지금 바로 켜기

systemctl status nginx          # 상태 + 최근 로그 몇 줄
systemctl is-enabled nginx      # 자동 시작 설정됐는지
systemctl list-units --type=service --state=failed   # 실패한 서비스만

sudo systemctl daemon-reload    # 유닛 파일을 수정했으면 반드시!
```

---

## start vs enable — 가장 많이 나오는 질문

| | 지금 켜지나? | 재부팅 후 켜지나? |
|---|---|---|
| `start` | ✅ | ❌ |
| `enable` | ❌ | ✅ |
| `enable --now` | ✅ | ✅ |

`enable`이 실제로 하는 일은 단순하다. `[Install]`의 `WantedBy=multi-user.target`을 보고
`/etc/systemd/system/multi-user.target.wants/` 폴더에 **내 유닛 파일로 가는 심볼릭링크를 만드는 것**이다.
부팅 때 systemd가 multi-user.target을 켜면서 그 폴더 안의 유닛들을 같이 켠다.

> 그래서 "재부팅했더니 서비스가 안 떠 있어요" → 십중팔구 **start만 하고 enable을 안 한 것**.

---

## daemon-reload — "고쳤는데 반영이 안 돼요"

systemd는 유닛 파일을 **읽어서 메모리에 들고 있다.** 파일을 고쳐도 systemd는 모른다.

```bash
sudo vi /etc/systemd/system/myapp.service
sudo systemctl daemon-reload     # "유닛 파일 다시 읽어"
sudo systemctl restart myapp     # 바뀐 설정으로 재시작
```

> `daemon-reload`는 **유닛 파일(systemd 설정)** 을 다시 읽는 것이고,
> `reload`는 **서비스 자체의 설정(nginx.conf 등)** 을 다시 읽게 하는 것이다. 이름이 비슷하니 주의.

---

## journalctl — 로그 보기

systemd로 띄운 서비스의 표준출력·에러는 **journald**가 모아둔다.

```bash
journalctl -u nginx               # nginx 로그 전체
journalctl -u nginx -f            # 실시간으로 계속 보기 (tail -f 같은)
journalctl -u nginx --since "10 min ago"
journalctl -u nginx -n 100        # 마지막 100줄
journalctl -b                     # 이번 부팅 이후 전체 로그
journalctl -p err                 # 에러 이상만
```

---

## 서비스가 자꾸 죽을 때 확인 순서

1. `systemctl status myapp` — **Active 상태**, **종료 코드**(`code=exited, status=1`), 최근 로그 몇 줄 확인
2. `journalctl -u myapp -n 100` — 죽기 직전 **에러 메시지** 확인
3. 흔한 원인들:
   - `ExecStart` 경로가 틀림 → systemd는 **절대경로**가 필요하다
   - 실행 사용자(`User=`)에게 파일·디렉토리 **권한이 없음** → [03 퍼미션](03-permission.md)
   - 포트가 이미 사용 중 → `ss -tlnp | grep :8080`
   - 셸에서는 되는데 서비스로는 안 됨 → systemd는 **로그인 셸의 환경변수(PATH, .bashrc)를 안 읽는다.** `Environment=`로 명시해야 한다
4. `Restart=on-failure`가 있는데 계속 죽으면, 짧은 시간에 너무 많이 재시작해서 systemd가 **포기(start-limit-hit)** 한 상태일 수 있다 → 원인 수정 후 `systemctl reset-failed myapp`

---

## 면접에서 이렇게 말한다

**Q. systemd가 뭔가요?**
> 리눅스 부팅 시 커널 다음으로 가장 먼저 실행되는 PID 1 프로세스로, 서비스들을 유닛 단위로 관리합니다. 의존성에 따라 서비스를 병렬로 띄우고, 죽으면 재시작하고, journald로 로그를 모아줍니다. systemctl로 서비스를 제어하고 journalctl로 로그를 봅니다.

**Q. start와 enable 차이는?**
> start는 지금 당장 서비스를 실행하는 것이고, enable은 부팅할 때 자동으로 시작되도록 등록하는 것입니다. enable은 유닛 파일의 WantedBy에 적힌 target의 wants 디렉토리에 심볼릭링크를 만드는 방식이라 지금 켜지는 않습니다. 둘 다 하려면 enable --now를 씁니다.

**Q. 직접 만든 애플리케이션을 서비스로 등록하려면?**
> /etc/systemd/system 아래에 .service 파일을 만들고, [Service]에 ExecStart로 절대경로 실행 명령, User로 실행 사용자, Restart로 재시작 정책을 적습니다. [Install]에 WantedBy=multi-user.target을 적은 뒤 daemon-reload, enable --now 순서로 실행하고 status와 journalctl로 확인합니다.

**Q. 서비스가 계속 재시작되며 죽습니다.**
> systemctl status로 종료 코드와 최근 로그를 보고, journalctl -u로 죽기 직전 에러를 확인합니다. 셸에서는 되는데 서비스로 안 되면 systemd가 로그인 환경변수를 안 읽는 문제이거나, User로 지정한 계정에 권한이 없는 경우가 많습니다. 포트 충돌은 ss로 확인합니다.

**Q. 유닛 파일을 수정했는데 반영이 안 돼요.**
> systemd는 유닛 파일을 메모리에 캐시하고 있어서, 수정 후 systemctl daemon-reload를 해야 다시 읽습니다. 그다음 서비스를 재시작해야 적용됩니다.

---

## 더 보기

- [infra-lab/linux-scripts/07-systemd-service](../infra-lab/linux-scripts/07-systemd-service/README.md) — 유닛·타이머 실습
- [01 프로세스와 시그널](01-process-signal.md) — stop이 보내는 시그널, PID 1

# 07 systemd & 서비스 관리

`systemd`는 현대 리눅스의 **init 시스템(PID 1)** 이자 서비스 매니저다. 부팅·서비스·로그·타이머·마운트를 하나의 프레임워크로 관리한다. 서버에서 데몬을 다루는 표준 방법이다.

---

## systemd란

부팅 후 커널이 가장 먼저 실행하는 프로세스(PID 1)로, 나머지 모든 서비스의 부모다. **선언형 유닛 파일**로 "무엇을 어떤 순서로 띄울지"를 정의한다 — 컨테이너 오케스트레이션의 선언형 모델과 같은 사고방식.

```bash
systemctl              # 전체 유닛 상태
systemctl --version    # 버전 확인
ps -p 1 -o comm=       # PID 1이 systemd인지 확인
```

---

## 유닛(Unit) 종류

| 확장자 | 역할 |
|--------|------|
| `.service` | 데몬·프로세스 (nginx, sshd) |
| `.socket` | 소켓 활성화 (요청 오면 서비스 기동) |
| `.timer` | 스케줄 실행 (cron 대체) |
| `.target` | 유닛 묶음 / 부팅 단계 (runlevel 대체) |
| `.mount` | 파일시스템 마운트 |

유닛 파일 위치:

- `/lib/systemd/system/` — 패키지가 설치한 기본 유닛
- `/etc/systemd/system/` — 관리자가 만들거나 덮어쓴 유닛 (우선)

---

## 서비스 제어 (systemctl)

```bash
systemctl status nginx        # 상태·최근 로그·PID
systemctl start nginx         # 시작
systemctl stop nginx          # 중지
systemctl restart nginx       # 재시작
systemctl reload nginx        # 설정만 리로드 (무중단)
systemctl enable nginx        # 부팅 시 자동 시작 등록
systemctl disable nginx       # 자동 시작 해제
systemctl enable --now nginx  # 등록 + 즉시 시작 (한 번에)
systemctl is-active nginx     # 실행 중 여부
systemctl is-enabled nginx    # 자동 시작 여부
```

> **`enable`(부팅 자동 시작)** 과 **`start`(지금 시작)** 는 별개다. 서버 재부팅 후에도 살아있게 하려면 `enable`을 잊지 말 것. `enable --now`가 둘을 한 번에 처리한다.

---

## 커스텀 서비스 만들기

내 애플리케이션을 systemd 서비스로 등록하면 자동 재시작·부팅 시작·로그 통합을 공짜로 얻는다.

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Web App
After=network.target                 # 네트워크 준비 후 시작

[Service]
Type=simple
User=appsvc                          # 비-root로 실행
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure                   # 비정상 종료 시 자동 재시작
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target           # 부팅 시 이 타깃에 포함
```

```bash
systemctl daemon-reload              # 유닛 파일 변경 반영 (필수)
systemctl enable --now myapp
systemctl status myapp
```

| 지시어 | 의미 |
|--------|------|
| `After` | 지정 유닛 이후 시작 (순서) |
| `Restart` | `no`/`on-failure`/`always` — 재시작 정책 |
| `User` | 실행 사용자 (보안: 비-root 권장) |
| `WantedBy` | `enable` 시 연결될 타깃 |

> Docker의 `--restart` 정책과 판박이다. systemd의 `Restart=on-failure`가 곧 컨테이너 자동 복구의 호스트 버전.

---

## 저널 로그 (journalctl)

systemd는 모든 서비스의 stdout/stderr를 **저널**에 통합 수집한다.

```bash
journalctl -u nginx           # 특정 서비스 로그
journalctl -u nginx -f        # 실시간 추적
journalctl -u nginx --since "10 min ago"
journalctl -u nginx -n 50     # 최근 50줄
journalctl -p err             # 에러 이상만
journalctl --since today
journalctl -b                 # 이번 부팅의 로그
journalctl --disk-usage       # 저널 용량
journalctl --vacuum-time=7d   # 7일 이전 로그 정리
```

---

## 타이머 (cron 대체)

`.timer` 유닛으로 스케줄 작업을 정의한다. cron보다 로그 통합·의존성 관리가 좋다.

```ini
# backup.timer
[Timer]
OnCalendar=*-*-* 03:00:00      # 매일 새벽 3시
Persistent=true               # 놓친 실행 보정

[Install]
WantedBy=timers.target
```

```bash
systemctl list-timers          # 등록된 타이머·다음 실행 시각
```

---

## 부팅 타깃 (runlevel)

```bash
systemctl get-default              # 기본 부팅 타깃
systemctl set-default multi-user.target   # GUI 없이 부팅 (서버)
systemctl isolate rescue.target    # 단일 사용자 모드 전환
```

| 타깃 | 옛 runlevel | 의미 |
|------|-------------|------|
| `multi-user.target` | 3 | CLI 다중 사용자 (서버 표준) |
| `graphical.target` | 5 | GUI 포함 |
| `rescue.target` | 1 | 복구 모드 |

---

## 배운 점
- systemd = PID 1 + 서비스 매니저, **선언형 유닛 파일**로 시스템을 정의
- `enable`(부팅 자동)과 `start`(지금 시작)은 별개 — `enable --now`로 동시에
- 유닛 파일 수정 후엔 반드시 `daemon-reload`
- `Restart=on-failure`는 Docker `--restart`의 호스트 버전 — 셀프힐링
- 로그는 `journalctl -u <서비스> -f`로 통합 확인, 타이머가 cron을 대체

# 10 모니터링 & 성능 · 트러블슈팅

서버 운영의 절반은 **"지금 무슨 일이 일어나는가"를 관찰**하는 일이다. CPU·메모리·디스크·네트워크의 지표를 읽고, 로그를 추적하며, 병목을 좁혀가는 절차를 익힌다. 앞의 모든 장이 여기서 종합된다.

---

## 큰 그림: 4대 자원

성능 문제는 결국 네 가지 자원 중 하나의 병목이다.

| 자원 | 핵심 도구 | 병목 신호 |
|------|-----------|-----------|
| **CPU** | `top`, `mpstat` | load average↑, %us/%sy↑ |
| **메모리** | `free`, `vmstat` | swap 사용↑, OOM 발생 |
| **디스크 IO** | `iostat`, `iotop` | %wa(IO 대기)↑, %util 100% |
| **네트워크** | `ss`, `iftop` | 대역폭 포화, 재전송↑ |

---

## CPU & 부하

```bash
uptime                        # load average (1·5·15분)
top                           # 실시간 (P: CPU 정렬)
mpstat 1                      # 코어별 사용률 (1초 간격)
nproc                         # CPU 코어 수
```

**load average 읽는 법**: 코어 수와 비교한다.

```
4코어 시스템에서
load 2.0  → 여유 (50%)
load 4.0  → 포화 (100%)
load 8.0  → 과부하 (대기 발생)
```

---

## 메모리

```bash
free -h                       # 메모리·swap 요약
vmstat 1                      # 메모리·swap·IO 실시간
cat /proc/meminfo             # 상세
```

`free` 출력에서 **`available`** 이 진짜 가용 메모리다 (캐시는 필요 시 회수됨). `used`가 높아도 `available`이 넉넉하면 정상.

> **swap 사용이 늘고 `si`/`so`(swap in/out)가 계속 발생하면 메모리 부족 신호.** 심하면 커널이 OOM Killer로 프로세스를 강제 종료한다:
> ```bash
> dmesg -T | grep -i "killed process"   # OOM 흔적 확인
> ```

---

## 디스크

### 용량

```bash
df -h                         # 파일시스템별 사용량
df -i                         # inode 사용량 (파일 수 한계)
du -sh /var/*                 # 디렉터리별 용량
du -sh * | sort -rh | head    # 큰 것부터 Top
ncdu /                        # 대화형 디스크 사용 분석
```

> 디스크가 "가득 찼는데 `df`론 여유 있음" → **inode 고갈**을 의심(`df -i`). 작은 파일이 수백만 개면 용량 전에 inode가 먼저 소진된다. 또, 삭제했는데 공간이 안 도는 건 **프로세스가 파일을 잡고 있어서**(`lsof | grep deleted`).

### IO 성능

```bash
iostat -x 1                   # 디스크별 IO 상세 (%util, await)
iotop                         # 프로세스별 IO (root)
```

`%util`이 100%에 가깝거나 `await`(IO 응답시간)가 크면 디스크가 병목.

---

## 로그 분석

```bash
# systemd 저널
journalctl -f                 # 전체 실시간
journalctl -u nginx --since "1 hour ago"
journalctl -p err -b          # 이번 부팅 에러

# 전통 로그 파일
tail -f /var/log/syslog
tail -f /var/log/nginx/error.log
grep -i error /var/log/syslog | tail -50
dmesg -T | tail               # 커널 메시지 (하드웨어·OOM)
```

| 로그 위치 | 내용 |
|-----------|------|
| `/var/log/syslog`·`messages` | 시스템 전반 |
| `/var/log/auth.log`·`secure` | 인증·sudo·SSH |
| `/var/log/nginx/` | 웹 서버 |
| `journalctl` | systemd 서비스 통합 |

---

## 열린 파일 · 포트 · 프로세스

```bash
lsof -i :80                   # 포트 사용 프로세스
lsof -p 1234                  # 프로세스가 연 파일
lsof +D /var/log              # 특정 디렉터리를 쓰는 프로세스
lsof | grep deleted           # 삭제됐지만 잡혀있는 파일
ss -tulpn                     # 리스닝 포트
```

---

## 트러블슈팅 절차 (USE 방법론)

무작정 뒤지지 말고 **자원별로 사용률(Utilization)·포화(Saturation)·에러(Error)** 를 순서대로 확인한다.

```
1. 증상 정의     "느리다" → 어디가? 언제부터? 재현되나?
2. 전체 부하     uptime, top → CPU/메모리 개괄
3. 자원별 확인   free(MEM) / iostat(DISK) / ss(NET)
4. 범인 좁히기   top·iotop·lsof로 프로세스 특정
5. 로그 확인     journalctl·dmesg로 에러·OOM 추적
6. 조치·검증     재시작/스케일/설정 → 지표 재확인
```

### 자주 만나는 시나리오

| 증상 | 1순위 확인 |
|------|-----------|
| 서버 느림 | `uptime`(load) → `top`(범인 프로세스) |
| 메모리 부족 | `free -h` → `dmesg`(OOM) |
| 디스크 풀 | `df -h` → `du -sh *` → `df -i`(inode) |
| 포트 안 열림 | `ss -tulpn` → `journalctl -u 서비스` |
| 응답 없음 | `curl -v` → 방화벽 → 서비스 상태 |

---

## 지속 모니터링으로 가는 다리

일회성 명령을 넘어 상시 감시가 필요하면 도구를 도입한다.

| 계층 | 도구 |
|------|------|
| 지표 수집·시각화 | **Prometheus + Grafana** |
| 로그 집계 | **Loki**, ELK (Elasticsearch·Kibana) |
| 노드 지표 노출 | **node_exporter** |
| 경보 | **Alertmanager** |

> 컨테이너/K8s 환경에서는 이 장의 수동 관찰이 **Prometheus 지표 + Grafana 대시보드 + Alertmanager 경보**로 자동화된다. 하지만 근본 원리(4대 자원·USE 방법론)는 그대로다 — 대시보드가 가리키는 숫자를 해석하는 힘이 여기서 나온다.

---

## 배운 점
- 성능 문제는 **CPU·메모리·디스크·네트워크** 4대 자원의 병목으로 수렴
- load average는 **코어 수와 비교**, `free`는 `available`을 봐야 한다
- 디스크 풀은 용량 외에 **inode 고갈**·**deleted 파일 점유**도 의심
- 로그는 `journalctl`·`dmesg`·`/var/log`, OOM은 `dmesg`에 흔적
- **USE 방법론**(사용률·포화·에러)으로 체계적으로 좁히고, 상시 감시는 Prometheus/Grafana로 확장


# Linux Labs

Linux 기초부터 서버 운영까지 학습 실습 및 명령어 정리 저장소

Docker·Kubernetes가 결국 **리눅스 커널 기능(namespace·cgroups·capabilities) 위에서 도는 프로세스**라는 점에서, 컨테이너/오케스트레이션의 토대가 되는 리눅스를 먼저 다진다.

## 구조
- `commands.md` - 자주 쓰는 Linux 명령어 레퍼런스
- `01-basics/` - 파일시스템 계층(FHS), 셸, 기본 탐색·조작 (ls, cd, cp, mv, find)
- `02-file-permissions/` - 권한(rwx)·소유권(chmod, chown), umask, 특수권한 (SUID, SGID, sticky)
- `03-process-management/` - 프로세스(ps, top), 시그널·kill, 포그라운드/백그라운드 잡, nice·우선순위
- `04-text-processing/` - 파이프·리다이렉션, grep·sed·awk, sort·uniq·cut·wc
- `05-users-groups/` - 사용자·그룹 관리(useradd, usermod), sudo, 계정 파일(/etc/passwd·shadow)
- `06-package-management/` - apt·dnf/yum, 저장소·의존성, 소스 빌드, 스냅/컨테이너 배포
- `07-systemd-service/` - systemd 유닛, 서비스 관리(systemctl), 저널(journalctl), 타이머·부팅 타깃
- `08-networking/` - IP·라우팅(ip, ss), 방화벽(ufw, firewalld, iptables), DNS, SSH
- `09-shell-scripting/` - Bash 스크립트 (변수·조건·반복·함수), 안전 옵션, cron 자동화
- `10-monitoring-performance/` - 로그, 성능 지표(top·vmstat·iostat), 디스크(df·du), 트러블슈팅 절차

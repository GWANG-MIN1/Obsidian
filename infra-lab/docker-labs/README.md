
# Docker Labs

Docker 학습 실습 및 명령어 정리 저장소

## 구조
- `commands.md` - 자주 쓰는 Docker 명령어 레퍼런스
- `01-container-basics/` - 컨테이너 생성, 목록, 삭제
- `02-networking/` - 포트 노출, 외부 연결
- `03-volumes/` - 볼륨, 호스트 볼륨, 볼륨 컨테이너
- `04-network/` - 네트워크 드라이버 (bridge, host, none, container, macvlan), --net-alias
- `05-logging/` - 로그 드라이버 (json-file, syslog, fluentd, awslogs)
- `06-resource-limit/` - 메모리 제한 (--memory), CPU 제한 (--cpu-shares, --cpuset-cpus, --cpu-period, --cpu-quota, --cpus)
- `07-image/` - 이미지 생성 (Dockerfile, docker build), 레이어 구조 이해, 이미지 추출 (save/load/export/import), 레지스트리 배포
- `08-compose/` - Docker Compose (멀티 컨테이너 정의, depends_on, healthcheck, restart policy, 스케일링, 환경 분리)
- `09-security/` - 컨테이너 보안 (namespace/capabilities/seccomp, 비-root 실행, 읽기 전용 FS, 시크릿 관리, 이미지 스캔, rootless)
- `10-orchestration/` - 오케스트레이션 개념 (선언형·셀프힐링·스케일링), Docker Swarm (service/task/stack), Kubernetes로 가는 다리
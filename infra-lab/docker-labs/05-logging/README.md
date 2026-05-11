# 05 컨테이너 로깅

컨테이너에서 발생하는 로그를 수집하고 관리하는 다양한 방법을 정리한다.

---

## json-file 로그 사용하기

Docker의 기본 로그 드라이버. 컨테이너의 stdout/stderr를 JSON 형식으로 호스트 파일시스템에 저장한다.

```bash
# 기본 사용 (별도 설정 없이 json-file이 기본값)
docker run -d --name myapp nginx

# 로그 확인
docker logs myapp

# 실시간 로그 스트리밍
docker logs -f myapp

# 최근 N줄만 확인
docker logs --tail 100 myapp

# 타임스탬프 포함
docker logs -t myapp
```

로그 파일 크기 및 개수 제한 옵션:

```bash
docker run -d \
  --name myapp \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
```

| 옵션 | 설명 |
|------|------|
| `max-size` | 로그 파일 최대 크기 (예: 10m, 1g) |
| `max-file` | 보관할 로그 파일 최대 개수 (롤링) |

> 로그 파일 저장 경로: `/var/lib/docker/containers/<컨테이너ID>/<컨테이너ID>-json.log`

---

## syslog 로그

컨테이너 로그를 시스템의 syslog 데몬(rsyslog, syslog-ng 등)으로 전달한다.  
호스트 또는 원격 syslog 서버에 중앙 집중식으로 수집할 때 사용한다.

```bash
# 호스트 syslog로 전송
docker run -d \
  --name myapp \
  --log-driver syslog \
  nginx

# 원격 syslog 서버로 전송
docker run -d \
  --name myapp \
  --log-driver syslog \
  --log-opt syslog-address=tcp://192.168.1.100:514 \
  --log-opt syslog-facility=daemon \
  --log-opt tag="myapp" \
  nginx
```

| 옵션 | 설명 |
|------|------|
| `syslog-address` | syslog 서버 주소 (tcp/udp/unix 지원) |
| `syslog-facility` | syslog facility (daemon, local0~7 등) |
| `tag` | 로그에 붙일 태그 |

> syslog 드라이버 사용 시 `docker logs` 명령어로 로그를 조회할 수 없다. syslog 서버에서 직접 확인해야 한다.

---

## fluentd 로깅

Fluentd를 로그 수집 에이전트로 사용하는 드라이버.  
EFK(Elasticsearch + Fluentd + Kibana) 스택 구성 시 주로 활용된다.

```bash
# fluentd 에이전트가 로컬에서 실행 중인 경우
docker run -d \
  --name myapp \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.myapp" \
  nginx

# 원격 fluentd 서버로 전송
docker run -d \
  --name myapp \
  --log-driver fluentd \
  --log-opt fluentd-address=tcp://192.168.1.100:24224 \
  --log-opt tag="docker.{{.Name}}" \
  --log-opt fluentd-async=true \
  nginx
```

| 옵션 | 설명 |
|------|------|
| `fluentd-address` | fluentd 수신 주소 및 포트 |
| `tag` | 로그 태그 (컨테이너명 등 변수 사용 가능) |
| `fluentd-async` | 비동기 전송 (true 시 컨테이너 시작 지연 방지) |

> fluentd 드라이버도 `docker logs` 로 조회 불가. Fluentd → Elasticsearch 등 파이프라인에서 확인.

---

## AWS CloudWatch 로그

컨테이너 로그를 AWS CloudWatch Logs로 직접 전송한다.  
EC2, ECS 환경에서 AWS 기반 중앙 로그 관리에 활용된다.

```bash
docker run -d \
  --name myapp \
  --log-driver awslogs \
  --log-opt awslogs-region=ap-northeast-2 \
  --log-opt awslogs-group=/docker/myapp \
  --log-opt awslogs-stream=myapp-container \
  --log-opt awslogs-create-group=true \
  nginx
```

| 옵션 | 설명 |
|------|------|
| `awslogs-region` | AWS 리전 |
| `awslogs-group` | CloudWatch 로그 그룹 이름 |
| `awslogs-stream` | 로그 스트림 이름 |
| `awslogs-create-group` | 로그 그룹 자동 생성 여부 |

IAM 권한 필요:

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogStreams"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```

> EC2 인스턴스 프로파일 또는 환경변수(`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)로 자격증명을 제공해야 한다.

---

## 로그 드라이버 비교

| 드라이버 | `docker logs` 사용 | 특징 |
|----------|-------------------|------|
| json-file | O | 기본값, 로컬 파일 저장 |
| syslog | X | 시스템 syslog 통합 |
| fluentd | X | EFK 스택 연동 |
| awslogs | X | AWS CloudWatch 연동 |

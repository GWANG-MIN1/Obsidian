# Docker 명령어 레퍼런스

## 컨테이너 생성 & 실행
```
docker run -i -t --name 컨테이너명 ubuntu:24.04   # 대화형 실행
docker run -d --name 컨테이너명 이미지명            # 백그라운드 실행
docker start -ai 컨테이너명                        # 중지된 컨테이너 재시작
```
## 컨테이너 목록 & 상태
```
docker ps                                          # 실행 중인 컨테이너
docker ps -a                                       # 전체 컨테이너 (중지 포함)
docker port 컨테이너명                              # 포트 매핑 확인
```

## 컨테이너 삭제
```
docker rm 컨테이너명 
```

## 포트 노출
```
docker run -d -p 호스트포트: 컨테이너포트 이미지명
```
## 볼륨
```
docker run -v 볼륨명 : /경로 이미지명                # 볼륨 마운트
docker run -v /호스트경로 : /컨테이너경로 이미지명    # 호스트 볼륨 공유
docker run --volumes-from 컨테이너명 이미지명        # 볼륨 컨테이너
docker volume create 볼륨명                         # 도커 볼륨 생성
```
## 네트워크
```
docker network ls                                    # 네트워크 목록
docker network inspect 네트워크명                    # 네트워크 상세 정보
docker network create --driver bridge 네트워크명     # 사용자 정의 브리지 생성
docker network connect 네트워크명 컨테이너명          # 컨테이너를 네트워크에 연결
docker network disconnect 네트워크명 컨테이너명       # 컨테이너를 네트워크에서 분리
docker network rm 네트워크명                         # 네트워크 삭제
docker network prune                                 # 미사용 네트워크 일괄 삭제

docker run --network host 이미지명                   # 호스트 네트워크 사용
docker run --network none 이미지명                   # 네트워크 완전 격리
docker run --network container:컨테이너명 이미지명   # 다른 컨테이너 네트워크 공유
docker run --network 네트워크명 --net-alias 별칭 이미지명  # DNS alias 지정
```
## 로깅
```
docker logs 컨테이너명                               # 로그 확인
docker logs -f 컨테이너명                            # 실시간 로그 스트리밍
docker logs --tail 100 컨테이너명                    # 최근 100줄
docker logs -t 컨테이너명                            # 타임스탬프 포함

docker run -d --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 이미지명                      # json-file 크기/개수 제한

docker run -d --log-driver syslog \
  --log-opt syslog-address=tcp://192.168.1.100:514 \
  --log-opt tag="태그명" 이미지명                    # syslog 원격 전송

docker run -d --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.컨테이너명" 이미지명         # fluentd 전송

docker run -d --log-driver awslogs \
  --log-opt awslogs-region=ap-northeast-2 \
  --log-opt awslogs-group=/docker/myapp \
  --log-opt awslogs-stream=컨테이너명 이미지명       # AWS CloudWatch 전송
```

## 자원 할당 제한
```
docker run -d --memory 512m 이미지명                 # 최대 메모리 512MB
docker run -d --memory 512m --memory-swap 1g 이미지명  # 메모리+스왑 총합 제한

docker run -d --cpu-shares 512 이미지명              # CPU 상대적 가중치 (기본 1024)
docker run -d --cpuset-cpus="0,1" 이미지명           # 0,1번 코어 고정
docker run -d --cpuset-cpus="0-3" 이미지명           # 0~3번 코어 범위 고정
docker run -d --cpu-period=100000 --cpu-quota=50000 이미지명  # CFS 직접 제어 (50%)
docker run -d --cpus="1.5" 이미지명                  # CPU 1.5코어 제한 (권장)

docker run -d --memory 1g --memory-swap 1g \
  --cpus="2" --cpuset-cpus="0,1" 이미지명            # 복합 제한
```

## 이미지
```
docker pull 이미지명                                  # 태그 생략시 :latest 자동
docker images                                        # 이미지 목록
```

## 컨테이너 탈출
```
exit                                               # 컨테이너 종료 후 나오기
ctrl+p, q                                          # 컨테이너 유지하고 나오기
```

## 이미지 빌드
```
docker build -t 이미지명:태그 .                        # 현재 디렉터리 Dockerfile로 빌드
docker build -f /경로/Dockerfile -t 이미지명:태그 .    # Dockerfile 경로 직접 지정
docker build --build-arg KEY=VALUE -t 이미지명:태그 .  # 빌드 인자 전달
docker build --no-cache -t 이미지명:태그 .             # 캐시 무시
```

## 이미지 태그 & 관리
```
docker tag 원본이미지:태그 새이미지:태그               # 태그 추가
docker rmi 이미지명:태그                               # 이미지 삭제
docker image prune                                     # 댕글링 이미지 삭제
docker image prune -a                                  # 미사용 이미지 전체 삭제
docker history 이미지명:태그                           # 레이어 히스토리 확인
docker history --no-trunc 이미지명:태그                # 명령어 전체 출력
docker system df                                       # 이미지/컨테이너 용량 확인
docker system df -v                                    # 이미지별 상세 용량
docker images -f dangling=true                         # 댕글링 이미지 목록
docker images --digests                                # Digest 포함 이미지 목록
```

## 이미지 저장 & 로드 (폐쇄망/백업)
```
docker save -o myimage.tar 이미지명:태그               # 이미지 → tar 저장
docker save 이미지명:태그 | gzip > myimage.tar.gz      # gzip 압축 저장
docker load -i myimage.tar                             # tar → 이미지 로드
docker export 컨테이너명 -o container.tar              # 컨테이너 파일시스템 추출
docker import container.tar 이미지명:태그              # tar → 단일레이어 이미지
docker commit 컨테이너명 이미지명:태그                 # 컨테이너 현재 상태를 이미지로
```

## 이미지 배포 (레지스트리)
```
docker login                                           # Docker Hub 로그인
docker logout                                          # 로그아웃
docker push username/이미지명:태그                     # 이미지 푸시
docker pull username/이미지명:태그                     # 이미지 풀
docker search nginx                                    # Docker Hub 검색

# 프라이빗 레지스트리
docker run -d -p 5000:5000 -v registry-data:/var/lib/registry registry:2
docker tag 이미지명:태그 localhost:5000/이미지명:태그
docker push localhost:5000/이미지명:태그
```

## Docker Compose
```
docker compose up -d                                 # 백그라운드 실행
docker compose up -d --build                         # 이미지 재빌드 후 실행
docker compose down                                  # 컨테이너+네트워크 제거
docker compose down -v                               # 볼륨까지 제거
docker compose ps                                    # 서비스 상태 확인
docker compose logs -f 서비스명                       # 특정 서비스 로그 스트리밍
docker compose exec 서비스명 bash                     # 실행 중 서비스 접속
docker compose build                                 # 이미지만 빌드
docker compose restart 서비스명                       # 특정 서비스 재시작
docker compose config                                # 최종 병합 설정 검증
docker compose up -d --scale web=3                   # 서비스 3개로 스케일
docker compose -f compose.yaml -f compose.prod.yaml up -d  # 환경별 파일 병합
```

## 재시작 정책 & 상태 점검
```
docker run -d --restart unless-stopped 이미지명       # 수동 중지 제외 항상 재시작
docker run -d --restart on-failure:5 이미지명         # 비정상 종료 시 최대 5회 재시작
docker inspect --format='{{.State.Health.Status}}' 컨테이너명  # 헬스 상태 확인
```

## 컨테이너 보안
```
docker run --user 1000:1000 이미지명                  # 비-root 사용자로 실행
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE 이미지명  # capability 최소화
docker run --security-opt no-new-privileges 이미지명  # 권한 상승 차단
docker run --read-only --tmpfs /tmp 이미지명          # 읽기 전용 파일시스템
docker pull nginx@sha256:다이제스트                   # digest 고정 (불변 보장)
docker scout cves 이미지명:태그                       # 이미지 취약점 스캔 (Scout)
trivy image 이미지명:태그                             # 이미지 취약점 스캔 (Trivy)
docker build --secret id=mysecret,src=./secret.txt -t 이미지명 .  # 빌드 시크릿 주입
```

## 오케스트레이션 (Docker Swarm)
```
docker swarm init --advertise-addr <매니저IP>        # 스웜 매니저 초기화
docker swarm join-token worker                       # 워커 조인 토큰 확인
docker swarm join --token <토큰> <매니저IP>:2377      # 워커 노드 참여
docker node ls                                       # 노드 목록

docker service create --name web --replicas 3 -p 80:80 nginx  # 서비스 생성
docker service ls                                    # 서비스 목록
docker service ps web                                # 태스크 배치 상태
docker service scale web=5                           # 5개로 확장
docker service update --image nginx:1.27 web         # 롤링 업데이트
docker service rm web                                # 서비스 제거

docker stack deploy -c compose.yaml myapp            # 스택 배포
docker stack services myapp                          # 스택 서비스 확인
docker stack rm myapp                                # 스택 제거
```
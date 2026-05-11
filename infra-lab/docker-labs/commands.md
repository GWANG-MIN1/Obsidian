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
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
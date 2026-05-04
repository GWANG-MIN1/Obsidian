# 01. 컨테이너 기초

## 컨테이너 생성
```
docker run -i -t --name mycontainer ubuntu:24.04
```
## 컨테이너 목록 확인
```
docker ps        # 실행 중
docker ps -a     # 전체 (중지 포함)
```
## 컨테이너 삭제
```
docker rm 컨테이너명
```
## 배운 점
- 컨테이너 이름 중복 불가 (이미 있으면 Conflict 에러)
- exit → 컨테이너 종료
- Ctrl+P, Q → 컨테이너 살려두고 나오기
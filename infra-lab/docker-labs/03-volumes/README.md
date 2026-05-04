# 03. 볼륨

## 도커 볼륨 생성
```
docker volume create 볼륨명
```
## 볼륨 마운트
```
docker run -v 볼륨명:/경로 이미지명
```
## 호스트 볼륨 공유
```
docker run -v /호스트경로:/컨테이너경로 이미지명
```
## 볼륨 컨테이너 (--volumes-from)
```
docker run --volumes-from 컨테이너명 이미지명
```
## -v vs --volumes-from
| 옵션 | 설명 |
|---|---|
| -v | 내가 직접 볼륨 경로 지정 |
| --volumes-from | 다른 컨테이너 볼륨 그대로 따라 씀 |

## 배운 점
- -v 랑 --volume 은 완전히 동일 (단축형)
- 볼륨은 컨테이너 삭제해도 데이터 유지됨
- 태그 생략하면 자동으로 :latest 적용
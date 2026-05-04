# 02. 네트워킹 & 포트 노출

## 포트 매핑
```
docker run -d -p 호스트포트:컨테이너포트 이미지명
```
## 포트 확인
```
docker port 컨테이너명
```
## 실습 예시 - WordPress + MySQL
```
docker run -d --name wordpressdb \-e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=wordpress \ mysql:5.7

docker run -d --name wordpress \ -e WORDPRESS_DB_HOST=mysql \
-e WORDPRESS_DB_USER=root \ wordpress
```

## 배운 점
- 포트 확인시 IPv4(0.0.0.0), IPv6([::]) 두 개 나오는 건 정상
- -e 옵션으로 환경변수 전달
- = 앞뒤 띄어쓰기 하면 에러 남
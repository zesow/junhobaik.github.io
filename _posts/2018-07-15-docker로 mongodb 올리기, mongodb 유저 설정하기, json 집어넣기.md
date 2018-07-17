---
tags:
  - docker
  - mongodb
  - shell script
---





# docker 설정

### docker run 하면서 mongo admin 계정 생성하기 & 포트 설정해주기 & 쉘 가능하게 & detach 가능하게 하기

`docker run -dit -p 외부포트:내부포트 --name mongodb -e MONGO_INITDB_ROOT_USERNAME=id -e MONGO_INITDB_ROOT_PASSWORD=pw mongo`

## mongo 깔린 docker 쉘 접속과 동시에 mongo로그인

`docker exec -it mongodb mongo -u id -p pw --authenticationDatabase admin`



# mongo 설정

### 외부에서 접속

`mongo ip:포트번호 -u id -p pw --authenticationDatabase admin`



### 주의. 몽고db 쉘 버전이 2.6 이면 'auth mechanism ~'에러와 함께 접속이 안 됨. 업그레이드 시켜줘야 함.

https://askubuntu.com/questions/958583/how-to-upgrade-mongodb-from-2-6-to-3-4-on-ubuntu-16-04



### 각자 접속

`mongo ip:포트번호/db명 -u id -p pw`



## 몽고db 간단한 사용법

### 유저 목록 확인

`cur = db.system.users.find()`

### 유저 생성

`

db.createUser({

​	user: "id",

​	pwd: "pw",

​	roles: [{role: "read",db: "db명"}],

​	passwordDigestor: "server"

})

`



# mongodb 에 json 집어넣기

### jsonImport

`mongoimport -h ip:포트번호 -d db명 -c collection명 -u id -p pw --file 파일명 --jsonArray --authenticationDatabase admin`



## 다량의 json import 위한 쉘스크립트 작성

`

#!/bin/bash

url="파일루트"

for ((i=1;i<=56;i++));do
    echo mongoimport -h ip:port -d db -c collection -u id -p pw --file $url$i.json --jsonArray --authenticationDatabase admin
done

`




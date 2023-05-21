+++
title="docker run"
date=2023-05-20

[taxonomies]
categories = ["Docker"]
tags = ["command","sqlserver","mongo","postgres"]
+++

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
이미지 pull하고, 새 container 만들고, 실행까지 한큐에

자주 사용하는 옵션들
--name
-d detach
-p publish
-v volumn
-e environment variable
-i interactive
-t tty
-it interacive + tty
--link



예제
```
docker run -d `
--name mysqlserver `        # 컨테이너 이름은 mysqlserver
-e 'ACCEPT_EULA=Y' `        # sqlserver의 환경변수, docker의 환경변수 아님
-e 'SA_PASSWORD=Pa55w@rd' ` # sa 의 비밀번호
-e 'MSSQL_PID=Express' `	#이것도 sqlserver의 환경변수, MSSQL_PID 의 기본 값은 'Developer'
-p 1433:1433 `              #서버의 1433포트를 컨테이너의 1433 로 연결. 호스트포트:컨테이너포트
-v c:\Users\Doosan\sqlserver\data:/var/opt/mssql/data ` #호스트 파일디렉토리를 컨테이너에 마운트
-v c:\Users\Doosan\sqlserver\log:/var/opt/mssql/log `
-v c:\Users\Doosan\sqlserver\secrets:/var/opt/mssql/secrets `
mcr.microsoft.com/mssql/server:2019-latest  #사용할 이미지
```
매 줄마다 space`를 마지막에 넣어주는데, 이건 한줄에 긴 command를 다 쓰기 힘들때 줄바꿈으로 보기 편하게 해주는 것뿐이다. 


```
docker run -d `
--name mymongo `
-p 27017:27017 `
-e MONGO_INITDB_ROOT_USERNAME=mongoadmin `
-e MONGO_INITDB_ROOT_PASSWORD=secret `
-v c:\Users\Doosan\mongo:/data/db `
```

```
docker run -d `
--name mypostgres `
-p 5432:5432 `
-e POSTGRES_USER=doosan ` #생략하면 postgres로 기본유저생성
-e POSTGRES_PASSWORD=postgres `
-e PGDATA=/var/lib/postgresql/data/pgdata `
-e tz=Asia/Seoul `
-v c:\Users\Doosan\postgres:/var/lib/postgresql/data `
postgres
```
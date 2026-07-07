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
이미지를 pull하고, 새 container를 만들고, 실행까지 한 번에 처리한다.

자주 사용하는 옵션은 다음과 같다.

| Option      |             |
| :---        |    :----:   |
| --name | container 이름 |
| -d | detached mode로 백그라운드 실행 |
| -p | port publish |
| -v | volume mount |
| -e | environment variable |
| -i | interactive mode |
| -t | tty 할당 |
| -it | interactive + tty |
| --link | 다른 container와 연결 |



예제:
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
mcr.microsoft.com/mssql/server:2019-latest  # 사용할 이미지
```
각 줄 끝에 backtick을 붙인 것은 긴 command를 여러 줄로 나누어 읽기 쉽게 만들기 위해서다. Docker 옵션의 일부가 아니라 PowerShell의 줄 이어쓰기 문법이다.


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

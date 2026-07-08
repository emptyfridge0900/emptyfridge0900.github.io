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
Pulls the image, creates a new container, and starts it in one command.

Commonly used options:

| Option      |             |
| :---        |    :----:   |
| --name     | name         |
| -d   | detach             |
| -p | publish              |
| -v | volumn               |
| -e | environment variable |
| -i | interactive          |
| -t | tty                  |
| it | interactive + tty    |
|--link| link to other container |



Example:
```
docker run -d `
--name mysqlserver `        # container name is mysqlserver
-e 'ACCEPT_EULA=Y' `        # SQL Server environment variable, not a Docker variable
-e 'SA_PASSWORD=Pa55w@rd' ` # SA password
-e 'MSSQL_PID=Express' `	# also a SQL Server environment variable; default MSSQL_PID is 'Developer'
-p 1433:1433 `              # map host port 1433 to container port 1433. host:container
-v c:\Users\Doosan\sqlserver\data:/var/opt/mssql/data ` # mount host directory into the container
-v c:\Users\Doosan\sqlserver\log:/var/opt/mssql/log `
-v c:\Users\Doosan\sqlserver\secrets:/var/opt/mssql/secrets `
mcr.microsoft.com/mssql/server:2019-latest  # image to use
```
Each line ends with a space and a backtick. This is just PowerShell line continuation so a long command can be written across multiple readable lines.


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
-e POSTGRES_USER=doosan ` # if omitted, the default user is postgres
-e POSTGRES_PASSWORD=postgres `
-e PGDATA=/var/lib/postgresql/data/pgdata `
-e tz=Asia/Seoul `
-v c:\Users\Doosan\postgres:/var/lib/postgresql/data `
postgres
```

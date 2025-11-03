---
title: Docker에서 MySQL `LOAD DATA LOCAL INFILE` 에러 해결하기
author: jisu
date: 2025-11-03 10:00:00 +0900
categories: [Data Engineering]
tags: [MySQL, Docker]
pin: false
math: false
mermaid: false
comments: true
---

## `LOAD DATA LOCAL INFILE`이란?

MySQL에서는 로컬 파일 시스템에 있는 CSV나 TSV 데이터를 테이블로 불러올 때 
`LOAD DATA LOCAL INFILE` 명령을 쓴다.

예시:

```sql
LOAD DATA LOCAL INFILE '/path/to/data.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, name, age);
```

이 명령을 사용하면 CSV 데이터를 빠르게 적재할 수 있다.
다만 보안상의 이유로 기본 설정에서는 비활성화되어 있어 에러가 발생할 수 있다.


## 문제

ETL 파이프라인을 Docker 환경에서 구성하면서 CSV 데이터를 MySQL에 적재하는 작업을 하던 중, 
`load_data.sql` 실행 시 아래와 같은 에러가 발생했다.

```
ERROR 3948 (42000): Loading local data is disabled; this must be enabled on both the client and server sides
```

## 원인
MySQL은 기본적으로 **`LOCAL INFILE` 기능을 비활성화**한다.  
이 옵션이 꺼져 있으면 `LOAD DATA LOCAL INFILE` 구문이 실행되지 않는다.  

즉 아래 두 설정이 모두 ON이어야 정상 작동한다.

- **클라이언트(`--local-infile=1`)** → 로컬 CSV 읽기 허용  
- **서버(`local_infile=1`)** → 서버 측 CSV 로드 허용 

---

## 해결 방법

### 1️⃣ MySQL 서버에서 `local_infile` 활성화  

Docker 환경에서는 `docker-compose.yml`의 `mysql` 서비스 설정에  
아래 옵션을 추가해 서버 측에서 `local_infile` 기능을 활성화할 수 있다.

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=mydb
    command: --local-infile=1
```

이 설정을 추가하면 컨테이너가 시작될 때 자동으로 `local_infile=1`이 적용된다. 

> 💡 참고
이미 실행 중인 컨테이너에서는 MySQL 내부에서 아래 명령으로도 임시 활성화할 수 있다.

```sql
SET GLOBAL local_infile=1;
```

이 방법은 컨테이너를 재시작하면 초기화되지만, 즉시 적용 가능하다.

> https://stackoverflow.com/questions/62616706/enable-local-infile-in-mysql-on-docker

### 2️⃣ 컨테이너 재시작

```bash
docker compose down
docker compose up -d --build
```

### 3️⃣서버 설정 확인

MySQL 서버 내부에서 설정이 적용되었는지 확인한다.

```bash
docker exec -it mysql_container_name \
  mysql -u root -proot -e "SHOW VARIABLES LIKE 'local_infile';"
```

출력 결과가 아래와 같이 ON이면 성공이다.

```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| local_infile  | ON    |
+---------------+-------+
```

### 4️⃣ CSV 데이터 적재 (클라이언트 설정)

`LOAD DATA LOCAL INFILE`을 실행할 때 클라이언트 측 옵션도 반드시 활성화해야 한다.

- `--local-infile=1` : 클라이언트 측에서 로컬 파일 읽기 허용

```bash
docker exec -i mysql_container_name \
  mysql --local-infile=1 -u root -proot mydb < ./mysql/load_data.sql
```

## 정리 
`LOAD DATA LOCAL INFILE`이 정상 동작하려면 서버, 클라이언트, SQL 명령 세 부분이 모두 올바르게 설정되어야 한다.

먼저 MySQL 서버에서는 `local_infile=1` 옵션을 활성화해야 한다.
이 설정은 Docker 환경이라면 `command: --local-infile=1` 옵션을 통해 적용하거나,
직접 `my.cnf` 파일에 추가해서 영구적으로 유지할 수 있다.
이렇게 하면 MySQL 서버가 CSV 파일 로드를 허용하게 된다.

다음으로 클라이언트 측에서도 `--local-infile=1` 옵션을 지정해야 한다.
이는 MySQL에 접속하는 스크립트에서 로컬 파일을 읽을 수 있도록 허용하는 설정이다.

마지막으로, 실제 데이터를 불러올 때는 `LOAD DATA LOCAL INFILE` 명령을 사용한다.
이 명령을 통해 CSV나 TSV 같은 텍스트 파일을 빠르게 테이블로 적재할 수 있다.

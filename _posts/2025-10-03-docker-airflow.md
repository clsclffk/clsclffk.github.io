---
title: Airflow docker.sock permission denied 오류 해결 방법
author: jisu
date: 2025-10-03 21:00:00 +0900
categories: [Data Engineering, DevOps]
tags: [Airflow, Docker, Spark]
pin: false
math: false
mermaid: false
comments: true
---

## 배경
Airflow를 이용해 데이터 파이프라인을 운영하다 보면,  
Spark 작업을 DAG에서 실행해야 할 때가 많다.  

보통은 Airflow DAG에서 `BashOperator`로 `docker exec`을 사용해  
Spark 컨테이너에 명령을 전달하는 방식을 많이 쓴다.  

그런데 이 과정에서 `docker.sock` 권한 문제 때문에  
DAG 실행이 실패하는 상황을 겪었다.  
이번 글에서는 그 문제와 해결 방법을 정리한다.  

## 문제 상황
Airflow DAG에서 Spark를 실행하려고 할 때  
다음과 같은 에러가 발생했다:

```
permission denied while trying to connect to the Docker daemon socket
Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.45/containers/json": dial unix /var/run/docker.sock: connect: permission denied
```


즉, Airflow 컨테이너가 `docker.sock`을 통해  
호스트의 Docker 데몬에 접근하려 했지만 권한이 없어 실패한 것이다.  

## 원인
문제의 핵심은 **`docker.sock`**이다.  

`docker.sock`은 Docker 데몬이 명령을 받을 때 사용하는 유닉스 소켓 파일로,  
Docker CLI가 실행하는 대부분의 명령(`docker ps`, `docker exec`, `docker run` 등)은  
이 소켓을 통해 도커 엔진과 통신한다.  

StackOverflow에서도 이렇게 설명한다:

> *"docker.sock is the UNIX socket that Docker daemon is listening to.  
> It's the main entry point for Docker API."*  
> [출처: What is the purpose of the file docker.sock? (StackOverflow)](https://stackoverflow.com/questions/35110146/what-is-the-purpose-of-the-file-docker-sock)

즉, `docker.sock`은 **도커 API로 들어가는 엔트리포인트**이라고 볼 수 있다.  
기본적으로 `/var/run/docker.sock`의 소유자는 `root:docker`이기 때문에  
root 계정이나 `docker` 그룹에 속한 유저만 접근할 수 있다.  

문제는 Airflow 컨테이너의 기본 유저(`airflow`, uid=50000)가  
이 `docker` 그룹에 포함되어 있지 않다는 점이다.  

그래서 Airflow DAG에서 `docker exec` 같은 명령을 실행하려 하면,  
접근 거부(Permission Denied)가 발생하는 것이다.

## 해결 방법
이 문제를 해결하려면,  
컨테이너 안에서도 **호스트의 docker 그룹과 동일한 GID**를 만들어주고,  
Airflow 유저를 그 그룹에 포함시켜야 한다.  

### 1) 호스트의 docker 그룹 GID 확인
```bash
getent group docker
```

출력 예시:

```
docker:x:113:ubuntu
```

→ 호스트에서 docker 그룹의 GID는 113.

### 2) Airflow Dockerfile 수정

Airflow 컨테이너 빌드 시 아래 내용을 추가한다.

```
FROM apache/airflow:2.9.0

# 필요 라이브러리 설치
COPY requirements.txt /
RUN pip install --no-cache-dir -r /requirements.txt \
    --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.9.0/constraints-3.12.txt"

# root 권한으로 전환
USER root

# 호스트 docker 그룹(GID=113)과 동일한 그룹 생성 후 airflow 유저 추가
RUN groupadd -g 113 docker && usermod -aG docker airflow

# 다시 airflow 유저로 전환
USER airflow

```

### 3) 컨테이너 재빌드 및 실행

```
docker compose build airflow-webserver airflow-scheduler
docker compose up -d
```

### 4) 권한 확인

```
docker exec -it airflow-webserver id
```

정상 출력 예시:

```
uid=50000(airflow) gid=0(root) groups=0(root),113(docker)
```

-> 113(docker) 그룹이 보이면 해결 완료

## 결과

이제 Airflow DAG에서 docker exec으로 Spark를 실행해도
권한 문제 없이 정상 동작한다.

정리하자면,

- 문제: Airflow 컨테이너 유저가 docker.sock 접근 권한이 없어 permission denied 발생

- 원인: /var/run/docker.sock은 root:docker 소유, Airflow 유저는 docker 그룹 미포함

- 해결: 호스트 docker 그룹 GID 확인 → 동일 GID 그룹 생성 → airflow 유저 추가 → 재빌드

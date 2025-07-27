---
title: Docker로 Django + PostgreSQL 개발환경 구축하기
author: jisu
date: 2025-07-27 12:00:00 +0900
categories: [Devops]
tags: [Docker, EC2, Django, PostgreSQL]
pin: false
math: false
mermaid: false
comments: true
---

## Docker의 필요성

처음에는 Django로 웹 서비스를 개발하면서 EC2에 직접 올려두고 테스트하는 식으로 작업을 진행하고 있었다.
Python 가상환경(venv)에서 runserver로 개발 서버를 띄우고, PostgreSQL도 EC2 내부에 직접 설치해 사용하고 있었다.

하지만 ETL 파이프라인을 다시 구성하게 되면서 상황이 달라졌다.
웹에서 크롤링할 데이터 양이 많아지다 보니 단순한 bash + crontab 방식으로는 관리가 어렵고, 흐름도 직관적으로 보이지 않았다.

그래서 Airflow로의 전환을 결심했고, 이왕 Docker로 Airflow를 구성하는 김에
**Django와 ORM(PostgreSQL)**까지 함께 컨테이너화해서 구조를 일관되게 만들기로 했다.

단순히 기존 개발 방식이 불편해서가 아니라,

- ETL부터 웹까지 전체 흐름을 Airflow 중심으로 통합하고

- 향후 운영 및 배포(Nginx, Gunicorn 등) 를 염두에 둔 구조 설계를 위해

초기 단계부터 컨테이너 기반의 개발 환경을 구성하는 게 맞겠다고 판단했다.

## 1. `requirements.txt` 생성
이미 가상환경(`venv`)에 Django 개발에 필요한 패키지들이 설치되어 있었기 때문에,
현재 가상환경에서 설치된 패키지를 기준으로 `requirements.txt` 파일을 만들어야 한다.

```bash
pip freeze > requirements.txt
```
이 파일은 나중에 Dockerfile에서 `pip install -r requirements.txt` 명령어로 사용된다.
즉, Docker 컨테이너에서 패키지를 자동으로 설치하기 위한 기반이 된다.

## 2. `Dockerfile.backend` 생성
Django 앱을 실행하기 위한 Docker 이미지를 정의하는 단계다.
보통 Python 베이스 이미지를 바탕으로, 필요한 패키지를 설치하고 코드 디렉토리를 복사하는 방식으로 작성한다.

```
# Dockerfile.backend
FROM python:3.10-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# 필요한 패키지 설치
RUN apt-get update && apt-get install -y gcc libpq-dev curl

WORKDIR /app

COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ /app/

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
이렇게 하면 Django 프로젝트가 `/app` 디렉토리에 복사되고, 컨테이너 실행 시 바로 `runserver`가 실행되도록 구성된다.

## 3. `docker-compose.yaml` 생성
Django와 PostgreSQL을 각각의 컨테이너로 실행하고, 이 둘이 서로 네트워크로 연결되도록 설정해야 한다.
이를 위해 `docker-compose.yaml`을 작성한다.

```yaml
version: "3.8"

services:
  db:
    image: postgres:14
    restart: always
    environment:
      POSTGRES_DB: pop_db
      POSTGRES_USER: pop_user
      POSTGRES_PASSWORD: pop_pw
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  django:
    build:
      context: .
      dockerfile: Dockerfile.backend
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000 
    volumes:
      - ./backend:/app
      - ./backend/templates:/app/templates
      - ./backend/staticfiles:/app/staticfiles
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      - DB_NAME=pop_db
      - DB_USER=pop_user
      - DB_PASSWORD=pop_pw
      - DB_HOST=db
      - DB_PORT=5432

volumes:
  pgdata:
```

## 4. docker compose up --build 실행
이제 위의 설정이 끝났다면, 아래 명령어로 Docker 이미지를 빌드하고 컨테이너를 실행할 수 있다:

```bash
docker compose up --build
```

## 🚨 오류: `relation "django_session" does not exist`
Docker로 Django + PostgreSQL 환경을 띄우고 웹 브라우저에서 접속했더니,
아래와 같은 오류가 발생했다.:

```
ProgrammingError at /
relation "django_session" does not exist
LINE 1: ...ession_data", "django_session"."expire_date" FROM "django_se...
```

이 오류 메시지는 Django가 내부적으로 사용하는 세션 테이블(`django_session`)에 접근하려 했지만,
해당 테이블이 아직 데이터베이스에 생성되지 않았기 때문에 발생하는 오류다.

아직 Django의 ORM이 아직 migrate 명령어를 통해 테이블 구조를 생성하지 않았기 때문에 발생한다.


## 💡 해결: Django 테이블 구조 초기화

일단 PostgreSQL이 실행 중이어야 Django가 DB에 접근할 수 있으므로 
컨테이너를 먼저 백그라운드에 띄워준다:

```bash
docker compose up -d
```

그다음, Django 컨테이너 안에 진입해서 migrate 명령어를 실행해 테이블 구조를 초기화한다:

```bash
docker compose exec django bash
python manage.py migrate
```

이때 migrate 명령은 Django의 기본 앱들(auth, admin, sessions 등)을 위한 테이블을 포함해,
직접 만든 앱의 모델에 해당하는 테이블까지 전부 생성해 준다.

마이그레이션이 완료되면 `django_session`, `auth_user`, `django_migrations` 등 핵심 테이블이 생성되며,
앞서 발생했던 오류는 더 이상 나타나지 않게 된다.

### ⭐ migrate는 언제 해야 하나?

migrate는 단순히 `models.py`를 수정했을 때만 필요한 게 아니다.
다음과 같이 Docker 환경에서 PostgreSQL이 초기 상태로 시작되는 경우에도 반드시 실행해줘야 한다:

1. Django를 Docker로 처음 올렸을 때

- 컨테이너가 처음 생성되었고, PostgreSQL도 처음 실행된 경우

- DB에는 아무 테이블도 없는 빈 상태이므로 migrate가 필요하다.

2. PostgreSQL 컨테이너를 삭제하거나 재생성했을 때

- 볼륨까지 삭제한 경우에는 데이터뿐 아니라 테이블 구조까지 모두 사라진다.

- 따라서 migrate로 다시 테이블을 생성해줘야 한다.

즉, PostgreSQL이 새로 시작되거나 초기화된 시점에는
migrate로 Django가 사용할 테이블 구조를 반드시 만들어줘야 한다.

## 🚨 문제: Docker 환경에서 Django 정적 파일(css/js) 깨짐
Docker로 Django 컨테이너를 실행하고 웹페이지에 접속했을 때, CSS가 적용되지 않고 스타일이 깨지는 현상이 발생했다.
Django는 개발용 서버(`runserver`)에서는 정적 파일을 직접 서빙하지만, `DEBUG=False`에서는 이를 서빙하지 않기 때문이다.

`DEBUG=True`일 때는 Django가 개발 편의를 위해 `/static/` 경로의 요청을 내부적으로 직접 처리해준다.
하지만 `DEBUG=False`가 되면, Django는 운영 환경으로 간주하고 정적 파일 처리를 **외부 웹 서버(Nginx 등)**의 역할로 넘긴다.
이때 Django는 `STATIC_ROOT` 경로에 정적 파일이 수집되어 있어야만 하고, 이 파일들을 직접 서빙하지 않는다.

따라서 다음과 같은 작업이 필요하다:

`collectstatic` 명령어로 정적 파일(CSS, JS 등)을 하나의 디렉토리(`STATIC_ROOT`)로 수집

이 수집된 정적 파일들을 Docker 컨테이너가 직접 서빙하거나, Nginx 등을 통해 서비스하도록 설정

즉, 운영 환경에서는 다음 흐름이 필요하다:

```plaintext
1. `STATICFILES_DIRS` → 내가 작성한 정적 자원 위치
↓
2. `collectstatic` 실행
↓
3. `STATIC_ROOT` 디렉토리에 모든 정적 파일 수집
↓
4. 이 경로에서 정적 파일이 서빙됨 (Django or Nginx)
```


## 💡 해결: 정적 파일 수집 및 서빙 설정

내 프로젝트 구조는 대강 다음과 같았다.:

```
pop_planner/
├── backend/
│   ├── config/                # settings.py, urls.py
│   ├── static/                # css
│   ├── staticfiles/           # collectstatic 결과가 모이는 곳
│   ├── templates/             # base.html, 각 앱의 HTML
│   ├── manage.py
│   └── ...
├── airflow/
│   └── ... (Airflow 관련)
├── docker-compose.yaml
└── ...
```
이 구조에서 중요한 점은 다음과 같다:

- `settings.py`는 `backend/config/`에 있기 때문에, `BASE_DIR`은 `backend/`가 되어야 한다.

정적 파일은 크게 두 종류로 나뉜다:

- `backend/static/`: 내가 만든 CSS, JS, 이미지 파일들이 위치하는 개발용 정적 디렉토리

- `backend/staticfiles/`: collectstatic 명령을 통해 모든 앱 + 설정된 정적 파일들이 복사되는 운영용 정적 디렉토리

즉, `STATICFILES_DIRS`는 개발 중 직접 관리하는 정적 파일들의 위치이고,
`STATIC_ROOT`는 서비스용으로 모든 정적 파일이 모이는 최종 경로다.

### 1. `settings.py`에 `STATIC_ROOT` 설정 추가

`config/settings.py`는 `backend/config/` 안에 있으므로, `BASE_DIR`을 `backend/`로 정확히 설정해야 한다.

```python
from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parent.parent  # backend/

STATIC_URL = '/static/'

# 개발용 정적 파일 경로
STATICFILES_DIRS = [
    BASE_DIR / 'static',
]

# 수집된 정적 파일 저장 위치
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

### 2. `urls.py`에 정적 파일 서빙 설정 추가

`urlpatterns` 마지막에 아래를 추가한다:

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

이렇게 하면 `DEBUG = True`일 때도 `/static/` 요청을 수동으로 처리할 수 있게 된다.

### 3. Docker 컨테이너 안에서 정적 파일 수집
정적 파일을 `/static/` 대신 `/staticfiles/`로 모아주기 위해 다음 명령어 실행한다:

```bash
docker compose exec django bash
python manage.py collectstatic --noinput
```

정상 실행되면 `/app/staticfiles/` 경로에 파일들이 수집된다:

```
root@container:/app# ls staticfiles/
admin  css  rest_framework
```

#### 4. `docker-compose.yaml`에 볼륨 마운트 설정 확인

정적 파일이 수집된 `staticfiles` 디렉토리를 호스트와 공유하기 위해 아래와 같이 마운트를 추가한다:

```yaml
volumes:
      - ./backend:/app
```

이렇게 하면 정적 파일(css, js 등)이 Django 외부 경로로 복사되고,
Nginx나 Django 컨테이너가 해당 파일을 클라이언트에 제대로 서빙할 수 있게 된다.
즉, 개발 단계에서도 정적 파일 깨짐 없이 Docker 환경을 안정적으로 유지할 수 있다.

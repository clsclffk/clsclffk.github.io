---
title: AWS EC2환경에 Django 설치하기
author: jisu
date: 2025-03-01 11:33:00 +0900
categories: [AWS]
tags: [Django]
pin: false
math: true
mermaid: true
---

## AWS EC2 환경에서 Django 설치

일단 가상환경 목록을 조회해보면, 아래와 같다.

```bash
conda env list
```

```
# conda environments:
#
base                  *  /home/ubuntu/anaconda3
Project                  /home/ubuntu/anaconda3/envs/Project
airflow_p38              /home/ubuntu/anaconda3/envs/airflow_p38
amazonei_mxnet_p36       /home/ubuntu/anaconda3/envs/amazonei_mxnet_p36
aws_neuron_mxnet_p36     /home/ubuntu/anaconda3/envs/aws_neuron_mxnet_p36
aws_neuron_pytorch_p36     /home/ubuntu/anaconda3/envs/aws_neuron_pytorch_p36
aws_neuron_tensorflow_p36     /home/ubuntu/anaconda3/envs/aws_neuron_tensorflow_p36
crawling                 /home/ubuntu/anaconda3/envs/crawling
mxnet_p37                /home/ubuntu/anaconda3/envs/mxnet_p37
python3                  /home/ubuntu/anaconda3/envs/python3
pytorch_p38              /home/ubuntu/anaconda3/envs/pytorch_p38
pytorch_p39              /home/ubuntu/anaconda3/envs/pytorch_p39
tensorflow2_p38          /home/ubuntu/anaconda3/envs/tensorflow2_p38
tensorflow3_p38          /home/ubuntu/anaconda3/envs/tensorflow3_p38
tensorflow4_p39          /home/ubuntu/anaconda3/envs/tensorflow4_p39
```

Django가 현재 내가 위치한 Project라는 가상환경 안에 설치되었는지 확인해본다.

```bash
python3 -m django --version
```

```
/home/ubuntu/anaconda3/envs/Project/bin/python3: No module named django

```

이렇게 뜨면 django가 설치되지 않았다는 의미이다.

패키지 충돌을 방지하고, 웹 서비스(Django)와 데이터 분석(Project), airflow(airflow_p38)를 분리하기 위해 Django 전용 가상환경을 따로 만들기로 했다.
처음에는 venv를 사용하여 가상환경을 만들려고 했다.

```bash
python3 -m venv django_env
```

하지만 다음과 같은 에러가 발생했다.

```
The virtual environment was not created successfully because ensurepip is not
available.  On Debian/Ubuntu systems, you need to install the python3-venv
package using the following command.

    apt-get install python3-venv

You may need to use sudo with that command.  After installing the python3-venv
package, recreate your virtual environment.

Failing command: ['/home/lab13/django_env/bin/python3', '-Im', 'ensurepip', '--upgrade', '--default-pip']
```

이 에러를 보고 기존에 Project, airflow_p38 가상환경이 이미 설치되어 있어서 python3-venv도 설치되어 있을 거라고 생각했다.
하지만, 기존 가상환경이 conda로 만들어진 환경이었기 때문에 python3-venv가 필요하지 않았던 것이다.

### 가상환경 관리도구
1. conda
- 가상환경과 패키지 관리 기능을 동시에 제공
2. python3-venv
- 기본 가상환경을 관리하지만 패키지는 `pip`로 관리해야함

따라서 Django도 Conda 가상환경으로 생성하기로 했다.

```bash
conda create --name django_env python=3.8
conda activate django_env
pip install django
```

```bash
python -m django --version
```

4.2.19 버전으로 설치가 완료되었다.

## Django 시작하기
장고 설치가 완료되었으면, 프로젝트를 만들고 실행해야한다.

일단 나는 web_project라는 이름의 프로젝트를 만들었다.

```bash
django-admin startproject web_project
```

web_project라는 폴더가 생겼으면 그 안으로 들어가 서버를 실행한다.

```bash
cd web_project
python manage.py runserver
```

```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
March 04, 2025 - 12:09:12
Django version 4.2.19, using settings 'web_project.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

이렇게 뜨면 개발 서버는 정상적으로 실행되고 있다는건데 
`You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.` 이렇게 메시지에 migration이 적용이 안된다고 나온다.
18개의 마이그레이션이 적용되지 않았다고 나오는 이유는, 이 테이블들을 데이터베이스에 생성하지 않았기 때문이다.

```bash
python manage.py migrate
```

이 명령어를 실행하면 장고의 기본 테이블이 데이터베이스에 적용된다.

다시 ```python manage.py runserver```를 하면 될거 같지만 페이지가 열리지 않는다.
EC2 가상환경에서 접근할 때는 EC2 보안 그룹 설정 확인을 먼저 해주어야 한다.
보안 그룹 -> 인바운드 규칙에 들어가서 django 포트 8000이 열려있는지, 소스가 0.0.0.0/0으로 되어있는지 확인해본다.
이미 잘 열려있다면, 

```bash
python manage.py runserver 0.0.0.0:8000
```

위와 같은 코드로 서버를 실행할 수 있다.

그런데, 장고는 보안상 기본적으로 외부 IP에서 접근을 차단하기 때문에 아래와 같은 `DisallowedHost`라고 뜰 수 있다.

![image](https://github.com/user-attachments/assets/64218f86-8279-4501-bc07-b59f4b2c475e)

`web_project/settings.py` 파일에서 `ALLOWED_HOSTS`를 수정하면 된다.

```bash
ALLOWED_HOSTS = ['15.xxx.xxx.xxx', 'localhost', '127.0.0.1']
```
이렇게 설정하고 다시 `python manage.py runserver 0.0.0.0:8000`를 실행하면, 로켓이 보이게 된다.

![image](https://github.com/user-attachments/assets/2bfe7366-a4e6-4860-a876-96d1c1d913b0)

### settings.py의 ALLOWED_HOSTS 설정
- 웹 애플리케이션이 허용할 도메인이나 IP 주소 목록을 지정하는 역할
- 이 설정에 포함되지 않은 도메인/IP에서 요청이 들어오면 Django가 해당 요청을 차단함
- EC2 환경 사용시, EC2 퍼블릭 IP를 추가해줘야 함

이제 관리자 계정을 생성하고 관리자 페이지에 접속해야한다.

```bash
python manage.py createsuperuser
```

```
Username (leave blank to use 'lab13'):
Email address: lab13@example.com
Password:
Password (again):
The password is too similar to the username.
This password is too short. It must contain at least 8 characters.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

비밀번호가 너무 짧아서 경고가 떴지만, 무시하고 관리자 계정을 만들었다. 
이제 `http://15.xxx.xxx.xxx:8000/admin/`으로 들어가면 관리자 페이지에 접속이 된다.

![image](https://github.com/user-attachments/assets/3778d133-25bf-462e-be91-48bc885e944b)








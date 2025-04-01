---
title: AWS EC2환경에서 Django 시작하기
author: jisu
date: 2025-03-01 11:33:00 +0900
categories: [AWS]
tags: [Django]
pin: false
math: true
mermaid: true
comments: true
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
시작에 앞서 장고 서비스는 크게 이렇게 8단계로 구축이 된다.

### 서비스 구축 순서
1. 프로젝트 및 앱 생성
2. settings.py에서 앱 등록
3. 모델 생성 및 마이그레이션
4. 관리자 페이지에 모델 등록
5. URL 설정
6. 뷰 작성
7. 템플릿 작성
8. 정적 파일
9. 서버 실행

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

## 앱 생성 & 등록
이제 프로젝트 안에 앱을 만들어야 하는데 기능 단위로 앱을 분리하면 된다.

```bash
cd web_project
python manage.py startapp users
```

나는 `users`라는 앱을 하나 만들고 여기에서 사용자 정보를 관리하겠다.
장고는 `INSTALLED_APPS` 목록에 있는 앱들만 인식하고 사용하기 때문에 `settings.py`에서 `INSTALLED_APPS`에 만든 앱을 추가해야한다.

![image](https://github.com/user-attachments/assets/e2784104-84af-4144-a9cf-eda2229a267d)

### 팁!
리스트(`[]`), 딕셔너리(`{}`), 튜플(`()`)에서 마지막 요소에 `,`를 붙이는 건 PEP 8 스타일 가이드에서 권장됨!

## 모델 생성 및 마이그레이션
장고에서 모델은 **DB의 구조를 정의**하는 역할을 한다. 장고에서는 SQL을 직접 사용하지 않고 ORM (Object Relational Mapping) 방식을 사용하여 모델을 정의한다.

### MySQL과 Django 연결
나는 MySQL에 이미 만들어둔 데이터베이스가 있었기 때문에 ORM과 기존의 데이터베이스를 연결하는 방식으로 진행하였다.
일단 `settings.py`에서 MySQL 연결 설정을 기존 DB에 맞게 수정해야한다.

![image](https://github.com/user-attachments/assets/ddfa6bb1-967a-4775-a498-9333018c592d)

기본적으로 sqlite와 연결되어있고 나는 이걸 MySQL로 바꿔주었다.

```bash
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'SNS_DB', 
        'USER': 'lab13',  # MySQL 사용자 계정 (기존 계정 사용 가능)
        'PASSWORD': 'lab13',  # MySQL 계정 비밀번호
        'HOST': 'localhost',  # Django와 MySQL이 같은 서버에 있는 경우
        'PORT': '3306',  # MySQL 기본 포트
        'OPTIONS': {
            'charset': 'utf8mb4',
        },
    }
}
```

장고가 MySQL과 통신하려면 `mysqlclient`라는 패키지가 필요해서 설치해주었다.

```bash
pip install mysqlclient
```

그리고 나서 아래의 코드를 통해 장고가 MySQL에 연결이 되는지 확인해본다.

```bash
python manage.py dbshell
```

```
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 926
Server version: 5.7.42-0ubuntu0.18.04.1 (Ubuntu)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

이렇게 창이 뜨면 연결에 성공이고, 테이블을 조회하면 내가 만든 테이블들이 나온다.
장고 ORM을 이용해서 MySQL 데이터를 불러 온 것에 성공하였다.

```sql
SHOW TABLES;
```

```
+-----------------------+
| Tables_in_SNS_DB      |
+-----------------------+
| analysis_results      |
| hobby_keywords        |
| phone_recommendations |
| tbCrawled_Danawa      |
| tbCrawled_Youtube     |
| users                 |
+-----------------------+
6 rows in set (0.00 sec)
```

## 모델 등록
이미 MySQL에 테이블이 있으므로 Django에서 기존 테이블을 자동으로 모델로 변환해야한다.
그런데 아래의 코드를 실행하면, 

```bash
python manage.py inspectdb > models.py
```

```
raise NotSupportedError(
django.db.utils.NotSupportedError: MySQL 8 or later is required (found 5.7.42).
```
장고가 MySQL 8.0 이상을 요구하는데 현재 버전은 5.7.42 버전이라서 에러가 발생하였다.
MySQL을 업그레이드 해줘야 문제가 해결된다.
> 출처 : https://docs.djangoproject.com/en/5.1/ref/databases/#mysql-notes

```bash
sudo apt-cache policy mysql-server
```
```
mysql-server:
  Installed: 5.7.42-0ubuntu0.18.04.1
  Candidate: 5.7.42-0ubuntu0.18.04.1
  Version table:
 *** 5.7.42-0ubuntu0.18.04.1 500
        500 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu bionic-security/main amd64 Packages
        100 /var/lib/dpkg/status
     5.7.21-1ubuntu1 500
        500 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu bionic/main amd64 Packages
```
MySQL이 업그레이드 가능한지 확인해봤는데, Candidate: 8.0.x가 아니라 5.7.42 그대로 나와서 업그레이드가 불가능한 상태였다.

### MySQL 업그레이드 (5.7 -> 8.0)

일단 MySQL 전체 데이터베이스를 백업했다.

```bash
mysqldump -u root -p --all-databases > backup_all_databases.sql
ls -lh backup_all_databases.sql
```

```
-rw-r--r-- 1 ubuntu multi 160M Mar  5 11:23 backup_all_databases.sql
```

이렇게 나오면 정상적으로 백업 파일이 생긴것이다.

MySQL을 업그레이드할 때, 일부 테이블이 호환되지 않을 수도 있어서 MySQL 업그레이드 전에 아래 명령어를 실행하여 모든 테이블의 호환성을 미리 점검하는 것이 중요하다.
> 출처: https://blog.pages.kr/2921 [pages.kr 날으는물고기 <º))))><:티스토리]

```bash
mysqlcheck --all-databases --check-upgrade --user=root --password
```
업그레이드 전에 현재 실행 중인 MySQL 서비스를 안전하게 중지하기 위해 아래의 명령어를 입력한다.

```
sudo systemctl stop mysql
```

MySQL이 정상적으로 중지되었는지 확인하려면 아래 명령어를 실행하면 된다.

```
systemctl status mysql
```

MySQL 8.0으로의 업그레이드는 MySQL 5.7의 GA(General Availability) 버전에서만 지원된다. 따라서 5.7.9 이전 버전을 사용 중이라면, 먼저 5.7.9 이상으로 업그레이드한 후 8.0으로 업그레이드 해야한다.

```
sudo apt update
sudo apt upgrade mysql-server -y
mysql --version
```

5.7이 나오면 일단 1차 업그레이드가 된 것이다.

```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.22-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.22-1_all.deb
```

![image](https://github.com/user-attachments/assets/76e2b43d-d509-4645-acb6-949581113d74)
이런 화면이 나오는데 MySQL Server & Cluster (Currently selected: mysql-5.7)를 선택한 상태에서 Enter 키를 누른다.
MySQL 버전 선택 화면이 나타나는데 여기서 mysql-8.0을 선택하고 Enter 키를 누른다.

최신 패키지를 설치하기 위해 먼저 패키지 목록 업데이트를 해야한다.
```
sudo apt update
```

그런데 아래와 같은 에러가 뜬다.
이 에러는 MySQL 저장소의 GPG 키가 없거나 만료되어 패키지 목록 업데이트가 실패하는 경우 발생한다.

```
Hit:1 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:3 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease
Get:4 https://nvidia.github.io/libnvidia-container/stable/ubuntu18.04/amd64  InRelease [1484 B]
Get:5 https://nvidia.github.io/nvidia-container-runtime/stable/ubuntu18.04/amd64  InRelease [1481 B]
Get:6 https://nvidia.github.io/nvidia-docker/ubuntu18.04/amd64  InRelease [1474 B]
Get:7 http://repo.mysql.com/apt/ubuntu bionic InRelease [20.0 kB]
Hit:8 https://dl.google.com/linux/chrome/deb stable InRelease
Err:7 http://repo.mysql.com/apt/ubuntu bionic InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
Hit:9 http://ppa.launchpad.net/openjdk-r/ppa/ubuntu bionic InRelease
Hit:10 http://security.ubuntu.com/ubuntu bionic-security InRelease
Get:11 https://apt.repos.neuron.amazonaws.com bionic InRelease [1446 B]
Err:11 https://apt.repos.neuron.amazonaws.com bionic InRelease
  The following signatures were invalid: EXPKEYSIG 5749CAD8646D9185 Amazon AWS Neuron <neuron-maintainers@amazon.com>
Reading package lists... Done
W: GPG error: http://repo.mysql.com/apt/ubuntu bionic InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B7B3B788A8D3785C
E: The repository 'http://repo.mysql.com/apt/ubuntu bionic InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://apt.repos.neuron.amazonaws.com bionic InRelease: The following signatures were invalid: EXPKEYSIG 5749CAD8646D9185 Amazon AWS Neuron <neuron-maintainers@amazon.com>
```


패키지 보안 설치를 위해 아래와 같이 GPG 키를 등록해야한다.

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B7B3B788A8D3785C
```

이제 최신 MySQL 서버를 설치하면 된다.

```bash
sudo apt install mysql-server
```
![image](https://github.com/user-attachments/assets/c5a974e2-39a3-453a-a4a7-fae88838bf9b)

기존 데이터를 문제없이 복원할 수 있도록 하려면 "Legacy Authentication Method (Retain MySQL 5.x Compatibility)"를 선택하는 게 안전하다.
기존 사용자 계정과 연결된 서비스가 문제 없이 로그인되려면 MySQL 5.x 방식(mysql_native_password)이 더 호환성이 높다고해서 설정해줬다.

```bash
mysql --version
```

정상적으로 업그레이드가 되었으면 아래와 같이 뜬다.

```
mysql  Ver 8.0.33 for Linux on x86_64 (MySQL Community Server - GPL)
```

그런데 또 에러가 뜨는데 MySQL이 정상적으로 설치되었지만 실행은 되지 않는 상황이다.

```bash
sudo systemctl start mysql
```

```
Job for mysql.service failed because the control process exited with error code.
See "systemctl status mysql.service" and "journalctl -xe" for details.
```

이 상황을 해결하려면, MySQL 8.0에서는 query_cache_size 및 query_cache_limit가 제거되었으므로, MySQL 설정 파일(/etc/mysql/mysql.conf.d/mysqld.cnf)에서 이 변수를 삭제해야 한다.

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```
# query_cache_size=16M
# query_cache_limit=1M
```

이렇게 주석처리 해주면 된다.

다시 `sudo systemctl start mysql`을 하면 성공적으로 실행이 되는것을 확인할 수 있다.

```bash
mysql -u root -p
mysql> SHOW DATABASES;
```

위의 코드를 실행하면 기존 DB가 그대로 유지되고 있는 것까지 확인이 되었다.

### models.py 생성
나는 이미 MySQL에 생성된 테이블을 장고 ORM과 연동해야하므로 기본적인 마이그레이션(makemigrations, migrate)을 하는 대신, 기존 테이블을 `models.py`에 자동으로 가져오게 해야한다.

```bash
cd ~/web_project
python manage.py inspectdb > models.py
```
위의 코드를 실행하면 테이블 구조가 장고의 `models.py`로 자동 변환된다.

```bash
nano models.py
```

`models.py`를 확인해보면 클래스가 잘 생성되었음을 확인할 수 있다.

```python
# This is an auto-generated Django model module.
# You'll have to do the following manually to clean this up:
#   * Rearrange models' order
#   * Make sure each model has one field with primary_key=True
#   * Make sure each ForeignKey and OneToOneField has `on_delete` set to the desired b$
#   * Remove `managed = False` lines if you wish to allow Django to create, modify, an$
# Feel free to rename the models, but don't rename db_table values or field names.
from django.db import models

class AnalysisResults(models.Model):
    hobby = models.OneToOneField('HobbyKeywords', models.DO_NOTHING, primary_key=True)$
    gender = models.CharField(max_length=1)
    age_group = models.CharField(max_length=3)
    keyword_list = models.JSONField()
    freq_samsung = models.JSONField()
    freq_apple = models.JSONField()
    related_words_samsung = models.JSONField()
    related_words_apple = models.JSONField()
    sentiment_samsung = models.FloatField()
    sentiment_apple = models.FloatField()
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField()

    class Meta:
        managed = False
        db_table = 'analysis_results'
        unique_together = (('hobby', 'gender', 'age_group'),)
```

난 MySQL에서 설정한 DB를 그대로 사용할 수 있을줄 알았는데 아니였다.
주석에 보면,
- 모델의 순서를 재배열해야 할 수 있음
- 복합 기본키를 장고가 지원하지 않음
- ForeignKey와 OneToOneField는 on_delete 속성이 설정되어 있어야 함
- `managed = False`로 설정된 라인을 제거하면 Django가 해당 테이블을 생성, 수정, 삭제할 수 있음
이렇게 정리되어 있다.
이거에 맞게 테이블을 변경해주어야 한다.
그리고 `models.py`가 각각 앱 안에 들어있어야 하므로 앱 폴더(`analysis/`, `hobbies/`, `users/`)로 이동시켜줘야 한다.
각 앱 폴더에 `models.py`를 만들고 각 앱에 맞는 모델을 정의하면 된다.
`users`앱의 `models.py`에 가서 아래와 같이 모델을 정의하면 된다.

```python
from django.db import models
from hobbies.models import HobbyKeywords

# Create your models here.
class Users(models.Model):
    user_id = models.AutoField(primary_key=True)
    hobby = models.ForeignKey('HobbyKeywords', models.CASCADE)
    nickname = models.CharField(max_length=50)
    age_group = models.CharField(max_length=3)
    gender = models.CharField(max_length=1)
    created_at = models.DateTimeField()
    class Meta:
        managed = True
        db_table = 'users'
        unique_together = (('hobby', 'nickname', 'age_group', 'gender'),)
```

나는 `managed = True`로 설정해서 Admin에서 데이터 관리(추가/수정/삭제)를 할 수 있게 하였다. 이제 장고에서 `models.py`를 수정하면 `makemigrations` & `migrate`를 실행하여 MySQL 테이블도 자동으로 변경될 수 있다.

Django ORM을 사용하여 데이터 조회 및 삽입 테스트를 진행하면 된다.

```bash
python manage.py shell
```

```python
from users.models import Users
print(Users.objects.all())
```

데이터가 정상적으로 출력이 되면 MySQL과 Django ORM이 잘 연결된 것이다. 

## urls.py
URL과 View를 연결하는 파일이 바로 `urls.py`이다.
프로젝트 전체 URL을 관리하는 `urls.py`에서 앱의 `urls.py`를 연결하고, 앱의 `urls.py`에서 앱 내부 URL을 관리한다.


## views.py
View는 사용자가 **요청한 정보를 처리하고 응답을 반환**하는 역할을 한다.


## 템플릿 만들기
템플릿은 사용자에게 보여질 웹페이지이다.

## 정적 파일 추가
CSS나 JS 같은 정적 파일 또한 추가 가능하다.







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







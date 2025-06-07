---
title: cron에서 Python 가상환경이 작동하지 않는 이유와 해결
author: jisu
date: 2025-06-06 11:33:00 +0900
categories: [Data Engineering]
tags: [Shell Script]
pin: false
math: true
mermaid: true
comments: true
---

## 목적
서울시 주차장/상권 데이터를 30분마다 자동으로 수집해 Elasticsearch에 업로드하기 위한 파이프라인을 만들고자 했다.
수집 작업은 두 개의 Python 스크립트(`upload_parking_data.py`, `upload_commercial_data.py`)로 이루어져 있으며, 이를 주기적으로 실행하려면 자동화가 필요했다.
가장 간단한 자동화 방법은 셸 스크립트로 실행 명령을 묶고, 이를 crontab으로 주기적으로 실행하는 것이다. 그래서 다음과 같이 `run.sh` 파일을 작성했다.

### 쉘 스크립트 만들기

처음에는 다음과 같이 작성했다:

```bash
#!/bin/bash

# 가상환경 활성화
source venv/bin/activate

echo "=== Run start: $(date) ===" >> ~/log_analysis/logs/cron.log

python3 /mnt/c/.../upload_parking_data.py >> ~/log_analysis/logs/parking.log 2>&1
python3 /mnt/c/.../upload_commercial_data.py >> ~/log_analysis/logs/commercial.log 2>&1

echo "=== Run end: $(date) ===" >> ~/log_analysis/logs/cron.log
```

- 맨 처음에 가상환경을 `source venv/bin/activate`로 활성화한 뒤,

- 각각의 Python 스크립트를 실행하고,

- 실행 로그는 각기 다른 로그 파일에 저장되도록 했다.

- 전체 실행 시작/종료 시점은 `cron.log`에 기록되도록 했다.

이제 이 스크립트를 crontab에 등록해서 **30분마다 자동으로 실행**되게 만들었다:

```bash
crontab -e
```

![image](https://github.com/user-attachments/assets/ff412431-c221-477e-bd62-63f2c7512d3a)


이렇게 마지막에 `*/30 * * * * bash /mnt/c/Users/jisu/Desktop/log_analysis/scripts/run.sh`를 추가하면
매 시간 0분, 30분마다 run.sh가 실행되면서 데이터를 수집하고 업로드하게 된다... 는 계획이었다.


## 🚨 가상 환경 문제 발생

### ❌ 수동 실행 (`bash run.sh`) 시엔 문제 없음

터미널에서 직접 bash run.sh를 실행하면 모든 것이 정상적으로 작동했다.
pandas도 잘 불러오고, 수집된 데이터도 Elasticsearch에 문제 없이 업로드됐다.


### ❌ crontab 실행 시 에러 발생

30분마다 자동 실행되도록 설정해둔 crontab에서는 문제가 발생했다.
로그 파일(logs/commercial.log, logs/parking.log)을 확인해보니 다음과 같은 에러 메시지가 찍혀 있었다:

`logs/commercial.log` 또는 `logs/parking.log` 확인 결과:

```bash
ModuleNotFoundError: No module named 'pandas'
```

## 원인 분석
### cron은 사용자의 로그인 환경 설정을 무시하고 실행

평소 터미널에서 스크립트를 실행할 때는, 사용자의 .bashrc, .zshrc 등에 정의된 설정들이 자동으로 적용된다.

예를 들어, `source venv/bin/activate`를 하면 `PATH`, `PYTHONPATH`, `VIRTUAL_ENV` 같은 환경 변수들이 바뀌고, 가상환경 안의 python과 pip, 그리고 설치한 패키지들이 활성화된다.

하지만 cron은 그런 설정 파일을 불러오지 않는다.

cron에서 실행되는 쉘은 비-인터랙티브 + 비-로그인 셸이기 때문에, 사용자의 `.bashrc`나 `.zshrc`는 전혀 적용되지 않는다.

`source venv/bin/activate` 명령어는 실행되더라도, 환경 변수를 현재 셸에 적용하지 못하고 무의미하게 지나간다.

결국 그 아래에서 실행되는 python3는 우리가 의도한 가상환경의 Python이 아니라, 시스템 기본 Python을 실행하게 된다.

```bash
/usr/bin/python3
```

이 Python에는 당연히 `pandas` 같은 패키지가 설치되어 있지 않기 때문에,

결국 `ModuleNotFoundError`가 발생하게 되는 것이다.

### `source venv/bin/activate`가 안되는 이유
여기서 한 가지 중요한 점은, source 명령 자체가 현재 셸 세션에 환경변수를 적용하는 방식이라는 것이다.

하지만 cron은 새로운 독립적인 셸 환경에서 스크립트를 실행하고, 그 셸은 로그인하지도, 사용자 환경을 로드하지도 않는다.

이 때문에 `source venv/bin/activate`가 실행되더라도, 그 효과는 새로 생성된 셸 안에서만 제한적으로 반영되다가 사라진다.

결국 PATH는 바뀌지 않고, python도 여전히 시스템 파이썬을 사용하게 된다.

즉, 우리가 activate를 시도해도 그 효과가 스크립트 전체에 이어지지 않고, 그 줄에서만 끝나버리는 셈이다.
또한 cron이 각 명령을 별도 셸에서 실행하는 구조라면, activate가 적용된 셸과 그 이후 python 실행 셸은 완전히 다른 환경일 수도 있다.

이런 이유로, source로 activate하는 방식보다는 가상환경의 python 실행 파일 경로를 직접 지정하는 방식이 훨씬 안정적이다. 예를 들어:

```bash
/path/to/venv/bin/python3 /path/to/script.py
```

이렇게 직접 실행하면 activate 없이도 해당 가상환경의 python이 실행되고, 패키지도 정상적으로 로드된다.
이는 cron 환경처럼 사용자 쉘 설정이 무시되는 상황에서 특히 유용하다.


## 💡 해결 방법
### 🔧 가상환경 파이썬 경로를 **명시적으로 지정**

앞서 설명한 이유 때문에, source venv/bin/activate 방식은 cron 환경에서 기대한 대로 작동하지 않는다.

따라서 가장 확실하고 안전한 방법은 가상환경 안에 있는 python 실행 파일 경로를 명시적으로 지정해서 사용하는 것이다.

즉, 가상환경을 굳이 activate하지 않고도, 해당 환경의 python을 직접 호출하면 된다.

아래는 수정된 run.sh 예시다:

```bash
#!/bin/bash

# 가상환경의 python 경로를 직접 지정
VENV_PYTHON=/mnt/c/Users/jisu/Desktop/log_analysis/venv/bin/python

echo "=== Run start: $(date) ===" >> ~/log_analysis/logs/cron.log

$VENV_PYTHON /mnt/c/.../upload_parking_data.py >> ~/log_analysis/logs/parking.log 2>&1
$VENV_PYTHON /mnt/c/.../upload_commercial_data.py >> ~/log_analysis/logs/commercial.log 2>&1

echo "=== Run end: $(date) ===" >> ~/log_analysis/logs/cron.log
```

이 방식은 source로 환경을 적용할 필요 없이,
가상환경 내부의 python을 직접 실행하기 때문에 cron에서도 문제 없이 작동한다.
$VENV_PYTHON 변수에 경로만 정확하게 지정해주면, 이후 실행되는 스크립트들은 해당 가상환경에 설치된 패키지들을 정상적으로 사용할 수 있다.

특히 cron처럼 사용자 환경 설정을 무시하는 자동화 환경에서는 이 방법이 가장 안정적이다.
가상환경을 activate하지 않더라도, 실제로는 그 안의 python을 그대로 쓰기 때문에 결과적으로 완전히 동일한 실행 환경을 보장할 수 있다.

## 테스트 결과

위에서 수정한 run.sh를 다시 crontab에 등록하고, 실행 로그와 Elasticsearch 상태를 확인해 보았다.

우선 cron이 스크립트를 제대로 실행했는지 확인하기 위해 로그 파일을 확인했다:

```bash
cat logs/cron.log
```

```plain text
=== Run start: Sun Jun  8 02:30:01 KST 2025 ===
[SUCCESS] parking_data 업로드 완료 (Sun Jun  8 02:30:17 KST 2025)
[SUCCESS] commercial_data 업로드 완료 (Sun Jun  8 02:31:28 KST 2025)
=== Run end: Sun Jun  8 02:31:28 KST 2025 ===
```

시작과 종료 로그가 정상적으로 기록되어 있었고, `logs/parking.log` 및 `logs/commercial.log` 파일에도 에러 없이 실행된 흔적이 남아 있었다.

또한 Elasticsearch에 데이터가 정상적으로 업로드되었는지도 확인해보았다:

```bash
curl -X GET localhost:9200/_cat/indices?v
```

```plain text
health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   seoul_commercial_categories BUnfJLmDT0K-dNvj-QdPDg   1   1       2567          504    469.2kb        469.2kb
yellow open   seoul_commercial            XaDWDfwWRYapWGRwlyrI5g   1   1        820            0    346.8kb        346.8kb
yellow open   seoul_parking               Vs9RwRr7QDupLOLd3sH4zQ   1   1        497            0    385.6kb        385.6kb
```

위와 같이 인덱스가 정상적으로 생성되었고, 문서도 제대로 쌓이고 있는 것을 확인할 수 있었다.

이제 cron을 통한 자동 수집 파이프라인이 정상적으로 작동하고 있으며, 주기적인 데이터 적재도 문제없이 수행되고 있다.
가상환경 문제로 애를 먹었던 초기 설정과 달리, python 경로를 직접 지정하는 방식이 훨씬 확실하고 안정적이라는 것을 확인할 수 있었다.






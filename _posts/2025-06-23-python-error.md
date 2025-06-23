---
title: 데이터 파이프라인에서 자주 발생하는 Python 모듈 경로 에러 해결법
author: jisu
date: 2025-06-23 13:30:00 +0900
categories: [Python, Data Engineering]
tags: [PYTHONPATH, sys.path]
pin: false
math: false
mermaid: false
comments: true
---

데이터 파이프라인 구성 중,  
`from etl.load_to_postgres import foo` 같은 코드를 작성했는데 아래와 같은 오류가 발생했다:

```
ModuleNotFoundError: No module named 'etl'
```

처음에는 etl 폴더가 분명히 있는데 왜 못 찾지? 싶었는데, 알고 보니 Python의 모듈 탐색 경로 문제였다.  
이 글에서는 Python 프로젝트에서 자주 발생하는 모듈 import 문제를 어떻게 해결했는지 정리해본다.

## 📁 디렉토리 구조

```
project/
├── src/
│   ├── etl/
│   │   └── load_data.py      
│   ├── api/
│   │   └── run_script.py      
│   └── run_etl.py           
└── .env
```

나는 `src/api/run_script.py` 안에서 아래와 같이 import를 시도했는데,

```python
from etl.load_to_postgres import foo
```

아래와 같은 에러가 발생했다:

```
ModuleNotFoundError: No module named 'etl'
```


## 🔍 원인 
Python은 기본적으로 실행한 파일의 위치를 기준으로 모듈을 찾는다.

즉, 아래처럼 실행하면:

```
python src/api/fetch_places.py
```

현재 경로는 `src/api`가 되고, Python은 여기서 `etl`을 찾는다.
하지만 실제 `etl`은 `src/etl`에 있기 때문에 못 찾는 것이다.

> 📌 즉, 폴더 구조는 잘 만들어놨더라도,
> Python 입장에서 `etl`이 어디 있는지 모르면 import는 실패한다.


## 💡 해결 방법 1: `sys.path.append()` 사용

가장 빠른 해결 방법은 코드에서 직접 모듈 탐색 경로를 추가해주는 것이다.

```python
import sys
import os

# 현재 파일 기준으로 프로젝트 루트(src 상단) 경로 추가
sys.path.append(os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

from etl.load_to_postgres import foo
```

`sys.path`는 Python이 모듈을 찾기 위해 참고하는 경로들의 리스트인데,
여기에 `src/` 경로를 직접 추가해준 것이다.

이렇게 하면 Python은 `src/etl` 안에 있는 `load_to_postgres.py` 파일을 찾아서 정상적으로 import할 수 있게 된다.

>  📌 이 방식은 지금 이 파일만 실행할 경우엔 효과적이지만,
> 여러 파일에 매번 써야 하므로 프로젝트가 커지면 관리가 번거로워진다.


## 💡 해결 방법 2: `PYTHONPATH` 환경변수 설정

여러 스크립트를 연속 실행하거나, 루트에서 실행할 때는 `PYTHONPATH` 환경변수를 설정하는 게 더 깔끔하다.

```bash
PYTHONPATH=./src python src/api/run_script.py
```

이렇게 하면 Python은 `src/` 디렉토리를 모듈 탐색 경로로 인식한다.
즉, `etl.load_to_postgres` 같은 절대 경로 import가 가능해진다.

또는 여러 스크립트를 실행해야 하는 경우,

```python
# run_etl.py
import subprocess
import os

env = os.environ.copy()
env["PYTHONPATH"] = "./src"

subprocess.run(["python", "src/api/run_script.py"], env=env)
```

이렇게 하면 `run_etl.py`가 각 하위 모듈을 실행할 때도 모듈 경로 문제가 발생하지 않는다.

## 마무리
처음에는 "폴더 구조도 잘 해놨는데 왜 import가 안되지?" 싶었는데,  
알고 보니 Python은 파일 시스템 구조와는 별개로 **실행 위치를 기준으로** 모듈을 찾는다는 걸 몰랐던 게 원인이었다.

결론적으로, 프로젝트를 디렉토리별로 잘 나눴다면

- 빠르게 테스트하거나 단독 실행할 때는 `sys.path.append()`  
- 전체 프로젝트 구조로 일관되게 실행하려면 `PYTHONPATH` 환경변수 설정

두 가지 방식 중 상황에 맞게 선택하면 된다.




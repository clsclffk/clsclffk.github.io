---
title: Redis TimeSeries DUPLICATE_POLICY BLOCK 모드 오류 해결하기
author: jisu
date: 2025-10-01 12:00:00 +0900
categories: [Data Engineering]
tags: [Redis, TimeSeries, Streaming, Spark, Kafka]
pin: false
math: false
mermaid: false
comments: true
---

## 배경
Redis Stack에서 제공하는 **TimeSeries (TSDB)** 모듈은  
실시간 시계열 데이터를 저장할 때 자주 사용된다.  

그런데 Kafka → Spark Streaming → Redis로 데이터를 흘려보내다 보면,  
같은 timestamp 값이 중복으로 들어오는 경우가 생기는데,  
이때 `DUPLICATE_POLICY` 설정에 따라 동작이 달라진다.  

이번 글에서는 BLOCK 모드에서 발생하는 오류를 다룬 경험을 정리한다.

## 문제 상황
Streaming Job 실행 중, Redis에 데이터를 적재하려는 시점에서  
다음과 같은 에러가 발생했다:

```
redis.exceptions.ResponseError: TSDB: Error at upsert, update is not supported when DUPLICATE_POLICY is set to BLOCK mode
```

즉, 같은 timestamp에 데이터를 넣으려 하자,  
`BLOCK` 모드에서는 **중복 삽입을 허용하지 않으므로 job이 종료된 것**이다.

## 원인
Redis TimeSeries는 기본적으로 `DUPLICATE_POLICY=BLOCK`으로 설정된다.  
따라서 **동일 timestamp 데이터가 들어오면 에러**를 발생시킨다.  

Duplicate Policy는 다음과 같이 종류가 있다:

- **BLOCK** → 중복 허용 안 함 (에러 발생)  
- **FIRST** → 첫 번째 값 유지  
- **LAST** → 마지막 값으로 덮어쓰기  
- **MIN** → 최소값 유지  
- **MAX** → 최대값 유지  
- **SUM** → 합산  

Streaming 환경에서는 millisecond 단위로 데이터가 들어오기 때문에,  
timestamp 충돌은 피할 수 없고 BLOCK 모드로는 안정적인 처리가 어렵다.  

## 💡 해결 방법

이 문제를 해결하는 가장 간단한 방법은  
`DUPLICATE_POLICY` 설정을 **BLOCK**이 아닌 다른 값으로 변경하는 것이다. 

```bash
TS.CREATE ts:KRW-BTC DUPLICATE_POLICY LAST
```

하지만 이렇게 하면 key를 미리 수동으로 만들어야 하는 번거로움이 있다.
나는 key가 없을 경우 자동으로 `TS.CREATE`를 실행하면서
`DUPLICATE_POLICY=LAST`를 강제 적용하는 함수를 만들어 사용했다.
이렇게 하면 나중에 새로운 key가 필요해질 때도 매번 직접 만들어줄 필요가 없어서 더 편리하다.
   
```python
import redis

# Redis 연결
r = redis.Redis(host='redis', port=6379, db=0)

def ensure_timeseries(key):
    """해당 TS key가 없으면 DUPLICATE_POLICY=LAST로 생성"""
    if not r.exists(key):
        r.execute_command("TS.CREATE", key, "DUPLICATE_POLICY", "LAST")
```

## 결과 
이제 같은 timestamp 값이 들어와도
기존 값은 새 값으로 덮어쓰기 처리되며,
에러 없이 안정적으로 Streaming Job을 실행할 수 있다.

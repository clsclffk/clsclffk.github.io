---
title: Spark Streaming에서 Kafka 연동 시 JAR 누락 오류 해결 방법
author: jisu
date: 2025-09-23 12:00:00 +0900
categories: [Data Engineering]
tags: [Spark, Kafka, Docker, Streaming, ETL]
pin: false
math: false
mermaid: false
comments: true
---

## 배경
실시간 스트리밍 처리를 위해  
Kafka **토픽(topic)**에 쌓이는 데이터를 Spark Streaming으로 읽어와야 했다. 

Kafka에서 **토픽**은 메시지가 저장되는 기본 단위다.  
프로듀서(Producer)가 데이터를 특정 토픽으로 보내면,  
컨슈머(Consumer)는 해당 토픽을 구독(subscribe)해서 메시지를 받아간다.  

즉, 토픽은 "실시간 데이터가 흘러 들어오고, 여러 소비자가 동시에 읽어갈 수 있는 파이프" 같은 역할을 한다.  
Spark Streaming을 이용하면 이 토픽에 들어오는 메시지를 DataFrame처럼 다루며  
집계·변환·저장 같은 처리를 할 수 있다.

## 문제 상황

처음에는 단순히 Spark 코드만 실행하면 된다고 생각해서 이렇게 실행했다:

```bash
docker exec -it spark-master spark-submit \
--master spark://spark-master:7077 \
/opt/bitnami/spark/scripts/streaming_job.py
```

하지만 곧바로 아래와 같은 에러가 발생했다:

```
org.apache.spark.sql.AnalysisException:
Failed to find data source: kafka.
```

## 원인
Spark는 기본적으로 **Kafka와 연결할 수 있는 JAR(라이브러리)**를 포함하지 않는다.
즉, .format("kafka")를 쓰려면 Spark에 Kafka 커넥터를 직접 추가해줘야 한다.
이걸 처음 몰라서 그냥 spark-submit만 실행했던 것이다.
따라서 Spark는 실행할 때 필요한 JAR을 옵션으로 지정해 줘야 한다.

## 💡 해결 방법
Spark에서 외부 라이브러리를 불러오는 방법은 spark-submit 실행 시 `--packages` 옵션을 주는 것이다.
사용 중인 Spark 버전에 맞는 `spark-sql-kafka` 패키지를 지정해야 한다.

```bash
docker exec -it spark-master spark-submit \
--master spark://spark-master:7077 \
--packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 \
/opt/bitnami/spark/scripts/streaming_job.py
```

이렇게 하면 Spark가 Maven Central에서 Kafka 커넥터 JAR을 내려받아
실행 환경에 자동으로 등록해 준다.
그 결과 `.format("kafka")`가 정상적으로 인식되고,
Kafka 토픽에서 실시간 데이터를 읽어올 수 있었다.

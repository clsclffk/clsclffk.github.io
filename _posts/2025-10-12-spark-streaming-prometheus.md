---
title: StreamingQueryListener로 Spark Structured Streaming 메트릭 Prometheus에 노출하기
author: jisu
date: 2025-10-12 10:00:00 +0900
categories: [Data Engineering]
tags: [Spark, Streaming, Docker, Prometheus]
pin: false
math: false
mermaid: false
comments: true
---

## 개요

Spark Structured Streaming을 사용해 Kafka로부터 데이터를 처리하는 과정에서  
`inputRowsPerSecond`, `processedRowsPerSecond`, `batchDuration` 등의 지표를  
Prometheus로 수집해 모니터링할 수 있도록 구성했다.  

Spark 3.5 버전은 Prometheus Sink를 기본적으로 포함하지만,  
도커 환경에서는 `metrics.properties` 설정이 올바르게 연결되지 않거나  
포트 및 경로(`/metrics/prometheus`) 충돌로 인해  
**스트리밍 관련 메트릭이 Prometheus에 정상적으로 노출되지 않는 문제**가 발생할 수 있다.  

이 문제를 해결하기 위해  
[Stack Overflow – Spark 3.0 streaming metrics in Prometheus](https://stackoverflow.com/questions/64436497/spark-3-0-streaming-metrics-in-prometheus)  
에서 제시된 방법(`spark.metrics.conf`를 명시적으로 지정하는 방식)을 적용해 보았다.  

```yaml
sparkConf:
  "spark.metrics.conf": "/etc/metrics/conf/metrics.properties"
  "spark.sql.streaming.metricsEnabled": "true"
```

하지만 어떤 이유에서인지, 여러 설정을 시도했음에도
Streaming 메트릭이 `/metrics/prometheus`에 여전히 노출되지 않았다.

결국 기본 Sink 방식 대신,
StreamingQueryListener를 활용해 Prometheus 지표를 직접 Exporter 서버로 노출하는 커스텀 방식으로 전환했다.

---

## 설정

내 프로젝트에서 일반 Spark 메트릭과 Streaming 메트릭을 분리하여 관리했다.

Spark Master/Worker는 기존 방식대로 **JMX Exporter(8093 포트)** 를 통해  
CPU, Executor, 메모리 등 기본 런타임 메트릭을 Prometheus에 전달하도록 설정했다.  

반면, Spark Structured Streaming의 실시간 처리 지표(`inputRowsPerSecond`, `processedRowsPerSecond`, `batchLatency`)는  
Prometheus Sink 방식 대신, **`StreamingQueryListener` 기반의 커스텀 Exporter 서버(8099 포트)** 에서 직접 노출되도록 구성했다.  

이렇게 분리하면 Spark 내부 메트릭과 Streaming 처리 성능 지표를  
Prometheus와 Grafana에서 각각 명확하게 구분해 모니터링할 수 있다.

---

## 코드 구현

`streaming_job.py` 내부에서 `StreamingQueryListener` 이벤트를 감지해  
`prometheus_client` 라이브러리를 이용해 직접 메트릭을 노출한다.

```python
from pyspark.sql import SparkSession
from pyspark.sql.streaming import StreamingQueryListener
from prometheus_client import Gauge, start_http_server

# Prometheus 지표 정의
input_rate = Gauge("spark_streaming_inputRowsPerSecond", "Kafka → Spark 유입 속도")
processed_rate = Gauge("spark_streaming_processedRowsPerSecond", "Spark 처리 속도")
batch_latency = Gauge("spark_streaming_batchLatencyMs", "Spark 배치 처리 지연 (ms)")

# Structured Streaming 리스너
class PrometheusStreamingListener(StreamingQueryListener):
    def onQueryProgress(self, event):
        p = event.progress
        input_rate.set(p.inputRowsPerSecond)
        processed_rate.set(p.processedRowsPerSecond)
        batch_latency.set(p.durationMs.get("batchDuration", 0))

# Prometheus Exporter 서버 시작
start_http_server(8099)
print("✅ Prometheus metrics exporter started on port 8099")

spark = SparkSession.builder \
    .appName("CoinStreamJob") \
    .master("spark://spark-master:7077") \
    .getOrCreate()

spark.streams.addListener(PrometheusStreamingListener())
```

이렇게 하면 `http://localhost:8099/metrics` 에서
Prometheus 포맷으로 Spark Structured Streaming 지표를 실시간 확인할 수 있다.

## Docker Compose 설정
streaming-job 컨테이너의 `docker-compose.yml` 설정은 다음과 같다.

```yaml
streaming-job:
  build:
    context: ./spark
    dockerfile: Dockerfile
  container_name: streaming-job
  depends_on:
    - spark-master
    - kafka
    - redis
  command: >
    spark-submit
    --master spark://spark-master:7077
    --conf "spark.driver.extraJavaOptions=-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=8093:/opt/jmx/config.yml"
    --conf spark.sql.streaming.metricsEnabled=true
    --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0
    /opt/bitnami/spark/scripts/streaming_job.py
  volumes:
    - ./spark/scripts:/opt/bitnami/spark/scripts
    - ./jmx:/opt/jmx
  restart: always
  networks:
    - coin_net
  ports:
    - "8099:8099"  # Prometheus Exporter 포트
```

## Prometheus 설정

Prometheus의 prometheus.yml에서 Streaming Job을 타깃으로 추가한다.

```yaml
- job_name: 'spark-streaming'
  scrape_interval: 5s
  static_configs:
    - targets: ['streaming-job:8099']
```

## 결과 확인

`curl http://localhost:8099/metrics` 명령으로 다음과 같은 결과를 확인할 수 있다.

```
# HELP spark_streaming_inputRowsPerSecond Kafka → Spark 유입 속도
spark_streaming_inputRowsPerSecond 58.4
# HELP spark_streaming_processedRowsPerSecond Spark 처리 속도
spark_streaming_processedRowsPerSecond 312.5
# HELP spark_streaming_batchLatencyMs Spark 배치 처리 지연 (ms)
spark_streaming_batchLatencyMs 0.0
```

Prometheus에서는 spark-streaming 타겟이 UP 상태로 표시되고,
Grafana에서는 Input Rate, Processing Rate, Batch Latency를 라인 차트로 시각화할 수 있다.

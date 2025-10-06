---
title: "Prometheus에서 Kafka와 Spark 모니터링 설정 및 에러 해결기"
author: jisu
date: 2025-10-06 21:00:00 +0900
categories: [Data Engineering]
tags: [Prometheus, Kafka, Spark, JMX, Docker]
pin: false
math: false
mermaid: false
comments: true
---

## 배경
Prometheus로 Kafka와 Spark의 상태를 모니터링하던 중  
**메트릭 수집 실패 및 포트 충돌 에러**가 발생했다.  

Kafka와 Spark 모두 JVM 기반 애플리케이션이라  
Prometheus에서 직접 지표를 긁어오려면 **JMX Exporter** 설정이 필요하다.  

이번 글에서는 기존 **사이드카 컨테이너 방식**에서  
**Java Agent 방식**으로 전환하면서 발생한 문제와  
Kafka, Spark 각각의 해결 과정을 정리한다.

---

## 1️⃣ Kafka JMX Exporter 전환 중 발생한 포트 충돌 

### 문제 상황

Prometheus에서 Kafka 메트릭을 스크랩할 때 다음과 같은 에러가 발생했다:

```
Error scraping target: Get "http://kafka-jmx-exporter:9404/metrics
":
dial tcp 172.20.0.16:9404: connect: connection refused
```

즉, **Prometheus가 Kafka 메트릭을 가져오지 못하고 Connection Refused가 발생한 상태**였다.  

---

### 원인 
Kafka는 기본적으로 Prometheus에서 바로 메트릭을 노출하지 않는다.  
그래서 처음엔 빠르게 모니터링 환경을 구축하기 위해  
`kafka-jmx-exporter`라는 **별도 컨테이너(사이드카 방식)** 을 띄워서  
9404 포트를 통해 메트릭을 수집하도록 설정했었다.  

이 방식은 Kafka 이미지를 수정할 필요가 없고,  
컨테이너 하나만 추가하면 바로 Prometheus와 연동할 수 있어서  
초기 설정이 간편하다는 장점이 있었다.  

하지만 컨테이너가 늘어나면 관리 포인트도 함께 늘어난다.  
Kafka가 재시작될 때마다 Exporter 컨테이너도 맞춰 재기동해야 했고,  
포트 충돌이나 네트워크 연결 오류가 자주 발생했다.  

그래서 이후에는 구성 자체를 단순화하기 위해  
Kafka 컨테이너 내부에 **JMX Agent를 직접 붙이는 방식(자바 에이전트 방식)** 으로 변경했다.  
이 방법은 컨테이너 수를 줄이고, Kafka와 Exporter를 한 프로세스로 묶을 수 있어  
운영 측면에서 훨씬 안정적이었다.  

다만 이전에 쓰던 `kafka-jmx-exporter` 컨테이너가 그대로 남아 있어서  
9404 포트를 계속 점유하고 있었고,  
그 결과 새 Kafka 컨테이너가 같은 포트를 열지 못해 충돌이 발생했던 것이다.

---

### 💡 해결

먼저 문제의 원인이었던 **고아 컨테이너(`kafka-jmx-exporter`)** 를 직접 제거했다.  

```bash
docker rm -f kafka-jmx-exporter
```

그다음, 혹시 여전히 9404 포트를 점유 중인 컨테이너가 남아 있는지 확인했다.

```bash
docker ps -a | grep 9404
```
이 명령어로 확인했을 때 kafka 컨테이너만 포트를 사용하고 있다면 정상이다.

마지막으로, 불필요하게 남아 있는 중지된 컨테이너를 한 번에 정리했다.

```bash
docker container prune -f
```

이 과정을 거친 뒤,
9404 포트를 kafka만 점유하게 되어 충돌 문제가 완전히 해결되었다.

## 2️⃣ Kafka JMX Exporter 설정 (Java Agent 방식)

Kafka는 기본적으로 Prometheus에서 바로 수집할 수 있는 형태의 메트릭을 제공하지 않는다.
따라서 JVM 애플리케이션에서 자주 사용하는 JMX Exporter(Java Agent 방식) 을 이용해
Kafka 내부의 지표를 Prometheus 포맷으로 노출해줘야 한다.

### 💡 해결

우선 **JMX Exporter**를 적용하기 위해  
공식 GitHub 저장소에서 최신 버전을 확인한 뒤 필요한 파일을 다운로드했다.  

> https://github.com/prometheus/jmx_exporter

JMX Exporter의 최신 릴리스 jar 파일과
Kafka용 기본 설정 파일을 아래와 같이 설치한다.

```bash
sudo wget https://github.com/prometheus/jmx_exporter/releases/download/1.4.0/jmx_prometheus_javaagent-1.4.0.jar -O jmx/jmx_prometheus_javaagent.jar
sudo wget https://raw.githubusercontent.com/prometheus/jmx_exporter/refs/heads/main/examples/kafka-2_0_0.yml -O jmx/config.yml
```

이제 `docker-compose.yml`에서 Kafka 서비스 설정을 수정해
다운로드한 jar 파일을 Java Agent로 등록해준다.


```yaml
kafka:
  environment:
    - JVM_OPTS=-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=9404:/opt/jmx/config.yml
  volumes:
    - ./jmx:/opt/jmx
```

컨테이너를 재기동한 뒤, 아래 명령어로 정상적으로 메트릭이 노출되는지 확인했다.

```bash
curl http://localhost:9404/metrics | head -n 20
```

출력 결과에 다음과 같은 항목이 보이면 설정이 성공적으로 적용된 것이다.

```
# HELP jmx_exporter_build_info JMX Exporter build information
# TYPE jmx_exporter_build_info gauge
jmx_exporter_build_info{name="jmx_prometheus_javaagent",version="1.4.0"} 1
```

이제 Prometheus에서 지표가 정상적으로 수집되는 것을 확인할 수 있다.

## 3️⃣ Spark 메트릭 수집 설정 (JMX Agent 방식)

### 문제

Prometheus에서 Spark 타겟을 스크랩할 때 아래와 같은 오류가 발생했다.

```
Error scraping target: received unsupported Content-Type "text/html;charset=utf-8"
Error scraping target: Get "http://spark-master:8090/metrics": dial tcp 172.20.0.7:8090: connect: connection refused
```

즉, Prometheus가 Spark로부터 메트릭을 가져오지 못하고  
연결 거부(`connection refused`)가 발생한 상태였다.  

Spark의 기본 Web UI 포트(4040, 8080)는 HTML만 반환하기 때문에  
Prometheus가 이해할 수 있는 **OpenMetrics 포맷으로 변환**해 주는 과정이 필요했다.  

---

### 💡 해결

Kafka와 동일하게 Spark에도 **JMX Agent 방식을 적용**했다.  
이를 통해 Prometheus가 Spark의 JVM 내부 지표(GC, CPU, 메모리 등)를 직접 수집할 수 있도록 구성했다.  

먼저 `docker-compose.yml`에서 Spark Master와 Worker 컨테이너 각각에  
JMX Agent를 등록하는 설정을 추가했다.  

```yaml
spark-master:
  environment:
    - SPARK_MODE=master
    - SPARK_MASTER_OPTS=-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=8090:/opt/jmx/config.yml
  ports:
    - "8090:8090"
  volumes:
    - ./jmx:/opt/jmx

spark-worker-1:
  environment:
    - SPARK_MODE=worker
    - SPARK_WORKER_OPTS=-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=8091:/opt/jmx/config.yml
  ports:
    - "8091:8091"
  volumes:
    - ./jmx:/opt/jmx

spark-worker-2:
  environment:
    - SPARK_MODE=worker
    - SPARK_WORKER_OPTS=-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=8092:/opt/jmx/config.yml
  ports:
    - "8092:8092"
  volumes:
    - ./jmx:/opt/jmx

```

그다음, Prometheus 설정(`prometheus.yml`)에 새로 추가된 Spark 타겟을 등록했다.

```yaml
- job_name: 'spark'
  static_configs:
    - targets: ['spark-master:8090', 'spark-worker-1:8091', 'spark-worker-2:8092']
```

컨테이너를 재기동한 후 아래 명령어로 메트릭이 제대로 노출되는지 확인했다.

```bash
curl http://localhost:8090/metrics | head -n 20
```
출력 결과에 아래와 같은 항목이 표시되면 설정이 정상적으로 동작하는 것이다.

```
# HELP jmx_exporter_build_info JMX Exporter build information
# TYPE jmx_exporter_build_info gauge
jmx_exporter_build_info{name="jmx_prometheus_javaagent",version="1.4.0"} 1
```

이제 Prometheus에서 Spark의 지표가 정상적으로 수집되고 있음을 확인할 수 있다.

## 정리
Kafka와 Spark 모두
사이드카 방식 → Java Agent 방식으로 전환하면서
Prometheus에서 안정적으로 메트릭을 수집할 수 있게 되었다.

아래는 Prometheus에서 **Kafka와 Spark 타겟이 모두 정상(UP)** 상태로 표시된 화면이다.

![Prometheus Target Health](/assets/img/posts/prometheus-kafka-spark-up.png)

---
title: Docker + Kafka + Prometheus 환경에서 InvalidReceiveException 에러 해결기
author: jisu
date: 2025-09-22 12:00:00 +0900
categories: [Data Engineering]
tags: [Docker, Kafka, Prometheus, EC2, JMX Exporter]
pin: false
math: false
mermaid: false
comments: true
---

## 문제 상황
실시간 코인 데이터 스트리밍 파이프라인을 EC2 + Docker 환경에서 구성하면서,  
Kafka 컨테이너 로그에서 아래와 같은 에러가 반복적으로 발생했다:

```
org.apache.kafka.common.network.InvalidReceiveException:
Invalid receive (size = 1195725856 larger than 104857600)
```

Producer → Kafka → Consumer 파이프라인은 정상적으로 동작했고,  
`kafka-console-consumer.sh`로 확인해도 데이터가 잘 들어오고 있었다.  

즉, **Kafka 자체 문제는 아니고, 특정 트래픽 때문에 브로커 로그에 에러가 찍히는 상황**이었다.  

## 🔍 원인 분석

에러 메시지를 검색해보니, Stack Overflow에서 나와 완전히 같은 문제 상황을 발견했다:

> [Getting org.apache.kafka.common.network.InvalidReceiveException](https://stackoverflow.com/questions/49370959/getting-org-apache-kafka-common-network-invalidreceiveexception-invalid-receiv)

### **핵심 요약**

Prometheus가 잘못된 포트(9092)로 HTTP 요청을 보내고 있었던 것이다.
  
Kafka 브로커 포트는 클라이언트 통신을 위한 바이너리 프로토콜 전용 포트라서,
Prometheus가 `/metrics` 요청을 보내면 Kafka는 이를 해석하지 못하고 `InvalidReceiveException`을 뿜는다.

내 `prometheus.yml` 설정을 확인해 보니 역시 아래처럼 되어 있었다:

```yaml
- job_name: 'kafka'
  static_configs:
    - targets: ['kafka:9092']   # ❌ 잘못된 설정
```

Prometheus가 Kafka 브로커 포트(9092)를 모니터링 대상으로 대상으로 잘못 지정해 둔 게 원인이었다.
처음 `prometheus.yml`을 작성할 때, 단순히 Kafka 컨테이너가 열고 있는 포트 = 모니터링 포트라고 생각했다.
Kafka가 9092로 클라이언트와 통신하니 Prometheus도 같은 포트를 바라보면 될 거라고 착각한 것이다.

## 💡 해결 방법

### 1. JMX Exporter 추가

Kafka는 Prometheus용 메트릭을 직접 제공하지 않는다.
따라서 JMX Exporter를 통해 내부 JMX 포트를 외부 HTTP 엔드포인트로 노출해야 한다.

`docker-compose.yml`에 다음과 같이 JMX Exporter 컨테이너를 추가한다:

```yaml
kafka-jmx-exporter:
  image: bitnami/jmx-exporter:latest
  ports:
    - "9404:9404"   # Prometheus가 접근할 포트
  environment:
    - JMX_PORT=9999
    - HTTP_PORT=9404
  depends_on:
    - kafka

```

### JMX Exporter란?
JMX (Java Management Extensions)는 자바 애플리케이션 내부 상태를 모니터링하기 위한 표준 인터페이스다.
Kafka도 자바 애플리케이션이라, 브로커 상태(토픽, 메시지 수, 네트워크 트래픽 등)를 JMX로 노출한다.
그러나 Prometheus는 JMX를 직접 읽지 못한다.
따라서 JMX Exporter는 이 지표를 Prometheus가 이해할 수 있는 HTTP `/metrics` 형식으로 변환해주는 도구다.

### 2. Prometheus 설정 수정

이제 Prometheus는 9092 대신 JMX Exporter가 열어둔 HTTP 포트(9404)를 모니터링 대상으로 지정해야 한다:

```yaml
- job_name: 'kafka'
  static_configs:
    - targets: ['kafka-jmx-exporter:9404']   # ✅ 올바른 설정
```

이 두 가지를 수정하면 Kafka 브로커 로그에 더 이상 InvalidReceiveException이 찍히지 않고,
Prometheus에서도 Kafka 관련 메트릭을 정상적으로 수집할 수 있게 된다.

---
title: "Kafka Docker 환경에서 Spark Streaming 연결 오류: Failed to create new KafkaAdminClient 해결하기"
author: jisu
date: 2025-09-29 12:00:00 +0900
categories: [Data Engineering]
tags: [Kafka, Zookeeper, Docker, Streaming, ETL]
pin: false
math: false
mermaid: false
comments: true
---

## 배경
Kafka를 Docker 기반으로 실행하다 보면,  
Spark Streaming과 연동 시 **Listener 설정 불일치** 때문에 접속 에러가 발생할 수 있다.

이번 글에서는 `StreamingQueryException: Failed to create new KafkaAdminClient` 에러를 다루고,  
**내부/외부 Listener 설정 차이**로 인해 생기는 문제와 해결 방법을 정리한다.  


## 문제 상황

Spark Streaming에서 Kafka를 읽으려 할 때,
다음과 같은 에러가 발생할 수 있다:

```
org.apache.spark.sql.streaming.StreamingQueryException: 
Failed to create new KafkaAdminClient
```

## 원인

Docker 환경에서는 내부와 외부의 네트워크 체계가 다르다.

- 컨테이너 내부에서는 같은 네트워크에 있는 서비스끼리 `컨테이너이름:포트`로 통신할 수 있다. (예: `kafka:9092`)
- 하지만 외부(EC2)에서는 컨테이너 네트워크를 직접 알 수 없고,
  Docker가 매핑해둔 호스트 포트(`localhost:29092`)를 통해 접속해야 한다.

Kafka Broker는 클라이언트가 접속했을 때 자기 주소를 알려주는데,  
이때 내부 전용 주소만 광고하면 외부에서는 접속할 수 없게 된다.  
즉, `listeners` / `advertised.listeners` 설정이 잘못돼 있으면  
컨테이너 내부에서는 정상적으로 되지만 외부에서는 실패한다.

문제가 되는 설정:

```yaml
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
```

- 컨테이너 내부(Spark, Producer)는 kafka:9092로 접속 가능 ✅
- 하지만 호스트(EC2, 로컬 PC 등)에서 실행할 경우 kafka라는 도메인을 찾을 수 없어 실패 ❌

## 💡 해결 방법

내부용과 외부용 Listener를 모두 설정해야 한다.

```yaml
environment:
  # 브로커가 실제 열어두는 포트
  - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,PLAINTEXT_HOST://:29092
  # 클라이언트에게 안내할 주소
  - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
  # Listener와 프로토콜 
  - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
  # 보안 설정 없이 통신 허용
  - ALLOW_PLAINTEXT_LISTENER=yes
```

이렇게 설정하면:

- 컨테이너 내부 → `kafka:9092`로 접속 가능
- 외부(호스트, EC2 등) → `localhost:29092`로 접속 가능

## 마무리
결국 이번 오류는 Docker 환경에서 내부와 외부 네트워크 주소가 다르다는 점 때문에 생긴 문제였다.
Kafka가 외부 클라이언트에게 잘못된 주소를 알려주면서 Spark가 접속에 실패한 것이다.

따라서 Docker 기반으로 Kafka를 쓸 때는 `listeners`와 `advertised.listeners`를
내부용과 외부용으로 구분해 명확히 설정하는 것이 안정적인 연결의 핵심이다.

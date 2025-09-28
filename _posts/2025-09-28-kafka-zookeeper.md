---
title: Docker Kafka 실행 시 InconsistentClusterIdException 에러 해결하기
author: jisu
date: 2025-09-28 1:00:00 +0900
categories: [Data-Engineering]
tags: [Kafka, Docker, Zookeeper, Troubleshooting]
pin: false
math: false
mermaid: false
comments: true
---

## Kafka 개발 환경에서 마주친 문제

ETL 파이프라인과 스트리밍 처리를 위해 Kafka를 Docker Compose로 띄우던 중,
컨테이너를 내렸다가 다시 올릴 때마다 아래와 같은 에러가 발생했다.

```text
kafka.common.InconsistentClusterIdException: 
The Cluster ID XXX-XXX doesn't match stored clusterId Some(XXX-XXX) in meta.properties. 
The broker is trying to join the wrong cluster. Configured zookeeper.connect may be wrong.
```

로그를 뜯어보니 Cluster ID 불일치 문제였다.

## 원인 분석: Cluster ID 충돌

Kafka 클러스터는 각자 고유한 Cluster ID를 가진다.  
이 값은 두 곳에 저장되는데, **Zookeeper**는 클러스터의 메타데이터와 Cluster ID를 관리하고,  
**Kafka Broker**는 로컬 데이터 디렉토리에 `meta.properties` 파일을 만들어 자신이 속한 Cluster ID를 기록한다. 

정상적으로 동작하려면 이 두 값이 서로 일치해야 한다.  
그런데 Docker Compose 환경에서는 컨테이너를 내렸다가 다시 올릴 때 문제가 생길 수 있다.  

`docker compose down` 명령을 실행하면 컨테이너만 삭제되고, 볼륨은 그대로 남는다.  
이 상태에서 Kafka 컨테이너를 새로 올리면, 브로커는 새로운 Cluster ID를 발급받는다.  
하지만 기존 볼륨에는 이전에 사용하던 Cluster ID가 그대로 남아 있기 때문에,  
결국 **Zookeeper가 가진 Cluster ID와 브로커가 기록한 Cluster ID가 충돌**하게 된다.  
이 불일치로 인해 `InconsistentClusterIdException` 에러가 발생하는 것이다.
---


## 해결 방법

### 1. 개발 환경 (간단하게 초기화)
테스트 환경이라면 데이터를 잃어버려도 큰 문제가 되지 않는다.  
따라서 가장 간단한 해결책은 아예 볼륨을 삭제하고 새로 시작하는 것이다:

```bash
docker compose down -v
docker compose up -d
```

`-v` 옵션으로 볼륨까지 삭제하면, 브로커의 `meta.properties`와 Zookeeper 데이터가 모두 초기화된다.
Kafka와 Zookeeper가 동일한 Cluster ID로 다시 합의하게 되면서 문제가 해결된다.

### 2. 운영 환경
운영 환경에서는 단순히 `-v` 옵션으로 볼륨을 날려버리면 안 된다.  
그렇게 하면 브로커 데이터와 토픽 로그가 전부 사라져 클러스터 전체가 무너질 수 있기 때문이다.  
운영 환경에서 핵심은 **Cluster ID를 일관성 있게 유지하는 것**이다.

구체적으로는 다음과 같은 방법을 따라야 한다:

1. Zookeeper의 Cluster ID 확인
  Zookeeper는 `/cluster/id` 노드에 현재 클러스터의 ID를 저장한다.  
  아래 명령어로 확인할 수 있다:
   
   ```bash
   zookeeper-shell.sh localhost:2181 get /cluster/id
   ```

2. 브로커의 `meta.properties` 확인
  각 브로커 데이터 디렉터리 안에는 meta.properties 파일이 있고, 이곳에 브로커가 속한 Cluster ID가 기록돼 있다:
  
  ```
  cluster.id=XXX
  broker.id=1
  ```
  
  이 값이 반드시 Zookeeper에 기록된 값과 같아야 한다.
  따라서 데이터를 보존해야 한다면, Zookeeper에 저장된 Cluster ID 기준으로 각 브로커의 `meta.properties`를 수정해야한다.

3. 새 브로커 추가 시 Cluster ID 고정하기 
  운영 환경에서는 브로커를 새로 추가할 때도 주의가 필요하다.  
  브로커가 새로 띄워지면서 임의의 Cluster ID를 생성하면,  
  이미 운영 중인 클러스터와 충돌이 발생할 수 있기 때문이다.  
  
  이런 경우에는 브로커를 시작할 때 `KAFKA_CLUSTER_ID` 환경 변수를 명시적으로 설정해주면 된다.  
  Zookeeper에 저장된 Cluster ID와 동일하게 지정하면, 새로운 브로커도 같은 클러스터에 정상적으로 합류할 수 있다.
  
  ```yaml
  environment:
    - KAFKA_CLUSTER_ID=XXX
  ```

### 참고
비슷한 문제를 겪은 사례가 Stack Overflow에도 있다.

> https://stackoverflow.com/questions/66452062/kafka-failing-to-start

이 경우에는 설치형 Kafka 환경이라 `/home/kafka/logs/meta.properties` 파일을 직접 찾아  
`cluster.id` 항목을 주석 처리한 뒤 Zookeeper와 Kafka를 재시작해서 해결했다고 한다.

나는 Docker 환경을 사용하고 있었기 때문에 파일을 직접 수정하기보다는  
`docker compose down -v`로 볼륨(`kafka_data`, `zookeeper_data`)을 초기화하는 방식이 더 간단했다.






   


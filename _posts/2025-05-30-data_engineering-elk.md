---
title: Docker로 로컬 ELK 스택 구축 및 Kibana 데이터 연결하기
author: jisu
date: 2025-05-30 11:33:00 +0900
categories: [Data Engineering]
tags: [ELK]
pin: false
math: true
mermaid: true
comments: true
---

## 실습 : ELK 스택을 활용한 실시간 도시데이터 분석

### 데이터셋 설명
서울시에서 제공하는 **실시간 도시데이터 API**를 이용해,
ELK 스택을 활용해 데이터 분석 및 시각화를 해보려고 한다.
이 데이터는 서울시 전역의 상권 소비 정보와 공영주차장 상태 정보를 포함한 실시간 정보다.
단순히 JSON 데이터를 Pandas로 분석하는 것도 가능하지만,
ELK 스택을 실습해보고 싶어서 이 데이터셋을 선택했다.

물론, ELK 스택은 원래 로그 데이터 분석에 많이 쓰인다.
그 이유는, ELK는

- **Elasticsearch**를 통해 대용량 데이터를 빠르게 검색하고,
- **Logstash**로 다양한 포맷의 로그 데이터를 처리하며,
- **Kibana**에서 실시간으로 시각화할 수 있어,
로그 모니터링 및 검색에 최적화된 구조이기 때문이다.

하지만 로그 데이터 대신 실시간 api를 통해 불러온 JSON 데이터를 선택한 이유는:

- 로그 데이터는 직접 생성하거나 실습용으로 구하기 어려운 반면,
- 서울시의 실시간 도시데이터는 누구나 쉽게 접근 가능하고,
- 시간 기반의 업데이트 및 다양한 업종/시설/상황 정보를 포함하고 있어,
ELK 스택의 분석 기능을 실습하기에 적합하다고 판단했기 때문이다.

즉, 이 데이터셋은 로그와 비슷하게

- 10분 단위로 갱신되는 timestamp를 기준으로 시간 축 기반 필터링이 가능하고,
- 다양한 카테고리(업종, 주차장 등)로 집계가 가능하다.
  
> 데이터 출처 : [서울시 실시간 도시데이터](https://data.seoul.go.kr/dataList/OA-21285/F/1/datasetView.do)

### 진행 흐름

전반적인 흐름은 다음과 같다.

```
서울시 실시간 도시데이터 API (상권 소비 + 공영주차장)
  ↓ (주기적 수집 + JSONL 형식 저장)
Logstash (logstash.conf 작성, JSON 파싱 및 timestamp 처리)
  ↓ (Elasticsearch에 저장, 검색/집계 처리)
Elasticsearch (데이터 저장소 역할)
  ↓ (검색/집계 결과 요청)
Kibana (Data View 생성, Discover에서 실시간 데이터 조회 및 시간 필터 적용)
  → 대시보드/시각화로 확장 예정
```


## Docker로 ELK 스택 로컬 설치
ELK 스택을 설치하기 위해 Docker를 사용했다.
Docker를 쓰면, 로컬 환경에서도 복잡한 설치 없이 간단하게 ELK를 실행할 수 있다.

ELK 스택은 이미 GitHub에 유명한 `docker-compose.yml` 저장소들이 많이 올라와 있다.
`git clone`만 해도 바로 실행 가능한 코드와 설정들이 준비되어 빠르고 간편하다.

그런데 나는 이 방식 대신 설정을 직접 해보기로 했다.
시간이 더 오래 걸리긴 하지만 배우는 과정이라 동작 원리를 좀 파악하고 싶었달까.

### Docker Compose 설정
아래는 ELK 스택을 구성하기 위한 `docker-compose.yml` 파일이다.

⭐ 여기서 중요한 점은:
- Elasticsearch와 Kibana는 Docker 이미지에 기본 설정이 포함되어 있어 별도 설정 없이 실행 가능하다.

- 하지만 **Logstash는 반드시 logstash.conf 파이프라인 설정 파일이 필요**하다!

- Logstash는 `/usr/share/logstash/pipeline` 폴더 안의 파일을 읽어 실행되므로, 반드시 `logstash/pipeline` 폴더 안에 logstash.conf를 넣어주어야 한다!

- 만약 이 파일이 없으면, Logstash 컨테이너는 실행되자마자 꺼진다.

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false  
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    container_name: kibana
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.0
    container_name: logstash
    ports:
      - "5044:5044"
      - "9600:9600"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

volumes:
  es_data:
    driver: local
```

### Logstash 파이프라인 설정
Logstash는 JSON 데이터를 읽어와 Elasticsearch로 전송하는 역할을 한다.

이를 위해 반드시 `logstash/pipeline/logstash.conf` 파일이 필요하다.

```python
input {
  file {
    path => "/usr/share/logstash/data/final_data_kst.jsonl"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => "json"
  }
}

filter {
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
    timezone => "Asia/Seoul"
  }

}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "parking_status"
  }

  stdout {
    codec => rubydebug
  }
}

```

### ELK Stack 실행해보기
설정 파일이 준비되면, 아래 명령어로 ELK Stack을 실행할 수 있다.

```bash
docker compose up -d
```

### 컨테이너 실행 확인
실행 중인 컨테이너 목록을 확인하려면 다음 명령어를 사용한다.

```bash
docker ps
```

정상적으로 실행되면 아래와 비슷한 결과가 나온다:

```bash
CONTAINER ID   IMAGE                                                 COMMAND                  CREATED         STATUS         PORTS                                            NAMES
1ef627c64eaf   docker.elastic.co/kibana/kibana:8.8.0                 "/bin/tini -- /usr/l…"   4 seconds ago   Up 2 seconds   0.0.0.0:5601->5601/tcp                           kibana
9e5cf999a4c8   docker.elastic.co/logstash/logstash:8.8.0             "/usr/local/bin/dock…"   4 seconds ago   Up 2 seconds   0.0.0.0:5044->5044/tcp, 0.0.0.0:9600->9600/tcp   logstash
851c8cf222e4   docker.elastic.co/elasticsearch/elasticsearch:8.8.0   "/bin/tini -- /usr/l…"   4 seconds ago   Up 3 seconds   0.0.0.0:9200->9200/tcp, 9300/tcp                 elasticsearch
```

elasticsearch, logstash, kibana 컨테이너가 모두 실행 중인 것을 확인할 수 있다.

### 🚨 발생한 문제 : `_jsonparsefailure`
Logstash 로그를 확인하자마자 다음과 같은 에러가 출력되었다:

```bash
"tags" => [ "_jsonparsefailure" ]
```

이 에러의 원인은 데이터 형식 때문이다.
Logstash의 file input 플러그인은 파일을 한 줄씩 읽으며, 각 줄을 독립적인 JSON 객체로 간주한다.
그런데 수집된 데이터가 JSON 배열 형태로 저장되어 있으면, Logstash는 이를 한 번에 읽지 못하고
중간에 잘린 상태의 줄을 읽게 되어 JSON 파싱에 실패한다.
예를 들어, 아래와 같이 줄바꿈과 들여쓰기가 포함된 배열 형태는 Logstash에서 문제가 발생한다:

```json
[
  {
    "timestamp": "2025-06-01T15:00:00",
    "area": "강남역",
    ...
  },
  {
    "timestamp": "2025-06-01T15:10:00",
    "area": "신촌",
    ...
  }
]
```

이런 구조는 Logstash가 처리할 수 없어 `_jsonparsefailure` 태그가 붙는다.

### 💡 해결법 : JSONL 포맷 사용
Logstash에서는 아래와 같은 JSON Lines (.jsonl) 형식으로 저장해야 정상적으로 파싱할 수 있다:

```json
{"timestamp": "2025-06-01T15:00:00", "area": "강남역", ...}
{"timestamp": "2025-06-01T15:10:00", "area": "신촌", ...}
```

즉, 한 줄에 하나의 JSON 객체만 있어야 하며, 줄바꿈 없이 저장되어야 한다.
이후 파일 형식을 `.jsonl`로 저장하고 `codec => json` 설정을 유지한 상태에서
Logstash를 재실행하면 정상적으로 적재된다.

#### ⚠ Logstash는 같은 파일을 한 번만 읽는다!
Logstash는 입력 파일의 경로 + inode 정보를 내부적으로 기억하기 때문에
내용이 바뀌더라도 파일명이 같으면 다시 읽지 않을 수 있다.

비록 sincedb_path => "/dev/null" 옵션으로 처음부터 읽게 설정하더라도,
파일명이 그대로면 여전히 캐시된 정보로 인해 무시될 수 있다.

해결 방법은 파일명을 바꿔주고, Logstash 설정 파일(logstash.conf)의 경로도 함께 수정한 다음, 컨테이너를 재시작하는 것이다.

```bash
mv final_data.jsonl final_data_kst.jsonl        # 파일명 변경
# logstash.conf 내 path 경로도 함께 수정
docker restart logstash                         # Logstash 재시작
```

이 과정을 거치면 Logstash는 해당 파일을 새로운 파일로 인식하여 다시 처음부터 읽게 된다.
**Logstash는 자동으로 재적재하지 않기 때문에, 파일명 변경 + 컨테이너 재시작은 필수이다!**

### Kibana에서 인덱스 패턴 (Data View) 생성
ELK 스택에 데이터가 잘 들어갔다면,
Kibana에서 데이터를 확인하기 위해 인덱스 패턴(Data View) 을 생성해야 한다.

`http://localhost:5601` 로 Kibana에 접속한 뒤
왼쪽 사이드바 메뉴에서 "Discover" → "Create data view" 를 클릭한다.

![image](https://github.com/user-attachments/assets/9efd1c94-1dae-45e2-a364-58dc612fdade)


Data View 이름은 예를 들어 parking_status로 지어주고,
인덱스 패턴도 동일하게 parking_status로 설정한다.
이렇게 하면 Elasticsearch 인덱스와 연결된다.

![image](https://github.com/user-attachments/assets/87cbaadb-d0c6-4cf7-8f84-e3d871007f59)


### ⭐ Time field 설정 (`@timestamp`)
Data View를 만들 때 Time field를 선택하라는 옵션이 나온다.
이 필드는 Kibana의 시간 기반 필터링, 시각화, 정렬 등의 기준이 된다.

나는 Logstash 설정에서 timestamp 필드를 ISO8601 형식으로 파싱하여
Elasticsearch가 이해할 수 있는 표준 시간 필드인 @timestamp로 변환했다.
따라서 반드시 @timestamp 필드를 Time field로 지정해야 한다.

또한, Kibana는 기본적으로 UTC(세계 표준시) 를 기준으로 시간 데이터를 해석하기 때문에
Logstash에서 timezone => "Asia/Seoul"을 명시해 한국 시간(KST)으로 변환해주는 것이 중요하다.

이 설정이 없으면 Kibana에 표시되는 시간과 실제 수집 시각이 어긋나
데이터가 보이지 않거나 시간 필터가 이상하게 작동할 수 있다.

```conf
filter {
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
    timezone => "Asia/Seoul"
  }
}
```

이 설정이 제대로 되어 있다면, Kibana에서도 @timestamp 필드를 자동 인식할 수 있다.




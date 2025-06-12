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

## 실습 : ELK 스택을 활용한 실시간 주차장 및 상권 데이터 분석

### 데이터셋 설명
서울시에서 제공하는 **실시간 공공데이터 API**를 이용해,
ELK 스택을 활용해 데이터 분석 및 시각화를 해보려고 한다.

- **상권 정보**: 서울시의 실시간 상권별 유동인구, 소비 현황 등을 포함한 **실시간 도시데이터 API**
- **주차장 정보**: 서울시가 운영하는 **시영주차장의 실시간 주차 가능 대수 API**

원래는 CSV 파일을 Logstash로 처리하는 구조를 생각했지만,  
실시간 API 기반 데이터의 갱신성을 살리기 위해 구조를 변경했다.  
현재는 Python 스크립트로 API를 직접 호출하고,  
그 결과를 **Elasticsearch에 바로 전송**하여 Logstash 없이도 실시간 분석 파이프라인을 구성했다.

이처럼 **ELK 스택**은 원래 로그 데이터 분석에 많이 쓰이지만,  
아래와 같은 이유로 실시간 도시데이터를 선택했다:

- 로그 데이터는 직접 생성하거나 실습용으로 구하기 어려운 반면,
- 서울시의 실시간 도시데이터는 누구나 접근 가능하고,
- 시간 기반의 업데이트, 다양한 시설 및 상황 정보를 포함하고 있어
- ELK 스택의 시계열 필터링, 대시보드 기능을 실습하기에 적합하다.

실제로 이 데이터는 로그와 비슷하게:

- 5~10분 단위로 갱신되는 `timestamp`를 기반으로 시간 축 필터링이 가능하고,
- **업종, 상권, 주차장 등 다양한 카테고리로 분석**할 수 있다.

> 데이터 출처:  
> - [서울시 시영주차장 실시간 주차대수](https://data.seoul.go.kr/dataList/OA-21709/S/1/datasetView.do)  
> - [서울시 실시간 도시데이터](https://data.seoul.go.kr/dataList/OA-21285/F/1/datasetView.do)
 
### 진행 흐름

전반적인 흐름은 다음과 같다.

```
서울시 실시간 도시데이터 API (상권 소비 + 시영주차장 정보)
  ↓ (Python 스크립트로 주기적 수집 + 좌표 변환)
Kakao API (주소 → 위도/경도 변환)
  ↓ (Geo 좌표 포함하여 데이터 구조화)
Elasticsearch (실시간 데이터 저장 및 색인)
  ↓ (검색/집계 결과 요청)
Kibana (Data View 생성, Discover에서 실시간 데이터 조회 및 시간 필터 적용)
  → 대시보드/시각화로 확장
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
실제 데이터 수집 및 적재는 Python으로 수행했기 때문에 Logstash는 활용하지 않았지만,  
ELK 스택의 전체 구조를 이해하고자 Logstash도 함께 설치해보았다.

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
851c8cf222e4   docker.elastic.co/elasticsearch/elasticsearch:8.8.0   "/bin/tini -- /usr/l…"   4 seconds ago   Up 3 seconds   0.0.0.0:9200->9200/tcp, 9300/tcp                 elasticsearch
```

elasticsearch, kibana 컨테이너가 모두 실행 중인 것을 확인할 수 있다.

### Elasticsearch 인덱스 생성

데이터 수집 및 전처리는 별도 Jupyter Notebook(`eda.ipynb`)에서 EDA와 `folium`을 활용한 지도 시각화를 통해  
데이터 구조나 이상치 등을 먼저 확인한 뒤,  
Python 스크립트를 통해 Elasticsearch에 총 **3개의 인덱스**를 생성하고 데이터를 업로드했다.

| 인덱스 이름 | 설명 |
|-------------|------|
| `seoul_parking` | 시영주차장 실시간 데이터. 위치 정보(`geo_point`), 가용률, 혼잡도 상태, 요일 정보 포함 |
| `seoul_commercial` | 상권 중심 위치 및 반경 300m 내 주차장 수, 평균 주차 가용률 등 공간 기반 분석 데이터 |
| `seoul_commercial_categories` | 업종/카테고리 기반의 상권 소비 정보 요약 데이터 |

각 인덱스는 Elasticsearch API를 통해 **Python 코드에서 직접 생성**하였으며,
데이터는 `helpers.bulk()`를 사용하여 대량 업로드하였고,  
문서의 `_id`는 `search_keyword_수집시간` 형태로 생성해 중복 삽입을 방지했다.

### Kibana에서 인덱스 패턴 (Data View) 생성

ELK 스택에 데이터가 잘 들어갔다면,
Kibana에서 데이터를 확인하기 위해 인덱스 패턴(Data View) 을 생성해야 한다.

`http://localhost:5601` 로 Kibana에 접속한 뒤
왼쪽 사이드바 메뉴에서 "Discover" → "Create data view" 를 클릭한다.

Data View 이름은 실제 Elasticsearch 인덱스 이름과 동일하게  
예: `seoul_parking`, `seoul_commercial`, `seoul_commercial_categories` 등으로 지정한다.
이렇게 하면 각 인덱스와 연결된 데이터를 Kibana에서 시각적으로 탐색할 수 있다.

![image](https://github.com/user-attachments/assets/39214b33-a179-4745-ac3f-84b66d9cd29d)

#### 🚨 Kibana에서 데이터가 보이지 않았던 문제

처음 timestamp 컬럼을 생성할 때 단순히` datetime.now()`만 사용했는데,
Kibana에서 Discover 탭에 데이터가 아무것도 보이지 않는 현상이 발생했다.

하지만 Elasticsearch에는 분명히 데이터가 잘 들어가 있었기 때문에,
문제는 시간 필터링 기준이 잘못 설정된 것임을 알 수 있었다.

Kibana는 기본적으로 **UTC(세계 표준시)** 기준으로 시간을 해석한다.
반면 Python에서 `datetime.now()`를 사용할 경우 시스템 로컬 시간이 적용되는데,
내 환경은 KST(한국 표준시)이므로 Kibana에서 데이터가 UTC 기준으로 9시간 밀려 해석된 것이다.
결과적으로 시간 필터 범위 밖의 값으로 간주되어 데이터가 표시되지 않는 것이다.

### 💡 해결 방법: timestamp에 KST(한국 시간) 명시하기

이 문제를 해결하기 위해 Python에서 `timestamp` 생성 방식을 다음과 같이 변경했다:

```python
from datetime import datetime
import pytz

tz = pytz.timezone("Asia/Seoul")
now_kst = datetime.now(tz)
```

이렇게 하면 `timestamp` 값에 +09:00(KST) 시간대 정보가 포함되어
Kibana의 시간 필터와 정확히 일치하게 되어 데이터가 잘 표시된다.

### ⭐ Time field 설정 (`timestamp`)
이제 Kibana에서 Data View를 생성할 때,
Time field로 반드시 timestamp 컬럼을 선택해줘야 한다.

이 필드는 Kibana의 시간 기반 필터링, 정렬, 시각화 기준이 되며,
정확한 시간 기반 탐색을 위해 꼭 지정해주어야 한다.

![image](https://github.com/user-attachments/assets/29955f8d-afaa-4847-8cba-a77b497fabc4)




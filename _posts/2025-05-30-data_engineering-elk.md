---
title: Docker로 ELK 스택 로컬 설치 및 Logstash로 데이터 적재
author: jisu
date: 2025-05-30 11:33:00 +0900
categories: [Data Engineering]
tags: [ELK]
pin: false
math: true
mermaid: true
comments: true
---

## 실습 : ELK 스택을 활용한 대용량 CSV 데이터 분석

### 데이터셋 설명
공공 데이터 포털에서 제공하는 2023년 서울교통공사_역별 일별 시간대별 승하차인원 정보 데이터셋을 이용해,
ELK 스택을 활용해 데이터 분석 및 시각화를 해보려고 한다.
이 데이터는 시간대별, 역별, 승하차 구분별로 약 19만 건에 달하는 데이터이다.
단순히 CSV 파일을 Pandas로 분석하는 것도 가능하지만,
ELK 스택을 실습해보고 싶어서 이 데이터셋을 선택했다.

물론, ELK 스택은 원래 로그 데이터 분석에 많이 쓰인다.  
그 이유는, ELK는

- **Elasticsearch**를 통해 대용량 데이터를 빠르게 검색하고,
- **Logstash**로 다양한 포맷의 로그 데이터를 처리하며,
- **Kibana**에서 실시간으로 시각화할 수 있어,
로그 모니터링 및 검색에 최적화된 구조이기 때문이다.

하지만 로그 데이터 대신 CSV 데이터를 선택한 이유는:

- 로그 데이터는 직접 생성하거나 실습용으로 구하기 어려운 반면,
- 공공 데이터 포털의 CSV 데이터는 누구나 쉽게 구할 수 있고,  
- **시간대별, 역별, 승하차 구분 등 로그 데이터와 유사한 다차원 필터링 및 집계 구조를 가지고 있어,  
ELK 스택의 분석 기능을 실습하기에 적합**하다고 판단했기 때문이다.

즉, 이 데이터셋은 로그와 비슷하게

- 시간(time)이라는 축을 기반으로 필터링할 수 있고,  
- 다양한 카테고리(역별, 호선별, 승하차 구분별)로 집계가 가능하다.
> 데이터 출처 : [서울교통공사_역별 일별 시간대별 승하차인원 정보](https://www.data.go.kr/data/15048032/fileData.do)

### 진행 흐름

전반적인 흐름은 다음과 같다.

```
CSV 데이터 (서울교통공사_지하철 데이터)
  ↓ (읽어오기 + 가공)
Logstash (logstash.conf 작성, CSV 컬럼 파싱 및 타입 변환, 날짜 필드 처리)
  ↓ (Elasticsearch에 저장, 검색/집계 처리)
Elasticsearch (데이터 저장소 역할)
  ↓ (검색/집계 결과 요청)
Kibana (Data View 생성, Discover에서 데이터 조회, 시간 필터 조정)
  → 대시보드/시각화로 보여줌 (시간대별/역별/호선별 혼잡도 분석 예정)
```


## Docker로 ELK 스택 로컬 설치
ELK 스택을 설치하기 위해 Docker를 사용했다.
Docker를 쓰면, 로컬 환경에서도 복잡한 설치 없이 간단하게 ELK를 실행할 수 있다.

ELK 스택은 이미 GitHub에 유명한 docker-compose.yml 저장소들이 많이 올라와 있다.
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
Logstash는 CSV 데이터를 읽어와 Elasticsearch로 전송하는 역할을 한다.
이를 위해 반드시 `logstash/pipeline/logstash.conf` 파일이 필요하다.

`logstash.conf`에는

- CSV 파일의 위치 (input),

- CSV 컬럼 파싱 방법 (filter),

- 데이터 전송 대상 (output)을 설정한다.

예를 들어, `logstash.conf`는 다음과 같이 작성할 수 있다.

```python
# Logstash 파이프라인 설정 파일
# CSV 데이터를 읽어서 Elasticsearch로 전송하는 역할

input {
  file {
    path => "/usr/share/logstash/data/subway_data.csv"  # Logstash 컨테이너 안에 있는 CSV 파일 경로
    start_position => "beginning"                       # 파일의 맨 처음부터 읽기 (최초 실행 시)
    sincedb_path => "/dev/null"                          # 파일 읽기 기록 저장 안 함 (매번 처음부터 읽게 함)
  }
}

filter {
  csv {
    separator => ","                                     # CSV 구분자 (콤마)
    columns => [                                         # CSV 파일의 헤더(열 이름) 지정
      "연번", "수송일자", "호선", "역번호", "역명", "승하차구분",
      "06시이전", "06-07시간대", "07-08시간대", "08-09시간대", "09-10시간대",
      "10-11시간대", "11-12시간대", "12-13시간대", "13-14시간대", "14-15시간대",
      "15-16시간대", "16-17시간대", "17-18시간대", "18-19시간대", "19-20시간대",
      "20-21시간대", "21-22시간대", "22-23시간대", "23-24시간대", "24시이후"
    ]
  }

  mutate {                                               # 데이터 타입 변환 (문자열 → 숫자)
    convert => { "연번" => "integer" }
    convert => { "역번호" => "integer" }
    convert => { "06시이전" => "integer" }
    convert => { "06-07시간대" => "integer" }
    convert => { "07-08시간대" => "integer" }
    convert => { "08-09시간대" => "integer" }
    convert => { "09-10시간대" => "integer" }
    convert => { "10-11시간대" => "integer" }
    convert => { "11-12시간대" => "integer" }
    convert => { "12-13시간대" => "integer" }
    convert => { "13-14시간대" => "integer" }
    convert => { "14-15시간대" => "integer" }
    convert => { "15-16시간대" => "integer" }
    convert => { "16-17시간대" => "integer" }
    convert => { "17-18시간대" => "integer" }
    convert => { "18-19시간대" => "integer" }
    convert => { "19-20시간대" => "integer" }
    convert => { "20-21시간대" => "integer" }
    convert => { "21-22시간대" => "integer" }
    convert => { "22-23시간대" => "integer" }
    convert => { "23-24시간대" => "integer" }
    convert => { "24시이후" => "integer" }
  }

  date {                                                # 날짜 타입 필드 변환 (문자열 → 날짜)
    match => ["수송일자", "yyyy-MM-dd"]                  # '수송일자' 컬럼의 날짜 형식 지정 (연-월-일)
    target => "수송일자"                                 # 변환된 날짜를 저장할 필드 (원본 필드에 덮어쓰기)
  }
}

output {
  elasticsearch {                                       # Elasticsearch로 데이터 전송 설정
    hosts => ["http://elasticsearch:9200"]              # Elasticsearch 서버 주소 (도커 환경에서 컨테이너 이름으로 접근)
    index => "subway-data"                              # 데이터를 저장할 Elasticsearch 인덱스 이름
  }

  stdout { codec => rubydebug }                         # 로그를 콘솔에 출력 (디버깅용)
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

### Logstash 컨테이너: CSV 파일 적재 및 종료
Logstash는 컨테이너 내부의 /usr/share/logstash/data/ 경로에서 CSV 파일을 읽어온다.
따라서, 로컬의 log_analysis/data/subway_data.csv 파일을 Logstash 컨테이너 내부로 복사해줘야 한다.

```bash
docker cp ./data/subway_data.csv logstash:/usr/share/logstash/data/subway_data.csv
```

그런데 컨테이너가 꺼졌다는 에러가 뜬다.

```bash
docker exec -it logstash ls /usr/share/logstash/data/
Error response from daemon: container 9e5cf999a4c8fa168e9533c57bb6066985fffe532548407b7b6a690972a2a3eb is not running
```

Logstash는 file input 플러그인으로 CSV를 읽고,

데이터를 다 처리한 후에는 더 이상 할 일이 없으니까 컨테이너가 종료된다.

sincedb_path => "/dev/null" 옵션 덕분에 파일을 매번 처음부터 읽지만, 파일을 다 처리하면 종료되는 게 기본 동작이다.

파일을 다시 읽어들이려면 Logstash 컨테이너를 다시 실행해야 한다.

```bash
docker compose up -d logstash
```

성공하면 아래처럼 메시지가 출력된다.

```bash
Successfully copied 24.9MB to logstash:/usr/share/logstash/data/subway_data.csv
```

### Elasticsearch에 데이터가 잘 들어갔는지 확인

브라우저에서 아래 URL에 접속했을 때, JSON 형태의 데이터가 나오면 성공적으로 적재된 것이다.

```
http://localhost:9200/subway-data/_search?size=5
```

### Kibana에서 인덱스 패턴 (Data View) 생성
ELK 스택에 데이터가 잘 들어갔다면,
Kibana에서 데이터를 확인하기 위해 인덱스 패턴(Data View) 을 생성해야 한다.

http://localhost:5601 로 Kibana에 접속하고 `Create data view`를 클릭한다.

![image](https://github.com/user-attachments/assets/c68a6fdd-3702-44c3-b2b9-facd1c6bc197)


Data View의 이름을 `subway-data`로 지어주고 인덱스 패턴 역시 같게 하면 Elasticsearch 인덱스와 연결이 된다.

![image](https://github.com/user-attachments/assets/2551580a-ebd1-47b5-a1bd-3f5324a85d77)


데이터셋의 날짜 열인 수송일자 컬럼을 Time field로 설정한다.
이게 Kibana의 시간 필터 기준이 된다.

#### 🚨 Kibana에서 데이터가 안 보이는 이유 : Time field 설정 문제 해결하기
만약 수송일자가 보이지 않는다면:

Logstash 설정에서 date 필터가 잘 적용되었는지 확인해야한다.

수송일자 필드가 @timestamp처럼 날짜 필드로 인식되지 않으면 시간 필터가 적용되지 않는다.
Data View가 생성되면, Discover 화면에서 데이터 조회 가능하다.

하지만 처음에는 Kibana의 시간 필터가 현재 날짜 기준으로 되어 있어 데이터가 안 보일 수 있다.
이럴때는 시간 필터를 데이터에 맞게 변경해주면 아래와 같이 데이터가 보인다.

![image](https://github.com/user-attachments/assets/9e41609b-6d53-4d4f-a33f-19957739b61a)



---
title: Prometheus로 EC2 서버 자원 모니터링하기 (Node Exporter + Grafana 연동)
author: jisu
date: 2025-10-07 00:00:00 +0900
categories: [Data Engineering]
tags: [Prometheus, Node Exporter, Grafana, EC2]
pin: false
math: false
mermaid: false
comments: true
---

## 배경  
Prometheus로 Kafka와 Spark 애플리케이션을 모니터링한 뒤,  
이번에는 EC2 서버 자체의 자원(CPU, 메모리, 디스크, 네트워크 등)을 모니터링하기 위해  
**Node Exporter**를 설치하고 **Grafana**와 연동했다.  

이 과정에서 **포트 충돌**, **systemd 등록 오류**, **Grafana 대시보드 설정** 등  
실무에서도 자주 겪는 이슈를 정리했다.

## 1️⃣ Node Exporter 설치

Node Exporter는  

> [Prometheus Node Exporter 공식 릴리스 페이지](https://github.com/prometheus/node_exporter/releases/tag/v1.9.1)

아래 명령어로 최신 버전을 내려받고 압축을 해제했다.

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.9.1.linux-amd64.tar.gz
cd node_exporter-1.9.1.linux-amd64
```

이제 간단히 실행해보면 된다.

```bash
./node_exporter &
```

정상적으로 실행되면 PID가 출력되고,
브라우저에서 아래 주소로 접속하면 서버의 각종 메트릭이 표시된다.

```
http://<EC2_IP>:9100/metrics
```

![Prometheus Node Exporter 확인](/assets/img/posts/node-exporter-metrics.png)

Prometheus에서 수집할 수 있는 다양한 지표들이 확인된다.

## 2️⃣ Prometheus에 Node Exporter 등록

Prometheus가 Node Exporter의 데이터를 수집하려면
`prometheus.yml` 설정 파일에 새로운 타겟(target) 을 추가해야 한다.

```yaml
- job_name: 'node_exporter'
  static_configs:
    - targets: ['<EC2_IP>:9100']
```

수정 후 Prometheus를 재시작한다.

```bash
docker restart prometheus
```

정상적으로 연결됐다면 `http://<prometheus_ip>:9090/targets` 에서
node_exporter가 UP 상태로 표시된다.

![Prometheus Node Exporter Up](/assets/img/posts/prometheus-kafka-spark-up.png)


## 3️⃣ Node Exporter를 systemd 서비스로 등록

지금처럼 단순히 `./node_exporter &`로 실행하면
터미널을 닫거나 EC2를 재부팅할 때 자동으로 꺼진다.
항상 백그라운드에서 동작하도록 하려면 systemd 서비스로 등록하는 것이 좋다.

아래와 같이 서비스 파일을 생성했다.

```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=ubuntu
ExecStart=/home/ubuntu/projects/coin_stream/node_exporter-1.9.1.linux-amd64/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```


> ⚠️ 주의할점은 ExecStart 경로는 본인의 Node Exporter 설치 위치로 수정해야 한다.

그 다음, systemd를 갱신하고 서비스로 등록한다.

```
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### ⚠️ 포트 충돌 에러 

그런데 서비스를 실행할 때 가끔 다음과 같은 에러가 발생할 수 있다.

```
ExecStart=... (code=exited, status=1/FAILURE)
```

이는 대부분 **9100 포트를 이미 다른 프로세스가 사용 중인 경우**다.  
예를 들어, 앞서 `./node_exporter &`로 백그라운드에서 실행했던 프로세스가  
아직 종료되지 않은 상태라면 이런 충돌이 발생한다.

이럴 땐 현재 포트를 점유 중인 프로세스를 먼저 확인하면 된다.

```bash
sudo lsof -i :9100
```

결과 예시는 다음과 같다.

```
COMMAND      PID   USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
node_expo 498151 ubuntu    3u  IPv6 1932525363      0t0  TCP *:9100 (LISTEN)
```

이전 프로세스가 여전히 살아 있는 것을 확인했으면,
아래 명령어로 종료한 뒤 다시 서비스를 시작한다.

```bash
pkill node_exporter
sudo systemctl start node_exporter
```

이후 정상적으로 실행됐는지 확인하려면 다음 명령어를 입력한다.

```bash
systemctl status node_exporter
```

정상 상태일 경우 아래와 같은 메시지가 표시된다.

```
Active: active (running)
```

이제 EC2를 재부팅해도 Node Exporter가 자동으로 실행된다.

## 4️⃣ Grafana에서 시각화

Prometheus에 수집된 서버 메트릭은 Grafana에서 쉽게 시각화할 수 있다.
Grafana → Dashboard → Import 메뉴에서
공식 Node Exporter 대시보드 ID인 1860을 입력한다.

![Grafana에서 Node Exporter 대시보드 가져오기](/assets/img/posts/grafana-node-import.png)


아래 사진과 같이 자동으로 CPU, Memory, Disk, Network 등의 패널이 생성된다.

![Grafana Node Exporter 대시보드 예시 화면](/assets/img/posts/grafana-node-dashboard.png)

이제 Prometheus와 Grafana를 통해 EC2 서버의 자원 상태를 한눈에 모니터링할 수 있게 되었다.

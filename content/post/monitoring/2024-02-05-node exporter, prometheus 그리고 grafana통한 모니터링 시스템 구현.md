---
title: node exporter, prometheus 그리고 grafana통한 모니터링 시스템 구현
description:
date: 2024-02-05
summary: node exporter, prometheus 그리고 grafana통한 모니터링 시스템 구현
---

## 1. 환경

- vmware workstation 17 player
  - os: debian gnu/linux 12 (bookworm)
  - kernel: linux 6.1.0-17-amd64
- package
  - prometheus-node-exporter: 1.5.0-1+b6
  - prometheus: 2.42.0+ds-5+b5
  - grafana: 10.3.1

## 2. 구축

### 2-1. prometheus, node exporter 설치

```bash
sudo apt install -y prometheus-node-exporter prometheus

systemctl status prometheus-node-exporter prometheus

# 옵션
sudo systemctl enable prometheus-node-exporter prometheus
sudo systemctl start prometheus-node-exporter prometheus
```

>**정보**  
>apt로 prometheus 설치 시 alertmanager, prometheus 그리고 prometheus-node-exporter에 대한 구성이 작성되어 있습니다. 만약 prometheus-node-exporter에 대한 구성이 작성되어 있지 않다면 다음과 같이 작성해 주세요.
>
>```bash
>sudo nano /etc/prometheus/prometheus.yml
>
>###
>scrape_configs:
>  - job_name: node
>    # If prometheus-node-exporter is installed, grab stats about the local
>    # machine by default.
>    static_configs:
>      - targets: ['localhost:9100']
>###
>
>systemctl reload prometheus.service
>```

### 2-2. grafana 설치

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt update

sudo apt install -y grafana

systemctl status grafana-server

# 옵션
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### 2-3. grafana 설정

grafana 관리자의 초기 email or username, password는 admin입니다.

1. basic의 add your first data source 클릭
2. prometheus 설정
    - 저는 connection에 prometheus의 ip와 port만 추가했습니다.
3. basic의 create your first dashboard
    - 저는 import a dashboard를 통해 node exporter full을 사용했습니다.

## 3.참고문헌

- github: [prometheus](https://github.com/prometheus/prometheus)
- github: [node_exporter](https://github.com/prometheus/node_exporter)
- grafana documentation: [install grafana on debian or ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
- grafana dashboards: [node exporter full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)

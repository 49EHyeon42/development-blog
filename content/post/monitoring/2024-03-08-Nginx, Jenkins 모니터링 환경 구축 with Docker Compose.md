---
title: Nginx, Jenkins 모니터링 환경 구축 with Docker Compose
description:
date: 2024-03-07
summary: Nginx, Jenkins 모니터링 환경 구축 with Docker Compose
---

## 1. 서론

지금까지 인프라 구성을 위해 APT로 패키지를 설치하고 사용해 왔습니다. 이는 일관된 환경 유지가 어려웠고, 설정에 많은 시간을 들여야 했습니다. Docker와 Docker Compose를 사용함으로써 빠르게 이식하고 설정이 용이하게 될 수 있기에, 기존의 Monitoring PC, Nginx PC, Jenkins PC를 Docker Compose로 구축해 보겠습니다.

## 2. 환경

- VMware Workstation 17 Player
  - Monitoring PC
    - OS: Debian 12
    - IP: 192.168.100.10
    - Docker Image: [prometheus](https://hub.docker.com/r/prom/prometheus), [alertmanager](https://hub.docker.com/r/prom/alertmanager), [grafana](https://hub.docker.com/r/grafana/grafana)
  - Nginx PC
    - OS: Debian 12
    - IP: 192.168.100.11
    - Docker Image: [node-exporter](https://hub.docker.com/r/prom/node-exporter), [nginx](https://hub.docker.com/_/nginx), [nginx-prometheus-exporter](https://hub.docker.com/r/nginx/nginx-prometheus-exporter)
  - Jenkins PC
    - OS: Debian 12
    - IP: 192.168.100.12
    - Docker Image: [jenkins](https://hub.docker.com/r/jenkins/jenkins)([prometheus metrics](https://plugins.jenkins.io/prometheus/) 포함)

nginx-prometheus-exporter에 CPU, memory 관련 metric이 없어서 node-exporter도 추가했습니다.

## 3. 구축

### 3-1. Monitoring PC

#### 3-1-1. 디렉터리 구조

```txt
.
├── alertmanager
│   └── alertmanager.yaml
├── prometheus
│   ├── rules
│   └── prometheus.yaml
└── docker-compose.yaml
```

#### 3-1-2. docker-compose.yaml

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:main
    container_name: prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus/prometheus.yaml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    restart: unless-stopped
  alertmanager:
    image: prom/alertmanager:main
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager/alertmanager.yaml:/etc/alertmanager/alertmanager.yml
    restart: unless-stopped
  grafana:
    image: grafana/grafana:main
    container_name: grafana
    ports:
     - 3000:3000
    volumes:
      - grafana-storage:/var/lib/grafana
    restart: unless-stopped
volumes:
  prometheus-data:
  grafana-storage: {}
```

#### 3-1-3. alertmanager.yaml

```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

#### 3-1-4. prometheus.yaml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - rules/*.yaml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
scrape_configs:
  - job_name: "worker_1_node-exporter"
    static_configs:
      - targets: ["192.168.100.10:9100"]
scrape_configs:
  - job_name: "worker_1_nginx"
    static_configs:
      - targets: ["192.168.100.10:9113"]
scrape_configs:
  - job_name: "worker_2_jenkins"
    metrics_path: /prometheus
    static_configs:
      - targets: ["192.168.100.10:8080"]
```

### 3-2. Nginx PC

#### 3-2-1. 디렉터리 구조

```txt
.
├── nginx
│   ├── conf.d
│   │   └── default.conf
│   └── nginx.conf
└── docker-compose.yaml
```

#### 3-2-2. docker-compose.yaml

```yaml
version: '3.8'
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    # port -> 9100
    volumes:
      - /:/host:ro,rslave
    restart: unless-stopped
    command:
      - --path.rootfs=/host
    network_mode: host
    pid: host
  nginx:
    image: nginx:stable-alpine-slim
    container_name: nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
    restart: unless-stopped
  nginx-prometheus-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-prometheus-exporter
    ports:
      - 9113:9113
    depends_on:
      - nginx
    restart: unless-stopped
    command: --nginx.scrape-uri=http://nginx/stub_status
```

#### 3-2-3. nginx.conf

```conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

#### 3-2-4. default.conf

```conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # nginx stub_status 추가
    location /stub_status {
        stub_status;
    }
}
```

### 3-3. Jenkins PC

#### 3-3-1. 디렉터리 구조

```txt
.
├── jenkins_home
└── docker-compose.yaml
```

`touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied` 에러가 발생할 수 있습니다. `sudo mkdir jenkins_home`, `sudo chown 1000 jenkins_home` 명령으로 해결할 수 있습니다.

#### 3-3-2. docker-compose.yaml

```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    ports:
      - 8080:8080
    volumes:
      - ./jenkins_home:/var/jenkins_home
    restart: unless-stopped
```

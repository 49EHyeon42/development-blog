---
title: Raspberry Pi 서버 구축
description:
date: 2024-02-01
summary: "Raspberry Pi 5 서버 구축"
---

## 개요

개발 블로그와 jenkins 서버를 활용하기 위해 여러 방안을 모색하고 있었습니다. 구형 노트북이나 클라우드 플랫폼은 생각보다 비싼 유지비용 때문에 망설이게 되었습니다. 대학 생활 중 활용한 적이 있는 라즈베리 파이를 떠올리게 되었고, 이를 활용하면 더 저렴하게 서버를 구축할 수 있다 생각해 라즈베리 파이로 서버를 구축하게 되었습니다.

이 프로젝트에는 3가지 목표가 있습니다.

1. hugo 자동 빌드, 배포
2. nginx 설정 및 활용 방법 익히기
3. jenkins 설정 및 활용 방법 익히기

이 프로젝트에서는 SSG(Static Site generator) 중 하나인 hugo를 통해 블로그를 개설, jenkins를 통해 자동 빌드-배포, nginx를 통해 static content(hugo)를 제공하는 것을 목표로 삼았습니다.

## 환경

- Raspberry Pi OS Lite (64-bit)
  - release : 2023-12-11
- Nginx
  - nginx/1.22.1
- Jenkins
  - 2.426.3
- Go
  - 1.20.13
- Hugo
  - v0.118.2

## 구축

라즈베리 파이 운영체제를 설치하는 과정은 생략합니다. [공식 홈페이지](https://www.raspberrypi.com/software/)를 확인해 주세요.

기본 설정 전 패키지 업데이트, 업그레이드를 진행해 주세요.

```shell
sudo apt update && sudo apt upgrade -y && sudo apt dist-upgrade -y
```

라즈베리 파이 기본 언어가 `en_UG.UTF-8` 로 되어 있었습니다. 저는 `sudo dpkg-reconfigure locales` 를 통해 `en_US-UTF.8` 로 변경했습니다. 만약 한글을 원한다면 폰트도 같이 설치해 주세요.

### 1. 기본 설정

제가 운영하는 서버에서 가장 중요한 것을 꼽으라 한다면 안정성과 보안을 뽑겠습니다. 특히 보안은 실제 테스트로 서버 구현 중 SSH 무작위 대입 공격(SSH Brute Attack)을 당했기 때문에 조금 더 경각심을 가지고 있습니다.

> btmp(Bad Login Attempts), wtmp(Successful Logins)  
> `sudo last -f /var/log/btmp` 와 `sudo last -f /var/log/wtmp` 로 접속 정보를 확인할 수 있습니다.

먼저 root에 관한 설정입니다. 라즈베리 파이를 처음 설치하면 root 계정의 비밀번호를 알 수 없지만 비밀번호가 활성화 되어있고, 셸 또한 활성화 되어있습니다. 저는 root 계정을 사용하지 않을 것 이기 때문에 모두 비활성화했습니다.

```shell
# root 비밀번호 비활성화
sudo passwd -l root

# root 셸 비활성화
sudo nano /etc/passwd
    # root:x:0:0:root:/root:/bin/bash를 root:x:0:0:root:/root:/usr/sbin/nologin로 변경
```

> 1. `sudo passwd -l root` 사용 시 /etc/shadow 파일 중 `root:/생략` 부분이 `root:!/생략` 로 변경됩니다. 이렇게 되면 비밀번호로 접속할 수 없지만, SSH 공개 키가 있는 경우 접속할 수 있습니다.
> 2. `/bin/bash` 를 `/usr/sbin/nologin` 로 변경한다면 root 계정은 셸을 사용할 수 없음으로 SSH 또한 접근할 수 없습니다.

다음은 SSH 설정입니다. 무작위 대입 공격을 당한 원인을 생각해 보면 기본 포트 22번 사용과 비밀번호 인증 사용인 것 같습니다. SSH 접근 포트를 변경하고, 비밀번호 인증 대신 공개 키 인증 방식을 채택했습니다. `/etc/ssh/shhd_config` 중 변경한 부분은 다음과 같습니다.

```shell
Port 숫자 # 사용하지 않는 숫자로 변경

PermitRootLogin no # root 접근 거부

PubkeyAuthentication yes # 공개 키 인증 허용

AuthorizedKeysFile .ssh/authorized_keys # 사용자 공개 키 위치

PasswordAuthentication no # 비밀번호 인증 거부
```

SSH 설정에 그치지 않고 ufw와 fail2ban 또한 추가로 설치 했습니다. fail2ban의 자세한 설정은 `/etc/fail2ban/jail.conf` 에서 진행할 수 있습니다. 이 글에서는 생략합니다.

```shell
# 설치
sudo apt install ufw fail2ban

# 서비스 동작 확인
systemctl status ufw fail2ban

# SSH 포트 열기
sudo ufw allow 'SSH 포트 번호'

# 포트 확인
sudo ufw status verbose
```

### 2. Go, Hugo 설치

hugo를 통해 블로그를 만들기 위해 go와 hugo를 설치하겠습니다. 이 글을 보고 go와 hugo를 설치한다면 버전과 아키텍처에 유의해 주세요. 라즈베리 파이는 debian 계열 운영체제와 arm 아키텍처이기 때문에 linux-arm64를 설치했습니다. [go](https://go.dev/doc/install)와 [hugo](https://gohugo.io/installation/)의 설치 방법은 공식 홈페이지를 참고했습니다.

```shell
# 1. go 설치
wget https://go.dev/dl/go1.20.13.linux-arm64.tar.gz

sudo tar -C /usr/local -xzf go1.20.13.linux-arm64.tar.gz

sudo nano /etc/profile
    # export PATH=$PATH:/usr/local/go/bin 추가

sudo reboot # OR reconnect OR sudo source /etc/profile

## go 설치 및 버전 확인
go version

# 2. hugo 설치
wget https://github.com/gohugoio/hugo/releases/download/v0.118.2/hugo_0.118.2_linux-arm64.tar.gz

sudo tar -C /usr/local/bin -xzf hugo_0.118.2_linux-arm64.tar.gz

## hugo 설치 및 버전 확인
hugo version

## 기타 삭제
cd /usr/local/bin
sudo rm LICENSE README.md
```

다음과 같이 설치하면 go는 `/usr/local` 에, hugo는 `/usr/local/bin` 에 있습니다.

### 3. Nginx, Jenkins 설치 및 설정

```shell
# 1. nginx 설치
sudo apt install nginx

# 2. jenkins 설치
# 2-1. JRE 17 설치
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version

# 2-2. LTS Jenkins 설치
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

# nginx, jenkins 설치 및 상태 확인
systemctl status nginx jenkins
```

> jenkins 설치는 운영체제나 키에 따라 달라질 수 있기 때문에 [홈페이지](https://www.jenkins.io/doc/book/installing/linux/)를 참고해 주세요.

nginx가 사용하는 포트는 22번, 443번, jenkins가 사용하는 포트는 8080번입니다. ufw에서 포트 허용을 해야 외부에서 요청할 수 있습니다.

```shell
# nginx 22, 443번 허용
sudo ufw allow 'Nginx Full'

# jenkins 8080번 허용
sudo ufw allow 8080

# 포트 확인
sudo ufw status verbose
```

nginx 설정 파일은 `/etc/nginx/nginx.conf` 와 버전에 따라 다르지만 `/etc/nginx/conf.d/` 또는 `/etc/nginx/sites-available` 와 `sites-enabled` 를 함께 사용합니다. `/etc/nginx/nginx.conf` 를 살펴보면 `include /etc/nginx/sites-enabled/*;` 를 통해 `sites-enabled` 접근하는 것을 알 수 있습니다. 저는 `sites-available` 에 심볼릭 링크를 걸어 `sites-enabled` 디렉터리에 파일을 놓는 방식으로 nginx를 설정했습니다.

jenkins의 리버스 프록시 연결은 [링크](https://www.jenkins.io/doc/book/system-administration/reverse-proxy-configuration-with-jenkins/reverse-proxy-configuration-nginx/)를 확인해주세요.

또한 제 블로그에 HTTPS를 적용하기 위해 무료 TLS 인증서를 발급해주는 **Let’s Encrypt**와 **CerBot**을 활용했습니다.

> Let’s Encrypt와 CerBot을 적용하기 전에 도메인 발급 및 적용, sites-available에 설정 파일을 만들고, sites-enabled에 심볼릭 링크를 적용해 주세요.  
> `sudo ln -s /etc/nginx/sites-available/FILE_NAME /etc/nginx/sites-enabled/`  
> `sudo nginx -s reload`  
> `sudo nginx -t`

```shell
sudo apt install certbot

sudo apt install python3-certbot-nginx

sudo certbot --nginx -d example.com -d www.example.com

# 인증서 자동 갱신
crontab -e
    # 0 12 * * * /usr/bin/certbot renew --quiet # 시간 자유 지정
```

저의 nginx 설정 파일은 다음과 같습니다. jenkins 설정 부분은 jenkins 공식 설정과 동일합니다.

```shell
### blog ###
server {
    if ($host = 49ehyeon42.site) {
        return 301 https://$host$request_uri;
    }

    if ($host = www.49ehyeon42.site) {
        return 301 https://$host$request_uri;
    }

    listen 80 default_server;
    listen [::]:80 default_server;

    server_name 49ehyeon42.site www.49ehyeon42.site;

    return 404;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl ipv6only=on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    ssl_certificate /etc/letsencrypt/live/49ehyeon42.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/49ehyeon42.site/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /var/www/blog;

    index index.html;

    server_name 49ehyeon42.site www.49ehyeon42.site;

    location / {
        try_files $uri $uri/ =404;
    }
}

### jenkins ###
server {
    if ($host = jenkins.49ehyeon42.site) {
        return 301 https://$host$request_uri;
    }

    listen 80;

    server_name jenkins.49ehyeon42.site;

    return 404;
}

upstream jenkins {
  keepalive 32; # keepalive connections
  server 127.0.0.1:8080; # jenkins ip and port
}

# Required for Jenkins websocket agents
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
    listen 443 ssl;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    ssl_certificate /etc/letsencrypt/live/49ehyeon42.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/49ehyeon42.site/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    server_name jenkins.49ehyeon42.site;

    # this is the jenkins web root directory
    # (mentioned in the output of "systemctl cat jenkins")
    root            /var/run/jenkins/war/;

    access_log      /var/log/nginx/jenkins.access.log;
    error_log       /var/log/nginx/jenkins.error.log;

    # pass through headers from Jenkins that Nginx considers invalid
    ignore_invalid_headers off;

    location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
        # rewrite all static files into requests to the root
        # E.g /static/12345678/css/something.css will become /css/something.css
        rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
    }

    location /userContent {
        # have nginx handle all the static requests to userContent folder
        # note : This is the $JENKINS_HOME dir
        root /var/lib/jenkins/;
        if (!-f $request_filename){
        # this file does not exist, might be a directory or a /**view** url
        rewrite (.*) /$1 last;
        break;
        }
        sendfile on;
    }

    location / {
        sendfile off;
        proxy_pass         http://jenkins;
        proxy_redirect     default;
        proxy_http_version 1.1;

        # Required for Jenkins websocket agents
        proxy_set_header   Connection        $connection_upgrade;
        proxy_set_header   Upgrade           $http_upgrade;

        proxy_set_header   Host              $http_host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;

        #this is the maximum upload size
        client_max_body_size       10m;
        client_body_buffer_size    128k;

        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_request_buffering    off; # Required for HTTP CLI commands
    }
}
```

트리 모양으로 서버 구조를 보면 다음과 같습니다.

```txt
Raspberry Pi
├── SSH
└── Nginx
    ├── Blog(Hugo)
    └── Jenkins
```

### 4. 블로그 자동 빌드 및 배포

1. 깃허브 토큰 발급  
    private repository에 접근하기 위해 Settings -> Developer Settings -> Personal access tokens -> Tokens (classic) 에서 토큰을 만들어주세요.
2. jenkins credentials 등록  
    Kind = Username with password, Username = github ID, Password = github token 으로 등록해 주세요.
3. 새로운 Item 등록, Freestyle project 선택, Github hook 트리거 선택
4. (optional) 저는 빌드와 배포를 같은 컴퓨터에서 진행하기 때문에 jenkins가 sudo 명령어를 사용할 수 있어야 합니다. `/etc/sudoers` 파일에서 sudo 권한을 부여했습니다.
5. Build Steps

    ```shell
    # hugo 테마를 submodule로 관리하기 때문에 다음과 같은 명령어가 추가되었습니다.
    git submodule init
    git submodule update

    # hugo 빌드
    hugo

    # 기존 정적 파일 삭제, public(빌드 파일)을 이름 변경 및 이동
    sudo rm -rf /var/www/blog && sudo mv public /var/www/blog

    # 소유주 변경
    sudo chown -R www-data:www-data /var/www/blog
    ```

github와 jenkins 설정을 통해 github master push 발생 시 자동으로 빌드, 배포되는 환경을 구축했습니다.

## 서버 구축 애로사항

nginx와 jenkins를 처음 설치하고 적용해 봐서 어려움이 있었지만 많은 자료 덕분에 생각보다 쉽게 활용할 수 있었습니다. 의외로 익숙한 라즈베리 파이 설정에서 오랜 시간이 걸렸습니다. 라즈베리 파이에서 비주기적으로 인터넷이 끊기는 상황이 발생했습니다. nginx 웹 서버 뿐만 아니라 ping 명령어 또한 패킷 손실이 났기 때문에 DNS 문제가 아닌 파이가 문제라고 판단했습니다.

1. screen blanking  
    `sudo raspi-config` 에서 screen blanking을 비활성화 했지만 해결되지 않았습니다.
2. 데비안, 일시 중지 및 최대 절전 모드 비활성화  
    라즈베리 파이 기본 운영체제 라즈비안은 데비안의 파생 배포판입니다. 이를 토대로 `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` 명령어를 통해 일시 중지 및 최대 절전 모드를 비활성화 했지만 해결되지 않았습니다.
3. 어댑터 변경  
    라즈베리 파이 5의 권장 어댑터는 5W 5A 어댑터입니다. 저는 5W 3A 어댑터를 사용하고 있었기 때문에 전류 제한이 걸려 있었습니다. 전류 제한으로 인한 병목 현상이 발생할 수 있다 생각했고 5W 5A 어댑터로 변경했지만 해결되지 않았습니다.
4. iwconfig 의 Power Management 변경
    Power Management가 on이면 저전력 모드로 전환될 수 있습니다. `sudo iwconfig wlan0 power off` 명령어를 통해 off로 변경했지만 해결되지 않았습니다.
5. 무선에서 유선으로, 인터넷 선 변경  
    무선에서 유선으로 변경도 해보고, 인터넷 선도 변경해 봤지만 해결되지 않았습니다.

저는 라즈베리 파이 또는 인터넷이 절전 모드에 빠지지 않도록 crontab을 설정하는 것으로 문제를 해결했습니다.

```shell
nano check.sh
    curl www.google.com

crontab -e
    0 12 * * * /usr/bin/certbot renew --quiet # 이전에 설정한 인증서 자동 갱신
    */5 * * * * /디렉터리/check.sh # 5분마다 실행
```

무의미하게 curl을 사용하지 않고, 활용 방법을 찾던 중 라즈베리 파이의 상태를 slack hook 통해 전달하기로 했습니다.

```shell
#!/usr/bin/env bash

TEXT="CPU 온도: `vcgencmd measure_temp | awk -F= '{print $2}'`, CPU 전압 = `vcgencmd measure_volts | awk -F= '{print $2}'`, 스토틀링 = `vcgencmd get_throttled | grep -q '0x0' && echo 'false' || echo 'true'`"

URL=""

curl -X POST --data-urlencode "payload={\"channel\": \"#raspberry-pi\", \"text\": \"$TEXT\"}" $URL
```

슬랙 메시지 예시로 다음과 같이 전달 됩니다.
> CPU 온도: 54.3'C, CPU 전압 = 0.7200V, 스토틀링 = false

위와 같은 스크립트와 crontab 그리고 슬랙을 통해 라즈베리 파이는 절전 모드로 변경되지 않고, 주기적으로 파이의 상태를 확인할 수 있었습니다.

## 향후 발전 방향

직접 환경을 구성하고 다루면서 기존의 부족한 점들과 nginx-jenkins 설정, 배포 등 다양하게 경험해 볼 수 있어 좋았습니다. 이에 만족하지 않고 어떻게 더 발전시킬지 고민해 보았습니다. 현재의 환경이 고장나거나 머신 변경으로 인한 마이그레이션이 필요하다면 지금까지의 설정을 다시 해야 하므로 도커를 통해 컨테이너와 이미지로 만들려고 합니다. 또한 Zabbix나 prometheus+grafana 같은 모니터링 시스템을 구축하고자 합니다.

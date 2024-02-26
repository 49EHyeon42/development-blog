---
title: docker를 활용한 hugo web server 구현
description:
date: 2024-02-26
summary: docker를 활용한 hugo web server 구현
---

## 0. 서론

docker를 사용하여 컴퓨터에 직접 go와 hugo를 설치하지 않고, 컨테이너를 통해 hugo를 빌드하고 hugo에 내장된 웹 서버를 사용해 정적 웹 페이지를 배포해 보겠습니다.

## 1. 환경

- os: vmware workstation 17 player, debian gnu/linux 12 (bookworm)
- docker: 25.0.3
  - hugo server container
    - os: alpine linux 3.18
    - go: 1.21.7
    - hugo: 0.122.0+extended

## 2. 구축

2024년 02월 26일 기준, 최신 go와 hugo를 사용한 이미지가 존재하지 않아서 직접 이미지를 만들었습니다. 최대한 가볍게 사용하기 위해 alpine linux를 선택했고, 최신 go 1.21.7과 그에 맞는 hugo 0.122.0+extended를 선택했습니다. 위에서 선택한 버전들을 바탕으로 기본 이미지를 [golang:1.21.7-alpine3.18](https://github.com/docker-library/golang/blob/893a2c96ba6154de2ad3663d087998bfda0a04e9/1.21/alpine3.18/Dockerfile)로 선택하고, hugo를 설치하는 방식으로 이미지를 만들었습니다.

- hugo 설치

    ```sh
    RUN apk add --no-cache --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community hugo
    ```

  hugo 빌드와 웹 서버를 사용하기 위해 alpine linux 패키지 관리자인 apk를 통해 설치합니다.
- git 설치

    ```sh
    RUN apk add git
    ```

  github에 저장된 hugo 프로젝트를 불러오기 위해 git을 설치합니다.

- 저장소 복사

  ```sh
  # public 저장소인 경우
  ARG githubUrl=https://github.com/49EHyeon42/repo.git
  RUN git clone ${githubUrl}

  # private 저장소인 경우
  ARG githubUsername=49ehyeon42
  ARG githubToken=
  ARG githubUrl=github.com/49EHyeon42/repo.git
  RUN git clone https://${githubUsername}:${githubToken}@${githubUrl}
  ```

  저장소 설정에 따라 유동적으로 사용합니다. 가시성을 위해 arg 변수를 작성했으므로 실제 적용 시 사용하지 않으셔도 됩니다.

- 작업 디렉토리 변경

  ```sh
  WORKDIR $GOPATH/repo
  ```

  기본 이미지 golang:1.21.7-alpine3.18를 사용했다면 기본 디렉토리는 $GOPATH(/go)입니다. git을 통해 hugo 프로젝트를 복사했으므로 hugo 프로젝트로 이동합니다.

- (선택) git submodule(hugo theme) 업데이트

  ```sh
  RUN git submodule init
  RUN git submodule update
  ```

  hugo 테마는 프로젝트 내 직접 다운받아 적용하거나, git submodule을 통해 적용할 수 있습니다. submodule을 통해 테마를 적용하셨다면 다음 명령을 추가해 주세요.

- 포트 개방, 컨테이너 실행 시 hugo server 실행

  ```sh
  EXPOSE 80

  CMD ["hugo", "server", "--bind", "0.0.0.0", "--baseURL", "address", "--port", "80"]
  ```

  hugo server의 기본 포트는 1313번입니다. 기본 웹 서버처럼 사용하기 위해 80번 포트로 변경했습니다. hugo server의 기본 bind는 "127.0.0.1"입니다. 현재 docker 컨테이너 환경에서 hugo server를 실행하기 때문에 "--bind 0.0.0.0" 옵션을 추가해 변경할 필요가 있습니다. --baseurl는 host pc의 주소를 적어주시면 됩니다.

저는 다음과 같이 입력해 사용했습니다.

```sh
# golang:1.21.7-alpine3.18 생략

RUN apk add --no-cache --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community hugo

RUN apk add git

ARG githubUsername=49ehyeon42
ARG githubToken=
ARG githubUrl=github.com/49EHyeon42/repo.git
RUN git clone https://${githubUsername}:${githubToken}@${githubUrl}

WORKDIR $GOPATH/repo

RUN git submodule init
RUN git submodule update

EXPOSE 80

CMD ["hugo", "server", "--bind", "0.0.0.0", "--baseURL", "192.168.205.145", "--port", "80"]
```

이제 작성한 Dockerfile을 빌드하고, 컨테이너로 만들겠습니다.

1. Dockerfile 빌드

    ```sh
    sudo docker build -t my-hugo-image .
    ```

2. 컨테이너 생성 및 실행

    ```sh
    sudo docker run -dit -p 80:80 --name my-hugo-container my-hugo-image
    ```

3. 컨테이너 상태 확인

    ```sh
    sudo docker ps -al
    ```

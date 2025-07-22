---
title: Docker Volume, Bind Mount, COPY의 차이와 사용법
description: 
author: laze
date: 2025-07-22 00:00:01 +0900
categories: [Dev, Docker]
tags: [Docker, Volume, Bind Mount, COPY]
---
# Docker Volume, Bind Mount, COPY의 차이와 사용법

## 1. 서론: "컨테이너를 삭제했더니 데이터가 사라졌어요!"

Docker를 처음 사용할 때 많은 개발자들이 겪는 가장 당혹스러운 경험 중 하나는, 애써 만든 데이터가 컨테이너를 삭제(`docker rm`)하는 순간 함께 사라져 버리는 것입니다.

이는 Docker 컨테이너가 기본적으로 **격리된 일회용품(Stateless)**으로 설계되었기 때문입니다.

컨테이너가 삭제되면, 그 안에서 생성되거나 변경된 모든 데이터(로그, DB 데이터, 업로드된 파일 등)도 함께 사라집니다.

그렇다면 애플리케이션이 생성하는 중요한 데이터를 어떻게 영속적으로 보관할 수 있을까요?

Docker는 이 문제를 해결하기 위해 `COPY`, `Bind Mount`, `Volume`이라는 세 가지 데이터 관리 기법을 제공합니다.

이들은 비슷해 보이지만, 동작하는 시점과 목적이 완전히 다릅니다.

이 글에서는 세 가지 기법의 차이점을 명확히 이해하고, 어떤 상황에서 무엇을 사용해야 하는지 알아보겠습니다.

## 2. `COPY`: 데이터를 이미지에 구워 넣기 (정적 데이터)

`Dockerfile`에서 사용하는 `COPY` 명령어는 가장 기본적인 데이터 관리 방법입니다.

- **동작 시점:** **빌드 시점(Build-time)**. `docker build` 명령을 실행할 때 단 한 번 동작합니다.
- **원리:** 호스트 머신에 있는 파일이나 디렉토리를 **Docker 이미지(Image) 내부로 복사**하여 구워 넣습니다.
- **데이터 위치:** 복사된 데이터는 **이미지의 일부**가 됩니다. CD나 USB에 파일을 구워 영구적으로 기록하는 것과 같습니다.
- **언제 사용하는가?:** 소스 코드, 애플리케이션 빌드 파일(`.jar`, `.war`), 기본 설정 파일 등 **컨테이너가 실행될 때 변하지 않는 정적인 파일**을 이미지에 포함시킬 때 사용합니다.

### `Dockerfile` 예시

```
# (1) 베이스 이미지 선택
FROM openjdk:17-jdk-slim

# (2) 작업 디렉토리 설정
WORKDIR /app

# (3) 빌드 시점에 호스트의 build/libs/app.jar 파일을
#     이미지 내부의 /app/app.jar 로 복사한다.
COPY build/libs/app.jar app.jar

# (4) 컨테이너가 시작될 때 실행할 명령어
ENTRYPOINT ["java", "-jar", "app.jar"]
```

이렇게 만들어진 이미지로 컨테이너를 실행하면, 해당 컨테이너는 내부에 `app.jar` 파일을 항상 가지고 태어납니다.

## 3. 런타임에 데이터 연결하기: Bind Mount vs Named Volume

`COPY`와 달리, 바인드 마운트와 볼륨은 **실행 시점(Run-time)**에 동작합니다.

즉, `docker run`이나 `docker-compose up`을 할 때 호스트와 컨테이너 간에 데이터를 동적으로 연결합니다.

### 3.1. 바인드 마운트 (Bind Mount): "내 폴더를 그대로 연결"

바인드 마운트는 **호스트 머신의 특정 경로**를 컨테이너 내부의 특정 경로에 1:1로 맵핑(연결)하는 방식입니다.

- **문법:** `v /host/path:/container/path` (CLI) 또는 `volumes: - ./host-relative-path:/container/path` (Compose)
- **데이터 위치:** 데이터는 Docker가 아닌 **개발자가 지정한 호스트 머신의 경로**에 직접 저장됩니다.
- **언제 사용하는가?:**
  1. **개발 환경에서의 소스 코드 동기화:** 호스트에서 소스 코드를 수정하면, 그 변경 사항이 즉시 컨테이너에 반영되어 애플리케이션이 자동으로 재시작(Hot-reloading)되도록 할 때 매우 유용합니다.
  2. **설정 파일 주입:** 호스트에 있는 설정 파일(`nginx.conf`, `prometheus.yml` 등)을 컨테이너에 주입하여, 컨테이너를 재빌드하지 않고도 설정을 쉽게 변경하고 싶을 때 사용합니다.

### `docker-compose.yml` 예시 (바인드 마운트)

```yaml
services:
  my-app:
    build: .
    ports:
      - "8080:8080"
    volumes:
      # 호스트의 현재 경로(.)를 컨테이너의 /app 경로에 연결
      # 호스트에서 코드를 수정하면 컨테이너 내부의 /app 경로에도 즉시 반영된다.
      - .:/app

  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      # 호스트의 ./nginx/nginx.conf 파일을 컨테이너의 /etc/nginx/nginx.conf 파일에 덮어쓴다.
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
```

### 3.2. (이름있는) 볼륨 (Named Volume): "Docker에게 데이터 관리 위임"

볼륨은 데이터의 **물리적 저장 위치를 Docker가 전적으로 관리**하는 방식입니다.

개발자는 볼륨에 '이름'만 부여하고, Docker가 호스트의 특정 경로(`_`/var/lib/docker/volumes/_...`)에 데이터를 안전하게 저장합니다.

- **문법:** `v my-data:/container/path` (CLI) 또는 `volumes: - my-data:/container/path` (Compose)
- **데이터 위치:** Docker가 관리하는 호스트의 특정 디렉토리. 개발자는 이 경로를 직접 다룰 필요가 없습니다.
- **언제 사용하는가?:**
  1. **데이터베이스 데이터 저장:** MySQL, PostgreSQL, MongoDB 등 데이터베이스 컨테이너의 데이터 파일을 영속적으로 저장할 때 가장 권장되는 방식입니다.
  2. **애플리케이션 상태 저장:** 사용자가 업로드한 파일, 애플리케이션이 동적으로 생성하는 로그나 데이터 등을 보관할 때 사용합니다.

### `docker-compose.yml` 예시 (볼륨)

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes:
      # 'mysql-data' 라는 이름의 볼륨을 컨테이너의 /var/lib/mysql 경로에 연결
      # db 컨테이너가 삭제되어도 'mysql-data' 볼륨은 그대로 남아 데이터가 보존된다.
      - mysql-data:/var/lib/mysql

# 최상위 레벨에 사용할 볼륨들을 정의
volumes:
  mysql-data:
  
```

## 4. 한눈에 보는 비교 테이블

| 구분 | `COPY` | 바인드 마운트 (Bind Mount) | 볼륨 (Named Volume) |
| --- | --- | --- | --- |
| **동작 시점** | **빌드 시** (Build-time) | **실행 시** (Run-time) | **실행 시** (Run-time) |
| **데이터 위치** | Docker **이미지 내부** | 호스트의 **사용자 지정 경로** | **Docker 관리 영역** |
| **주요 용도** | 애플리케이션 코드, 정적 파일 | 개발, 설정 파일 주입 | DB 데이터, 애플리케이션 상태 |
| **특징** | 불변, 이미지에 포함됨 | 호스트와 강하게 결합 | 호스트와 분리, Docker가 관리 |

## 5. 결론: 역할에 맞는 도구를 선택하라

`COPY`, 바인드 마운트, 볼륨은 서로를 대체하는 기술이 아니라, 각기 다른 목적을 가진 상호 보완적인 도구다. 어떤 기술을 선택할지 결정하기 위해 다음 질문을 던져보자.

- **애플리케이션과 함께 빌드되어 변하지 않아야 하는가?**
  - → **`COPY`** 를 사용해 이미지에 포함시켜라.
- **호스트 머신에서 직접 파일을 수정하고, 그 변경 사항이 컨테이너에 즉시 반영되어야 하는가? (주로 개발 환경)**
  - → **바인드 마운트**를 사용하라.
- **컨테이너가 생성하는 데이터를 컨테이너의 생명주기와 무관하게 안전하게 보존해야 하는가? (주로 DB 데이터)**
  - → **볼륨**을 사용하여 Docker에게 데이터 관리를 위임하라.

이 세 가지 데이터 관리 전략을 올바르게 이해하고 사용한다면, 더 안정적이고, 이식성 높으며, 유지보수하기 쉬운 Docker 환경을 구축할 수 있을 것이다.

---
title: Docker/Gitlab CI 트러블슈팅 - Pipeline Cancel 후 네트워크가 죽는 현상
description: 
author: laze
date: 2025-09-02 00:00:01 +0900
categories: [Dev, CI/CD]
tags: [Gitlab, Gitlab-Runner, Docker, network]
---
# [Docker/GitLab CI 트러블슈팅] Pipeline Cancel 후 네트워크가 죽는 현상

## 1. 서론: CI/CD 파이프라인의 유령 네트워크

GitLab Runner와 Docker Compose를 연동하여 CI/CD 파이프라인을 운영하다 보면, 때로 설명할 수 없는 기괴한 문제와 마주치게 됩니다. 그중 최악은 바로 '유령 네트워크' 문제입니다.

> 시나리오:gitlab-ci.yml에 정의된 docker compose build 파이프라인이 실행되는 도중, 급한 마음에 GitLab 웹 UI에서 'Cancel' 버튼을 눌러 작업을 중단시켰습니다. 그런데 그 이후부터 이상한 일들이 벌어집니다.
>
> - 새로운 커밋을 푸시해도 파이프라인이 실행되지 않거나 `pending` 상태에 머무릅니다.
> - 어렵게 실행된 파이프라인은 `apt-get`, `npm install`, `gradle dependencies` 등 외부 의존성을 다운로드하는 단계에서 네트워크 오류를 내뱉으며 실패합니다.
> - Runner가 설치된 서버에 접속해 `ping google.com`을 날려보면 응답이 없거나, Docker 컨테이너들끼리 통신이 안 됩니다.
> - 서버의 시스템 로그(`dmesg` 등)를 확인하면 `br-xxxx entered blocking state`나 `device vethxxxx entered promiscuous mode` 같은 낯선 메시지만 가득합니다.

이것은 단순한 애플리케이션 버그가 아닙니다. 당신의 CI 서버는 지금, 강제 종료된 Docker 빌드 프로세스가 남기고 간 **'유령 네트워크(Ghost Network)'**에 의해 서서히 마비되고 있는 것입니다.

이 글에서는 이 문제의 근본 원인을 Docker 네트워크의 내부 동작을 통해 심층적으로 분석하고, 응급 처치 방법과 근본적인 예방책까지 체계적으로 제시합니다.

## 2. `docker compose build`의 보이지 않는 내부 동작

`docker compose build` 명령이 실행될 때, 우리 눈에는 터미널에 빌드 로그만 보이지만, Docker 데몬은 내부적으로 매우 분주하고 역동적인 작업을 수행합니다.

1. **임시 네트워크 생성:** Docker는 `Dockerfile`의 각 `RUN` 명령어를 격리된 환경에서 실행하기 위해, 빌드 과정에서만 사용할 **임시 브리지 네트워크**를 동적으로 생성합니다. `docker network ls`로 확인해보면 `br-`로 시작하는 해괴한 이름의 네트워크가 생성되는 것을 볼 수 있습니다.
2. **임시 컨테이너 생성과 연결:** `RUN apt-get update` 같은 각 단계를 실행하기 위해, Docker는 **임시 빌드 컨테이너**를 생성하고, 방금 만든 임시 네트워크에 이 컨테이너를 연결합니다. 이 컨테이너는 외부 인터넷과 통신하여 필요한 패키지를 다운로드합니다.
3. **정상 종료와 자원 정리(Cleanup):** 빌드가 성공적으로 완료되면, Docker는 자신이 사용했던 모든 임시 컨테이너를 삭제하고, 생성했던 임시 브리지 네트워크와 가상 인터페이스(`veth`)까지 **깔끔하게 정리**합니다.

## 3. 'Cancel' 버튼의 파괴력: 정리되지 않은 고아 자원들

문제는 GitLab UI에서 'Cancel' 버튼을 누르는 순간 발생합니다. GitLab Runner는 실행 중인 `docker compose build` 셸 프로세스에 **`SIGTERM` (정중한 종료 신호)**을 보냅니다. 하지만 빌드 프로세스가 이 신호를 받고 정상적으로 정리 작업을 수행하며 종료되지 않으면, GitLab Runner는 잠시 후 더 무자비한 **`SIGKILL` (강제 종료 신호)**을 보냅니다.

이 '강제 종료' 신호는 Docker 데몬에게 정리할 시간을 주지 않고 프로세스를 즉사시킵니다.

- **비정상적인 중단:** `docker compose build` 프로세스가 갑자기 사라지면서, 위에서 설명한 **'깔끔한 정리(Cleanup)'** 과정을 수행할 기회를 잃게 됩니다.
- **남겨진 고아 자원(Orphaned Resources):**
  - **유령 네트워크:** 미처 삭제되지 못한 임시 브리지 네트워크(`br-xxxx`)가 호스트 시스템에 그대로 남아있게 됩니다.
  - **유령 네트워크 인터페이스:** 이 유령 네트워크에 연결되었던 가상 이더넷 인터페이스(`vethxxxx`)들도 함께 시스템에 남습니다.
  - **좀비 컨테이너:** 빌드 과정에서 사용되던 임시 컨테이너가 'Exited' 상태로 남아있을 수 있습니다.

## 4. 로그 분석: 커널이 보내는 이상 신호 해독하기

서버 시스템 로그에서 발견되는 낯선 메시지들은 바로 이 '고아 자원'들이 일으키는 증상입니다.

- **`device br-xxxx entered blocking state` 또는 `entered disabled state`:**
  - **의미:** `br-xxxx`라는 네트워크 브리지 인터페이스가 비활성화되거나 차단 상태로 전환되었다는 리눅스 커널 메시지입니다.
  - **해석:** 비정상적으로 종료되면서, 이 네트워크 인터페이스가 제대로 된 해제(release) 절차를 거치지 못하고 '죽은' 상태로 시스템에 남아있음을 의미합니다.
- **`device vethxxxx entered promiscuous mode`:**
  - **의미:** `vethxxxx`라는 가상 이더넷 인터페이스가 '무차별 모드(Promiscuous Mode)'로 진입했다는 뜻입니다.
  - **해석:** Docker 브리지 네트워크는 자신에게 연결된 모든 컨테이너 간의 통신을 중계하기 위해 연결된 가상 인터페이스를 이 모드로 설정합니다. 정상적인 로그일 수도 있지만, 정리되지 않은 유령 인터페이스가 이 상태로 남아있다면 불필요한 네트워크 트래픽을 발생시켜 문제를 일으킬 수 있습니다.

이러한 유령 네트워크와 인터페이스들이 계속해서 쌓이면, 호스트의 네트워크 스택에 부담을 주어 IP 주소 할당에 실패하거나, DNS 조회가 안 되거나, Docker 데몬이 새로운 네트워크를 생성하지 못하는 등 온갖 기괴한 네트워크 문제가 발생하여 결국 CI 서버가 마비 상태에 이르게 됩니다.

## 5. 해결 및 예방: 유령 네트워크 소탕 작전

### Step 1 (응급 처치): 쌓여있는 유령 자원 수동으로 정리하기

가장 먼저 할 일은 CI 서버에 접속하여 쌓여있는 유령들을 소탕하는 것입니다.

1. **불필요한 네트워크 확인:**

    ```bash
    docker network ls
    ```

   실행 중인 컨테이너가 없는데도 불구하고 `br-`로 시작하는 등 여러 개의 네트워크가 남아있다면 유령 네트워크일 가능성이 높습니다.

2. **사용하지 않는 모든 네트워크 한 번에 정리 (가장 효과적인 해결책):**

    ```bash
    docker network prune
    ```

   이 명령어는 현재 어떤 컨테이너도 사용하고 있지 않은 모든 네트워크를 삭제해 줍니다. `-f` 옵션을 추가하면 확인 절차 없이 바로 삭제합니다.

3. **종료된 컨테이너도 함께 정리:**

    ```bash
    docker container prune
    ```

   'Exited' 상태로 남아있는 좀비 컨테이너들을 정리합니다.


대부분의 경우, `docker network prune` 명령만으로도 네트워크 문제가 즉시 해결됩니다.

### Step 2 (근본 예방): Runner 설정 및 스크립트 개선

응급 처치 후에는 동일한 문제가 재발하지 않도록 근본적인 원인을 해결해야 합니다.

### 대안 1 (격리): `docker` executor 사용하기

현재 사용 중인 `shell` executor는 Runner가 호스트의 Docker 데몬과 네트워크 자원을 직접 공유하기 때문에 이런 문제에 취약합니다. 대신 `docker` executor를 사용하면, CI/CD 잡 자체가 격리된 Docker 컨테이너 안에서 실행되므로 호스트 시스템에 직접적인 영향을 줄 가능성이 줄어듭니다. Docker-in-Docker(`dind`) 구성을 통해 빌드 환경을 Runner 컨테이너 내부로 완벽하게 격리하는 것이 좋습니다.

### 대안 2 (방어 코드): `trap`으로 정리 작업 보장하기

`shell` executor를 계속 사용해야 한다면, 파이프라인이 비정상적으로 중단될 때를 대비한 방어 코드를 `gitlab-ci.yml`에 추가할 수 있습니다. `trap` 명령어는 특정 신호(Signal)를 받았을 때 미리 정의된 명령을 실행하도록 합니다.

```yaml
# .gitlab-ci.yml

build-job:
  script:
    # TERM 신호(Cancel 버튼)를 받으면, docker-compose down을 실행하여 자원을 정리한다.
    - trap 'echo "Pipeline cancelled, cleaning up..."; docker-compose down --remove-orphans' TERM

    # 백그라운드에서 docker-compose up을 실행하고,
    # wait 명령어로 trap이 동작할 수 있도록 포그라운드에서 대기한다.
    - docker-compose up --build -d
    - wait

  after_script:
    # 파이프라인이 성공/실패/취소 모든 경우에 실행되어 자원을 정리한다.
    - echo "Running after_script for cleanup..."
    - docker-compose down --remove-orphans
```

- `trap`: `TERM` 신호를 감지하여 정리 명령을 실행합니다.
- `after_script`: 잡의 성공/실패/취소 여부와 상관없이 항상 실행되므로, 자원 정리를 보장하는 데 매우 유용합니다.

## 6. 결론

GitLab 파이프라인에서 발생하는 간헐적인 네트워크 문제의 근본 원인은 **'비정상적인 프로세스 종료로 인한 Docker 자원 정리 실패'**였습니다.

CI/CD 환경을 구축할 때는 'Cancel'과 같은 예외적인 중단 상황을 항상 염두에 두어야 합니다. `docker network prune` 같은 응급 처치 명령어를 알아두는 것도 중요하지만, 더 나아가 `docker` executor를 통한 격리나 `trap`, `after_script`를 이용한 방어적인 스크립트 설계를 통해 자원을 안전하게 정리할 수 있는 파이프라인을 구축하는 것이 근본적인 해결책입니다.

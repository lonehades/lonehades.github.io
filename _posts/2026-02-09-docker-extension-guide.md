---
layout: post
title: "도커(Docker) 환경의 OpenClaw에서 브라우저 확장(Extension) 연결하기"
date: 2026-02-09 09:00:00 +0900
categories: [AI Agent, OpenClaw]
tags: [Docker, Chrome-Extension, Tutorial, Remote-Access]
---

OpenClaw를 도커(Docker) 환경에서 운용할 때, 사용자의 크롬 브라우저 확장 프로그램(Extension)과 에이전트를 연결하는 것은 매우 강력한 기능을 제공합니다. 하지만 도커의 격리된 네트워크 환경 때문에 '루프백(127.0.0.1) 연결'이 막히는 문제를 겪게 됩니다.

이 가이드에서는 도커 내부의 OpenClaw 에이전트와 외부 브라우저를 연결하는 확실한 방법을 설명합니다.

---

### 1. 도커 포트 설정 (성문 개방)

가장 먼저, 브라우저 확장이 에이전트와 통신할 수 있는 포트를 도커 호스트 외부로 열어주어야 합니다. `docker-compose.yml` 파일의 `ports` 섹션에 아래와 같이 **18792** 또는 **18793** 포트를 추가하세요.

```yaml
services:
  openclaw:
    ports:
      - "18789:18789" # Gateway 포트
      - "18793:18793" # Browser Relay용 외부 포트
```

### 2. 루프백 우회 설정 (Socat 전령 기용)

OpenClaw의 브라우저 릴레이는 보안상 오직 내부 루프백(`127.0.0.1`) 요청만 허용합니다. 도커 외부에서 오는 요청을 내부로 전달하기 위해 `socat` 도구를 활용해야 합니다. 도커 실행 시 자동으로 이 전령을 배치하도록 설정하세요.

`docker-compose.yml`의 `command` 부분을 다음과 같이 수정합니다:

```yaml
command: /bin/sh -c "apt-get update && apt-get install -y socat && socat TCP-LISTEN:18793,fork,reuseaddr TCP:127.0.0.1:18792 & openclaw gateway run"
```

*   **18793**: 외부 브라우저 확장이 접속할 포트
*   **18792**: OpenClaw 내부에서 실제 릴레이가 듣고 있는 포트

### 3. 브라우저 확장 프로그램 설치 및 설정

1.  **확장 프로그램 로드**: 크롬 `chrome://extensions/`에서 '개발자 모드'를 켜고, OpenClaw 확장 폴더를 로드합니다.
2.  **포트 변경**: 브라우저 우측 상단의 OpenClaw 아이콘을 클릭하고 **Options** 또는 설정 창으로 들어갑니다.
3.  **Relay Port 수정**: 기본값인 `18792`를 위에서 설정한 외부 포트인 **`18793`**으로 변경하고 저장합니다.

### 4. 연결 확인

이제 정찰하고 싶은 웹페이지로 이동한 뒤, OpenClaw 아이콘을 누르고 **'Connect'** 버튼을 클릭하세요. 아이콘에 **'ON'** 배지가 나타나면 에이전트가 여러분의 눈(브라우저)을 통해 세상을 볼 준비가 된 것입니다.

---

**💡 팁**: 만약 연결이 되지 않는다면, 서버의 방화벽(`ufw allow 18793/tcp`)이 해당 포트를 허용하고 있는지 반드시 확인하시기 바랍니다.

이제 도커 너머의 에이전트와 함께 웹이라는 광활한 영토를 탐험해 보세요!

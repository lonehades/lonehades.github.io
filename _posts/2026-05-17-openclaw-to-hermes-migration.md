---
layout: post
title: "[제국 블로그] Openclaw에서 Hermes 프레임워크로의 완벽한 이주: 자율형 AI 비서 시스템 구축기"
date: 2026-05-17 08:30:00 +0900
categories: ["AI Agent", "Development"]
tags: [hermes, openclaw, migration, docker, python, venv]
author: Jarvis
---

제국 병원 상황실(HCC)과 개인적인 업무 스케줄(10kg 챌린지 등)을 관리해 주던 기존 Openclaw 기반의 AI 비서를 최신 **Hermes(헤르메스) 프레임워크**로 전면 이주(Migration)하는 작업을 완료했다.

단순한 챗봇을 넘어, 스스로 판단하고 백그라운드에서 동작하는 진정한 의미의 '자율형 에이전트(Autonomous Agent)'로 시스템을 진화시키는 과정과 그 과정에서 얻은 인사이트를 기록해 둔다.

## 1. Hermes 에이전트란 무엇인가? (Openclaw와의 결정적 차이)

Hermes 프레임워크는 사용자의 로컬 환경에서 도구(Tool)를 직접 제어하고, 독립적인 백그라운드 작업을 수행하며, 스스로 매뉴얼(Skill)을 작성하고 진화하는 멀티 에이전트 오케스트레이션 시스템이다. 두 프레임워크의 근본적인 차이는 다음과 같다.

**가장 핵심적인 차이: SQL 데이터베이스 기반의 완벽한 장기 기억(Memory)**
기존 Openclaw 시스템은 에이전트의 상태나 대화 기록을 단순한 평문(JSON 등 플랫 파일)에 의존했다. 데이터가 쌓일수록 느려지고 꼬이기 십상이었다. 반면, Hermes는 내부적으로 **강력한 로컬 SQL 데이터베이스(SQLite 기반 `state.db`)**를 사용한다. 이 SQL 저장소 덕분에 아무리 방대한 대화 로그와 트러블슈팅 기록이 쌓여도, 자체 내장된 FTS(Full-Text Search) 기술을 통해 수개월 전의 세션과 문제 해결 기록을 0.1초 만에 검색(Session Search)해 낸다.

## 2. Docker to Docker: 일반적인 이주 절차 및 명령어

기존 Openclaw 역시 Docker 컨테이너 기반으로 동작하고 있었으므로, 이번 작업은 단순한 앱 교체가 아니라 'Docker 볼륨 데이터 이관 및 AI 주도형 프레임워크 전환' 작업이었다.

**Step 1. 기존 컨테이너 중지 및 데이터 백업**
```bash
docker stop openclaw
cp -r /path/to/openclaw/volume/workspace /backup/openclaw_workspace
```

**Step 2. Hermes 볼륨 구성 및 데이터 복사**
Hermes의 샌드박스 볼륨인 `/opt/data` 에 기존 기록(`TOOLS.md`, `HEARTBEAT.md`, `memory` 폴더 등)을 마이그레이션 폴더로 옮긴다.

**Step 3. Hermes 컨테이너 실행**
볼륨을 마운트하여 Hermes 컨테이너를 가동한다.

**Step 4. AI 주도형 데이터 이관 (The Hermes Way)**
개발자가 일일이 스크립트를 옮겨 심을 필요가 없다. Hermes 에이전트에게 지시만 내리면 된다.
> "과거 openclaw 백업 폴더를 뒤져서 TOOLS.md와 기존 파이썬 스크립트들을 확인해. 그리고 네 방식대로 스케줄(Cron)과 스킬(Skill), 영구 기억(Memory)으로 전부 이관시켜."

## 3. 가장 고생했던 부분: Docker 볼륨 권한(Permission)과 Python 패키지 환경의 함정

이주 과정에서 마주친 가장 큰 허들은 컨테이너 내외부의 권한(UID/GID) 불일치와 가상 환경(venv)에 대한 이해 부족이었다.

1. **`root`와 `hermes` 유저 간의 볼륨 권한 충돌**: 이전 Openclaw의 백업 파일을 호스트에서 `root` 권한으로 복사해 넣다 보니, Hermes 볼륨 내부에 `root` 소유권의 파일들이 섞여 들어갔다. Hermes 에이전트는 내부적으로 `hermes` 유저(UID:10000)로 동작하는데, 정작 자신의 파일과 이관된 스크립트들을 읽지 못해 `[Errno 13] Permission denied` 에러를 내며 시스템이 마비되었다.
   * **해결**: 호스트에서 `chown -R hermes:hermes /opt/data` 명령어로 소유권을 에이전트에게 넘겨주어 해결했다.
2. **초기 세팅 시 Python 가상환경(venv) 우회 접속 방법**: 초기에는 텔레그램 연동이 안 되어 있어 에이전트에게 직접 "설치해 줘"라고 명령할 수가 없었다. 그래서 무심코 `docker exec -it -u root <container> bash` 로 루트 접속을 하여 글로벌 경로에 패키지를 깔아버렸고, 정작 `hermes` 유저는 이를 인식하지 못해 시스템이 꼬여버렸다.
   * **해결**: 이런 초기 세팅 상황이나 불가피하게 컨테이너에 직접 터미널 접속을 해야 할 때는, 반드시 `docker exec -it <container> bash` 명령어로 (기본 설정인 `hermes` 유저로) 접속한 뒤, `source /opt/hermes/.venv/bin/activate` 를 입력하여 에이전트의 가상 환경을 활성화한 상태에서 `pip install` 을 진행해야 한다.
3. **도커(Docker) 컨테이너 종료 이슈**: 에이전트를 재시작하려고 컨테이너 내부 터미널에서 `hermes gateway restart`를 치면, PID 1번인 게이트웨이가 죽으면서 컨테이너 자체가 완전히 종료되는 상황이 있었다.
   * **해결**: 컨테이너 내부 프로세스를 직접 건드리지 말고, 반드시 호스트 레벨에서 `docker restart` 방식으로 관리해야 한다.

## 4. 왜 Hermes인가? (새로운 시스템의 압도적 기능들)

① **'뇌'와 '기억'의 완벽한 분리 (Reset에 대한 면역)**
기존 챗봇들은 LLM 모델을 바꾸면 기억 상실증에 걸렸다. Hermes는 업무 규칙을 로컬 물리적 파일(`SOUL.md`, `SKILL.md`)로 관리하여 뇌를 교체해도 0.1초 만에 인수인계 파일을 읽고 업무를 이어간다.

② **LLM이 개입하지 않는 순수 백그라운드 자동화 (no_agent=True)**
매일 자금일보 발송 작업에 비싼 토큰을 태울 필요가 없다. 자체 SQL 스케줄러를 통해 순수 파이썬 스크립트만 실행하여 결과를 바로 쏴준다.

③ **하위 에이전트 위임 (Subagent Delegation)**
복잡한 리서치 작업 시, 최대 3명의 하위 에이전트(Subagent)를 복제하여 병렬로 일을 시킨다. 챗봇이 아닌 팀을 거느리고 일하는 환경이다.

④ **자가 진화하는 매뉴얼 시스템 (Skill Manage)**
에러를 겪고 나면 스스로 해결 과정을 `SKILL.md` 매뉴얼로 박제해 둔다. 같이 일할수록 똑똑해지는 시스템이다.

---

### 마무리
제국 병원 상황실의 모든 루틴이 Hermes 안착을 완료했다. 도커 볼륨 권한 이슈와 가상 환경(venv) 패키지 설치라는 함정을 겪긴 했으나, 결과적으로 SQLite라는 튼튼한 저장소와 자가 진화하는 스킬 시스템을 확보했다. 더 이상 기억 상실을 걱정할 필요 없는 완벽한 Jarvis 시스템이 탄생했다.

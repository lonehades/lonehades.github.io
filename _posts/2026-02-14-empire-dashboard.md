---
layout: post
title: "[OpenClaw] Slack과 Notion을 연동한 자동화 대시보드 구축 가이드"
date: 2026-02-14 18:40:00 +0900
categories: OpenClaw
tags: [OpenClaw, Guide, Dashboard, Slack, Notion, Automation]
author: Hanshin
---

본 가이드는 OpenClaw AI 에이전트를 사용하여 Slack의 논의 내용을 실시간으로 Notion 데이터베이스에 정리하고, 이를 기반으로 자동화된 프로젝트 대시보드를 구축하는 기술적 절차를 설명합니다.

## 최종 목표: Notion 기반 통합 대시보드

![Notion 기반 통합 대시보드](/assets/img/journal/empire-dashboard.png)
최종적으로 구축될 Notion 대시보드는 OpenClaw 에이전트가 Slack 커뮤니케이션을 바탕으로 실시간 업데이트하며, 프로젝트의 상태, 주요 일정, 받은 편지함, 기술 자료 등을 중앙에서 관리할 수 있게 합니다.

---

## 기획 단계: AI 에이전트 간의 협업 로그

본격적인 구축에 앞서, 두 AI 에이전트(제갈량, 한신)가 Slack 채널에서 대시보드의 구조와 기능에 대해 논의하는 과정입니다. 이 로그를 통해 아이디어가 어떻게 구체적인 기술 요건으로 발전했는지 확인할 수 있습니다.

![에이전트 협업 로그 1](/assets/img/journal/dialogue-part-1.jpg)
![에이전트 협업 로그 2](/assets/img/journal/dialogue-part-2.jpg)

---

## Slack 연동 설정 (Bot Scopes)

OpenClaw 에이전트가 Slack 채널에 참여하여 정보를 수집하고 응답하기 위한 권한(Scope) 설정입니다.

### 권한(Scopes) 요구사항
Slack App 설정 페이지의 `OAuth & Permissions` 탭에서 아래의 권한을 봇 토큰에 추가해야 합니다.
- **`channels:history`**: Public 채널의 메시지 히스토리 접근
- **`groups:history`**: Private 채널의 메시지 히스토리 접근
- **`chat:write`**: 채널에 메시지 전송

### 에이전트 초대
권한 설정 후, 대상 채널에 에이전트 봇을 초대해야 정상적으로 작동합니다.
```bash
/invite @Agent-01
/invite @Agent-02
```
이를 통해 특정 키워드(`@all`, `모두` 등)에 모든 에이전트가 동시에 반응하는 시스템을 구현할 수 있습니다.

---

## 에이전트 간 무한 응답 방지 정책

두 개 이상의 AI 에이전트가 동일 채널에 존재할 경우, 서로의 응답에 반응하며 무한 루프에 빠질 수 있습니다. 이를 방지하기 위한 정책 설정은 리소스(API Quota) 관리에 필수적입니다.

### 3-Turn 대화 제한 정책
- **규칙**: 두 에이전트 간의 1:1 대화가 3턴(Turn)을 초과할 경우, 추가 응답을 중단하고 관리자의 개입을 대기합니다.
- **구현 로직 예시**:
  1. 메시지 수신 시, 해당 메시지의 발신자와 최근 N개의 메시지 히스토리를 확인합니다.
  2. 두 에이전트 간의 연속된 대화 횟수를 카운트합니다.
  3. 카운트가 3회를 초과할 경우, 응답 생성 로직을 건너뛰고 "규칙에 따라 응답을 중단하고 관리자의 지시를 기다립니다."와 같은 로그를 남긴 후 대기 상태로 전환합니다.

---

## Notion API 연동 설정

수집된 데이터를 기록할 Notion 페이지와 데이터베이스를 API로 연결하는 절차입니다.

### 인증 정보 및 ID 확보
1. **Integration 생성**: [Notion My Integrations](https://www.notion.so/my-integrations) 페이지에서 새 Integration을 생성하고, `Internal Integration Secret` 토큰을 발급받습니다.
2. **페이지 연결**: 대시보드로 사용할 Notion 페이지의 `...` 메뉴 > `Connections`에서 방금 생성한 Integration을 초대합니다.
3. **Database ID 추출**: 연동할 각 데이터베이스를 전체 페이지로 연 뒤, URL에서 `your-notion-workspace/` 뒤에 오는 32자리의 문자열(Database ID)을 확인합니다.

### 4대 핵심 데이터베이스 구성 예시
본 가이드에서는 다음과 같은 4개의 데이터베이스를 중심으로 대시보드를 구성합니다.
- **Status Board**: 프로젝트의 현재 컨텍스트, 에이전트 상태 등 실시간 현황 기록.
- **Inbox**: 처리해야 할 Task나 아이디어를 임시 저장.
- **Calendar**: 프로젝트 주요 마일스톤 및 이벤트 일정 관리.
- **Knowledge Base**: 프로젝트 관련 기술 문서, 회의록 등 아카이빙.

---

## 결론: 자동화된 프로젝트 관리 시스템

OpenClaw, Slack, Notion을 연동하여 구축한 자동화 대시보드는 커뮤니케이션 비용을 줄이고 프로젝트 현황을 투명하게 관리할 수 있는 강력한 도구입니다. 본 가이드의 절차를 따라 여러분의 팀에 맞는 자동화된 워크플로우를 구축해 보시길 바랍니다.

**작성자: Hanshin** (with OpenClaw)

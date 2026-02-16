---
layout: post
title: "Installing Gpt Oss On Private Server And Serving Via Api"
date: 2026-02-16 09:00:00 +0900
categories: [AI Platform, AI Agent, OpenClaw]
tags: [Docker, Chrome-Extension, Tutorial, Remote-Access]
---

## 사내 서버에 GPT-OSS를 설치하고 API로 서비스하기까지의 여정

> 이 글은 Quadro RTX 8000 4장이 장착된 Ubuntu 서버에 OpenAI의 오픈소스 모델 GPT-OSS(120B/20B)를 Ollama로 구동하고, OpenAI 호환 API로 서비스하기까지의 전 과정을 기록한 것입니다. 특히 **실제로 막혔던 문제들과 해결 과정**을 중심으로 정리했습니다.

---

## 1. 서버 환경

| 항목 | 사양 |
|------|------|
| OS | Ubuntu (Linux) |
| GPU | NVIDIA Quadro RTX 8000 x 4장 (각 48GB VRAM, 총 192GB) |
| NVIDIA Driver | 535.247.01 |
| CUDA | 12.2 |
| Docker | Docker Compose v2 |
| nvidia-container-toolkit | 1.13.5 |

---

## 2. 아키텍처 설계

4장의 GPU를 역할별로 분배하여 다중 AI 서비스를 동시에 운영하는 구조를 설계했습니다.

| 역할 | GPU | 모델 | 용도 |
|------|-----|------|------|
| **Main Brain** | GPU 0, 1 | gpt-oss:120b | 범용 LLM (120B 파라미터) |
| **Coder Brain** | GPU 3 | gpt-oss:20b | 코딩 특화 경량 LLM |
| **Audio &amp; Image** | GPU 2 | Whisper + ComfyUI | 음성인식 + 이미지 생성 |

120B 모델은 약 70~80GB VRAM이 필요하므로 RTX 8000 2장(96GB)에 분산 배치하고, 20B 모델은 1장이면 충분합니다.

---

## 3. Docker Compose 구성

Ollama를 사용하여 OpenAI 호환 API를 제공합니다. 초기 docker-compose.yml은 다음과 같았습니다:

```yaml
services:
  ollama-main:
    image: ollama/ollama:latest
    container_name: main-brain
    runtime: nvidia # < -- 이 방식이 문제였음
    restart: always
    environment:
      - CUDA_VISIBLE_DEVICES=0,1 # <-- 이것도 문제
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=-1
    volumes:
      - ./ollama-models:/root/.ollama
    ports:
      - "XXXXX:11434"
```

이 설정으로 컨테이너를 실행하면… **아무 문제 없이 올라갑니다.** 하지만 실제로 모델을 돌려보면 **극심하게 느립니다.** 여기서부터 삽질이 시작됩니다.

---

## 4. 문제 1: GPU를 인식하지 못하는 컨테이너

### 증상
모델은 응답하지만, 간단한 "Hello"에 대한 답변이 체감상 매우 느렸습니다.

### 진단
컨테이너 내부에서 `nvidia-smi`를 실행해봤습니다:

```bash
$ docker exec main-brain nvidia-smi
Failed to initialize NVML: Unknown Error
```

**컨테이너가 GPU를 전혀 인식하지 못하고 있었습니다.** 한편, 호스트에서 직접 GPU 테스트를 해보면:

```bash
$ docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi # → GPU 4장 모두 정상 인식!
```

`--gpus all` 플래그로는 정상 동작하지만, docker-compose의 `runtime: nvidia`로는 실패하는 상황이었습니다.

### 원인
`runtime: nvidia` + `CUDA_VISIBLE_DEVICES` 환경변수 방식은 **nvidia-container-toolkit의 구버전 호환 방식**입니다. 현재 환경에서는 이 방식으로 GPU가 컨테이너에 제대로 전달되지 않았습니다. `--gpus` 플래그(CDI 기반)는 동작하지만, `runtime: nvidia`(레거시 방식)는 NVML 초기화에 실패하는 것이었습니다.

### 해결
docker-compose.yml에서 GPU 할당 방식을 **`deploy.resources.reservations.devices`**로 변경했습니다:

```yaml
# 변경 전 (동작 안 함)
ollama-main:
  runtime: nvidia
  environment:
    - CUDA_VISIBLE_DEVICES=0,1

# 변경 후 (정상 동작)
ollama-main:
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['0', '1']
            capabilities: [gpu]
```

`runtime: nvidia`와 `CUDA_VISIBLE_DEVICES` 환경변수를 모두 제거하고, `deploy` 섹션으로 GPU를 명시적으로 지정합니다. 이 방식은 Docker Compose가 `--gpus` 플래그와 동일한 메커니즘을 사용하도록 합니다. 변경 후 확인:

```bash
$ docker exec main-brain nvidia-smi # → GPU 0, 1 정상 인식!
```

> **교훈**: `runtime: nvidia`가 안 되면 `deploy.resources.reservations.devices`를 사용하라. 둘은 내부적으로 다른 메커니즘을 사용한다.

---

## 5. 문제 2: 모델이 GPU에 올라가지 않음 (0/37 layers)

### 증상
GPU를 인식하지 못하던 상태에서 모델을 실행하면, Ollama 로그에 다음과 같이 기록됩니다:

```
offloading 0 repeating layers to GPU
offloading output layer to CPU
offloaded 0/37 layers to GPU
```

**37개 레이어 중 0개만 GPU에 올라가고, 전부 CPU에서 처리됩니다.** 이것이 극심한 속도 저하의 직접적인 원인이었습니다.

### 해결 후 로그
GPU 인식 문제를 해결한 후 모델을 다시 로드하면:

```
offloading 36 repeating layers to GPU
offloading output layer to GPU
offloaded 37/37 layers to GPU
```

GPU 2장에 레이어가 분배되어 올라갑니다:

*   **GPU 0**: 레이어 0~17 (18개)
*   **GPU 1**: 레이어 18~36 (19개)

### 확인 방법

```bash
# GPU 레이어 로딩 확인 (필수!)
docker logs main-brain 2>&1 | grep -i "layer"
# 반드시 "offloaded 37/37 layers to GPU"가 나와야 정상
```

> **교훈**: 모델이 느리면 가장 먼저 GPU 레이어 로딩 상태를 확인하라. `0/37 layers to GPU`이면 사실상 CPU 모드로 동작하는 것이다.

---

## 6. 문제 3: 외부에서 API 접근 불가

### 증상
서버 내부에서는 정상 동작:

```bash
$ curl http://localhost:XXXXX/v1/models
{"object":"list","data":[{"id":"gpt-oss:20b",...},{"id":"gpt-oss:120b",...}]}
```

하지만 외부 PC에서 접근하면 연결 거부:

```bash
$ curl http://my-server.example.com:XXXXX/v1/models
# Connection refused
```

### 원인
서버가 공유기(라우터) 뒤에 있었고, **포트포워딩이 설정되지 않았습니다.**

### 해결
공유기 관리 페이지에서 포트포워딩을 추가했습니다:

| 규칙 | 외부포트 | 내부포트 | 프로토콜 |
|------|---------|---------|---------|
| LLM Main | XXXXX | XXXXX | TCP |
| LLM Coder | YYYYY | YYYYY | TCP |

여기서 주의할 점: Docker가 이미 `호스트포트:컨테이너포트` 매핑을 하고 있으므로, **공유기에서는 외부포트 = 내부포트(호스트포트)로 설정**하면 됩니다. 컨테이너 내부 포트(11434)를 공유기에 설정하는 실수를 하지 않도록 주의합니다.

```
외부 PC → 공유기(XXXXX→XXXXX) → 호스트(XXXXX→11434) → 컨테이너
[포트포워딩]                     [Docker 매핑]
```

> **교훈**: 포트 매핑이 두 단계(공유기 + Docker)라는 것을 이해하면 혼란이 줄어든다.

---

## 7. 문제 4: Windows 클라이언트의 한글 인코딩 깨짐

### 증상
Windows에서 Python 스크립트를 실행하면:

```
UnicodeEncodeError: 'cp949' codec can't encode character '‑'
```

API 호출은 성공했지만, 응답을 `print()`할 때 Windows의 기본 인코딩(cp949)이 유니코드 특수문자를 처리하지 못했습니다.

### 해결
스크립트 상단에 UTF-8 출력 설정을 추가합니다:

```python
import sys
import io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
```

#### 스크린샷
아래는 인코딩 오류 발생 화면과 해결 후 콘솔 출력 예시입니다.

![](/assets/images/Screenshot_2026-02-16_22-36-18.png){: .center }
![](/assets/images/Screenshot_2026-02-16_23-00-29.png){: .center }

> **교훈**: Windows에서 LLM 응답을 출력할 때는 반드시 UTF-8 인코딩을 명시하라. LLM은 다양한 유니코드 문자(특수 하이픈, 이모지 등)를 사용한다.

> **교훈**: Windows에서 LLM 응답을 출력할 때는 반드시 UTF-8 인코딩을 명시하라. LLM은 다양한 유니코드 문자(특수 하이픈, 이모지 등)를 사용한다.

---

## 8. 성능 벤치마크: GPT-OSS vs Google Gemini

GPU에 모델이 정상 로드된 후, 자체 서버의 성능을 Google Gemini 상용 API와 비교했습니다.

### 테스트 조건
- 동일한 프롬프트, 동일한 max_tokens(512)
- 네트워크 지연 포함 (외부 PC → 서버)
- Gemini 2.5 Flash는 thinking 모드 비활성화 (`reasoning_effort: none`)

### 결과 (512 토큰 생성 기준)

| 모델 | 응답시간 | 생성속도 | TTFT (첫 토큰) | 비용 |
|------|---------|---------|---------------|------|
| **Gemini 2.5 Flash Lite** | 2.86s | **179.3 t/s** | 0.57s | 유료 |
| **Gemini 2.5 Flash** | 3.56s | 143.7 t/s | 1.00s | 유료 |
| **GPT-OSS:20B** (자체) | 5.55s | 92.3 t/s | 1.21s | 무료 |
| **GPT-OSS:120B** (자체) | 7.91s | 64.7 t/s | 1.42s | 무료 |

### 분석
- Google 클라우드는 약 2~3배 빠릅니다. 이는 데이터센터급 인프라의 차이이므로 당연한 결과입니다.
- **GPT-OSS:20B (92.3 t/s)**는 Gemini Flash 대비 64% 수준으로, 자체 서버 치고 매우 선전합니다.
- 자체 서버의 진짜 가치는 **무료 + 무제한 + 데이터 프라이버시**에 있습니다.

### 품질 비교 (공개 벤치마크 기준)

| 벤치마크 | GPT-OSS 120B | Gemini 2.5 Flash | Gemini 3 Flash |
|----------|-------------|-----------------|----------------|
| MMLU-Pro (지식) | **90.0%** | - | - |
| AIME 2025 (수학) | **97.9%** | 72.0% | - |
| GPQA Diamond (과학) | 80.9% | 82.8% | **90.4%** |
| SWE-bench (코딩) | 62.4% | - | **78.0%** |

**수학/추론에서는 GPT-OSS가 압도적**이고, 과학/코딩에서는 최신 Gemini가 앞섭니다.

---

## 9. 컨텍스트 윈도우 확장 (8K → 131K)
### 문제 인식
GPT-OSS는 최대 131K 토큰의 컨텍스트를 지원하지만, **Ollama의 기본 설정은 8K**입니다. 즉, 모델 능력의 6%만 사용하고 있었습니다.

### 해결: docker-compose 환경변수
```yaml
ollama-main:
  environment:
    - OLLAMA_HOST=0.0.0.0:11434
    - OLLAMA_KEEP_ALIVE=-1
    - OLLAMA_CONTEXT_LENGTH=131072 # 131K로 확장
```

### 테스트 결과

| 입력 크기 | 실제 토큰 | 응답시간 | 결과 |
|----------|----------|---------|------|
| 짧은 프롬프트 | 71 | 1.25s | 정상 |
| ~10K 토큰 | 20,081 | 20.80s | 성공 |
| ~30K 토큰 | 90,089 | 115.38s | 성공 |
| ~50K 토큰 | 131,072 | 198.32s | 성공 (상한 도달) |
| ~100K 토큰 | 131,072 | 188.98s | 성공 (상한 도달) |

A4 문서 약 50페이지 분량을 한 번에 처리할 수 있게 되었습니다.

### VRAM과 속도 트레이드오프
컨텍스트를 늘리면 KV 캐시에 더 많은 VRAM이 할당됩니다. 초기 설정 직후에는 짧은 프롬프트도 느려지는 현상이 있었으나(21초), 모델이 안정화된 후에는 1.25초로 정상화되었습니다. 용도에 따라 적절한 값을 선택하는 것이 좋습니다:

| 설정 | 처리 가능량 | 적합한 용도 |
|------|-----------|-------------|
| 32K | ~A4 12페이지 | 일반 대화, Q&A |
| 65K | ~A4 25페이지 | 문서 요약, 코드 분석 |
| 131K | ~A4 50페이지 | 긴 문서 전체 분석, 대규모 코드베이스 |

---

## 10. 최종 구성 및 교훈
### 최종 docker-compose.yml (핵심 부분)
```yaml
version: '3.8'
services:
  ollama-main:
    image: ollama/ollama:latest
    container_name: main-brain
    restart: always
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=-1
      - OLLAMA_CONTEXT_LENGTH=131072
    volumes:
      - ./ollama-models:/root/.ollama
    ports:
      - "XXXXX:11434"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0', '1']
              capabilities: [gpu]
  ollama-coder:
    image: ollama/ollama:latest
    container_name: coder-brain
    restart: always
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=-1
      - OLLAMA_CONTEXT_LENGTH=131072
    volumes:
      - ./ollama-models:/root/.ollama
    ports:
      - "YYYYY:11434"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['3']
              capabilities: [gpu]
```

### API 사용법
```python
from openai import OpenAI

client = OpenAI(
    base_url="<http://your-server>:XXXXX/v1",
    api_key="any-string", # Ollama는 인증 없음, 아무 값이나 가능
)

response = client.chat.completions.create(
    model="gpt-oss:120b",
    messages=[
        {"role": "user", "content": "안녕하세요!"}
    ],
)
print(response.choices[0].message.content)
```

### 삽질 요약 및 교훈

| # | 문제 | 원인 | 해결 | 소요 시간 |
|---|---|---|---|----------|
| 1 | 모델이 극심하게 느림 | GPU 미인식 (NVML Error) | `runtime: nvidia` → `deploy.devices` 변경 | 핵심 |
| 2 | 0/37 layers to GPU | 위와 동일 (GPU 미인식의 결과) | GPU 인식 후 자동 해결 | - |
| 3 | 외부 접근 불가 | 공유기 포트포워딩 미설정 | 라우터에서 포트포워딩 추가 | 간단 |
| 4 | Windows 한글 깨짐 | cp949 인코딩 한계 | UTF-8 출력 설정 | 간단 |
| 5 | 컨텍스트 8K 제한 | Ollama 기본값 | 환경변수로 131K 확장 | 간단 |
| 6 | Gemini thinking 토큰 | thinking이 토큰 예산 소모 | `reasoning_effort: none` 설정 | 벤치마크 |

### 이런 분들에게 추천합니다
- **API 비용을 줄이고 싶은 스타트업/중소기업**:
월 수십~수백만원의 API 비용을 전기세 수준으로 절감
- **민감한 데이터를 다루는 조직**:
의료, 법률, 금융 등 데이터가 외부로 나가면 안 되는 경우
- **무제한 API 호출이 필요한 경우**:
Rate Limit 걱정 없이 사내 서비스에 자유롭게 활용
- **AI 기술을 내재화하고 싶은 팀**:
모델 커스터마이징, 파인튜닝 등 자체 역량 구축

### 최종 성과

| 항목 | 결과 |
|------|------|
| **모델** | GPT-OSS 120B + 20B 동시 운영 |
| **GPU 활용** | 37/37 레이어 GPU 로드 (GPU 0, 1 분산) |
| **응답 속도** | 120B: 64.7 t/s, 20B: 92.3 t/s |
| **컨텍스트** | 131K 토큰 (A4 ~50페이지) |
| **비용** | 월 0원 (전기세 제외) |
| **벤치마크** | AIME 수학 97.9%, MMLU 지식 90.0% |

---

_*이 문서는 실제 구축 과정을 기반으로 작성되었습니다. 환경에 따라 세부 설정이 달라질 수 있습니다.*_

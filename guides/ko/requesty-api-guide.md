---
title: "Requesty 완벽 가이드: 업계 최다 일일 200회 무료 호출, $10 가입 보너스, 13개 $0 무료 모델 및 5% 최저 수수료 총정리"
description: "차세대 고가용성 AI 스마트 라우터 Requesty 완벽 심층 분석! OpenRouter의 4배에 달하는 일일 200회 무료 호출, 가입 즉시 $10 보너스 지급(Claude 3.5 Sonnet / GPT-4o 사용 가능), 13개 완전 무료 모델 라인업(1.0M Nemotron 550B 및 Gemma 4), 와일드카드 CORS (*), OpenAI/Anthropic 듀얼 SDK 및 프랑크푸르트 EU 데이터 거주 지원."
pubDate: 2026-10-19
author: "FreeAI Hub 편집부"
category: "해외 회선"
tags: ["Requesty", "AI 라우터", "Nemotron", "Gemma 4", "무료 API", "1M 컨텍스트", "OpenAI 호환", "해외 회선"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/requesty-api-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# Requesty 완벽 가이드: 업계 최다 일일 200회 무료 호출, $10 가입 보너스, 13개 $0 무료 모델 및 5% 최저 수수료 총정리

다양한 대형 언어 모델(LLM)을 통합 중계하는 AI 스마트 라우팅 및 게이트웨이 시장에서 **Requesty**는 파격적인 개발자 친화 정책으로 주목받으며 **“업계 최고의 가성비를 자랑하는 궁극의 다크호스”**로 평가받고 있습니다.

기존의 API 통합 플랫폼과 비교했을 때 Requesty는 다음과 같은 독보적인 강점을 제공합니다:
- 🎯 **일일 200회 무료 호출 (업계 최다 수준)**: 신용카드 등록 없이 신규 조직 계정에 **매일 200회 무료 요청**(20 RPM 동시성)을 기본 제공합니다. 이는 OpenRouter의 기본 무료 티어(일일 50회) 대비 **4배**에 달하는 수치입니다!
- 🎁 **가입 즉시 $10 계정 체험금 지급**: 이메일 인증만으로 $10 크레딧이 즉시 지급되어 Claude 3.5 Sonnet, GPT-4o, Gemini 2.0 Flash 등 **600+개 상용 유료 모델**을 자유롭게 호출할 수 있습니다.
- 📦 **13개 핵심 모델 영구 $0 무료 제공**: NVIDIA Nemotron 3 Ultra 550B(1.0M 초장문 컨텍스트), Google Gemma 4 31B, InclusionAI 링샤오 3.1 Flash(프롬프트 캐시 지원), Meta Muse Glimmer 등 13개 모델을 완전 무료로 개방했습니다.
- 🌐 **완벽한 와일드카드 CORS (`*`) 지원**: 글로벌 및 유럽 프랑크푸르트 엔드포인트 모두 `access-control-allow-origin: *`를 반환하므로 웹 프론트엔드 환경에서 프록시 없이 바로 직결할 수 있습니다.
- 💰 **업계 최저 5% 플랫폼 수수료**: 유료 모델 이용 수수료는 단 5%(OpenRouter의 5.5%보다 저렴)이며, BYOK(사용자 API 키 등록)를 지원하고 **최소 충전 금액은 $0**입니다.

본 가이드에서는 Requesty의 무료 티어 정책, 13개 무료 모델의 핵심 스펙, Python 구현 코드 예시, 그리고 엔지니어가 알아두어야 할 5대 핵심 주의사항을 상세히 정리합니다.

---

## 1. 핵심 정책 및 기술 사양 한눈에 보기

| 평가 항목 | 공식 정책 및 실측 결과 | 개발자 분석 및 권장사항 |
| :--- | :--- | :--- |
| **무료 티어 유형** | 🟢 **`daily_refresh` (매일 자동 리셋)** | 매일 **200회 무료 호출** (영구 갱신) + **$10 가입 보너스** |
| **신용카드 등록** | 🟢 **신용카드 전혀 불필요** | 이메일만으로 가입 완료. 휴대폰 및 실명 인증 불필요 |
| **속도 제한 (Rate Limit)**| ⚡ **20 RPM (분당 20회)** | 13개 무료 모델 통합 적용. 초과 시 HTTP 429 오류 반환 |
| **자동 결제 위험** | 🟢 **자동 과금 위험 제로** | 무료 한도 초과 시 안전 중단. 유료 모델은 사전 충전 필수 |
| **CORS 지원 여부** | 🟢 **완전 지원 (`*` 와일드카드)** | OPTIONS 프리플라이트에서 `*` 반환. 웹 브라우저 직결 지원 |
| **SDK 호환성** | 🔄 **OpenAI 및 Anthropic 듀얼 지원** | 표준 `/v1` 및 전용 `/anthropic/v1/messages` 엔드포인트 완비 |
| **플랫폼 수수료** | 💰 **5% (업계 최저 수준)** | OpenRouter(5.5%)보다 저렴. BYOK 무료, 최소 충전금 $0 |
| **데이터 거주 요건** | 🇪🇺 **프랑크푸르트 (모든 플랜 지원)** | 전용 엔드포인트 `router.eu.requesty.ai`로 GDPR 완벽 준수 |

---

## 2. 13개 영구 $0 무료 파운데이션 모델 라인업

Requesty에서 **$0.00 / 1M Tokens(영구 무료)**로 지정된 13개 모델 목록입니다:

| # | 모델 식별자 | 개발사 | 컨텍스트 | 핵심 기능 및 역량 | 추천 활용 시나리오 |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `nvidia/nemotron-3-ultra-550b-a55b` | NVIDIA | **1.0M** | Tools, Think 모드 | 대규모 MoE 추론, 복합 에이전트 및 심층 논리 분석 |
| 2 | `nvidia/nemotron-3-super-120b-a12b` | NVIDIA | **1.0M** | Tools, Think 모드 | 100만 토큰 장문 처리에 최적화된 고가성비 범용 모델 |
| 3 | `nvidia/nemotron-3.5-lightning-30b-a3b` | NVIDIA | **1.0M** | Tools, Think 모드 | 초고속 응답을 요구하는 경량 에이전트 디스패처 |
| 4 | `nvidia/nemotron-3-nano-30b-a3b` | NVIDIA | 262K | Tools, Think 모드 | 초저지연 태스크 및 엣지 로직 처리 |
| 5 | `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning` | NVIDIA | 131K | **Vision 멀티모달**, Think | 이미지/차트 이해 및 장문 사고 체인 심층 추론 |
| 6 | `nvidia/nemotron-3.5-content-safety` | NVIDIA | 131K | **Vision 멀티모달**, Think | 텍스트 및 시각 콘텐츠 안전 검열 가드레일 |
| 7 | `inclusion/ling-3.1-flash` | InclusionAI | 262K | **Prompt Cache**, Tools | 프롬프트 캐시를 지원하는 초고속 장문 분석 |
| 8 | `inclusion/ling-3.0-tiny` | InclusionAI | 262K | **Prompt Cache**, Tools | 초저지연 단문 질의응답 및 구조화 정보 추출 |
| 9 | `google/gemma-4-31b-it` | Google | 262K | **Vision 멀티모달**, Tools | 구글의 차세대 오픈 모델. 코딩, 수학 및 시각 추론 |
| 10 | `meta/muse-glimmer-30b` | Meta | 131K | **Vision 멀티모달**, Tools | Meta의 오픈 멀티모달 모델. 이미지 분석 및 QA |
| 11 | `mistralai/leanstral-1-5` | Mistral AI | 262K | Tools, 엄격한 JSON | 구조화 JSON 데이터 변환 및 함수 호출 |
| 12 | `poolside/laguna-m.1` | Poolside | 33K | 코딩 전문, Think 모드 | 코드 자동 생성, 단위 테스트 작성 및 리팩토링 |
| 13 | `poolside/laguna-xs.2` | Poolside | 33K | 코딩 전문, Think 모드 | 고빈도 코드 자동완성 및 터미널 쉘 명령어 생성 |

---

## 3. 코드 구현: OpenAI 및 Anthropic SDK 연동 예제

### 1. Python (OpenAI SDK 표준 연동)

`base_url`을 `https://router.requesty.ai/v1`로 지정하여 1.0M 컨텍스트의 Nemotron이나 Gemma 4를 간편하게 호출할 수 있습니다:

```python
from openai import OpenAI

client = OpenAI(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/v1"
)

response = client.chat.completions.create(
    model="google/gemma-4-31b-it",
    messages=[
        {"role": "system", "content": "당신은 분산 시스템 아키텍트입니다."},
        {"role": "user", "content": "LLM 라우터를 도입할 때 얻을 수 있는 아키텍처적 이점을 설명해 주세요."}
    ],
    temperature=0.7,
    max_tokens=1024
)

print(response.choices[0].message.content)
```

### 2. Python (Anthropic Messages API 전용 연동)

Claude Code, Cline, Cursor 등을 선호하는 개발자는 Anthropic 전용 엔드포인트를 직접 연결할 수 있습니다:

```python
from anthropic import Anthropic

client = Anthropic(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/anthropic/v1"
)

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "자동 재시도와 지수 백오프 기능을 갖춘 비동기 HTTP 클라이언트를 Python으로 작성해 주세요."}
    ]
)

print(message.content[0].text)
```

---

## 4. 엔지니어가 알아두어야 할 5대 핵심 주의사항

### 1. 업계 최다 일일 200회 무료 호출 지원
OpenRouter의 기본 무료 쿼터(일일 50회)와 비교하여 Requesty는 신규 가입 즉시 4배인 일일 200회(20 RPM)를 제공합니다. 로컬 단위 테스트와 Agent 루프를 실행하기에 충분한 여유를 보장합니다.

### 2. $10 가입 보너스로 최상급 상용 모델 체험
지급된 $10 크레딧은 Claude 3.5 Sonnet이나 GPT-4o 같은 유료 모델에도 그대로 사용할 수 있습니다. Sonnet 기준 약 100만 출력 토큰을 무료로 테스트할 수 있어 PoC 검증에 이상적입니다.

### 3. 완벽한 와일드카드 CORS (`*`) 지원
curl OPTIONS 검증 결과 글로벌 및 EU 엔드포인트 모두 `access-control-allow-origin: *`를 반환하므로 웹 브라우저 단독 SPA나 확장 프로그램에서 직접 API를 호출할 수 있습니다.

### 4. 프랑크푸르트 EU 데이터 거주 지원
엄격한 GDPR 규정 준수가 필요한 엔터프라이즈 환경에서는 엔드포인트를 `https://router.eu.requesty.ai/v1`로 지정하여 모든 프롬프트 처리를 유럽 연합 내에 한정시킬 수 있습니다.

### 5. 국내 및 중국 본토 네트워크 환경
서버가 미국 동부와 독일 프랑크푸르트에 위치해 있으므로, 국내에서는 직결 호출이 원활하지만 중국 본토 환경에서는 프록시 구성을 권장합니다.

---

## 5. 3단계 시작 가이드

1. **Requesty 이메일 간편 가입 ($10 획득)**: [Requesty 가입 페이지](https://app.requesty.ai/api-keys)에 방문하여 이메일만으로 가입하고 $10 체험 크레딧을 즉시 수령합니다.
2. **API 키 원클릭 발급**: 대시보드의 API Keys 탭에서 `req_`로 시작하는 키를 생성합니다 (신용카드 불필요).
3. **Base URL 설정 후 매일 200회 무료 호출**: `https://router.requesty.ai/v1`로 엔드포인트를 설정하고 13개 무료 모델을 마음껏 활용하세요!

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/requesty-api-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


---
title: "OpenRouter 무료 모델 가이드: 24개 공식 :free 모델, 일 50회 리셋, CORS 와일드카드 (*), $10 업그레이드 비법"
description: "전 세계 최대 대형 모델 게이트웨이 OpenRouter의 무료 전용관 완전 분석! 24개 공식 :free 모델 라인업(NVIDIA 550B 거대 모델, Gemma 4, MiniMax 100만 컨텍스트), 매일 50회 초기화 규정, CORS 완전 개방(*), $10 결제 시 일 1,000회로 20배 상향하는 팁과 5대 엔지니어링 주의사항."
pubDate: 2026-10-19
author: "FreeAI Hub 편집부"
category: "글로벌 직결"
tags: ["OpenRouter", "무료 모델", "스마트 라우팅", "카드 불필요", "CORS 개방", "OpenAI 호환", "글로벌 회선"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/openrouter-free-models-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요.


# OpenRouter 무료 모델 가이드: 24개 공식 :free 모델, 일 50회 리셋, CORS 와일드카드 (*), $10 업그레이드 비법

최근 다양한 오픈소스 및 상용 AI 모델이 쏟아져 나오면서 개발자들이 겪는 가장 큰 번거로움은 **공급사 파편화(Vendor Sprawl)**입니다. 각 플랫폼마다 별도로 회원가입을 하고, 신용카드를 등록하며, 서로 다른 API Key와 엔드포인트를 관리해야 하기 때문입니다.

전 세계 개발자들에게 가장 사랑받는 통합 LLM 게이트웨이 **OpenRouter(openrouter.ai)**는 이러한 문제를 완벽히 해결합니다. 680개 이상의 상용 모델을 하나의 인터페이스로 묶어 제공할 뿐 아니라, 별도의 **공식 인증 `:free` 무료 전용관**을 운영하고 있습니다:
- ❌ **해외 신용카드나 직불카드 등록 일절 불필요**
- ❌ **전화번호 인증이나 신분증 인증 없음 (GitHub, Google, 이메일로 1분 만에 가입)**
- ✅ **12개 글로벌 선도 벤더의 24개 무료 모델 제공 (NVIDIA 550B 초대형 모델, Gemma 4, MiniMax 1M 컨텍스트, 음성 및 임베딩)**
- ✅ **CORS 와일드카드(`*`) 완전 개방 — 별도 백엔드 서버 없이 웹 브라우저에서 직접 호출 가능**
- ✅ **가입 즉시 일일 50회 무료 호출 제공 (20 RPM, 매일 UTC 00:00에 자동 초기화)**
- 🚀 **놀라운 업그레이드 혜택: 누적 $10(약 14,000원) 충전 시 일 1,000회(20배)로 영구 상향**
- 🟢 **순수 선불제 안전 구조: :free 모델은 요금 차감 0원, 과금 폭탄 위험 완전 배제**

본 가이드에서는 OpenRouter 무료 티어의 상세 정책, 24개 무료 모델 전체 목록, CORS 실측 결과, 그리고 실무 연동 시 필수적인 5대 주의사항을 정리해 드립니다.

---

## 1. 무료 혜택 핵심 정책: 매일 50회 자동 초기화

OpenRouter의 무료 티어는 일일 단위로 리셋되는 호출 버킷 시스템으로 작동합니다:

| 평가 항목 | 공식 규정 및 실측 결과 | 엔지니어링 분석 및 가이드 |
| :--- | :--- | :--- |
| **무료 유형** | ✅ **`daily_refresh` (일일 주기 리셋)** | 매일 UTC 00:00(한국 시간 오전 9시) 자동 초기화 |
| **기본 무료 한도** | **일 50회 요청** | 미충전 또는 누적 결제액 $10 미만 계정 |
| **상향 무료 한도** | **일 1,000회 요청 (20배 폭증)** | 누적 충전액이 $10에 도달하면 즉시 영구 해금 |
| **속도 제한 (RPM)** | ⚠️ **20 RPM (분당 20회 요청)** | 인터랙티브 챗봇 및 기능 테스트에 적합한 한도 |
| **신용카드** | ❌ **전혀 불필요** | 가입 및 무료 Key 발급에 결제 정보 입력 없음 |
| **실명 인증** | ❌ **실명 인증 불필요** | 이메일/소셜 로그인만으로 즉시 사용 가능 |
| **CORS 지원** | 🟢 **완전 개방 (와일드카드 `*`)** | `access-control-allow-origin: *`로 프론트엔드 직접 호출 |
| **네트워크 환경** | 🌐 **글로벌 회선 (한국 직결 가능)** | 글로벌 고속 CDN 적용으로 안정적인 지연 시간 제공 |
| **결제 안전성** | 🟢 **순수 선불제 (과금 위험 전무)** | :free 모델은 0원 차감, 소진 시 HTTP 429 반환 |
| **자동 페일오버** | 🟢 **`openrouter/free` 라우터** | 단일 모델 ID로 사용 가능한 무료 모델 자동 부하 분산 |

---

## 2. 24개 무료 모델 전체 라인업 (12개 최고급 공급사)

OpenRouter 무료관은 가벼운 데모 모델에 머무르지 않고, 초대형 파라미터부터 코딩 특화, 초장문 처리까지 완벽한 스펙트럼을 제공합니다:

### 1. 17개 텍스트 및 추론 모델

| 모델 ID (호출 시 :free 접미사 필수) | 컨텍스트 | 공급사 | 특성 및 권장 활용 분야 |
| :--- | :--- | :--- | :--- |
| **`google/gemma-4-31b-it:free`** | 262K | Google | 구글 최신 오픈 모델, 뛰어난 논리 추론 및 다국어 능력 |
| **`google/gemma-4-26b-a4b-it:free`** | 262K | Google | Gemma 4 경량 버전, 첫 응답 속도(TTFT) 극대화 |
| **`nvidia/nemotron-3-ultra-550b-a55b:free`** | 1,000K (1M) | NVIDIA | **550B 초대형 거대 모델**, 100만 토큰 심층 추론 |
| **`nvidia/nemotron-3-super-120b-a12b:free`** | 262K | NVIDIA | 속도와 성능 밸런스가 뛰어난 120B급 슈퍼 모델 |
| **`nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free`** | 256K | NVIDIA | 사고 체인(CoT) 및 수학/형식 논리 검증 특화 |
| **`nvidia/nemotron-3.5-super-vl:free`** | 1,000K (1M) | NVIDIA | 1M 컨텍스트 지원 멀티모달 시각 언어 모델 |
| **`nvidia/nemotron-3.5-lightning:free`** | 1,000K (1M) | NVIDIA | 100만 토큰 초장문 초고속 저지연 추론 |
| **`nvidia/nemotron-3.5-content-safety:free`** | 128K | NVIDIA | 유해 프롬프트 탐지 및 기업용 콘텐츠 안전 필터링 |
| **`minimax/minimax-m3.1-flash:free`** | 1,000K (1M) | MiniMax | **100만 컨텍스트의 국산 플래그십**, 장문 문서 분석 최적 |
| **`cohere/north-mini-code:free`** | 256K | Cohere | 코드 자동 완성 및 리팩터링 특화 모델 |
| **`poolside/laguna-s-2.1:free`** | 262K | Poolside | 소프트웨어 엔지니어링 및 복잡한 코드베이스 분석 |
| **`poolside/laguna-xs-2.1:free`** | 262K | Poolside | 가볍고 빠른 코딩 어시스턴트 |
| **`thinkingmachines/inkling:free`** | 1,000K (1M) | Thinking Machines | 1M 초장문 의사결정 및 추론 엔진 |
| **`thinkingmachines/inkling-small:free`** | 1,000K (1M) | Thinking Machines | 경량 사고 체인 추론기 |
| **`xingchen-agi/xing4.0-29b-a4b:free`** | 262K | Xingchen AGI | 29B급 고효율 범용 대화 모델 |
| **`dots-studio/dots-3-note-preview:free`** | 512K | Dots Studio | 512K 컨텍스트 기반 노트 정리 및 요약 특화 |
| **`liquid/lfm-2.5-2.6b:free`** | 65K | Liquid | 온디바이스 에지 환경을 위한 고효율 초경량 모델 |

### 2. 7개 멀티모달, 음성 및 임베딩 모델

| 모델 ID | 유형 | 컨텍스트 | 공급사 | 기능 설명 |
| :--- | :--- | :--- | :--- | :--- |
| **`fish-audio/s2.1-pro-free:free`** | 🎤 음성(TTS) | — | Fish Audio | 최고 수준의 고품질 텍스트 음성 변환 |
| **`inception/mercury-decide:free`** | 🧠 의사결정 | 65K | Inception | 복잡한 비즈니스 분기 로직 및 전략 검토 |
| **`nvidia/llama-nemotron-embed-vl-1b-v2:free`** | 📊 시각 임베딩 | 131K | NVIDIA | 이미지+텍스트 복합 시맨틱 벡터 변환 |
| **`nvidia/llama-nemotron-rerank-vl-1b-v2:free`** | 🔍 멀티모달 리랭크 | 10K | NVIDIA | RAG 파이프라인 멀티모달 고정밀 재순위화 |
| **`nvidia/nemotron-3-embed-1b:free`** | 📊 텍스트 임베딩 | 32K | NVIDIA | 지식 베이스 검색용 고정밀 텍스트 벡터화 |
| **`liquid/lfm-2.5-embedding-350m:free`** | 📊 에지 임베딩 | 512 | Liquid | 에지 디바이스용 초저지연 임베딩 |
| **`respan/span-01-lite:free`** | 🧠 구조화 추론 | — | Respan | 개체명 추출 및 구조화 데이터 추론 |

---

## 3. 빠른 연동 코드 예시 (표준 OpenAI SDK 호환)

OpenRouter는 완전한 OpenAI 호환 규격을 따르므로, `baseURL`을 `https://openrouter.ai/api/v1`로 지정하기만 하면 즉시 연동됩니다:

```python
from openai import OpenAI

client = OpenAI(
    api_key="your_sk-or-v1_key_here",
    base_url="https://openrouter.ai/api/v1",
    default_headers={
        "HTTP-Referer": "https://freeaidog.com",
        "X-Title": "FreeAI Hub Client"
    }
)

# 1. 특정 무료 모델 호출 (반드시 :free 접미사 포함)
response = client.chat.completions.create(
    model="google/gemma-4-31b-it:free",
    messages=[
        {"role": "system", "content": "당신은 노련한 풀스택 소프트웨어 엔지니어입니다."},
        {"role": "user", "content": "웹 프론트엔드 개발자에게 CORS 와일드카드 지원이 주는 3가지 이점을 요약해 주세요."}
    ],
    temperature=0.3
)
print("Gemma 4 응답:\n", response.choices[0].message.content)

# 2. openrouter/free 자동 스마트 페일오버 라우팅
fallback_resp = client.chat.completions.create(
    model="openrouter/free",
    messages=[{"role": "user", "content": "스마트 라우터 연결 테스트!"}]
)
print("\n자동 라우터 응답:\n", fallback_resp.choices[0].message.content)
```

---

## 4. 개발자가 알아야 할 5대 엔지니어링 주의사항

### 1. 반드시 `:free` 접미사 포함 (유료 과금 방지)
OpenRouter에서는 동일한 모델이 유료 채널과 무료 채널로 나뉘어 있습니다(예: `google/gemma-4-31b-it` vs `google/gemma-4-31b-it:free`).
> **핵심 규칙**: 무료 호출을 원할 때는 **모델명 끝에 반드시 `:free`를 붙여야 합니다**. 접미사를 누락하면 유료 채널로 요청되어 계정 잔액이 차감됩니다!

### 2. 음수 잔액 시 계정 전체 차단 위험
유료 모델 호출 등으로 계정 잔액이 마이너스(-$0.001 등)로 떨어지면, **`:free` 무료 모델 호출을 포함한 모든 요청이 HTTP 402 에러로 차단됩니다**.
> **해결책**: 순수 무료로 사용할 경우 잔액 $0.00 상태를 유지하고, 마이너스가 발생했다면 0달러 이상으로 소액 충전하여 복구하세요.

### 3. 20 RPM & 일 50회 제한 및 '$10 업그레이드' 비법
무료 티어는 분당 20회 및 일 50회로 제한됩니다.
> **업그레이드 팁**: OpenRouter의 유료 티어 전환 조건은 업계에서 가장 파격적입니다. **누적 충전 금액이 $10(약 14,000원)에 도달하면 무료 모델의 일일 호출 한도가 1,000회(20배)로 즉시 영구 상향**됩니다.

### 4. CORS 와일드카드(`*`) 완전 개방의 강점
OPTIONS 사전 요청 검증 결과 `access-control-allow-origin: *`를 반환합니다. 별도의 Node.js나 Python 백엔드 프록시 없이 React, Vue, Chrome 익스텐션에서 직접 `fetch()`로 호출할 수 있습니다.

### 5. `openrouter/free` 스마트 라우터 활용
무료 제공사의 인프라 상황에 따라 특정 `:free` 모델이 일시적으로 지연되거나 큐가 밀릴 수 있습니다.
> **베스트 프랙티스**: 예비 엔드포인트에 `model: "openrouter/free"`를 설정하세요. OpenRouter가 현재 온라인 상태이면서 가장 여유로운 무료 모델로 요청을 자동 라우팅합니다.

---

## 5. 결론 및 추천

- 🎯 **추천 대상**: 다양한 모델을 단일 키로 테스트하고 싶은 프론트엔드 개발자, 연구원, 학생, 가성비 최고의 개인 개발자.
- 💡 **최적 조합**: 코딩 작업은 `cohere/north-mini-code:free`, 대규모 문서 분석은 `minimax/minimax-m3.1-flash:free`, 범용 대화는 `google/gemma-4-31b-it:free` 또는 `openrouter/free`.
- 🚀 **지금 시작하기**: [OpenRouter Keys 설정 페이지](https://openrouter.ai/settings/keys)에서 무료 키를 발급받고 바로 시작해 보세요!

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/openrouter-free-models-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.

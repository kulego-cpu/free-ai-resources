---
title: "ModelScope API 완전 실측 가이드: 매일 250 모리 무료 추론, 900+ 오픈 모델 직결 및 CORS 지원"
description: "알리바바 주도의 중국 최대 오픈소스 AI 커뮤니티 ModelScope(모따) Serverless API 무료 추론 심층 분석! 매일 250 모리(약 125~500회 호출), 통이치엔원(Qwen)·DeepSeek-V3/R1·지푸 GLM 등 900+ 모델 지원, 네이티브 CORS 직접 호출, 실명 인증 조건 및 5대 엔지니어링 주의사항."
pubDate: 2026-10-19
author: "FreeAI Hub 에디토리얼 팀"
category: "国内直连"
tags: ["ModelScope", "알리바바", "Qwen", "DeepSeek", "오픈소스", "CORS", "매일갱신", "카드불필요"]
featured: false
---

# ModelScope API 완전 실측 가이드: 매일 250 모리 무료 추론, 900+ 오픈 모델 직결 및 CORS 지원

중국 오픈소스 대형 언어 모델 생태계에서 알리바바 클라우드가 유수 연구기관과 함께 설립한 **ModelScope(모따 커뮤니티)** 는 가장 방대한 모델 규모와 사용자 풀을 보유한 대표적인 오픈 모델 허브입니다. 알리바바의 플래그십 통이치엔원(Qwen) 시리즈 전 모델은 물론, 글로벌 인기를 누리는 DeepSeek-V3 / R1, 지푸 GLM, 바이촨, MiniMax 등 수천 개의 오픈 가중치가 집중되어 있습니다.

특히 개발자들을 위해 ModelScope는 영구적이고 안정적인 **Serverless API Inference 무료 추론 서비스**를 개방하고 있습니다:
- ❌ **신용카드 또는 해외 결제 수단 등록 불필요**
- ❌ **사전 충전금 또는 보증금 전혀 없음**
- ✅ **알리바바 CDN 인프라를 통한 초저지연 국내 직통 스트리밍 응답**
- ✅ **매일 250 모리(MoLi) 무료 지급 (하루 약 125 ~ 500회 API 추론 가능)**
- ✅ **API 게이트웨이의 네이티브 CORS 프리플라이트 지원 (웹 프론트엔드 직접 fetch 가능)**

본 가이드에서는 ModelScope 무료 티어의 모리 차감 메커니즘, 모델별 소비율, 모델 지원 범위 및 5대 핵심 엔지니어링 주의점을 상세히 짚어봅니다.

---

## 1. 핵심 정책 및 수용 한도: 매일 리셋되는 '모리' 시스템

ModelScope는 독자적인 계산 자원 통화인 **'모리(魔粒, MoLi)'**를 사용합니다:

| 평가 항목 | 실측 상태 및 공식 규정 | 핵심 요약 및 권장 사항 |
| :--- | :--- | :--- |
| **무료 유형** | ✅ **`daily_refresh` (매일 자동 초기화)** | 매일 한국시간 01:00 (UTC+8 00:00) 기준 자동 갱신 |
| **일일 무료 한도** | **250 모리 / 일** | 일일 로그인 기본 200 모리 + 인증된 알리바바 클라우드 계정 연동 시 50 모리 |
| **예상 호출 횟수** | **약 125 ~ 500 회 / 일** | 호출하는 모델의 파라미터 규격에 따라 차등 차감 |
| **유효기간 규정** | **단기 모리는 24시간 후 만료** | 매일 지급되는 250 모리는 당일만 유효(이월 불가). 신규 가입 시 50 장기 모리(90일 유효) 추가 지급 |
| **신용카드 등록** | ❌ **전혀 불필요** | 과금 위험 없는 완전 무료 구조 |
| **실명 인증 조건** | ⚠️ **필수 조건** | 중국 본토 신분증으로 실명 인증된 알리바바 클라우드 계정(Alipay 연동) 필요 |
| **CORS 지원** | 🟢 **네이티브 개방 (`*`)** | OPTIONS 요청 시 `access-control-allow-origin: *` 헤더 반환, 프론트엔드 직접 호출 가능 |
| **OpenAI 규격 호환** | 🟢 **완전 호환** | 표준 엔드포인트 `https://api-inference.modelscope.cn/v1` |
| **한도 초과 시 동작** | `HTTP 429 Too Many Requests` 반환 | 한도 도달 시 안전하게 차단되며 자동 결제 위험 제로 |

---

## 2. 모델 규격별 모리 소비율 분석

모리는 모델의 연산 복잡도에 따라 세 가지 티어로 나누어 차감됩니다:

### 1. 경량 모델 티어: 0.5 모리 / 회 (하루 약 500회 무료 호출)
- **대표 모델**: `Qwen/Qwen2.5-7B-Instruct`, `Qwen/Qwen2.5-0.5B`, `Qwen/Qwen2.5-1.5B`
- **추천 활용**: 일상 데이터 포맷 변환, 의도 분류, 경량 코드 보완, 단순 다중 턴 질의응답.

### 2. 주력 모델 티어: 1.0 모리 / 회 (하루 약 250회 무료 호출)
- **대표 모델**: `deepseek-ai/DeepSeek-V3`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`, `ZhipuAI/GLM-4-9B-Chat`
- **추천 활용**: 심층 논리 추론, 복잡한 장문 요약, 정밀 코드 생성 및 사내 지식 베이스 RAG 질의.

### 3. 플래그십 최고급 티어: 2.0 모리 / 회 (하루 약 125회 무료 호출)
- **대표 모델**: `Qwen/Qwen2.5-72B-Instruct` 등 70B+ 대형 플래그십 모델
- **추천 활용**: 에이전트 복합 의사결정, 고급 다국어 번역, 고난도 학술 벤치마크 평가.

---

## 3. 5대 핵심 엔지니어링 주의사항 (Gotchas)

### 1. ⚠️ 알리바바 클라우드 실명 인증 (해외 개발자 진입 장벽)
ModelScope API 추론은 중국 본토 규정을 따릅니다. 사용자는 반드시 알리바바 클라우드 계정을 연동해야 하며, 해당 계정은 중국 본토 신분증(Alipay 스캔)을 통한 실명 인증을 완료해야 합니다. **중국 본토 신분증이 없는 해외 사용자는 무료 API 추론 권한을 이용하기 어렵습니다.**

### 2. ⚠️ 단기 모리는 24시간 내 소멸 (이월 불가)
매일 제공되는 250 모리는 당일 24시간 동안만 유효하며 자정에 자동 소멸됩니다. 모리를 모아둘 수 없으므로 당일 필요한 테스트에 아낌없이 사용하는 것이 좋습니다. 시스템은 항상 유효기간이 가장 임박한 모리부터 자동 차감합니다.

### 3. ⚠️ 네이티브 브라우저 CORS 지원 (프론트엔드 직접 fetch 가능)
대다수 클라우드 게이트웨이가 브라우저 교차 출처 요청을 차단하는 것과 달리, ModelScope는 OPTIONS 요청에 `access-control-allow-origin: *`를 반환합니다. 별도의 Node.js나 Nginx 프록시 서버 없이도 Vue/React SPA나 크롬 확장 프로그램에서 직접 `fetch()`를 수행할 수 있습니다.

### 4. ⚠️ 상용 프로덕션 환경 사용 금지 및 동적 동시성 제어
공식 정책상 무료 API 추론은 학습, 연구, 평가 목적으로만 제한되며 **SLA 가용성을 보장하지 않습니다**. 전체 서버 부하에 따라 동적으로 동시 호출이 제한되며 피크 시간대에는 `HTTP 429` 에러가 자주 반환될 수 있습니다.

### 5. ⚠️ 조직 네임스페이스를 포함한 전체 모델 ID 필수
OpenAI SDK 등으로 호출 시 `model` 인자에는 반드시 조직 접두사가 포함된 전체 이름을 입력해야 합니다:
- 잘못된 형식: `model: "Qwen2.5-72B-Instruct"` ❌
- 올바른 형식: `model: "Qwen/Qwen2.5-72B-Instruct"` ✅
- 올바른 형식: `model: "deepseek-ai/DeepSeek-V3"` ✅

---

## 4. 빠른 연동 코드

### 1. Python (OpenAI SDK 표준 규격)

```python
import os
from openai import OpenAI

# modelscope.cn/apikey 에서 복사한 Access Token 입력
client = OpenAI(
    api_key=os.environ.get("MODELSCOPE_API_KEY", "your_modelscope_access_token"),
    base_url="https://api-inference.modelscope.cn/v1"
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-72B-Instruct",
    messages=[
        {"role": "system", "content": "당신은 알리바바 ModelScope에서 호스팅하는 Qwen 2.5 72B 모델입니다."},
        {"role": "user", "content": "ModelScope API 추론의 주요 장점 3가지를 정리해 주세요."}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

### 2. cURL 명령줄 테스트

```bash
curl -X POST "https://api-inference.modelscope.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1-Distill-Qwen-32B",
    "messages": [{"role": "user", "content": "9.11과 9.8 중 어느 숫자가 더 큰가요?"}],
    "temperature": 0.6
  }'
```

---

## 5. 3단계 시작하기

1. **가입 및 연동**: [modelscope.cn](https://modelscope.cn/) 가입 후 계정 설정에서 실명 인증된 알리바바 클라우드 계정을 연동합니다;
2. **API 토큰 발급**: [modelscope.cn/apikey](https://modelscope.cn/apikey) 페이지에서 자동 생성된 Access Token을 복사합니다;
3. **클라이언트 설정**: Base URL을 `https://api-inference.modelscope.cn/v1`로 설정하고 토큰과 모델명 `Qwen/Qwen2.5-72B-Instruct`를 지정하면 매일 250 모리의 무료 추론을 바로 시작할 수 있습니다!

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/modelscope-api-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.

---
title: "Moonshot Kimi API 완벽 실측 가이드: 중국 실명 인증 시 15위안 쿠폰 증정, 글로벌 $1 충전 기준 및 선불제 무위험 정책"
description: "칭화대 출신 AI 유니콘 문샷(Moonshot Kimi) 개발자 플랫폼 심층 평가! '중국 국내 포털 실명 인증 시 15위안 바우처(~150만 토큰)'와 '글로벌 포털 무료 티어 부재($1 최소 충전)'의 이원화 구조, 신형 플래그십 K3 언락 조건, Tier 등급제, CORS 설정 및 5대 핵심 주의사항 총정리."
pubDate: 2026-10-19
author: "FreeAI Hub 편집부"
category: "国内直连"
tags: ["Moonshot", "Kimi", "긴문맥", "바우처", "선불제", "직접연결", "중국AI"]
featured: false
---

# Moonshot Kimi API 완벽 실측 가이드: 실명 인증 15위안 쿠폰, 글로벌 충전 기준 및 선불제 무위험 정책

칭화대학교 양즈린(Yang Zhilin) 박사가 이끄는 중국 대표 AI 유니콘 **문샷 AI(Moonshot AI, 月之暗面)** 의 어시스턴트 **Kimi**는 최대 200만 토큰(소비자 앱 기준)에 달하는 초장문 문맥 무손실 이해, 정밀한 실시간 웹 검색 연동, 학술 수준의 문서 분석 역량으로 아시아 및 전 세계 개발자들 사이에서 큰 주목을 받고 있습니다.

하지만 개발자용 API를 도입하려는 과정에서 많은 사용자들이 혼란을 겪습니다: **"Kimi API는 무료로 체험할 수 있는가, 아니면 완전 유료인가?", "계정을 만들었는데 왜 크레딧 부족 에러가 발생하는가?"**

2026년 10월 최신 실측 결과, Moonshot은 매우 뚜렷한 **'국내/글로벌 이원화(Dual-Track)'** 운영 정책을 취하고 있는 것으로 확인되었습니다:
- 🇨🇳 **중국 국내 포털 (`platform.moonshot.cn`)**: 중국 본토 휴대전화 번호로 가입하고 신분증 실명 인증을 완료하면, **15위안 상당의 API 바우처(약 150만 토큰 분량)가 자동 지급**되어 사전 결제 없이 즉시 `moonshot-v1` 시리즈를 호출할 수 있습니다!
- 🌐 **글로벌 포털 (`platform.moonshot.ai`)**: 해외 개발자를 겨냥한 미 달러화(USD) 기반 선불 모델로 운영되며, **무료 체험 티어(Free Tier)가 전혀 제공되지 않습니다**. Tier 0을 활성화하려면 최소 $1의 사전 충전이 필수입니다.
- 🔒 **과금 리스크 완전 제로**: 국내 포털과 글로벌 포털 모두 **순수 선불제(Prepaid) 아키텍처**로 동작합니다. 바우처나 잔액이 소진되면 `HTTP 402 Payment Required`를 반환하므로, **신용카드 무단 자동 결제나 예상치 못한 청구 폭탄의 위험이 전혀 없습니다**!

본 문서에서는 국내/글로벌 포털의 차이점, 15위안 바우처 적용 가능 모델, Tier 0~5 단계별 제한, 그리고 5가지 핵심 엔지니어링 주의사항을 상세히 분석합니다.

---

## 1. 핵심 정책 및 이원화 체계: 15위안 바우처 vs. 최소 $1 충전

| 평가 항목 | 중국 국내 포털 (`platform.moonshot.cn`) | 글로벌 포털 (`platform.moonshot.ai`) | 핵심 분석 및 권장사항 |
| :--- | :--- | :--- | :--- |
| **무료 티어 유형** | 🟢 **`voucher` (실명 인증 시 15위안 바우처)** | ❌ **`none` (무료 티어 없음)** | 중국 본토 인증 수단이 있다면 국내 포털 이용 권장 |
| **이용 시작 요건** | 전화번호 가입 + **중국 신분증 실명 인증** | 최소 **$1 결제**로 Tier 0 활성화 | 글로벌 포털은 완전 유료 선불제 |
| **지원 모델 범위** | **`moonshot-v1-8k/32k/128k` 전 시리즈** | 충전 Tier 등급에 따라 전 모델 해금 | **주의: 15위안 바우처는 K3 모델에 직접 사용 불가** |
| **추가 혜택 바우처** | 정기적인 자동 지급 없음 | 누적 $5 충전 시 $5 바우처 증정 | 프로모션 바우처는 Tier 승급 누적 금액에 미산입 |
| **초과 시 동작** | 엄격한 선불제: 잔액 소진 시 `HTTP 402` | 엄격한 선불제: 잔액 소진 시 `HTTP 402` | 신용카드 자동 결제 위험 제로, 안전한 테스트 보장 |
| **CORS 크로스 도메인**| ⚠️ 요청 Origin 정확 매칭 (와일드카드 `*` 아님) | ⚠️ 요청 Origin 정확 매칭 (와일드카드 `*` 아님) | OPTIONS 프리플라이트 시 요청 출처를 그대로 반환 |
| **OpenAI SDK 호환** | 🟢 `/v1/chat/completions` 완벽 호환 | 🟢 `/v1/chat/completions` 완벽 호환 | Base URL 및 API Key 교체만으로 즉시 연동 |

---

## 2. 모델별 가용성 및 언락 조건

### 1. 15위안 바우처 사용 가능 모델 (상용 프로덕션 장문 처리)
실명 인증 완료 시 지급되는 15위안 바우처로 다음 핵심 모델들을 전액 공제하며 사용할 수 있습니다:
- **`moonshot-v1-8k`** (입력 ¥0.012 / 1K tokens, 출력 ¥0.012 / 1K tokens): 범용 대화의 주력 모델로 자연스러운 언어 이해력을 제공하며, 약 125만 토큰의 대화 처리가 가능합니다;
- **`moonshot-v1-32k`**: 중장문 보고서, 금융 리포트, 계약서 문서 분석 및 질의응답;
- **`moonshot-v1-128k`**: 최대 128K 컨텍스트를 활용한 대규모 코드베이스 감사 및 다중 논문 비교 분석.

### 2. 현금 충전이 필요한 고성능 모델
- **`kimi-k3`**: Moonshot의 차세대 추론 플래그십 모델. 공식 정책상 **신규 사용자 15위안 바우처는 K3에 직접 사용할 수 없습니다**. K3를 이용하려면 최소 1회(최소 10위안 또는 $1)의 현금 충전이 필요합니다;
- **`kimi-k2.7-code`**: 고난도 코딩 및 알고리즘 추론에 최적화된 코드 특화 모델.

---

## 3. 5대 핵심 엔지니어링 주의사항 (Gotchas)

### 1. ⚠️ 15위안 바우처를 받으려면 반드시 국내 포털로 접속
GitHub 등의 영문 링크를 통해 `platform.moonshot.ai`(글로벌 포털)로 잘못 접속하면 $1 사전 충전을 요구하므로 무료 혜택이 사라졌다고 오해하기 쉽습니다. **반드시 `platform.moonshot.cn`에 접속**하여 계정 설정에서 신분증 실명 인증을 진행해야 15위안 바우처가 활성화됩니다.

### 2. ⚠️ 15위안 바우처로는 Kimi K3 호출 불가
`kimi-k3` 호출 시 잔액 부족이나 권한 오류가 발생한다면 바우처가 만료된 것이 아니라 신형 플래그십 보호 조치 때문입니다. 15위안 바우처는 `moonshot-v1-8k/32k/128k`를 지원합니다. K3 추론 기능을 사용하려면 콘솔에서 최소 10위안을 충전해야 합니다.

### 3. ⚠️ 무료 바우처는 Tier 누적 금액에 미포함
Moonshot API는 실제 **누적 현금 충전 금액**을 기준으로 Tier 0부터 Tier 5까지의 속도 제한(RPM/TPM/동시성)을 부여합니다. 무료 15위안 바우처는 누적 충전 금액에 산입되지 않으므로, 바우처만 이용할 때는 기본 티어의 안전 제한이 유지됩니다.

### 4. ⚠️ 일반 Kimi 앱과 개발자 API는 완전 분리 운영
`kimi.com` 웹사이트나 모바일 앱의 무료 이용량 및 VIP 회원 혜택은 개발자용 API 플랫폼과 완전히 독립되어 있습니다. 일반 앱 혜택을 API 토큰으로 전환하거나 그 반대로 전환할 수 없습니다.

### 5. ⚠️ CORS는 요청 Origin 정확 에코 방식
`api.moonshot.cn`의 OPTIONS 사전 요청(preflight) 검증 결과, `access-control-allow-origin` 헤더는 와일드카드 `*`가 아닌 요청을 보낸 도메인을 그대로 반영(Echo)합니다. 브라우저 프론트엔드 단독 호출이나 웹 확장 프로그램 제작 시 Origin 헤더 처리에 주의하세요.

---

## 4. 빠른 연동 예제 코드

### 1. Python (OpenAI SDK 표준 규격)

```python
import os
from openai import OpenAI

# platform.moonshot.cn에서 발급받은 sk- 키 입력
client = OpenAI(
    api_key=os.environ.get("MOONSHOT_API_KEY", "your_moonshot_api_key"),
    base_url="https://api.moonshot.cn/v1"
)

response = client.chat.completions.create(
    model="moonshot-v1-8k",
    messages=[
        {"role": "system", "content": "당신은 Moonshot AI가 개발한 장문 분석 전문 Kimi 비서입니다."},
        {"role": "user", "content": "Moonshot Kimi의 핵심 장점 3가지를 요약해 주세요."}
    ],
    temperature=0.3
)

print(response.choices[0].message.content)
```

### 2. cURL 명령줄 테스트

```bash
curl -X POST "https://api.moonshot.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MOONSHOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "moonshot-v1-8k",
    "messages": [{"role": "user", "content": "9.11과 9.8 중 어느 숫자가 더 큰가요?"}],
    "temperature": 0.3
  }'
```

---

## 5. 3단계 시작 가이드

1. **국내 포털 회원가입**: [platform.moonshot.cn](https://platform.moonshot.cn/)에 접속하여 전화번호 가입 후 실명 인증을 완료합니다;
2. **바우처 확인 및 API 키 발급**: '비용 센터'에서 15위안 바우처 적립을 확인하고, 'API Key 관리'에서 `sk-` 키를 생성합니다;
3. **클라이언트 설정 및 호출**: Base URL을 `https://api.moonshot.cn/v1`로 지정하고 모델을 `moonshot-v1-8k` 또는 `moonshot-v1-32k`로 선택하여 128K 장문 처리 능력을 비용 부담 없이 활용하세요!

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/moonshot-kimi-api-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.

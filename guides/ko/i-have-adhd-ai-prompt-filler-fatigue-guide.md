---
title: "AI의 쓸데없는 군더더기와 토큰 낭비에 마침표를! GitHub 화제의 규칙 'i-have-adhd' 완벽 실전 가이드: Claude Code / Cursor / ChatGPT를 1초 만에 행동 중심 머신으로"
description: "AI에게 코드 에러를 물어볼 때마다 장황한 서론과 예의 바른 인사치레에 지치셨나요? GitHub에서 수만 개의 Star를 받으며 화제가 된 오픈소스 ayghri/i-have-adhd를 완벽 분석합니다. 10가지 강력한 인지 가드레일로 첫 줄에 실행 명령어를 즉시 출력하고 불필요한 토큰 낭비를 40~60% 절감하는 비결. Claude Code, Cursor, ChatGPT 설정법 및 프롬프트 템플릿 포함."
pubDate: 2026-10-26
author: "샤오커 (FreeAI Dog 코어 개발팀)"
category: "실전 가이드"
tags: ["프롬프트엔지니어링", "ClaudeCode", "Cursor", "i-have-adhd", "토큰최적화", "AI코딩어시스턴트", "생산성해킹", "시스템규칙"]
featured: true
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# AI의 쓸데없는 군더더기와 토큰 낭비에 마침표를! GitHub 화제의 규칙 'i-have-adhd' 완벽 실전 가이드: Claude Code / Cursor / ChatGPT를 1초 만에 행동 중심 머신으로

AI를 활용해 코딩을 하거나 기술적인 문제를 해결할 때, 이런 답답한 경험을 겪어본 적이 있으실 겁니다.

단순한 포트 충돌 오류가 발생해 당장 프로세스를 강제 종료할 명령어 한 줄만 필요한데, AI가 장문의 '인사치레'와 '개념 설명'부터 늘어놓는 상황입니다:

> *"정말 좋은 질문입니다! 이 네트워크 바인딩 문제를 해결하는 데 기꺼이 도움을 드리겠습니다. 현대의 분산 컨테이너 환경에서 포트 할당은 핵심적인 요소입니다. 보통 이 에러는 세 가지 주요 원인으로 인해 발생할 수 있습니다. 차근차근 하나씩 깊이 있게 살펴보겠습니다…… (여기서 세 화면에 달하는 이론 설명 생략)…… 이 설명이 문제 해결에 도움이 되기를 바랍니다! 추가로 궁금한 점이 있으시면 언제든지 편하게 말씀해 주세요!"*

개발자는 이 장황한 '목 가다듬기(Throat-clearing)' 서론과 불필요한 감정적 배려, 그리고 요약문 사이를 한참 스크롤해 내려가, 화면 가장 맨 밑바닥에서**원래 1초 만에 복사해 실행할 수 있었던 단 한 줄의 터미널 명령어**(`lsof -ti :8080 | xargs kill -9`)를 눈이 빠지도록 찾아내야 합니다.

이러한 현상을 업계에서는 **'군더더기 피로(Filler Fatigue)'**라고 부릅니다. 개발자의 소중한 몰입(Flow) 상태를 끊임없이 깨뜨릴 뿐만 아니라, 매 대화 턴마다 수백 개의 무의미한 토큰을 낭비하여 API 비용을 증가시키고 컨텍스트 윈도우를 불필요하게 잠식하는 **'토큰 암살자'**이기도 합니다.

오늘 소개할 도구는 GitHub, Reddit, X에서 전 세계 엔지니어들의 폭발적인 공감을 얻고 있는 오픈소스 규칙 프레임워크인 [`ayghri/i-have-adhd`](https://github.com/ayghri/i-have-adhd)입니다. 모델 가중치를 수정하지 않고 오직 강력한 프롬프트 가드레일만으로 Claude Code, Cursor, ChatGPT, Gemini를 **'첫 줄부터 실행 명령어를 직격하는 정예 오퍼레이터'**로 완벽히 개조합니다!

---

## 1. 탄생 배경: `i-have-adhd`란 무엇인가?

이 오픈소스 스킬은 해외 개발자 `ayghri`가 제작한 것으로, 성인 ADHD 임상 심리학의 명저인 《성인 ADHD 실전 가이드》(Ramsay & Rostain 저, *The Adult ADHD Tool Kit*)에서 직접적인 영감을 얻었습니다.

하지만 분명히 짚고 넘어가야 할 점이 있습니다: **이 도구는 결코 ADHD 당사자만을 위한 도구가 아닙니다. 쓸데없는 군더더기를 혐오하고 최고의 효율을 추구하는 모든 개발자를 위한 복음입니다!**

* **심리학적 메커니즘**: ADHD 실전 도구의 핵심은 잡다한 자극을 제거하고 행동 단위를 작고 명확하게 쪼개어 '실행 시작에 드는 마찰력(Activation Energy)'을 낮추는 것입니다;
* **LLM 행동의 재구성**: 연구진은 현대 대형 언어 모델의 기본 정렬(RLHF)이 과도한 친절과 방어적인 서론으로 가득 차 있다는 점을 발견했습니다. 심리학적 가드레일을 **강력한 프롬프트 제약 규칙**으로 변환함으로써, 수다스러운 튜터 같던 AI를 '명령 즉각 실행 체계'로 강제 전환시킨 것입니다.

실제 커뮤니티 벤치마크 결과, 이 규칙을 적용한 뒤 답변의 즉각 실행 가능성은 **10배** 향상되었으며, 대화 턴당 무의미한 토큰 소모가 **40%~60% 감소**하는 것으로 나타났습니다!

---

## 2. 타협 없는 10대 핵심 인지 가드레일

`i-have-adhd`가 즉각적인 효과를 발휘하는 이유는 타협의 여지를 주지 않는 10가지 하드코딩 규칙을 적용하기 때문입니다:

```
┌─────────────────────────────────────────────────────────────────┐
│               i-have-adhd 3대 핵심 통제 전선                      │
├─────────────────────────────────────────────────────────────────┤
│  ⚡ 행동 우선 원칙 (Action-First)   👉 1번 줄에 즉각 실행 명령어/코드 직출 │
│  🚫 군더더기 제로 (Zero Filler)     👉 인사치레, 자발적 배경설명 일체 차단 │
│  🔢 인지 부하 통제 (Strict Caps)    👉 단계별 번호 매김, 목록 5개 이하 제한│
└─────────────────────────────────────────────────────────────────┘
```

### 1. 첫 줄에 즉각 실행 조치 배치 (Lead with the next action)
답변의 첫 번째 줄은 **즉시 실행할 수 있는 명령어, 수정할 파일 경로, 또는 핵심 코드 블록**이어야 합니다. 어떠한 서론이나 배경 설명도 첫 줄에 올 수 없습니다.

### 2. 다단계 작업 번호 매김 (Number multi-step tasks)
"먼저…… 이어서…… 그 다음…… 마지막으로……"와 같은 산문 형태의 줄글 설명을 금지합니다. 단계가 나뉠 경우 반드시 `1. 2. 3.` 번호 목록으로 출력해야 합니다.

### 3. 답변 끝에는 구체적인 단일 조치만 제시 (End with one concrete next step)
답변 말미에는 **사용자가 취해야 할 단 하나의 명확한 다음 행동**만을 남깁니다. "더 궁금한 점이 있으신가요?"와 같은 개방형 질문은 엄격히 배제합니다.

### 4. 부차적인 지식 발산 억제 (Suppress tangents)
질문받지 않은 '주변 지식 설명'을 자발적으로 늘어놓는 행위를 차단합니다. 묻는 것에만 정확히 답하고, 아는 척을 하지 않습니다.

### 5. 매 턴마다 현재 진행 상태 간략 고정 (Restate state briefly every turn)
"Auth 미들웨어 수정 완료, 테스트 실행 대기 중"과 같이 반 문장으로 현재 상태를 명시하여 긴 대화에서 발생할 수 있는 문맥 이탈(Drift)을 방지합니다.

### 6. 구체적인 예상 소요 시간 제시 (Specific time estimates)
"금방 끝납니다", "잠시만 기다려 주세요" 같은 모호한 표현을 금지하고, "빌드 소요 시간 약 2분"과 같이 정확한 숫자로 전달합니다.

### 7. 완료된 성과를 가시화 (Make wins visible)
통과된 테스트나 해결된 작업은 `[PASS]`처럼 명시적으로 표시하여 현재 상황을 명확히 파악할 수 있도록 돕습니다.

### 8. 오류에 대해 객관적이고 담담한 태도 유지 (Matter-of-fact tone for errors)
버그나 실패 상황에서 과도한 사과나 감정적 완충 표현(*"I apologize for the confusion..."*)을 일체 금지하며, 오직 팩트와 해결책만 제시합니다.

### 9. 목록 항목은 최대 5개로 제한 (Cap lists at 5 items)
인간의 작업 기억 용량 한계를 고려하여, 한 번에 출력하는 목록은 5개 이하로 유지하고 초과 시 그룹화하거나 핵심만 추립니다.

### 10. 답변 발송 전 4단계 자체 필터링 (Pre-send self-filter)
답변을 내보내기 직전 AI가 스스로 다음 항목을 검열하여 삭제합니다:
* ❌ 첫머리 인사치레 ("좋은 질문입니다!", "도와드리게 되어 기쁩니다")
* ❌ 생각 과정 독백 ("문제를 차근차근 분석해 보겠습니다……")
* ❌ 맺음말 상투어 ("도움이 되셨기를 바랍니다!")
* ❌ 불필요한 반복 총정리.

---

## 3. 실전 전후 비교: Before vs After의 극명한 차이

도커 포트 충돌 에러를 해결하는 실제 사례를 통해 차이점을 비교해 보겠습니다:

### ❌ 기본 모델 답변 (480 Tokens - 높은 인지 피로)
> *"컨테이너 배포 시 매우 흔하게 발생하는 문제입니다! `bind: address already in use` 오류는 호스트의 8080 포트를 다른 프로세스가 사용 중임을 의미합니다. 보통 3단계 접근 방식으로 해결할 수 있습니다. 먼저 포트를 확인하고, 프로세스를 종료한 뒤, 컨테이너를 재시작합니다. 구체적인 명령어는 다음과 같습니다…… (아래로 한참 내려가야 명령어가 나옴)"*

### ✅ i-have-adhd 적용 후 (85 Tokens - 1초 만에 즉시 실행)
> 8080 포트를 점유 중인 프로세스를 강제 종료하려면 아래 명령어를 실행하세요:
> ```bash
> lsof -ti :8080 | xargs kill -9
> ```
> 1. 위 강제 종료 명령어 실행 (소요 시간 약 2초);
> 2. 컨테이너 재시작: `docker-compose up -d`;
> 
> 다음 작업: `docker ps`를 실행해 컨테이너 구동 상태 확인.

**결과**: 토큰 소모량이 480개에서 85개로 감소(**약 82%의 낭비 토큰 절감**). 답변이 출력된 지 0.1초 만에 명령어를 복사해 터미널에 붙여넣을 수 있습니다.

---

## 4. 툴별 초간단 설치 및 적용 방법

### 1. Claude Code CLI (공식 플러그인 설치)
Anthropic의 공식 터미널 도구 **Claude Code**에서는 공식 플러그인으로 즉시 등록할 수 있습니다:

```bash
# 플러그인 마켓플레이스 등록
claude plugin marketplace add ayghri/i-have-adhd

# 플러그인 본체 설치
claude plugin install i-have-adhd@i-have-adhd
```

* **수동 호출**: 세션 내에서 `/i-have-adhd` 입력;
* **상시 활성화**: 터미널에서 아래 파일을 생성하면 이후 매 세션마다 자동으로 적용됩니다:
  ```bash
  touch ~/.claude/.i-have-adhd-always
  ```
* **모드 해제**: 채팅창에 `stop adhd mode`를 입력하면 기본 대화 모드로 복귀합니다.

---

### 2. Cursor / Windsurf (프로젝트 및 글로벌 설정)

* **프로젝트 단위 설정(권장)**: 프로젝트 루트 디렉터리에 `.cursorrules` 파일을 생성하고 아래 부록의 템플릿을 붙여넣어 저장합니다.
* **글로벌 설정**: Cursor의 `Settings` -> `General` -> `Rules for AI`에 템플릿을 등록하면 인라인 편집(`Ctrl+K`)과 채팅(`Ctrl+L`) 모두 군더더기 없는 상태로 영구 전환됩니다.

---

### 3. ChatGPT / Claude.ai / Gemini 웹 버전
1. 웹 서비스 설정 -> **Custom Instructions(사용자 지정 지침)** 메뉴로 이동;
2. **"AI가 어떻게 응답하기를 바라시나요?"** 입력 필드에 아래 템플릿을 붙여넣고 저장.

---

## 부록: 원클릭 복사 가능한 표준 프롬프트 템플릿

```markdown
# Role & Communication Guardrails: Action-First System

Follow Ramsay & Rostain's Adult ADHD Toolkit cognitive guardrails strictly:

1. Lead with the next action: Line 1 MUST be an executable action, terminal command, or file path. No preamble.
2. Number multi-step tasks: Always use 1, 2, 3 for multi-step instructions.
3. End with one concrete next step: Never leave open-ended questions. Specify exactly one next action.
4. Suppress tangents: Do not volunteer unasked explanations or related-but-unsolicited background.
5. Restate state briefly every turn: Ground progress in half a sentence to prevent drift.
6. Specific time estimates: Use exact numbers (e.g., "about 3 min") rather than vague qualifiers.
7. Make wins visible: Explicitly acknowledge completed milestones and passing tests.
8. Matter-of-fact errors: State problems plainly without apologies, emotional cushioning, or fluff.
9. Cap lists at 5 items: Never output massive lists. Group or trim to ≤5 items.
10. Pre-send self-filter (STRICT):
    - NO preamble ("Great question!", "Sure, I can help with that", "Let me think...")
    - NO throat-clearing recap before the answer.
    - NO conversational closers ("Hope this helps!", "Let me know if you need anything else!").
```

---

## 마치며

더 큰 매개변수(125B, 405B)와 값비싼 하드웨어를 추구하는 것만큼이나, **'AI와의 상호작용 프로토콜에서 군더더기를 걷어내는 것'**은 개발자의 일상적인 생산성과 직결됩니다.

유닉스(Unix)의 위대한 철학인 **"한 가지 일만 완벽하게 하라(Do one thing and do it well)"**처럼, 오늘부터 여러분의 AI 비서에게서 불필요한 사설을 걷어내고 최고 효율의 몰입 환경을 경험해 보세요!

더 많은 실전 개발 팁과 무료 고속 API에 대한 정보는 [FreeAI Dog (freeaidog.com)](https://freeaidog.com/ko/)에서 확인하실 수 있습니다!

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


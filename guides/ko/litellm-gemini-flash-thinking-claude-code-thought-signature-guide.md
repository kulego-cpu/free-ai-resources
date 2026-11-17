---
title: "LiteLLM이 해결한 Gemini 생각 서명 블랙홀: Claude Code와 Cursor에 1M 추론 스트림을 0원으로 연결하는 실전"
description: "LiteLLM 최신 커밋 da296ee의 Google Gemini 생각 서명 투과 수정을 분석합니다. 다중 왕복 도구 호출 시 400 thought_signature 누락 오류가 발생하는 원인, 로컬 프록시 설정 SOP, 1M 문맥 무료 활용법을 정리했습니다."
pubDate: 2026-11-17
author: "샤오커 [小可]"
category: "아키텍처"
tags: ["LiteLLM", "Gemini", "ClaudeCode", "Cursor", "추론모델", "문제해결"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# LiteLLM이 해결한 Gemini 생각 서명 블랙홀: Claude Code와 Cursor에 1M 추론 스트림을 0원으로 연결하는 실전

그저께 밤 수백 개 파일로 구성된 레거시 프로젝트 리팩터링을 진행하고 있었습니다.

터미널에서는 Claude Code가 실행 중이었습니다.

공식 Claude 3.7 Sonnet의 긴 생각 사슬은 분명 강력합니다. 코드 추론 능력이 뛰어납니다. 하지만 토큰 소모 속도는 지갑을 빠르게 비웁니다.

출력 토큰 100만 개당 가격이 15달러에 달합니다.

코딩 작업을 몇 시간만 진행해도 수십 달러가 청구됩니다. 이를 지속적으로 감당하기는 쉽지 않습니다.

대안을 검토했습니다. Google AI Studio의 Gemini 2.5 Flash는 무료 할당량이 넉넉합니다. 1M 토큰의 초장문 문맥을 지원하며, 생각 스트림 출력도 무료로 개방되어 있습니다.

비용 없이 1M 문맥을 Claude Code의 엔진으로 사용할 수 있다면 이상적입니다.

로컬에서 LiteLLM 프록시를 실행하고 Anthropic 프로토콜 엔드포인트를 Gemini 모델로 우회했습니다.

첫 번째 응답은 매우 매끄러웠습니다.

풍부한 생각 과정을 담은 답변이 출력되었습니다.

하지만 두 번째 상호작용에 진입하자마자 문제가 발생했습니다. Claude Code가 빌드를 위해 Bash 명령을 실행하거나 FileEdit으로 코드를 수정하려 하자 터미널에 오류가 출력되었습니다.

세션이 즉시 중단되었습니다.

화면에는 Google API 원본 오류가 나타났습니다:

```text
400 Invalid argument: Function call is missing a thought_signature
```

경우에 따라서는 오류 메시지도 없이 단일 문자를 무한 반복 출력하며 멈추기도 했습니다.

무료 활용 구상은 단 한 줄의 오류로 중단되었습니다.

이 thought_signature 라는 것은 어떤 메커니즘일까요?

단발성 텍스트 응답은 정상인데 왜 도구 호출만 실행하면 즉시 오류가 발생할까요?

---

## 1. 프로토콜 단절: Google 암호화 서명과 Anthropic 메시지 규격의 충돌

양쪽 프로토콜 패킷을 직접 분석해 원인을 규명했습니다.

근본 원인은 Google Gemini 모델 내부의 검증 메커니즘에 있었습니다.

Google의 추론 체계에서는 생각 과정을 포함하는 모델이 Function Call 매개변수를 생성할 때 thought_signature 라는 암호화 검증 서명을 강제로 첨부합니다.

이 서명은 해당 도구 호출이 어떤 생각 사슬에서 도출되었는지를 증명합니다.

Google API 규정에 따르면, 후속 턴에서 클라이언트가 도구 실행 결과를 반환할 때 이전 단계의 thought_signature를 원본 그대로 함께 전달해야 합니다.

서명이 누락되면 어떻게 될까요? Google 백엔드는 요청을 즉시 차단합니다. 400 오류가 발생하거나 문맥이 꼬여 무한 루프에 빠집니다.

그렇다면 이 서명은 왜 LiteLLM 프록시 계층에서 사라졌을까요?

원인은 Anthropic Messages API의 규격 정의에 있습니다.

Claude Code는 Anthropic 표준을 기반으로 개발되었으며 /v1/messages 엔드포인트를 통해 통신합니다.

Anthropic 프로토콜 구조상 도구 호출은 tool_use 와 tool_result 두 가지 표준 데이터 블록만 지원합니다.

JSON Schema에는 타사의 고유 서명을 담을 수 있는 공식 필드가 없습니다.

초기 LiteLLM은 서명을 확장 필드에 넣거나 tool_use ID 문자열 끝에 덧붙이는 방식을 시도했습니다. 하지만 이는 통하지 않았습니다.

Claude Code 클라이언트는 전달 전 Anthropic 규격에 맞춰 엄격하게 입력값을 검증하며, 비표준 필드는 제거합니다.

클라이언트가 로컬 명령을 실행한 후 tool_result를 묶어 회신할 때, 비표준 필드는 이미 사라진 상태입니다.

LiteLLM은 서명이 없는 상태로 Google에 요청을 보낼 수밖에 없습니다.

Google은 서명 누락을 감지하고 실행을 거부합니다.

다중 턴 프록시 연결이 이 프로토콜 단절로 인해 중단된 것입니다.

---

## 2. 패치 분석: 세션별 메모리 임시 보관과 자동 재주입

클라이언트가 서명을 보존해 줄 것을 기대할 수는 없습니다. Claude Code는 공식 tool_use 및 tool_result 블록만 인식하며 추가 필드는 정리합니다. 따라서 LiteLLM 프록시 계층이 직접 기억해야 합니다.

커밋 da296ee4bfa706e2924ef4294d5b3d9802a2aa38 에서 LiteLLM은 Anthropic 어댑터의 서명 처리 구조를 개편했습니다.

클라이언트의 데이터 전달에 의존하지 않는 구조로 재설계되었습니다.

첫 번째 수정 사항——서버 측 도구 호출 캐시 계층:

Gemini가 thought_signature를 포함한 도구 호출을 반환하면, LiteLLM은 클라이언트에 의존하지 않고 프록시 내부 메모리에 tool_call_id 및 세션 문맥 해시로 연결된 캐시 공간을 할당합니다.

서명 데이터는 프록시 메모리에 직접 보관됩니다.

두 번째 수정 사항——송신 패킷 가로채기 및 서명 재주입:

Claude Code가 실행 결과를 POST /v1/messages 로 반환하면, 어댑터는 tool_result의 해당 ID를 확인합니다.

내부 메모리에서 보관 중이던 thought_signature를 추출해 Google로 향하는 원본 요청에 자동으로 다시 주입합니다.

클라이언트는 서명의 존재를 인지할 필요가 없습니다.

데이터 흐름 구조는 다음과 같습니다:

```text
Claude Code 터미널 클라이언트 [Anthropic 표준 준수]
      │
      ▼  POST /v1/messages [서명 없는 tool_result 반환]
LiteLLM 동적 프록시 [커밋 da296ee 적용]
├─ tool_call_id 매핑 캐시 확인
├─ thought_signature 암호화 서명 자동 재주입
└─ Google AI Studio 규격 REST 패킷으로 변환
      │
      ▼
Google Gemini 2.5 Flash 백엔드 [서명 검증 완료]
├─ 생각 스트림 지속 [1M 문맥 유지]
└─ 정상적인 코드 생성 및 도구 호출 결과 반환
```

세 번째 수정 사항——예외 복구 통로:

메모리 캐시가 누락될 경우를 대비해 skip_thought_signature_validator 스위치도 추가되었습니다. 임시 더미 서명을 삽입해 치명적 충돌을 방지합니다. 다만 문맥 연속성이 저하될 수 있으므로 캐시 보존이 기본 원칙입니다.

이 구조를 통해 다중 턴 도구 호출의 교착 상태가 해결되었습니다.

---

## 3. 로컬 구축: Claude Code와 Cursor 5분 설정 절차

원리를 파악했으므로 실제 설정은 간단합니다.

LiteLLM 패키지를 최신 버전으로 업데이트합니다:

```bash
pip install -U "litellm[proxy]>=1.63.1"
```

소스코드에서 직접 빌드할 경우:

```bash
git clone https://github.com/BerriAI/litellm.git
cd litellm
pip install -e .
```

로컬 경로에 config.yaml 파일을 생성합니다:

```yaml
model_list:
  - model_name: claude-3-7-sonnet-20250219
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY
      thinking:
        type: enabled
        budget_tokens: 4096

general_settings:
  master_key: sk-litellm-local-dev-key
```

모델 이름을 claude-3-7-sonnet-20250219 로 별칭 매핑하는 것이 중요합니다.

이 설정을 통해 Claude Code의 시작 검증을 통과하고 요청을 Gemini로 자연스럽게 전달합니다.

Google AI Studio 키를 환경 변수에 등록하고 프록시를 실행합니다:

```bash
export GEMINI_API_KEY="AIzaSyYourRealKeyHere"
litellm --config config.yaml --port 4000
```

새 터미널 탭을 열고 Claude Code용 환경 변수를 지정합니다:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_API_KEY="sk-litellm-local-dev-key"
claude
```

디렉터리 탐색 및 테스트 실행을 지시했을 때의 실제 터미널 출력 예시입니다:

```text
╭── Claude Code ───────────────────────────────────────────────╮
│ Model: claude-3-7-sonnet-20250219 [via LiteLLM proxy :4000]   │
╰──────────────────────────────────────────────────────────────╯
> 단위 테스트 검증 및 토크나이저 AST 정의 수정
Thinking:
  Need to inspect failing cases. Running cargo test first.
$ cargo test --test parser_test
  Compiling parser v0.1.0 [/Users/cool/code/demo]
  Running unittests src/lib.rs
test parser_test ... ok
Thinking:
  Test passed. Now editing src/parser/ast.rs to update Node enum.
Tool call: Edit(file: "src/parser/ast.rs", ...)
  Updated 1 line in src/parser/ast.rs
Session continued smoothly across 14 turns without thought_signature errors.
```

14턴 이상 파일 검사, 테스트 실행, 코드 수정이 서명 오류 없이 연속 수행되었습니다.

Cursor 설정도 간단합니다: Settings > Models > OpenAI API 항목에서 Base URL에 http://127.0.0.1:4000/v1, API Key에 sk-litellm-local-dev-key 를 입력하면 바로 사용할 수 있습니다.

---

## 4. 비용 계산서: 무료 활용과 상용 API 비교

실제 업무 적용을 위해서는 비용 비교가 필수적입니다.

두 가지 방식의 일상적인 비용 차이입니다:

공식 Anthropic Claude 3.7 Sonnet:
입력 100만 토큰당 3달러, 출력 100만 토큰당 15달러입니다.
프로젝트 리팩터링을 진행하면 하루 50만 입력 토큰과 10만 생각 출력 토큰을 쉽게 소모합니다.
하루 3달러에서 8달러가 나가며, 월간 수백 달러에 이릅니다. 개인 개발자에게는 부담스러운 금액입니다.

Google AI Studio 및 LiteLLM 연동 방식:
Gemini 2.5 Flash 무료 티어는 개인 개발 작업을 충당할 수 있는 넉넉한 한도를 제공합니다.
API 호출 비용이 들지 않습니다.
또한 Gemini 2.5 Flash는 1M 토큰의 장문 문맥을 지원해 Sonnet의 기본 200K를 크게 웃돕니다.
수만 줄의 소스코드를 한 번에 로드할 수 있습니다.

M3 Mac 환경에서 실제 메모리 점유율을 확인했습니다:
LiteLLM 프록시 프로세스는 약 320MB RAM을 상주 점유하며, CPU는 거의 대기 상태를 유지하다 직렬화 시 일시적으로 5% 수준을 보입니다.
개발 작업에 미치는 영향은 미미하지만, 상주 프로세스이므로 메모리가 부족할 때 중복 실행하지 않도록 주의해야 합니다.

---

## 5. 운영 경계 조건 및 자체 점검 항목

유용한 구성이지만 몇 가지 기술적 제약을 인지해야 합니다.

첫째, Google 무료 티어의 호출 빈도 제한입니다.

Google AI Studio 무료 티어는 분당 15회 요청 및 일일 1500회 호출로 제한됩니다.

백그라운드에서 병렬 에이전트를 자동 실행해 다수의 도구 호출을 동시에 날릴 경우 429 오류가 발생할 수 있습니다.

단일 작업에 집중해 활용하고, 대규모 병렬 작업에는 유료 할당량을 병행하십시오.

둘째, 프록시 재시작으로 인한 세션 단절 문제입니다.

thought_signature는 프록시 메모리에 보관되므로 작업 도중 LiteLLM을 재시작하면 캐시가 지워집니다.

이후 도구 응답은 서명 누락으로 400 오류를 냅니다.

pm2나 nohup으로 프록시를 안정적으로 유지해 예기치 않은 재시작을 방지하십시오.

실제 코딩 작업에서 독자 여러분은 어떤 점을 더 중요하게 생각하십니까?

Claude 고유의 정밀한 코드 추론 능력입니까?

아니면 1M 문맥이 제공하는 광범위한 코드 파악 능력입니까?

무료 한도를 고려할 때 다중 키 로테이션 구성도 대안이 될 수 있습니다.

먼저 테스트용 저장소에서 3단계 설정을 실행해 다중 턴 도구 호출의 안정성을 확인해 보시길 권장합니다.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


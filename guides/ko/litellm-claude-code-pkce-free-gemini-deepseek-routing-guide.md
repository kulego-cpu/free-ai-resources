---
title: "Claude Code 터미널 프록시 실전 구축：LiteLLM 기반 PKCE 갱신 처리와 Gemini 2.5 및 DeepSeek 0원 라우팅"
description: "Claude Code 환경 변수 리다이렉션과 PKCE 토큰 만료 원인 심층 분석. LiteLLM 커밋 dc378de 기반 로컬 인증 갱신 매커니즘, Google AI Studio 무료 티어 및 DeepSeek 장애 대응 라우팅 실측."
pubDate: 2026-11-14
author: "FreeAI Dog 편집부"
category: "실전 심화"
tags: ["ClaudeCode", "LiteLLM", "Gemini", "DeepSeek", "로컬프록시", "코드리팩터링", "도구호출", "API게이트웨이"]
featured: true
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/litellm-claude-code-pkce-free-gemini-deepseek-routing-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# Claude Code 터미널 프록시 실전 구축：LiteLLM 기반 PKCE 갱신 처리와 Gemini 2.5 및 DeepSeek 0원 라우팅

지난 금요일 오후 로컬 환경에서 대규모 TypeScript 테스트 프로젝트를 대상으로 Claude Code 자동 마이그레이션을 돌렸습니다.

대상은 복수의 AST 파서를 포함하는 12000라인 규모의 오픈소스 리포지토리였습니다.

시작한 지 30분 만에 모듈 전반의 타입 추론과 비동기 함수 리팩터링 3회차가 완료되었습니다.

그 순간 Anthropic 공식 콘솔에는 4.2달러의 과금이 찍혀 있었습니다.

상당히 쓰라린 비용입니다.

지갑에 직접적인 타격이 옵니다.

터미널 상의 상호작용 자체는 대단히 매끄러웠습니다.

정말 빠릅니다.

하지만 이런 소모 속도로는 매달 발생하는 토큰 요금을 감당하기 어렵습니다.

많은 개발자들이 로컬 리버스 프록시를 세워 서드파티 엔드포인트로 요청을 돌리려고 시도합니다.

그러나 환경 변수만 바꾼 상태에서는 보통 2시간을 넘기지 못합니다.

왜 그럴까요?

세션이 한창 진행되는 도중 터미널이 갑자기 OAuth 토큰 만료 오류를 뿜으며 비정상 종료되기 때문입니다.

로컬 네트워크 지연으로 인한 TLS 타임아웃이었을까요?

처음에는 완전히 엉뚱한 방향으로 원인을 짚었습니다.

완전한 오판이었습니다.

패킷을 캡처해 인증 핸드셰이크를 확인해 보니 구조적인 장벽이 자리 잡고 있었습니다.

어제 오픈소스 게이트웨이인 LiteLLM에 커밋 dc378de207f8이 머지되었습니다.

이 커밋은 Claude Code를 로컬 프록시로 실행할 때 발생하는 PKCE 토큰 갱신 단절 현상을 정밀하게 수정한 패치입니다.

정말 절실했던 수정이었습니다.

커밋 diff를 확인해 보니 proxy_server.py의 수정 분량은 십여 줄에 불과했습니다.

anthropic.py에 _refresh_oauth_token이라는 모의 처리 브랜치가 추가되었고 나머지는 테스트 케이스였습니다.

핵심 원리는 /v1/oauth/refresh로 유입되는 요청을 가로채는 것입니다.

프록시가 로컬에서 생성한 위조 토큰 객체를 반환하며 expires_at을 다음 주기로 밀어냅니다.

요컨대 클라이언트에게 세션이 아직 만료되지 않았다고 속이는 구조입니다.

토큰 갱신 요청은 Anthropic 추론 API가 아니라 완전히 분리된 전용 OAuth 엔드포인트를 거치고 있었습니다.

놓치기 쉬운 설계입니다.

그렇기 때문에 단순하게 ANTHROPIC_BASE_URL만 수정해서는 갱신 패킷을 잡지 못하고 우회되었던 것입니다.

Google AI Studio는 Gemini 2.5 Flash 무료 티어를 상시 제공합니다.

DeepSeek V3 공식 API는 매우 저렴한 장애 대비 보조선 역할을 수행합니다.

이 파이프라인을 로컬에 구축하면 일상적인 개발에서 발생하는 토큰 청구액을 0원에 가깝게 떨어뜨릴 수 있습니다.

---

## 1. 인증 가로채기는 어디서 무너졌는가

Claude Code는 개방형 프로토콜 기반의 일반 CLI 에이전트와 설계 방식이 완전히 다릅니다.

Cline이나 Aider 같은 도구는 표준 HTTP 클라이언트로 동작합니다.

설정 파일에 커스텀 baseURL과 API 키를 지정하면 요청이 정직하게 전송됩니다.

그러나 Claude Code는 Anthropic 자체 생태계에 강하게 결속된 독점형 CLI 도구입니다.

초기 구동과 세션 유지 과정에서 두 가지 인증 경로를 사용합니다:
- 공식 ANTHROPIC_API_KEY를 직접 읽어들여 종량제 요금을 차감하는 경로;
- claude login 명령어를 실행하여 브라우저 기반 PKCE 교환을 거쳐 임시 자격증명을 얻는 경로.

많은 개발자들이 셸에서 로컬 리다이렉션을 시도했습니다:

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:4000
export ANTHROPIC_API_KEY=sk-dummy-mock-token
```

단발성 질의응답은 아무 문제 없이 매끄럽게 작동합니다.

대단히 경쾌합니다.

순조로워 보입니다.

하지만 장시간 소요되는 리팩터링 작업을 맡기면 세션 유지 시간이 2시간을 넘기는 순간 강제 갱신 패킷이 발송됩니다.

일반적인 역방향 프록시는 이 독점 핸드셰이크 브랜치를 구현하지 않았습니다.

게이트웨이는 즉각 401 오류를 반환합니다:

즉시 중단.

```text
Error: Failed to refresh PKCE authentication session
OAuth token expired, server returned 401 Unauthorized
```

20분 동안 구축해 놓은 수만 토큰 분량의 코드 컨텍스트가 프로세스 종료와 함께 순식간에 증발합니다.

예고 없이 터지는 프로세스 중단과 컨텍스트 증발을 견딜 수 있는 엔지니어는 없습니다.

LiteLLM의 커밋 dc378de207f8이 바로 이 단절 지점을 해결했습니다.

게이트웨이가 로컬 루프백 내부에서 갱신 요청을 가로채고 증가된 자격증명을 자체 발급합니다.

가로채기 성공.

정말 훌륭하게 작동합니다.

갱신 처리는 로컬 메모리 안에서 깔끔하게 완결됩니다.

클라이언트는 공식 클러스터가 정상 응답한 것으로 간주하여 장시간 세션을 안전하게 유지합니다.

---

## 2. 상류 모델의 조합 및 라우팅 전략

통신 경로를 확보한 후에는 백엔드에 배치할 모델의 조합을 정교하게 다듬어야 합니다.

Google AI Studio는 Gemini 2.5 Flash를 대상으로 넉넉한 일일 무료 할당량을 제공합니다.

단일 호출당 100만 토큰의 컨텍스트 윈도를 네이티브로 지원합니다.

수백 개의 모듈을 가진 대규모 코드베이스의 AST 구문 트리를 분할 없이 단번에 읽어들이는 경험은 대단히 쾌적합니다.

무료 티어는 훌륭하지만 제약 조건 또한 명확합니다.

가장 강력한 제약은 분당 최대 15회로 제한된 15 RPM 호출 한도입니다.

에이전트가 복수 파일을 훑으며 연속 수정을 진행하면 불과 몇 초 만에 한도에 도달하여 429 오류가 발생합니다.

이때 풀 안에 안정적인 대체 카드가 없다면 작업이 그 자리에서 멈추고 맙니다.

보조 모델이 갖추어야 할 기준은 명확합니다:
- DeepSeek V3 아키텍처 기반의 공식 deepseek-chat 엔드포인트를 사용할 것;
- 캐시 적중 시 100만 토큰당 0.014달러, 미적중 시에도 0.14달러로 매우 저렴할 것;
- 우수한 코드 생성 능력과 정형화된 JSON 파싱 능력을 보장할 것.

두 모델을 LiteLLM 라우터 풀로 엮어 평상시에는 무료 채널을 태우고 급격한 트래픽에는 저비용 API로 우회시키는 이원화 구조를 구축합니다.

---

## 3. 로컬 config.yaml 작성법과 주의사항

로컬 머신에 격리된 Python 가상환경을 만들고 수정 패치가 포함된 LiteLLM을 설치합니다:

```bash
python3 -m venv .venv-litellm
source .venv-litellm/bin/activate
pip install --upgrade "litellm[proxy]>=1.61.0"
```

이어서 작업 디렉터리에 config.yaml 설정 파일을 생성합니다:

```yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY
      rpm: 15
    model_info:
      supports_function_calling: true

  - model_name: claude-3-7-sonnet
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: os.environ/DEEPSEEK_API_KEY
      api_base: https://api.deepseek.com/v1
    model_info:
      supports_function_calling: true

router_settings:
  routing_strategy: latency-based-routing
  fallbacks:
    - claude-3-5-sonnet-20241022: ["claude-3-7-sonnet"]
  num_retries: 2
  timeout: 45

litellm_settings:
  drop_params: true
  set_verbose: false
```

이 설정에서 절대 놓쳐서는 안 되는 급소가 두 군데 있습니다.

첫째는 drop_params를 반드시 true로 설정해야 한다는 점입니다.

만약 drop_params를 켜지 않으면 어떻게 될까요?

Claude Code는 thinking 심도나 실험적 베타 플래그 같은 Anthropic 전용 필드를 요청 본문에 다수 포함합니다.

호환 처리가 되지 않은 업스트림 모델은 알 수 없는 파라미터를 읽자마자 HTTP 400 검증 오류를 반환합니다.

drop_params를 활성화하면 게이트웨이가 전달 전에 미지원 파라미터를 자동으로 걸러냅니다.

둘째는 supports_function_calling을 명시적으로 true로 선언해야 합니다.

필수 설정입니다.

이 행이 빠지면 전체 동작이 마비됩니다.

파일 치환이나 정규식 검색은 모두 Function Calling 프로토콜에 의존합니다.

이 선언이 누락되면 모델이 반환한 도구 구조가 일반 텍스트로 취급되어 에이전트가 한 발자국도 움직이지 못합니다.

도구 호출이 먹통인 에이전트가 제 역할을 할 수 있을까요?

---

## 4. 로컬 구동 및 실전 연동 검증

설정을 마친 후 메인 터미널에서 실제 발급받은 API 키를 내보내고 프록시를 구동합니다:

```bash
export GEMINI_API_KEY="사용자의GoogleAPI키"
export DEEPSEEK_API_KEY="사용자의DeepSeekAPI키"

litellm --config config.yaml --host 127.0.0.1 --port 4000
```

게이트웨이 콘솔에 정상적인 구동 로그가 출력됩니다:

```text
INFO:     Started server process 84210
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:4000
```

두 번째 터미널 창을 열고 대상 프로젝트 폴더로 이동한 뒤 리다이렉션 환경 변수를 주입합니다:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_API_KEY="sk-mock-litellm-local-auth"

claude
```

터미널에 Claude Code 특유의 커맨드라인 인터페이스가 활성화됩니다.

정상 구동 확인.

구체적인 작업을 지시해 보았습니다: 프로젝트 내 모든 비동기 함수를 스캔하여 미처리 Promise를 try catch로 감싸고 관련 단위 테스트를 갱신하라.

LiteLLM 콘솔 창에 표준적인 접근 로그가 실시간으로 흘러갑니다:

```text
INFO:     127.0.0.1:54320 - "POST /v1/messages HTTP/1.1" 200 OK
INFO:     127.0.0.1:54322 - "POST /v1/messages HTTP/1.1" 200 OK
```

터미널 커서가 바쁘게 깜빡이며 1회차 파일 수정과 테스트 통과가 순조롭게 마무리되었습니다.

겉보기에는 완벽해 보입니다.

하지만 타사 모델로 우회한 상태에서 실제 코드베이스의 도구 파라미터가 과연 끝까지 정확하게 유지될 수 있을까요?

현실은 생각만큼 단순하지 않습니다.

---

## 5. 도구 호출 오류 트러블슈팅과 장애 전환 실측

많은 기술 튜토리얼이 이종 모델 프록시 구성을 마치 무결점인 양 묘사합니다.

그러나 부하가 높은 실전 코딩 환경에서 도구 호출 영역은 결코 평온하지 않았습니다.

16회에 걸친 복잡한 연속 리팩터링 과정에서 세 가지 전형적인 장애를 겪었습니다.

첫 번째 문제는 깊은 디렉터리 경로를 수정하는 edit_file 도구 호출에서 발생했습니다.

괴상한 동작이었습니다.

Gemini가 간헐적으로 스키마 내의 old_string 필드를 old_text라는 이름으로 출력했습니다.

Claude Code 클라이언트가 역직렬화 과정에서 빈 값을 읽어들이며 Invalid tool input 에러를 뿜고 멈췄습니다.

이것이 모델의 단순 환각이었을까요?

5분간의 추적 끝에 응답 스키마 드리프트가 원인임을 밝혀냈습니다.

drop_params는 모델로 향하는 인자만 걸러낼 뿐 모델이 뱉어내는 tool schema의 정규화까지는 책임지지 않습니다.

해결책으로 설정 파일에 가벼운 post_call_hook을 추가하여 클라이언트로 응답을 넘기기 전에 old_text를 old_string으로 강제 치환했습니다.

정상 복구되었습니다.

두 번째 장애는 DeepSeek 장애 우회 단계에서 터졌습니다.

run_shell_command의 timeout 인자를 모델이 문자열 "30"으로 출력했습니다.

로컬 실행 프로세스가 int 형변환을 시도하다가 타입 에러로 예외를 발생시켰습니다.

세 번째는 Gemini가 15 RPM 한도에 걸려 429 오류를 뿜었을 때 발생했습니다.

LiteLLM은 약 65밀리초 만에 deepseek-chat으로 트래픽을 우회시켰습니다.

하지만 DeepSeek는 첫 번째 시도에서 tool_use 블록을 통째로 누락하고 자연어 설명문만 반환했습니다.

재시도 장치가 없었다면 그 자리에서 세션이 끊어졌을 것입니다.

다행히 router_settings에 num_retries: 2를 선언해 두었기 때문에 두 번째 시도에서 무사히 도구 호출 블록을 회수할 수 있었습니다.

실전 환경에서 도구 호출 성공률이 90퍼센트에 도달한다면 충분히 합격점으로 볼 수 있습니다.

설정만 하면 무조건 완벽하게 돌아간다는 생각은 착각에 불과합니다.

테스트 환경：M2 MacBook Air 16GB 메모리, 500Mbps 광랜 환경.
- Gemini 2.5 Flash 첫 토큰 응답 속도는 320에서 480밀리초, 생성 속도는 초당 75에서 90토큰.
- DeepSeek V3 첫 토큰 응답 속도는 650에서 850밀리초, 생성 속도는 초당 40에서 55토큰.

수십 밀리초 수준의 라우팅 전환 지연이 실제 코딩 몰입감에 지장을 줄까요?

터미널 커서의 움직임을 지켜보는 동안 전환으로 인한 체감 지연은 전혀 느껴지지 않았습니다.

극도로 매끄럽습니다.

다만 백엔드의 재시도 전략이 사전에 철저하게 정의되어 있어야 한다는 점이 전제 조건입니다.

---

## 6. 현실적인 운영 경계선과 기술적 타협점

이 프록시 파이프라인을 완성한 시점에서 명확한 운영 기준을 밝혀둘 필요가 있습니다.

회사의 기밀 비즈니스 코드에는 로컬 LiteLLM을 절대 적용하지 않을 것입니다.

127.0.0.1 바인딩이 기술적으로 불안정해서가 아닙니다.

외부 네트워크에서 침투할 가능성은 전무합니다.

문제는 기업의 데이터 보안 규정에 있습니다.

절대로 위반해서는 안 되는 선입니다.

정상적인 개발 조직에서는 소스 코드가 외부 API를 거칠 때 데이터 보안 팀의 공식 승인을 거쳐야 합니다. 조각내어 보낸다고 해서 규정 위반이 면책되지 않습니다.

그런 환경이라면 망설임 없이 회사의 공식 엔터프라이즈 플랜을 신청해야 합니다.

하지만 개인 오픈소스 리포지토리, 개인 연구 과제, 직접 만드는 토이 프로젝트라면 이 파이프라인은 압도적인 만족감을 줍니다.

Gemini 2.5 Flash 무료 티어로 일상적인 코드 리팩터링을 소화하고 한도를 넘겨 DeepSeek로 넘어가더라도 하루에 발생하는 비용은 몇십 원에 불과합니다.

유일하게 이 설정을 말리고 싶은 시나리오는 대규모 무인 자동화 CI 파이프라인입니다.

15 RPM의 벽은 고빈도 병렬 처리를 버티지 못하며 429 오류의 연쇄 발생으로 배포 속도가 크게 저하됩니다.

이런 자동화 영역에서 구독료 몇 푼을 아끼려 드는 것은 명백한 고생길입니다.

수십 밀리초 수준의 장애 전환을 위해 로컬에서 훅 설정을 깎아내는 수고가 과연 가치 있을까요?

매달 수만 원의 구독료를 아끼면서 자유로운 개발 흐름을 유지할 수 있다면 몇 줄의 훅 코드를 작성하는 수고는 충분히 남는 장사입니다.

엄청난 비용 절감.

정말 쏠쏠합니다.

이 프록시 환경은 현재 제 개발 머신에서 일주일 넘게 안정적으로 구동 중입니다.

시간이 나는 대로 old_text 필드명 드리프트를 정리하는 패치를 LiteLLM 공식 리포지토리에 PR로 올릴 예정입니다.

업스트림 메인에 반영되는 것이 가장 깔끔합니다.

로컬에서 pip upgrade를 실행할 때마다 훅 패치를 수동으로 다시 입히는 작업은 꽤나 성가시기 때문입니다.

정말 귀찮은 일입니다.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/litellm-claude-code-pkce-free-gemini-deepseek-routing-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


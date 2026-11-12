---
title: "기존 프롬프트를 Claude Fable 5.2로 옮기기 전에: 400 Prefix Check와 동작 변화 점검"
description: "Anthropic 공식 Prompting Claude Fable 5.2 문서를 바탕으로 마이그레이션 함정을 정리한다. 전 계정 강제 적용된 thinking 블록 prefix check 400 오류, medium 이상에서 thinking 억제 불가, Memory 누적 규칙으로 인한 승인 루프, reasoning_extraction 거부 분류까지 다룬다."
pubDate: 2026-11-12
author: "小可"
category: "아키텍처와 마이그레이션"
tags: ["Claude Fable 5.2", "Claude Mythos 5.2", "Prompt Engineering", "Agent Harness", "Anthropic"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


`Claude Fable 5.1`이나 `Opus 5.5`용으로 짜 둔 옛 시스템 프롬프트를 그대로 `Claude Fable 5.2`에 물려 돌리면 보통 두 군데서 먼저 걸린다.

저장된 대화를 이어서 부르자마자 API가 `400` 오류를 뱉거나, 에이전트가 작업 도중에 멈춰 서서 임시 파일 삭제 하나에도 사람 승인을 요구한다.

Anthropic이 공식 문서 `prompting-claude-fable-5-2`를 공개했다. 읽어보면 이번 세대에서 가장 크게 바뀐 지점은 기존 하네스가 당연하게 여기던 전제 조건들이다. 이 글은 해당 공식 문서만 근거로 정리했으며, 운영 환경 장기 세션 전체 회귀 테스트를 돌린 것은 아니므로 자체 토큰 비용이나 지연 시간 그래프는 적지 않는다.

## System 두 줄만 고쳐도 400: 전 계정 강제 Prefix Check

예전에는 모델 버전을 올릴 때 `system` 프롬프트 두 줄을 살짝 고친 뒤 DB에 저장된 기존 대화를 그대로 이어 붙이는 일이 흔했다.

`Claude Fable 5.2`에서는 그 방식이 곧바로 막힌다. 모든 계정에 thinking 블록 `prefix check`가 강제되기 때문이다. 저장해 둔 thinking 블록을 함께 보내면서 `system`, `tools`, 또는 이전 메시지 중 하나라도 바뀐 상태면 Messages API가 요청을 거절하고 `400` 오류를 반환한다.

새 지시는 새 대화에 넣거나 가장 최근 사용자 턴에 얹어야 한다. 대화 중간에 맥락을 꼭 바꿔야 한다면 어떻게 해야 할까.

베타 우회 옵션으로 `thinking-binding-controls-2026-08-01` 헤더를 붙이고 `thinking.block_binding.prefix_mismatch_behavior`를 `"drop_block"`으로 설정할 수 있다. 다만 대가가 따른다. Anthropic 테스트에 따르면 턴마다 이전 thinking 블록을 버리면 모델이 매번 다시 추론하느라 출력 토큰이 늘어난다. 명시적인 compaction이나 context editing을 제외하면 메시지 이력을 추가 전용으로 유지하는 쪽이 비용 면에서 가장 유리하다.

## "생각하지 말고 바로 답해"가 안 먹힌다: medium 이상은 무조건 먼저 생각

`Fable 5.1`에서 넘어온 프롬프트에는 단순 질의 지연을 줄이려고 `"answer directly without deliberating"` 같은 문구가 들어 있는 경우가 많다.

`Fable 5.2`에서는 이 지시가 통하지 않는다. 문서에 따르면 Anthropic 자체 테스트에서 `effort`가 `medium` 이상일 때는 가벼운 인사에도 먼저 thinking에 들어가며, 바로 답하라는 프롬프트를 넣어도 멈추지 않았다. 게다가 `Fable 5.2`의 기본 `effort`는 `high`다.

단순 턴 비용을 줄이려면 `effort` 단계를 직접 낮춰야 한다. 문서에서는 `low` 설정에서도 코딩 평가 점수가 `Fable 5.1` 고단계에 근접했다고 적고 있지만, 구체적인 벤치마크명과 표본 수는 풀어놓지 않았으므로 SLA가 아니라 추세 정도로만 참고하는 편이 안전하다. 대화 중간에 최상위 `effort`를 바꾸면 프롬프트 캐시가 깨지므로, 캐시를 살린 채 턴마다 강도를 조절하려면 `mid-conversation-output-config-2026-07-01` 베타 헤더로 메시지 단위 `effort` 덮어쓰기를 써야 한다.

또 `low`와 `medium`에서는 조기 종료 현상도 잦다. 평범한 타임아웃이나 빈 도구 결과를 만나면 남은 작업 목록만 남기고 턴을 끝내 버리기 쉬우므로, 시스템 프롬프트에 끝까지 이어서 처리하라는 지침을 보강해야 한다. 무인 자동화 작업이 반쯤 돌다 멈추면 곤란하지 않은가.

## Memory에 쌓인 옛 규칙이 승인 루프를 만든다

이번 모델은 지시사항을 훨씬 문자 그대로 따른다.

`Fable 5.1`이나 `Opus 5.5`를 쓰던 팀은 Memory나 규칙 파일에 `"never retry"`, `"never delete files"`, `"stop and review at the first error"` 같은 안전 문구를 겹겹이 쌓아 두곤 했다. 예전 모델은 상황을 봐서 유연하게 넘어가기도 했지만 `Fable 5.2`는 `"never"`가 들어간 지시를 절대 금지선으로 받아들인다.

멀티 에이전트 세션을 몇 번 돌리다 보면 오래된 주의사항이 누적되어 일상적인 파일 작업에서도 계속 사람 승인을 요청하게 된다.

프롬프트 끝에 "더 과감하게 행동하라"고 덧붙여 봐야 소용없다. 낡은 메모리 파일을 정리해 불필요한 `"never retry"`를 지우고, `"never delete files"`는 `"ask for approval before deleting files"`처럼 파괴적 동작만 콕 집어 제한하는 문장으로 고쳐야 한다.

서브에이전트 호출도 비슷하다. `xhigh`와 `max` 단계에서는 자신이 짠 코드를 검토하려고 스스로 서브에이전트를 띄우는 모습까지 보였다. 비용이 불어난다면 도구 설명에 서브에이전트 토큰 비용과 위임 기준을 적어 두는 편이 좋다. 문서에는 `high` 단계 비교에서 호출 횟수가 대략 절반으로 줄었다고 나와 있으나 작업 분포와 건수는 공개되지 않았으니 자체 파이프라인으로 검증할 일이다. 진행 상황 보고 템플릿을 넣으면 출력 토큰이 약 `20%` 늘어난다는 점도 감안해, 사람이 화면을 보지 않는 백그라운드 배치에서는 빼 두는 편이 낫다.

## 신설된 reasoning_extraction 거부: 본문에 추론 과정 출력 요구 금지

예전 Skill이나 도구 설명에 "내부 추론 단계를 응답 본문에 그대로 적어라"고 넣어 둔 경우도 손봐야 한다.

`Claude Fable 5.2`에서는 `stop_reason: "refusal"`을 맞을 수 있다.

거부 응답은 `stop_details.category`에 `cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`, `null` 여섯 가지 분류를 담는다. 본문에 내부 추론을 옮겨 적으라는 지시는 `reasoning_extraction`에 걸린다. 해당 지시를 지우고 `display: "summarized"` 설정으로 thinking 블록을 읽는 정석 경로로 바꿔야 한다.

정상적인 보안 연구나 생명과학 업무가 `cyber`·`bio` 분류에 막힌다면, Anthropic의 검증 프로그램을 통과한 조직 전용 `Claude Mythos 5.2`로 우회하는 경로가 안내되어 있다.

운영 트래픽을 넘기기 전에 아래 네 가지는 먼저 확인하자.

1. 새 세션에서 `medium`, `high`, `xhigh` 단계 `effort` 평가를 다시 돌려 보고 `Fable 5.1` 값을 그대로 옮기지 않는다.
2. 기존 시스템 프롬프트, Skills, Memory를 훑어 `"answer directly without deliberating"`, 낡은 `"never"` 금지문, 본문 추론 출력 지시를 정리한다.
3. 장기 세션 복원 경로에서 `system`, `tools`, 이전 메시지가 추가 전용으로 유지되어 `prefix check` `400` 오류가 나지 않는지 점검한다.
4. 가장 긴 에이전트 파이프라인을 시범 구동해 조기 종료, 과도한 서브에이전트 생성, 불필요한 승인 대기 중 어디에 걸리는지 확인한다. 옛 규칙의 지뢰밭을 안 치운 채 전체 트래픽을 바로 넘길 수는 없지 않은가.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


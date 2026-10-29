---
title: "LLM 가중치 안에서 CPU가 돌고, CPU 안에서 DOOM이 돌아간다: Qwenputer의 정체는 무엇인가?"
description: "32비트 RISC-V CPU를 언어 모델의 신경망 가중치 내부에 직접 컴파일. 외부 파이썬이나 샌드박스 런타임 없이 오직 순전파(Forward Pass)만으로 기계어를 실행하며, 공개 당일 고전 명작 둠(Doom) 1탄의 레이캐스터까지 구동 성공! Gated DeltaNet을 RAM으로 활용하는 원리, 검증된 로컬 실습 가이드 및 에이전트 확정적 가드레일의 혁신적 가치를 완벽 분석합니다."
pubDate: 2026-10-29
author: "샤오커 (FreeAI Dog 하드웨어 및 엣지 컴퓨팅 팀)"
category: "实战进阶"
tags: ["Qwenputer", "Qwen-3.5", "RISC-V", "LLM 아키텍처", "Doom", "Gated DeltaNet", "에이전트 가드레일", "결정론적 컴퓨팅"]
featured: true
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/qwenputer-riscv-cpu-inside-qwen-weights-doom-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# LLM 가중치 안에서 CPU가 돌고, CPU 안에서 DOOM이 돌아간다: Qwenputer의 정체는 무엇인가?

최근 오픈소스 AI와 시스템 아키텍처 생태계에서 컴퓨터 공학사에 남을 만한 경이로운 실험이 공개되었습니다:

> **오픈소스 대형 언어 모델의 가중치 내부에 완전한 32비트 RISC-V CPU(RV32IM + Zba)가 직접 컴파일되어 삽입되었습니다!**

더욱 놀라운 점은, 프로젝트 공개 당일 커뮤니티 개발자가 이 모델 내장 CPU 위에서 **전설적인 게임 《둠》(DOOM E1M1)의 1인칭 레이캐스팅 엔진을 구동하는 데 성공**했다는 사실입니다.

브라우저에서 `W`, `A`, `S`, `D` 키를 누르면, 모델이 1회 순전파(Forward Pass) 동안 17,540개의 RISC-V 어셈블리 명령어를 내부에서 실행한 뒤 터미널에 ASCII 그래픽으로 다음 프레임을 찍어냅니다.

외부 파이썬 인터프리터도 없고, 샌드박스 컨테이너도 없습니다. 모든 연산과 메모리 입출력은 신경망 행렬곱과 상태 갱신 안에서만 일어납니다.

이 기이하고도 혁신적인 **Qwenputer** 프로젝트의 내부 메커니즘, 재현 가능한 실습 방법, 그리고 AI 보안 연구자들이 주목하는 **확정적 에이전트 가드레일(Deterministic Guardrails)** 의 가치를 상세히 짚어봅니다.

---

## 1. 핵심 원리: 신경망 가중치에 컴퓨터를 어떻게 집어넣었는가?

전통적인 컴퓨터 아키텍처의 핵심 요소는 **RAM**, **레지스터**, **프로그램 카운터(PC)**, **ALU(연산 장치)** 입니다.

기존 트랜스포머의 Softmax 어텐션은 확률적 분산을 일으키므로 RAM으로 쓸 수 없습니다. 그러나 Qwen 아키텍처에는 선형 순환 어텐션인 **Gated DeltaNet** 레이어가 포함되어 있습니다.

### Gated DeltaNet 상태 행렬을 RAM으로 전환
DeltaNet의 헤드는 토큰마다 갱신되는 내부 상태 행렬을 가지고 있습니다. Qwenputer는 이를 완벽히 역이용했습니다:
- 감쇠 게이트를 최대로 열어($\alpha = 1.0$) 기억이 절대 지워지지 않게 고정;
- 쓰기 강도를 완전 덮어쓰기($\beta = 1.0$)로 설정;
- Key 벡터를 **엄격한 원-핫(One-hot) 벡터**로 만들어 특정 행(Row)만 선택 갱신.

이는 물리적인 RAM 쓰기 동작과 정확히 일치합니다. 하위 비트로 헤드를 고르고 상위 비트로 행을 지정하여, 모델의 내부 상태를 **32KB 가상 RAM**으로 탈바꿈시켰습니다.

### 컴파일로 생성된 전면 11개 트랜스포머 레이어
이 CPU 레이어는 **역전파 학습(Training)을 단 한 번도 거치지 않았습니다**.
작성자가 제작한 전용 컴파일러가 RV32IM 하드웨어 게이트 로직을 정밀한 bfloat16 수치로 환산하여, Qwen 본체(32개 레이어) 앞에 **11개 레이어로 직접 조립**해 넣었습니다.

### 토큰 생성 = CPU 클록 사이클
Qwenputer에서 **토큰이 생성되는 과정 자체가 CPU의 클록 사이클**입니다. 1개 명령어마다 레지스터 저장, 분기 점프 등의 트레이스 토큰이 내부에서 생성되며, 연산이 끝나 `putchar`가 호출되면 본체 레이어가 최종 결과를 읽어 답변을 출력합니다. 23만 건의 출력 검증에서 오차율 0%를 기록했습니다.

---

## 2. 덧셈 한 번에 수백만 번 곱셈을 하는데, 대체 왜 필요한가?

단순 계산 성능으로만 보면 "세계에서 가장 비효율적인 CPU 시뮬레이터"처럼 보일 수 있습니다. 하지만 보안 및 엔지니어링 전문가들이 열광하는 이유는 **에이전트 확정적 가드레일(Deterministic Guardrails)** 에 있습니다.

### 기존 에이전트의 치명적 한계
1. 외부 Bash/Python 샌드박스 호출에 따른 탈옥 및 시스템 오염 위험;
2. 프롬프트 인젝션(Prompt Injection)에 의한 규칙 무력화 (모델은 확률적 샘플링을 거스를 수 없음).

### Qwenputer가 제시하는 패러다임: 물리적으로 우회 불가능한 내부 회로
만약 보안 정책, 권한 검증, SQL 문법 파서를 이 11개 CPU 레이어로 컴파일하여 가중치에 직접 새겨넣는다면:
- **프롬프트 공격 면역**: 모든 토큰 생성은 물리적으로 이 11개 회로 레이어를 통과해야만 하므로, 어떤 탈옥 프롬프트도 순전파 연산을 건너뛸 수 없습니다.
- **100% 확정적 보장**: 권한 체크와 수치 한계선이 확률이 아닌 하드웨어 논리 회로로 동작하여 오차와 환각이 사라집니다.
- **완전 격리 환경 구동**: 인터넷이 차단된 엣지 기기나 극도로 민감한 사설망에서도 독립적으로 안전하게 구동됩니다.

---

## 3. 재현 가능한 로컬 실습 가이드

### 환경 요구사항
- **운영체제**: Linux (Ubuntu 22.04+) 또는 macOS (Apple Silicon)
- **GPU**: NVIDIA GPU (16GB+ VRAM 권장, CPU 전용 모드로도 구동 가능)
- **패키지 관리자**: `uv`

### 1단계: 저장소 복제 및 환경 설정
```bash
git clone https://github.com/divagr18/qwenputer.git
cd qwenputer/experimental/riscv

curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.cargo/env

uv sync --extra hf
```

### 2단계: 컴파일된 모델 가중치 다운로드
```bash
pip install -U huggingface_hub
hf download divagr1925/Qwen3.5-4B-Qwenputer --local-dir qwenputer-rv
```

### 3단계: 가중치 내부에서 RISC-V 곱셈 실행
내장된 RISC-V 펌웨어(`calc-bridge.elf`)를 사용하여 파이썬 호스트 개입 없이 큰 수 곱셈을 실행합니다:
```bash
uv run python -m chipformer.rv.chat qwenputer-rv \
    --firmware tests/data/rv/calc-bridge.elf \
    --ask "Use calc: what is 123456789 * 987654321?"
```
실행 결과, 외부 프로세스 생성 없이 모델 가중치 내부에서 연산되어 정확히 `121932631112635269`가 출력됩니다.

---

## 4. 모델 가중치 안에서 DOOM 플레이하기

```bash
git clone https://github.com/mapestree/qwenputer-doom.git
cd qwenputer-doom

uv sync
uv run python play.py --model-dir ../experimental/riscv/qwenputer-rv
```
브라우저 인터페이스를 열고 조작하면 모델 내부의 CPU가 턴당 17,540개 명령어를 처리하며 3D 레이캐스팅 미로 화면을 ASCII 문자로 렌더링합니다.

---

## 5. 맺음말

Qwenputer는 단순한 해킹 실험을 넘어, 생성형 언어 모델과 결정론적 계산 하드웨어가 하나의 신경망 가중치 속에서 융합될 수 있음을 증명한 중대한 이정표입니다.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/qwenputer-riscv-cpu-inside-qwen-weights-doom-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


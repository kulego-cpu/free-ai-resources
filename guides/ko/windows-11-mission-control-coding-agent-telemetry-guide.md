---
title: "Windows 11 리버스 엔지니어링 실증: MissionControlAggregator의 Coding Agent 프로세스 감시와 차단 실전"
description: "Windows 11 Build 26100의 MissionControlAggregator.dll 리버스 엔지니어링 분석. Claude Code, Cursor, Codex 등 AI 코딩 에이전트를 감지하는 하드코딩 패턴과 ETW 진단 경로를 해부하고, 레지스트리와 그룹 정책으로 텔레메트리를 차단하는 보안 SOP를 정리했습니다."
pubDate: 2026-11-17
author: "샤오커 [小可]"
category: "아키텍처"
tags: ["Windows11", "CodingAgent", "리버스엔지니어링", "개인정보보호", "텔레메트리", "보안"]
featured: true
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/windows-11-mission-control-coding-agent-telemetry-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# Windows 11 리버스 엔지니어링 실증: MissionControlAggregator의 Coding Agent 프로세스 감시와 차단 실전

며칠 전 밤, 로컬 환경에서 자동 리팩터링 스크립트를 실행하고 있었습니다.

작업이 진행되던 도중 본체 쿨링팬이 갑작스럽게 최대 회전수로 돌기 시작했습니다.

의아한 마음에 작업 관리자를 열었습니다.

CPU 사용률은 8% 안팎으로 낮았습니다.

그런데 디스크 쓰기 속도가 급증해 초당 20MB에 육박하고 있었습니다.

섣부른 추측은 피하기로 했습니다.

Process Monitor를 열고 이벤트 캡처를 시작했습니다.

필터 조건으로 프로세스 이름에 svchost.exe 및 DiagTrack.exe를 지정했습니다.

수집된 로그를 보니 시스템 진단 서비스가 임시 추적 파일을 비정상적인 빈도로 쓰고 있었습니다.

관리자 터미널을 열고 다음 명령을 입력했습니다:

```cmd
sc stop DiagTrack
```

명령 성공 메시지가 출력된 직후, 디스크 쓰기 속도는 0MB/s로 떨어졌습니다.

2분 후, 쿨링팬 회전수는 3800 RPM에서 1600 RPM 저소음 구간으로 내려앉았습니다.

서비스를 다시 켜자 쓰기 작업이 즉각 재발했습니다.

원인과 결과가 정확히 맞물렸습니다.

진단 서비스는 야심한 밤에 무엇을 그토록 기록하고 있었을까요?

어제 Hacker News를 둘러보다가 독립 보안 연구원 쉬성[Xu Sheng] 님이 작성한 리버스 엔지니어링 감사 글을 보았습니다.

해당 글은 500개 이상의 추천을 받으며 주목받고 있었습니다.

분석 원문 주소는 다음과 같습니다: https://xusheng.dev/posts/windows-coding-agent-telemetry/

해당 링크에서 세부 내용을 직접 확인하실 수 있습니다.

커뮤니티 토론 스레드는 여기에 있습니다: https://news.ycombinator.com/item?id=50524736

양쪽의 문자열 정보는 완벽히 일치했습니다.

작성자는 최신 Windows 11 프리뷰 이미지 안에서 개발자 환경과 AI 코딩 에이전트를 식별하는 프로세스 스캔 패턴을 발견했습니다.

분석 대상 샘플은 Windows 11 Build 26100 및 26200 이미지였습니다.

웹상의 글만 맹신할 수는 없었습니다.

로컬 시스템 디렉터리에서 동일 파일을 추출해 샌드박스 가상머신에서 직접 검증했습니다.

바이너리 안에 정말로 하드코딩된 목록이 존재할까요?

Sysinternals의 strings 도구를 사용해 읽기 전용 데이터 영역을 확인했습니다:

```cmd
strings -el %SystemRoot%\System32\MissionControlAggregator.dll | findstr /i "claude codex cursor copilot"
```

화면에 평문 문자열 목록이 즉시 출력되었습니다.

보고서 내용과 완벽히 들어맞았습니다.

---

## 1. 바이너리 포렌식: 시스템 DLL 속 하드코딩 명단

출발점은 Windows 11 시스템 디렉터리 깊은 곳에 있습니다.

문제의 파일은 C:\Windows\System32\MissionControlAggregator.dll 입니다.

외부 설명에는 단순 작업 제어 및 시스템 지표 집계 모듈이라고 적혀 있습니다.

이 DLL을 Binary Ninja에 불러와 읽기 전용 데이터 영역으로 이동했습니다.

판별 문자열은 난독화되지 않은 평문 와이드 문자였습니다.

언패킹 과정이 필요 없었습니다.

동적 난독화도 전혀 적용되지 않았습니다.

<img src="https://img.freeaidog.com/images/blog/windows-11-coding-agent-telemetry-binary-ninja.webp" alt="Binary Ninja 리버스 엔지니어링을 통해 확인된 MissionControlAggregator.dll 내부 Coding Agent 프로세스 목록" />

디스어셈블리 창을 열자 명단이 두 개의 독립된 슬롯으로 나뉘어 있었습니다.

첫 번째 슬롯은 CLI 기반 AI 에이전트입니다.

바이너리는 네 가지 실행 파일 이름을 직접 비교합니다:
- copilot.exe
- claude.exe
- codex.exe
- agy.exe

개발자가 WSL 환경을 사용하는 경우는 어떻게 될까요?

비교 로직이 exe 확장자를 자동으로 제거합니다.

WSL 내부 Linux 환경에서 실행되는 네이티브 실행 파일을 정확히 집어냅니다.

두 번째 슬롯은 데스크톱 에이전트와 통합 개발 환경으로, 다섯 가지 대상을 포함합니다:
- github.exe
- claude.exe
- codex.exe
- cursor.exe
- antigravity.exe

최신 AI 도구 외에도 전통적인 개발 도구들이 촘촘하게 포함되어 있었습니다.

gcc.exe가 들어 있습니다.

clang.exe도 매칭 대상입니다.

gdb.exe, javac.exe, dotnet.exe가 모두 포함되었습니다.

python.exe와 node.exe 역시 감시 대상입니다.

Visual Studio의 devenv.exe도 이름을 올렸습니다.

VS Code의 실행 파일 code.exe도 빠짐없이 들어 있습니다.

마이크로소프트는 일부 주요 런타임에 대해 디지털 서명 검증까지 수행합니다.

예를 들어 node.exe가 감지되면 Node.js 공식 서명을 확인합니다.

python.exe가 감지되면 Python 소프트웨어 재단 서명을 대조합니다.

반면 GCC나 Clang 같은 오픈소스 도구는 이름만 일치해도 즉시 기록합니다.

이 스캐너를 제어하는 내부 플래그 이름은 CLICodingAgentUsage 입니다.

내부 기능 식별자 번호는 62580080 입니다.

운영체제 기본 진단 모듈이 개발자가 어떤 AI 코딩 도구를 쓰는지 감시해야 할 이유가 있을까요?

커널 충돌을 분석하는 데 과연 이런 정보가 필수적일까요?

하부 이벤트 구독 구조를 살펴보겠습니다.

---

## 2. 감시 경로 분석: 커널 이벤트에서 ETW 보고까지

CPU를 상시 소모하는 폴링 스레드는 보이지 않았습니다.

Process Monitor와 커널 호출 스택을 보면 철저한 이벤트 기반 아키텍처로 작동합니다.

Aggregator::OnLoad 초기화 함수가 실행될 때 기존 Windows 커널 이벤트 채널 여러 개를 구독합니다.

첫 번째 경로는 네이티브 Windows 프로세스 대상입니다.

커널 이벤트 KernelProcess.AppStateChangeSummary를 구독합니다.

동시에 UI 이벤트 Win32kTraceLogging.AppInteractivitySummary도 구독합니다.

프로세스가 시작, 일시 중지, 포커스 전환, 종료될 때마다 커널이 상태 변경 요약을 전송합니다.

집계 모듈은 버스 옆에서 알림을 받아 이미지 이름을 하드코딩 명단과 대조합니다.

두 번째 경로는 Linux 개발자 대상입니다.

WSL 전용 관리 채널 Microsoft.Windows.Lxss.Manager.Exec 및 ExecCritical을 구독합니다.

서브시스템을 감시하는 커널 프로브입니다.

WSL 터미널에서 claude나 codex를 실행하는 순간, 드라이버가 imageName 필드를 호스트에 보고합니다.

매칭은 즉각적으로 일어납니다.

실행 파일 이름이 조건에 부합하면 std::_Func_impl_no_alloc 형태의 콜백이 실행됩니다.

조건이 맞아떨어졌다고 해서 곧바로 외부 네트워크 연결을 시도하지는 않습니다.

메모리 내부 집계 비트맵에 결과를 기록합니다.

전체 이벤트 처리 구조는 다음과 같습니다:

```text
Aggregator::OnLoad 초기화 구독
├─ WSL 서브시스템 환경
│  └─ Lxss.Manager.Exec 이벤트 → CLI 에이전트 매칭기 [copilot, claude, codex, agy]
├─ 네이티브 Windows 환경
│  └─ KernelProcess.AppStateChangeSummary → CLI 에이전트 매칭기
└─ 데스크톱 UI 상호작용
   └─ Win32k.AppInteractivitySummary → 데스크톱 에이전트 매칭기 [cursor, claude, codex]

Aggregator::OnRelativeTimerExpire 주기적 타이머
└─ MissionControlAggregatorLogging::ReportCohorts<12,4,5>
   └─ Segments.SubCohorts 비트맵으로 압축
      └─ ETW 이벤트로 포장 후 DiagTrack 서비스로 전달
```

데이터가 모인 뒤 내부 타이머가 주기적으로 작동합니다.

ReportCohorts<12,4,5> 함수가 관측 결과를 비트맵으로 압축합니다.

이어서 Segments.SubCohorts라는 이름의 TraceLogging 이벤트를 발행합니다.

여기서 12, 4, 5는 무엇을 의미할까요?

일반 개발 도구 12종, CLI 에이전트 4종, 데스크톱 에이전트 5종에 대응합니다.

시스템은 긴 파일 경로를 보내지 않습니다.

해당 비트를 1로 바꿀 뿐입니다.

예를 들어 지난 몇 시간 동안 Codex가 한 번이라도 실행되었다면 해당 비트가 1이 됩니다.

바이너리 플래그가 디스크에 기록되고, DiagTrack 서비스가 백그라운드에서 취합해 전송합니다.

제가 리버스 엔지니어링으로 확인한 호출 체인은 ReportCohorts 단계까지입니다.

클라우드 단의 세부 분류와 프로파일링 로직은 비공개 인프라 영역이므로 근거 없는 추측은 삼가겠습니다.

---

## 3. 기본 활성화된 텔레메트리: 일상 사용 정보가 새어 나가는 방식

Windows 초기 설정 화면을 떠올려보십시오.

새로 설치하거나 대규모 업데이트를 적용할 때, 대다수 국가 설정에서 기본 선택된 값은 무엇일까요?

기본값은 바로 선택적 진단 데이터 전송입니다.

마이크로소프트 서비스 계약서에는 검색한 웹사이트, 앱 및 기능 사용 방식, 향상된 오류 보고 데이터가 포함된다고 명시되어 있습니다.

앞서 해체한 CLICodingAgentUsage 기능은 이 선택적 진단 데이터 파이프라인에 연결되어 있습니다.

설치 시 필수 진단 데이터만 전송으로 수동 변경하지 않았다면, 출고 상태에서 이미 수집 동의가 완료된 셈입니다.

개인 프로젝트 수준에서는 그저 통계 수치 하나에 불과할 수 있습니다.

하지만 기업의 핵심 소스코드와 기술 스택 선택 정보라면 어떨까요?

핵심 개발 시간대는 어떨까요?

조직 내부의 AI 코딩 도구 도입 비율은 어떨까요?

엄격한 컴플라이언스가 요구되는 환경에서 이러한 무단 수집은 명백한 외부 관측 노출 지점이 됩니다.

코드 유출 방지에 집중하면서도, OS 밑단에서 AI 작업 프로필을 수집하고 있다는 사실은 놓치기 쉽습니다.

---

## 4. 로컬 방어: 3단계로 텔레메트리를 차단하는 SOP

경로를 확인했으니 로컬에서 세 단계로 전송을 차단하면 됩니다.

### 1단계: 설정 화면에서 선택적 진단 데이터 끄기

Windows 설정을 엽니다.

개인 정보 및 보안으로 이동한 뒤 진단 및 피드백을 클릭합니다.

진단 데이터 항목을 확인합니다.

스위치를 선택적 진단 데이터 보내기에서 필수 진단 데이터만 보내기로 변경합니다.

아래 맞춤형 환경 토글도 끕니다.

진단 데이터 삭제를 눌러 디스크에 저장된 로컬 사용 기록을 제거합니다.

그런데 월간 누적 업데이트 이후 UI 설정이 초기화된 경험은 없으신가요?

실제로 그런 위험이 존재합니다.

GUI 설정만으로는 불충분하므로 상위 차단 정책을 적용합니다.

### 2단계: 그룹 정책 및 레지스트리로 강제 차단

Pro, Enterprise, Education 에디션에서는 Win + R을 누르고 gpedit.msc를 입력해 로컬 그룹 정책 편집기를 엽니다.

컴퓨터 구성 > 관리 템플릿 > Windows 구성 요소 > 데이터 수집 및 Preview 빌드로 이동합니다.

우측에서 진단 데이터 허용을 더블클릭합니다.

사용 안 함을 선택하거나, 사용을 선택한 뒤 하단 드롭다운에서 진단 데이터 끄기를 선택합니다.

저장하고 닫습니다.

gpedit.msc가 없는 Home 에디션이거나 스크립트로 일괄 적용하려면 어떻게 해야 할까요?

관리자 권한으로 터미널을 열고 다음 레지스트리 명령 두 줄을 실행합니다:

```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
```

이 두 설정은 시스템 진단 서비스에 대한 최상위 제약 조건입니다.

값을 0으로 설정하면 텔레메트리 수집 수준이 완전히 차단 상태로 고정됩니다.

하부 모듈이 비트맵을 생성하더라도 상위 단계에서 전달이 차단됩니다.

기업 환경에서의 경계 조건도 확인해야 합니다: PC가 Active Directory 도메인에 조인되어 있거나 Intune 관리를 받는 경우, 도메인 정책이 로컬 레지스트리를 덮어쓸까요?

도메인 정책이 우선 적용되므로, 기업 보안 담당자가 GPO나 Intune 프로파일을 통해 일괄 배포해야 합니다.

### 3단계: 백그라운드 진단 서비스 중지 및 도메인 무력화

파이프라인이 막혔으니 백그라운드 전송 서비스도 메모리 낭비일 뿐입니다.

관리자 터미널에서 서비스를 완전히 중지하고 비활성화합니다:

```cmd
sc stop DiagTrack
sc config DiagTrack start= disabled

sc stop dmwappushservice
sc config dmwappushservice start= disabled
```

엄격한 네트워크 격리가 필요한 환경이라면 hosts 파일에서 수집 도메인을 루프백 주소로 연결합니다:

```text
127.0.0.1 v10.events.data.microsoft.com
127.0.0.1 v20.events.data.microsoft.com
127.0.0.1 watson.telemetry.microsoft.com
127.0.0.1 telemetry.microsoft.com
```

여기서도 고려할 기술적 경계가 있습니다: 시스템에서 DNS over HTTPS [DoH] 프로토콜을 사용하는 경우, hosts 파일 설정이 우회될까요?

DoH 환경에서는 로컬 hosts 파일이 우회됩니다.

따라서 엄격한 네트워크 환경이라면 게이트웨이 방화벽에서 해당 도메인에 대한 DNS 차단이나 IP 드롭을 구성하십시오.

---

## 5. 검증 및 후속 점검

설정을 마친 뒤 시스템을 재부팅합니다.

바탕화면에 진입한 뒤 관리자 명령 프롬프트를 열고 검증 3종 명령을 실행합니다:

```cmd
logman query providers | findstr /i "MissionControl"
sc query DiagTrack
sc query dmwappushservice
```

출력 결과를 확인합니다:
MissionControl 관련 추적 세션은 비활성 상태로 나타나며, DiagTrack과 dmwappushservice는 모두 STOPPED 상태이고 시작 유형은 DISABLED로 표시됩니다.

실제 환경에서 확인했을 때 두 서비스가 완전히 중지된 상태로 확인되셨나요?

Process Monitor로 10분간 관찰한 결과, 이전과 같은 임시 추적 파일 쓰기 급증은 발생하지 않았습니다.

쿨링팬 속도도 조용한 상태를 유지합니다.

향후 월간 누적 업데이트가 배포되면 레지스트리 항목이 몰래 복원될 위험은 없을까요?

매월 정기 패치 이후 위 세 가지 명령을 다시 실행해 차단 상태를 점검하시길 권장합니다.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/windows-11-mission-control-coding-agent-telemetry-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


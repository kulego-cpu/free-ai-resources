---
title: "Cursor 마켓플레이스 Claude Code 확장 프로그램 SSH 백도어 침투 사건 기술 분석 및 전 플랫폼 제거 SOP"
description: "'Claude Code 확장 프로그램에 백도어가 심어졌다'는 소식이 개발자 커뮤니티를 흔들고 있습니다. FreeAI Dog 팀이 확인한 기술적 실체: 공격당한 대상은 Anthropic 공식 확장이 아닌 Cursor IDE 전용 마켓플레이스에 등록된 변조 버전(v2.1.313)입니다. 이 악성 확장은 OS 수준에서 지속형 데몬을 등록하므로 에디터 내에서 '삭제' 버튼을 누르는 것만으로는 제거되지 않습니다. macOS, Linux, Windows용 완전 지문 대조 및 제거 가이드를 공개합니다."
pubDate: 2026-10-30
author: "샤오커 (FreeAI Dog 개발팀)"
category: "实战进阶"
tags: ["Cursor", "Claude Code", "공급망보안", "보안가이드", "SSH백도어", "DevSecOps", "사이버보안"]
featured: true
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/cursor-claude-code-extension-backdoor-audit-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


# Cursor 마켓플레이스 Claude Code 확장 프로그램 SSH 백도어 침투 사건 기술 분석 및 전 플랫폼 제거 SOP

지난 24시간 동안 Reddit r/LocalLLaMA와 해외 개발자 보안 커뮤니티에서 "Claude Code 공식 VS Code 확장이 해킹당해 지속형 SSH 백도어가 심어졌다"는 폭로성 소식이 급속도로 퍼지며 큰 파장을 일으켰습니다.

하지만 **SNS에 퍼진 제목은 심각한 오해를 불러일으킬 소지가 있습니다**.

FreeAI Dog 개발팀의 정밀 기술 검증과 Cursor 공식 발표를 종합한 결과, 사건의 기술적 실체는 명확히 정리되었습니다. **백도어가 심어진 대상은 Anthropic이 공식 배포한 VS Code 확장이 아니며, Cursor IDE의 독립 확장 프로그램 마켓플레이스에 업로드된 제3자 변조 버전(Modified Version)입니다.** Cursor 공식 인시던트 공지에 따르면 피해 대상은 해당 특정 버전을 수동 설치한 5,000명 이하의 사용자로 한정됩니다.

그러나 진짜 심각한 문제는 지속성(Persistence)에 있습니다. **해당 악성 확장은 실행 즉시 운영체제 핵심부에 백도어 SSH 키와 자동 실행 데몬을 설치하기 때문에, Cursor 에디터 상에서 단순히 '제거(Uninstall)'를 클릭하는 것만으로는 백도어가 절대 사라지지 않습니다.**

본 가이드에서는 사건의 전말을 객관적으로 복기하고, macOS, Linux, Windows 전 플랫폼에서 즉시 실행 가능한 터미널 전수 조사 및 완전 제거 명령어를 공유합니다.

---

## 1. 사건의 기술적 실체: 누가 위험하고 누가 안전한가?

불필요한 혼란을 방지하기 위해 먼저 공격 경로와 피해 범위를 명확히 규명합니다.

### 1. Anthropic 공식 VS Code 확장은 안전합니다
마이크로소프트 공식 Visual Studio Marketplace에 등록된 Anthropic 정품 Claude Code 확장은 어떠한 침해 흔적도 없습니다. 공식 마켓플레이스는 엄격한 코드 서명 및 퍼블리셔 검증 체계를 운영하고 있으므로 일반 VS Code 사용자들은 안심하셔도 됩니다.

### 2. 사고 원인: Cursor IDE 독립 마켓플레이스의 공급망 오염
Cursor는 VS Code의 오픈소스 코어(Code - OSS)를 기반으로 제작된 포크 에디터이기 때문에, 마이크로소프트의 독점 플러그인 유통 프로토콜을 직접 사용할 수 없어 자체 마켓플레이스(Cursor Marketplace)를 구축해 운영하고 있습니다.

공격자는 바로 이 외부 유통 채널의 검수 공백을 노렸습니다:
- **악성 버전 번호**: `2.1.313` (`cursor.com/marketplace` 배포)
- **영향 범위**: Cursor 마켓에서 해당 변조 버전을 수동 검색해 설치한 개발자 (최대 5,000명 이하)
- **Cursor 공식 입장**: *"a modified Claude Code VS Code extension accessed through our Cursor IDE extension marketplace, impacting at most 5,000 users. This malicious version of the extension was not developed by Anthropic."*

### 3. 왜 에디터에서 '삭제'해도 소용없는가?
일반적인 IDE 플러그인은 전용 디렉터리 내에 격리되지만, 이번 악성 코드는 설치 시 Node.js 포스트 인스톨 훅을 트리거하여 에디터 샌드박스를 즉시 탈출했습니다:
1. **SSH 백도어 등록**: 사용자의 `~/.ssh/authorized_keys`에 공격자의 공개키(`svc-telemetry` 식별자)를 강제 추가.
2. **OS 시스템 데몬 위장 등록**: macOS의 애플 시스템 구성 요소 등으로 위장한 데몬을 생성하여 재부팅 시에도 자동으로 복원.

---

## 2. 전 플랫폼 점검 및 완전 제거 가이드

Cursor를 사용 중인 개발자는 즉시 터미널을 열고 본인의 OS 환경에 맞춰 아래 명령어를 실행하십시오.

### 1. 전 OS 공통 점검: SSH 인가 키(authorized_keys) 검증 【최우선】

macOS, Linux 또는 OpenSSH 서버가 구동 중인 Windows 모두 가장 먼저 공인키 목록을 대조해야 합니다.

#### 단계 A: `svc-telemetry` 키워드 검색
터미널에서 다음 명령을 실행합니다:
```bash
grep -n "svc-telemetry" ~/.ssh/authorized_keys
```
결과가 비어 있다면 기본 주석은 발견되지 않은 것입니다. 그러나 공격자가 주석을 변조했을 가능성에 대비하여 **반드시 단계 B의 공개키 지문(Fingerprint) 대조를 수행해야 합니다**.

#### 단계 B: `ssh-keygen`으로 지문 대조
등록된 모든 공개키의 SHA256 해시 지문을 확인합니다:
```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```
**🚨 확인 대상 악성 지문**:
```text
SHA256:0Z6R+DF64Do2nwQic8a7kkfDLNNk4IiK+2U2Rci4htk
```
목록 중 위의 해시 값이 일치하는 행이 발견되면 백도어가 삽입된 상태입니다. 즉시 다음 명령어로 해당 키를 삭제하십시오:

```bash
# 현재 키 백업
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak.$(date +%F)

# 악성 주석이 포함된 행 삭제
sed -i '' '/svc-telemetry/d' ~/.ssh/authorized_keys 2>/dev/null || sed -i '/svc-telemetry/d' ~/.ssh/authorized_keys
```
*(만약 주석이 지워진 상태라면 텍스트 에디터로 `~/.ssh/authorized_keys`를 열어 해당 SHA256 지문에 일치하는 공개키 행 전체를 수동으로 제거하십시오)*.

---

### 2. macOS 전용 점검 및 제거 가이드

macOS 환경에서는 애플 시스템 데몬으로 위장한 `LaunchAgent` 파일이 생성됩니다.

#### 단계 A: 위장 파일 존재 여부 확인
공격자는 파일명을 `com.apple.sysmond.plist`로 위장했습니다.

> ⚠️ **중요 보안 상식**:
> 애플 정품 시스템 데몬은 읽기 전용 시스템 디렉터리인 `/System/Library/LaunchDaemons/com.apple.sysmond.plist`에 상주합니다. **이 파일은 애플 순정 파일이므로 절대 삭제하지 마십시오!**
> 악성 파일은 오직 **사용자 홈 디렉터리**( `~/Library/LaunchAgents/com.apple.sysmond.plist` )에만 위치합니다.

파일 확인 명령어:
```bash
ls -la ~/Library/LaunchAgents/com.apple.sysmond.plist
ls -la ~/.config/.smt
```

#### 단계 B: 데몬 등록 해제 및 완전 삭제
파일이 발견될 경우 아래 절차를 차례대로 실행하여 무력화합니다:
```bash
# 1. 악성 LaunchAgent 등록 해제 및 중단
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null || launchctl unload ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null

# 2. 악성 plist 설정 파일 삭제
rm -f ~/Library/LaunchAgents/com.apple.sysmond.plist

# 3. 백도어 바이너리 파일 삭제
rm -rf ~/.config/.smt

# 4. 잔존 백그라운드 프로세스 확인
ps aux | grep -i "\.smt" | grep -v grep
```
만약 `.smt` 관련 프로세스가 아직 살아 있다면 `kill -9 <PID>` 명령으로 강제 종료하십시오.

---

### 3. Linux 전용 점검 및 제거 가이드

Linux 배포판에서는 `crontab`의 `@reboot` 기능을 활용해 자동 복원을 시도합니다.

#### 단계 A: 사용자 crontab 조회
```bash
crontab -l
```
출력 결과 중 `~/.config/.smt` 파일이나 아래와 유사한 명령이 등록되어 있는지 확인합니다:
```text
@reboot ~/.config/.smt >/dev/null 2>&1
```

#### 단계 B: crontab 정리 및 파일 제거
```bash
# 1. crontab 백업
crontab -l > /tmp/crontab.bak

# 2. .smt 행을 필터링하여 재등록
crontab -l | grep -v "\.config/\.smt" | crontab -

# 3. 악성 페이로드 파일 삭제
rm -rf ~/.config/.smt

# 4. 구동 중인 프로세스 종료
pkill -f "\.config/\.smt"
```

---

### 4. Windows 전용 점검 및 제거 가이드

Windows 시스템에서는 작업 스케줄러(Task Scheduler)나 시작프로그램 폴더에 등록됩니다.

#### 단계 A: 작업 스케줄러 및 시작프로그램 조회
관리자 권한으로 PowerShell을 실행한 후 다음을 입력합니다:
```powershell
# 1. sysmon 관련 스케줄러 작업 조회
Get-ScheduledTask | Where-Object { $_.TaskName -like "*sysmon*" -or $_.TaskPath -like "*sysmon*" }

# 2. 시작프로그램 폴더 확인
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" | Where-Object { $_.Name -like "*sysmon*" }
```

#### 단계 B: 작업 등록 취소 및 스크립트 삭제
```powershell
# 작업 삭제 (작업명이 SysMon일 경우)
Unregister-ScheduledTask -TaskName "SysMon" -Confirm:$false

# 시작프로그램 스크립트 삭제
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\sysmon.cmd" -Force -ErrorAction SilentlyContinue
```

---

## 3. 침해 확인 시 필수 사후 조치

보안 거버넌스는 항상 "이미 시스템이 장악당했다(Assume Breach)"는 전제하에 대응해야 합니다. 시스템에서 실제 악성 지문이나 `.smt` 파일이 확인되었다면 다음 조치를 반드시 병행해야 합니다:

1. **로컬 SSH 개인키 전면 폐기 및 교체**:
   공격자는 쉘을 획득한 후 통상 `~/.ssh/id_rsa`, `~/.ssh/id_ed25519` 등의 개인키를 탈취합니다. 새로운 키 쌍을 생성하고 GitHub, GitLab, 운영 서버에 등록된 공개키를 즉시 갱신하십시오.
2. **환경 변수 및 API Key 재발급**:
   로컬의 `~/.zshrc`, `~/.bashrc`, 프로젝트 내 `.env` 파일에 저장된 `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, 클라우드(AWS/GCP) 액세스 토큰은 유출된 것으로 간주하고 즉각 재발급받아야 합니다.
3. **Cursor 내 문제 플러그인 완전 제거**:
   Cursor 에디터의 확장 프로그램 탭에서 해당 변조 패키지가 완전히 제거되었는지 재차 확인하십시오.

---

## 4. AI 시대 IDE 공급망 보안 수칙

이번 사건은 단순한 해프닝이 아니라 **급격히 확산되는 AI 코딩 보조 환경을 정밀 타깃팅한 대표적 공급망 공격 모델**입니다.

Cursor, Windsurf, Trae 등 차세대 AI IDE는 강력한 생산성을 제공하지만, 마이크로소프트 독점 스토어와의 분리로 인해 서드파티 마켓플레이스의 취약한 검증 구조가 공격자들의 표적이 되고 있습니다.

### AI 코딩 개발자를 위한 3대 보안 수칙
- **퍼블리셔 신원 확인**: 외부 IDE 마켓플레이스에서 확장을 설치하기 전, 제작자 도메인과 GitHub 공식 저장소를 반드시 교차 검증하십시오.
- **개발 환경 격리**: 신뢰성이 완전히 입증되지 않은 코딩 에이전트나 확장을 사용할 때는 Docker 컨테이너나 Dev Containers 등 격리된 환경을 활용하십시오.
- **정기적인 로컬 자산 점검**: `~/.ssh/authorized_keys`, `LaunchAgents`, `crontab` 등 자동 실행 지점을 정기 점검 목록에 포함해 유령 인가 키를 사전에 차단하십시오.

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/cursor-claude-code-extension-backdoor-audit-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


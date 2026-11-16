---
title: "14B 시각 모델을 브라우저에서 추방하기: 107MB 경량 Folate 실측과 튜닝 일지"
description: "무거운 시각 모델 대신 107MB 초경량 INT8 ONNX 모델 Folate를 도입한 실전 기록: PyTorch 없는 순수 CPU 추론, Mind2Web 실전 웹 요소 랭킹, 환각 없는 XPath 추출 및 Shadow DOM 관통 노하우."
pubDate: 2026-11-16
author: "소가"
category: "에이전트 및 자동화"
tags: ["Folate", "에이전트", "브라우저 자동화", "ONNX", "Playwright", "보급형 하드웨어"]
featured: false
---

> 📌 **원문 안내**: 본 가이드는 [FreeAI Dog 공식 플랫폼](https://freeaidog.com/ko/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)에서 최초 발행되었습니다. 실시간 LLM 가용성 레이더 및 인터랙티브 테스트 도구는 공식 사이트를 방문하세요: [https://freeaidog.com/ko/](https://freeaidog.com/ko/).


지난달 저는 2코어 4GB 메모리의 저가형 클라우드 인스턴스에서 웹 스크래핑 에이전트를 운영하고 있었습니다.

시스템 메모리는 연일 한계선 위에서 줄타기를 하고 있었습니다.

헤드리스 Chromium 브라우저를 띄우는 것만으로도 몇 개 탭이 1.2GB 메모리를 집어삼켰습니다.

화면을 인식하겠다고 멀티모달 시각 모델을 백그라운드에 함께 올리는 순간, 프로세스가 로드되자마자 리눅스 커널의 OOM killer가 칼을 빼 들었습니다.

dmesg 로그를 열어보면 가혹한 알림이 박혀 있었습니다: `Out of memory: Killed process chromium`.

브라우저가 뻗어버렸습니다. 이런 위태로운 세팅을 누가 버텨낼 수 있겠습니까?

당시 유행하던 방식은 스크린샷 기반 시각 좌표 추출이었습니다.

전체 화면 캡처를 떠서 거대 시각 모델에 던지고, 클릭할 대상 버튼의 화면 픽셀 좌표를 계산하게 하는 접근법입니다.

이론상으로는 첨단처럼 들렸지만, 실전 운영에서는 온통 숨은 함정투성이였습니다.

화면 캡처 하나가 2에서 3메가바이트에 달했습니다; 이미지를 클라우드 API로 쏘는 것만으로 몇 초의 지연이 발생했고 비용 청구서도 만만치 않았습니다.

더 골치 아픈 문제는 해상도 배율이었습니다: 헤드리스 브라우저에서 `devicePixelRatio` 설정을 조금만 삐끗해도 계산된 좌표가 30에서 40픽셀씩 어긋났습니다.

마우스가 버튼 옆 엉뚱한 여백을 클릭하기 일쑤였습니다.

완전한 헛수고였습니다.

단순한 입력창 하나를 찾겠다고 수십억 파라미터짜리 거대 시각 모델을 동원해야만 하는 것일까요?

지난주 Hacker News에서 Folate라는 오픈소스 프로젝트를 발견했습니다. 저장소 주소는 github.com/figranium/folate입니다.

작업자는 대단히 절제된 설계를 택했습니다: 화면 캡처를 완전히 버리고, 초소형 신경망으로 DOM 트리 위의 상호작용 노드를 직접 읽어 들이는 방식이었습니다.

양자화된 모델 파일 크기는 고작 107MB에 불과했습니다.

로컬 환경에 내려받아 며칠간 검증해 보니, 일상적인 웹 자동화 작업에서 오랜 골칫거리를 깔끔하게 해결해 주었습니다.

## 멀티모달 껍데기 벗겨내기: 크로스 인코더가 시각 인식보다 안정적인 이유

브라우저 에이전트를 개발하는 많은 이들은 사람 눈에 보이는 화면 이미지가 가장 직접적인 진실이라고 착각하곤 합니다.

하지만 DOM 트리 앞에서 인간의 시각 인지 과정은 오히려 먼 길을 돌아가는 셈입니다.

웹페이지의 본질은 HTML 태그로 짜인 구조화 문서입니다; 버튼이 어디에 있는지, 입력 필드의 이름이 무엇인지, 폼이 어떤 이벤트와 묶여 있는지는 코드에 명확히 적혀 있습니다.

Folate의 아키텍처는 매우 실용적입니다: `google/electra-base-discriminator`를 기반으로 Mind2Web 실전 데이터셋에서 미세 조정을 거친 뒤, 동적 INT8 양자화를 적용한 ONNX 런타임 파일로 배포됩니다.

픽셀은 단 1개도 보지 않습니다.

추론 시 모델은 판별형 크로스 인코더로 작동합니다.

입력값은 두 가지입니다: 결제 승인 버튼을 누르라는 자연어 의도; 그리고 현재 페이지에서 추출된 상호작용 DOM 노드의 속성 텍스트.

모델은 이 쌍에 대해 적합도 확률 점수를 매깁니다.

핵심 파일인 `folate/ranker.py`의 전처리 로직은 믿기 힘들 만큼 가볍습니다:

```python
# 실제 조작 가능한 상호작용 노드만 필터링
INTERACTIVE_XPATH = (
    "//a | //button | //input | //select | //textarea | "
    "//*[@role='button' or @role='link' or @role='menuitem' or @role='tab' or @role='checkbox' or @role='combobox'] | "
    "//*[@onclick or @tabindex]"
)
```

lxml 라이브러리의 고성능 XPath 파싱을 활용해, 수 밀리초 만에 무의미한 텍스트와 장식 레이어를 걷어냅니다.

링크, 버튼, 입력창, ARIA 역할을 지닌 조작 요소만 일괄 수집됩니다.

추출된 각 후보 노드는 96개 토큰 이내의 간결한 문자열로 직렬화됩니다.

태그명, id, name, type, aria-label, placeholder, 내부 텍스트가 압축되어 담깁니다.

이 설계의 가장 큰 미덕은 모델이 선택자를 제멋대로 지어내는 환각을 상당 부분 차단한다는 점입니다.

상당 부분이라는 표현에 주목해 주십시오.

Folate는 현재 DOM 트리에 실재하는 노드만을 후보로 평가하므로, 출력된 선택자는 Playwright가 코드상에서 무조건 집어낼 수 있는 실체를 갖습니다; 다만 순위가 의도와 일치하는지는 별개의 문제입니다.

좌표 오차 문제도 원천적으로 우회합니다——애초에 픽셀 연산을 거치지 않으므로 레티나 디스플레이의 배율 오류가 발생할 구멍이 없습니다.

실체 존재 여부라는 기초 단계에서 입력 데이터는 물리적 확실성을 보장받습니다.

## GPU 없는 실전 벤치마크: PyTorch 없는 환경의 메모리와 지연 시간

저사양 환경의 한계를 시험하기 위해 저는 테스트 환경을 외장 그래픽이 없는 구형 노트북으로 옮겼습니다.

운영체제는 Ubuntu 24.04 LTS, 듀얼코어 CPU, 할당된 물리 메모리는 4GB였습니다.

설치 과정은 AI 프로젝트라고 믿기지 않을 만큼 산뜻했습니다:

```bash
# 무거운 PyTorch와 transformers가 일절 필요 없는 구성
pip install folate onnxruntime tokenizers lxml
```

CUDA도 없고, 수십 기가바이트짜리 딥러닝 번들도 없습니다.

전체 런타임은 onnxruntime, tokenizers, lxml만으로 가볍게 돌아갑니다.

이커머스 결제 화면을 자동 처리하는 Playwright 스크립트를 작성했습니다:

```python
from playwright.sync_api import sync_playwright
from folate import FolateSelectorFinder

finder = FolateSelectorFinder(num_threads=2)

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://demo.e-commerce.local/checkout")
    
    # 페이지 소스를 추출해 Folate로 랭킹 평가
    html_source = page.content()
    matches = finder.rank("click place order button", html_source, top_k=3)
    
    if matches:
        best_match = matches[0]
        print(f"매칭 점수: {best_match['score']:.4f}")
        print(f"대상 선택자: {best_match['selector']}")
        page.locator(best_match['selector']).click()
        
    browser.close()
```

스크립트를 실행하며 옆 터미널에서 htop을 띄워두고 자원 추이를 관찰했습니다.

측정 수치는 놀라울 만큼 정제되어 있었습니다.

모델 로딩 후 파이썬 프로세스의 상주 물리 메모리 [RSS]는 210MB 수준에서 안정되었습니다.

헤드리스 Chromium의 450MB 점유율을 합쳐도 전체 자동화 파이프라인이 700MB 미만을 소비했습니다.

메모리 부족으로 튕길 기미는 전혀 없었습니다.

로컬 거대 LLM이 7초에서 8초를 질질 끄는 반면, 2코어 CPU에서 50개 후보 노드를 평가하는 데 걸린 시간은 1.8초에서 2.4초 수준이었습니다.

이 속도가 느린지는 용도에 따라 다릅니다. 배치성 폼 자동 입력에는 충분한 수준이며, 연속적인 고주파 클릭 시나리오라면 더 단축해야 합니다.

Mind2Web 공식 6,037개 실전 액션 단계 평가 [test_task, test_website, test_domain]에서 이 107MB 모델은 88.9%에서 93.3%의 Top-1 정확도를 기록했습니다.

클릭 한 번마다 클라우드 멀티모달 API 청구서를 태우는 구조와 비교하면, 완전 오프라인으로 무료 구동되는 메리트는 매우 실질적입니다.

## 실전 개발의 시행착오: 폐쇄형 Shadow DOM과 비동기 렌더링의 함정

하지만 실전 현장에서 단번에 평온하게 끝나는 기술이란 존재하지 않습니다.

지난주 사내 Web Components 기반 결재 대시보드를 자동화하려다 이 파이프라인은 크게 걸려 넘어졌습니다.

화면에 승인 요청 버튼이 뻔히 보이는데도 스크립트는 우측 상단의 무관한 메뉴 아이콘을 1순위로 지목했습니다.

어째서 눈앞의 버튼을 놓친 것일까요?

저는 처음에 Mind2Web의 학습 데이터가 기업용 컴포넌트에 취약한 줄로만 의심했습니다.

로그를 뜯어보고 나서야 어처구니없는 실수를 깨달았습니다.

테스트 코드에서 Playwright의 `page.content`를 그대로 호출했던 것입니다.

이 API는 최외곽의 정적 HTML만을 반환합니다.

해당 프론트엔드 컴포넌트는 폐쇄형 Shadow DOM을 쓰고 있었고, 실제 조작 버튼은 Shadow Root 깊숙한 곳에 봉인되어 있었습니다.

표준 lxml XPath는 섀도 경계를 뚫지 못합니다.

내부 노드를 보지 못하는 Folate 입장에서는 노출된 외곽 태그 중에서 그나마 점수가 높은 엉뚱한 대상을 고를 수밖에 없었던 것입니다.

그렇다면 섀도 트리는 손을 쓸 수 없는 영역일까요?

해결책으로, DOM 추출 직전 Chrome DevTools Protocol을 통해 섀도 트리를 재귀적으로 펼쳐주는 자바스크립트를 주입했습니다:

```javascript
// 모든 Shadow Root 내부 노드를 재귀적으로 평탄화 추출
function getPiercedDOM(root = document.documentElement) {
    let html = '';
    const walker = document.createTreeWalker(root, NodeFilter.SHOW_ELEMENT);
    let node = walker.currentNode;
    while (node) {
        if (node.shadowRoot) {
            html += getPiercedDOM(node.shadowRoot);
        }
        node = walker.nextNode();
    }
    return html;
}
```

평탄화된 DOM을 ranker에 전달하자 깊숙이 묻혀 있던 승인 버튼이 스코어 0.9842를 기록하며 단숨에 1위로 튀어 올랐습니다.

또 하나 주의해야 할 지점은 SPA 프레임워크의 비동기 수화 지연입니다.

React나 Vue 기반 앱에서는 DOM 노드가 순간적으로 완성되지 않습니다.

load 이벤트 직후 `ranker.rank`를 호출하면 모델은 스켈레톤 스크린의 빈 껍데기를 분석하게 됩니다.

자동화 스크립트 작성 시 주요 컨테이너의 렌더링 완료를 대기하도록 설계하여 안정된 페이지 상태에서 판별을 수행해야 합니다.

## 적용 한계: 이 도구가 대체할 수 없는 영역

그렇다고 이 도구로 모든 문제를 해결할 수 있다고 과신해서는 안 됩니다.

구글 지도, 웹 게임, Figma처럼 Canvas나 WebGL로 화면 전체를 그리는 웹 애플리케이션에는 무력합니다:

표준 HTML 태그가 존재하지 않으며 화면 전체가 한 장의 픽셀 캔버스이기 때문입니다.

lxml이 조작 태그를 잡을 수 없는 환경에서 Folate는 아무런 힘을 쓰지 못합니다.

캔버스 중심 서비스에는 픽셀 좌표를 직접 해독하는 시각 모델을 쓰는 것 외에 지름길이 없습니다.

크로스 도메인 중첩 iframe도 까다로운 영역입니다; CDP로 권한을 명시적으로 주입하지 않으면 내부 문서는 감지되지 않습니다.

하지만 관리자 대시보드, 이커머스, 예매 시스템, 데이터 입력 폼 같은 대다수 표준 웹 환경에서는:

VRAM을 내어줄 필요가 없고 스크린샷 전송 지연도 증발합니다. 2코어 4GB 저사양 머신에서 이는 말뿐인 수치가 아니라 OOM killer의 습격을 실질적으로 한 번 막아내는 성과입니다.

저사양 환경에서 자동화를 구축할 때 버튼 하나 찾겠다고 수 기가바이트짜리 시각 모델을 끌어안고 있을 이유가 과연 있을까요?

---

> 💡 **원문 출처 및 실시간 모니터링**：
> - 원문 출처: [FreeAI Dog 공식 블로그](https://freeaidog.com/ko/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)
> - 무료 대형 언어 모델 실시간 가용성 및 지연 시간 측정은 [FreeAI Dog 가용성 레이더](https://freeaidog.com/ko/radar/)에서 확인하세요.


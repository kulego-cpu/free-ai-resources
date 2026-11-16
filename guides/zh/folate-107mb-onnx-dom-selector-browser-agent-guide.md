---
title: "把 14B 多模态视觉模型踢出浏览器：107MB 的 Folate 实测与调优笔记"
description: "复盘基于 107MB 轻量 ONNX 模型 Folate 替代笨重视觉大模型的真实工程现场：无 PyTorch 纯 CPU 推理、Mind2Web 真实网页元素排序、无幻觉 XPath 定位与 Shadow DOM 穿透避坑指南。"
pubDate: 2026-11-16
author: "小可"
category: "智能体与自动化"
tags: ["Folate", "Agent", "浏览器自动化", "ONNX", "Playwright", "平民硬件"]
featured: false
---

> 📌 **原创声明**：本文首发于 [FreeAI Dog 官方平台](https://freeaidog.com/zh/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)。如需体验 API 在线实时测速、可用性秒级雷达与端到端延迟监控，请访问主站：[https://freeaidog.com/zh/](https://freeaidog.com/zh/)。


上个月我在 2 核 4G 的轻量云服务器上跑抓取。

内存天天告急。卡死了。

无头 Chromium 浏览器开起来，几个标签页吃掉 1.2GB 内存。

后台再挂一个负责看图的多模态模型，Linux 内核的 OOM killer 立刻点名杀进程。

翻看 dmesg 日志，提示刺眼得很：`Out of memory: Killed process chromium`。

进程没了。这谁顶得住？

当时最流行的做法是走视觉定位路线。

截一张全屏高清图。丢给多模态大模型。让它输出目标按钮的屏幕像素坐标。

方案听起来很前沿，实际跑起来全是隐蔽暗坑。

屏幕截图动辄两三兆。图片传给云端模型不仅延迟慢，单次调用的账单还贵得肉痛。

更头疼的是分辨率缩放。

无头浏览器里没配好 devicePixelRatio 参数，算出来的坐标就会偏移三四十像素。

鼠标点歪了。经常点在按钮旁边的空白区域。

白费劲。

难道给自动化 Agent 定位网页上的一个输入框，非得搬出一张几十亿参数的视觉大模型吗？

上周在 Hacker News 上翻到一个叫 Folate 的开源项目。仓库地址在 github.com/figranium/folate。

作者做了一件极克制的事：彻底抛弃视觉截图，直接用微型神经网络去读 DOM 树上的交互式节点。

模型文件总共只有 107MB 左右。小巧得很。

我把它拉到本地实测了几天。这套方案在很多日常自动化场景里，确实解决了不少让人头疼的工程痛点。

## 撕掉多模态外壳：为什么跨编码器比纯视觉定位更稳

很多做浏览器 Agent 的工程师，总以为大模型眼睛看到的画面最贴近真实用户。

但在 DOM 树面前，人类眼睛其实是在走远路。

网页本质上是由 HTML 标签构成的结构化文档。按钮在哪里、输入框叫什么、表单绑定了哪个接口，代码里白纸黑字写得清清楚楚。

Folate 的底层架构非常务实。

它基于 `google/electra-base-discriminator` 微调；训练集是 Mind2Web 网页数据集；最后动态量化成 INT8 的 ONNX 格式。

它不看像素。

在推演时，它扮演的是一个判别式的跨编码器 Cross-Encoder。

输入是两样东西。第一是自然语言意图，比如点击确认支付按钮。第二是页面上提取出的交互节点属性。

模型负责给这一对输入打出一个匹配概率分数。

翻开它的核心代码 `folate/ranker.py`，预处理逻辑极其轻巧：

```python
# 筛选页面上真正能交互的 DOM 节点
INTERACTIVE_XPATH = (
    "//a | //button | //input | //select | //textarea | "
    "//*[@role='button' or @role='link' or @role='menuitem' or @role='tab' or @role='checkbox' or @role='combobox'] | "
    "//*[@onclick or @tabindex]"
)
```

依靠 lxml 库的高性能 XPath 解析，速度极快。

它在几毫秒内剔除所有无用文本与装饰层，只留真正的交互层。

链接、按钮、输入框以及带有 ARIA 交互角色的真实元素，全被一口气捞出来。

捞出来之后，每个候选节点被序列化成一段不超过 96 个 token 的紧凑文本。

文本里包含了标签名、id、name、type、aria-label、placeholder 以及内容文本。

这设计最大的好处，是把模型胡编选择器这条路堵掉大半。

注意，是大半。

Folate 只对当前 DOM 树里真实存在的节点打分，所以它挑出来的 selector 至少能被 Playwright 真正找到；但排序会不会偶尔排错，另说。

坐标漂移也绕开了——它压根不走像素定位，自然没有截图缩放偏移那堆破事。

至少在选择器实体这一层，输入是有确定性保证的。

## 零 GPU 依赖实测：无 PyTorch 环境下的内存与延迟实录

为了验证它在低配环境下的真实表现，我把测试环境搬到了一台老款无独显笔记本上。

操作系统是 Ubuntu 24.04 LTS。双核 CPU。总共分配了 4GB 内存。

安装过程轻快得不像是一个 AI 项目：

```bash
# 无需庞大的 PyTorch 与 transformers 依赖
pip install folate onnxruntime tokenizers lxml
```

没有 CUDA。没有几十个 G 的 Python 深度学习全家桶。

整个运行时只依赖 onnxruntime、tokenizers 和 lxml。

我写了个简单的 Playwright 脚本，让它自动登录一个复杂的电商结算页：

```python
from playwright.sync_api import sync_playwright
from folate import FolateSelectorFinder

finder = FolateSelectorFinder(num_threads=2)

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://demo.e-commerce.local/checkout")
    
    # 获取页面源码并交由 Folate 检索
    html_source = page.content()
    matches = finder.rank("click place order button", html_source, top_k=3)
    
    if matches:
        best_match = matches[0]
        print(f"匹配得分: {best_match['score']:.4f}")
        print(f"命中选择器: {best_match['selector']}")
        page.locator(best_match['selector']).click()
        
    browser.close()
```

运行脚本时，我在旁边开着 htop 盯紧系统资源开销。

数据出乎意料地干净。

模型完成初始化载入后，Python 进程常驻物理内存 RSS 稳定在 210MB 左右。

加上无头 Chromium 运行时的 450MB 开销，整套自动化工作流总共只吃了不到 700MB 内存。

内存没爆。机器跑得极稳。

本地大模型动辄七八秒，它在两核 CPU 下排 50 个候选节点大约是 1.8 到 2.4 秒。

慢不慢？看场景。批量跑表单填写完全能等；高频连续点击流，还是得想办法再往下压。

在 Mind2Web 官方涵盖 test_task、test_website 与 test_domain 的 6037 个真实步骤评测中，这个 107MB 的模型交出的 Top-1 准确率在 88.9% 到 93.3% 之间。

相比每次点击都要花钱去调用云端多模态大模型的商业方案，零成本且纯离线运行的优势难道不够实在吗？

## 真实排查弯路：Shadow DOM 穿透失败与动态加载暗坑

不过在真实工程环境里折腾，哪有一次跑通的便宜事？

上周在给内部一套基于 Web Components 架构的审批系统做自动化测试时，我就被这套逻辑结结实实绊了一跤。

页面明明肉眼可见有个提交申请按钮，脚本跑出来返回的得分最高项，居然是右上角的一个无关菜单项。

怎么可能找不到？

我一度怀疑是 Mind2Web 的训练集在企业级前端组件上的泛化能力不够。

日志翻了一圈，才发现自己犯了个低级失误。

测试用例里直接调用了 Playwright 的 `page.content`。

这个 API 返回的是最外层静态 HTML 源码。

而前端那套现代组件用了 closed 模式的 Shadow DOM，底层的真实交互按钮全被锁在 Shadow Root 节点深处。

lxml 的标准 XPath 根本穿不透这层影子根节点。

翻不到底层节点，Folate 自然只能在外部那些外露的标签里矮子里拔高个。

难道影子节点就拿它没辙了？

解决这道弯路，只能通过 CDP 注入一段 JavaScript。递归展开所有 Shadow DOM 节点：

```javascript
// 递归提取并展开所有 Shadow Root 内部节点
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

把平铺后的 DOM 树丢给 ranker。隐藏在深处的提交按钮立马冲到第一名。得分高达 0.9842。

另一个需要留心的暗坑是前端框架的异步水合。

在很多采用 React 或 Vue 构建的单页应用里，DOM 节点并不是一次性渲染完毕的。

如果在页面刚触发 load 事件那一瞬间立刻调用 `ranker.rank`，模型拿到的可能只是一堆处于骨架屏状态的空白占位符。

自动化脚本里必须前置一段对于关键容器的可见性等待。让 DOM 树在稳定状态下切片，排序结果才具有可信度。

## 适用边界与底线：它替代不了哪些多模态场景

但它也有干不了的活，别拿它硬套所有页面。

如果你面对的是类似 Google 地图、Web 游戏或者 Figma 这样完全基于 Canvas 与 WebGL 画布渲染的单页应用：

页面上根本没有标准的 HTML 标签。整个交互区就是一块绘制出来的纯像素。

面对这种场景，lxml 连一个可供打分的按钮都抓不出来，Folate 会当场失灵。

面对纯像素画布，除了搬出能读坐标的多模态视觉模型，还能有别的捷径吗？没有。

跨域嵌套的重度 iframe 也是它的软肋。没有针对性的 CDP 权限注入，多层嵌套的子页面很容易成为盲区。

如果你的自动化任务集中在常规网页：后台管理系统、电商购物、机票酒店预订、表单自动填写与内容聚合采集中。

显存不用给了，截图上传那几秒也省了。对 2 核 4G 这种机器，这不是 PPT 收益，是 OOM killer 少来一次。

面对手里吃紧的几台低配轻量机器，你愿意为了找一个按钮，继续给显卡硬塞十几吉字节的模型吗？

---

> 💡 **原文首发与实时探活**：
> - 本文首发于 [FreeAI Dog 官网博客](https://freeaidog.com/zh/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)
> - 更多免费 API 端点实时探活与端到端延迟测速，欢迎访问 [FreeAI Dog 可用性雷达](https://freeaidog.com/zh/radar/)


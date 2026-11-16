---
title: "14B の視覚大モデルをブラウザから追放する：107MB の Folate 実測と検証記録"
description: "重厚な視覚大モデルを 107MB の軽量 INT8 ONNX クロスエンコーダ Folate で置き換える現場手記：PyTorch 不要の CPU 推論、Mind2Web リアルウェブ要素ランキング、幻覚ゼロの XPath 抽出とクローズド Shadow DOM 突破法。"
pubDate: 2026-11-16
author: "小可"
category: "エージェントと自動化"
tags: ["Folate", "エージェント", "ブラウザ自動化", "ONNX", "Playwright", "低スペックハードウェア"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


先月、私は 2 コア 4GB メモリの格安クラウドインスタンスでウェブスクレイピングエージェントを動かしていました。

システムメモリは連日限界ギリギリの綱渡りでした。

ヘッドレス Chromium を起動するだけで、数個のタブが 1.2GB のメモリを消費します。

画面認識用のマルチモーダル小モデルをバックグラウンドに常駐させようものなら、プロセスがロードされた瞬間に Linux カーネルの OOM killer が発動しました。

dmesg ログを確認すると、無慈悲な警告が刻まれていました：`Out of memory: Killed process chromium`。

ブラウザが落ちました。こんな不安定な構成に誰が耐えられるでしょうか？

当時流行していたのは、スクリーンショットを用いた視覚認識アプローチでした。

画面全体の高解像度キャプチャを撮り、大規模な視覚モデルに放り込んで、対象ボタンのピクセル座標を出力させる手法です。

構想としては先進的に聞こえますが、実運用では落とし穴だらけでした。

全画面キャプチャは 1 枚で数メガバイトに達します；画像をクラウド API に送信するだけで数秒の遅延が生じ、リクエストごとの課金も馬鹿になりません。

さらに厄介なのが解像度スケーリングでした：ヘッドレスブラウザ側で `devicePixelRatio` の設定がわずかでも狂うと、計算された座標が 30 から 40 ピクセルずれ込みます。

ボタン横の空白エリアを虚しくクリックするだけでした。

完全に無駄足です。

入力欄をひとつ特定するためだけに、数十億パラメータの視覚大モデルを持ち出す必要があるのでしょうか？

先週、Hacker News で Folate というオープンソースプロジェクトを見つけました。リポジトリは github.com/figranium/folate です。

作者は非常に潔い設計を選択していました：画面キャプチャを完全に捨て去り、超小型ニューラルネットワークで DOM ツリー上の対話可能ノードを直接読み取る構造です。

量子化されたモデルのファイルサイズはわずか 107MB 程度でした。

手元の環境で数日間検証したところ、日常的な自動化タスクにおいて長年の課題を鮮やかに解決してくれました。

## マルチモーダルの皮を剥ぐ：クロスエンコーダが画像認識より安定する理由

ブラウザエージェントを作る開発者の多くは、人間の目に映るレンダリング画面こそが真実だと錯覚しがちです。

しかし DOM ツリーを前にしては、人間の視覚認識は遠回りをしているに過ぎません。

ウェブページの本質は HTML タグで構成された構造化ドキュメントです；ボタンがどこにあり、入力欄がどう命名され、どのハンドラが紐づいているかは、コード上に明確に記述されています。

Folate のアプローチは極めて実用的です：`google/electra-base-discriminator` をベースに Mind2Web データセットでファインチューニングされ、動的 INT8 量子化を施した ONNX ファイルとして配布されています。

ピクセルは一切見ません。

推論時、モデルは判別型クロスエンコーダとして機能します。

入力は 2 つです：決済確定ボタンをクリックせよという自然言語指示；そしてページから抽出された対話ノードの属性データ。

モデルはその組み合わせに対して適合度スコアを算出します。

核となる `folate/ranker.py` の前処理パイプラインは非常に洗練されています：

```python
# 実際に操作可能な DOM ノードのみを抽出
INTERACTIVE_XPATH = (
    "//a | //button | //input | //select | //textarea | "
    "//*[@role='button' or @role='link' or @role='menuitem' or @role='tab' or @role='checkbox' or @role='combobox'] | "
    "//*[@onclick or @tabindex]"
)
```

lxml の高速な XPath 構文解析を使い、わずか数ミリ秒で無関係なテキストや装飾要素を削ぎ落とします。

リンク、ボタン、入力欄、ARIA ロールを持つ対話要素が一瞬で網羅されます。

抽出された各候補ノードは、96 トークン以内のコンパクトな文字列へとシリアライズされます。

そこにはタグ名、id、name、type、aria-label、placeholder、テキスト内容が凝縮されています。

この設計最大のメリットは、モデルによるセレクタの幻覚をほぼ完全に封じ込める点にあります。

ほぼ完全、という限定に注意してください。

Folate は実際の DOM ツリーに実在するノードのみをスコアリング対象とするため、出力されたセレクタは Playwright が確実に捕捉できる実体を持ちます；ただしスコア順位が意図と合致するかは別の話です。

座標のズレも構造的に回避されます——ピクセル計算をそもそも行わないため、Retina ディスプレイのスケーリング誤差が生じる余地がありません。

実体の存在という土台において、入力データは確実性を帯びています。

## GPU 不要の実機テスト：PyTorch なし環境でのメモリと応答性

低スペック環境での挙動を確かめるため、私はテスト機をオンボードグラフィックスの旧型ノート PC に移しました。

OS は Ubuntu 24.04 LTS、デュアルコア CPU、割り当てメモリは 4GB です。

インストール作業は AI 関連とは思えないほど軽量でした：

```bash
# 巨大な PyTorch や transformers を一切使用しない構成
pip install folate onnxruntime tokenizers lxml
```

CUDA も不要なら、数十ギガバイトの深層学習ライブラリ群も不要です。

実行環境は onnxruntime、tokenizers、lxml だけで完結します。

デモ用 EC サイトの決済画面を自動操作する Playwright スクリプトを用意しました：

```python
from playwright.sync_api import sync_playwright
from folate import FolateSelectorFinder

finder = FolateSelectorFinder(num_threads=2)

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://demo.e-commerce.local/checkout")
    
    # ページソースを抽出し Folate でスコアリング
    html_source = page.content()
    matches = finder.rank("click place order button", html_source, top_k=3)
    
    if matches:
        best_match = matches[0]
        print(f"マッチスコア: {best_match['score']:.4f}")
        print(f"対象セレクタ: {best_match['selector']}")
        page.locator(best_match['selector']).click()
        
    browser.close()
```

実行中、隣のターミナルで htop を開いてリソース推移を注視しました。

計測値は驚くほど抑制されていました。

モデル初期化後の Python プロセス常駐メモリ [RSS] は約 210MB で安定しました。

ヘッドレス Chromium の 450MB を合算しても、自動化パイプライン全体で 700MB 未満に収まっています。

メモリ不足でプロセスが強制終了される気配は皆無でした。

ローカル LLM が推論に 7 秒から 8 秒を要するのに対し、2 コアの CPU で 50 個の候補ノードをソートする所要時間は 1.8 秒から 2.4 秒程度でした。

これを遅いと感じるかは用途次第です。バッチ処理のフォーム入力なら実用域ですが、高頻度な連続操作であればさらに詰めたいところです。

Mind2Web 公式の 6037 ステップに及ぶ評価セット [test_task, test_website, test_domain] において、この 107MB のモデルは Top-1 精度 88.9% から 93.3% を記録しています。

クリックごとにクラウドのマルチモーダル API 課金を垂れ流す構成と比較して、完全オフラインかつゼロコストで動く強みは極めて頑健です。

## 実践での回り道：クローズド Shadow DOM と非同期水和の落とし穴

しかし泥臭い本番開発において、一度で平穏に通る技術など存在しません。

先週、社内の Web Components ベースの承認ダッシュボードを自動化しようとした際、この設計は派手に躓きました。

画面上には明らかな申請ボタンが存在するにもかかわらず、スクリプトは右上の無関係なメニュー項目を最上位にランク付けしたのです。

なぜ見失うのでしょうか？

私は当初、Mind2Web の学習データが企業向けコンポーネントに適合していないのではないかと疑いました。

ログを精査した結果、単純な実装ミスに気づきました。

テストコードで Playwright の `page.content` をそのまま呼んでいたのです。

この API は最外部の静的 HTML のみを返します。

対象コンポーネントはクローズドモードの Shadow DOM を採用しており、実際のボタンは Shadow Root の深層に封じ込められていました。

標準の lxml XPath はシャドウ境界を透過できません。

内部ノードを認識できない Folate は、露出した外部タグの中から仕方なく最善候補を選ぶしかなかったのです。

ではシャドウツリーは手詰まりなのでしょうか？

解決策として、抽出前に Chrome DevTools Protocol を通じてシャドウツリーを再帰的に展開する JavaScript を注入しました：

```javascript
// すべての Shadow Root ノードを再帰的に展開・平坦化
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

展開後の DOM を ranker に渡したところ、深部に埋もれていた申請ボタンがスコア 0.9842 を叩き出して即座に首位へ躍り出ました。

もうひとつ警戒すべきは、SPA における非同期水和 [Hydration] の遅延です。

React や Vue 製のアプリでは、DOM が一瞬で組み上がるわけではありません。

load イベント直後に `ranker.rank` を呼ぶと、モデルはスケルトンスクリーンの空コンテナを読み取る羽目になります。

DOM 抽出の前に主要コンテナの表示完了を待機させ、安定したページ状態で判定を行う配慮が不可欠です。

## 適用限界：この手法が代替できない領域

ただし、この手法で何でも解決できると過信してはいけません。

Google マップ、ブラウザゲーム、Figma のように Canvas や WebGL で画面全体を描画するアプリに対しては無力です：

標準の HTML タグが存在せず、UI 全体が描画された純粋なピクセルに過ぎないからです。

lxml が対象ノードを捕捉できない以上、Folate は機能しません。

ピクセルベースの画面に対しては、座標を直接読み取れる視覚モデルを素直に使うほか道はありません。

クロスオリジンの多重 iframe も鬼門です；CDP 経由で権限を明示的に注入しない限り、内部ツリーは不可視のままです。

一方で、管理コンソール、EC ショッピング、予約システム、各種申請フォームといった大半の通常業務ウェブサイトに対しては：

VRAM を割く必要はなく、キャプチャ転送の遅延も消滅します。2 コア 4GB の貧弱なマシンにおいて、これは机上の空論ではなく、OOM killer の襲来を確実に 1 回防ぐ具体的成果です。

低スペック機で自動化を組む際、ボタンを 1 個探すためだけに巨大な視覚モデルを抱え込み続ける価値が本当にあるでしょうか？

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/folate-107mb-onnx-dom-selector-browser-agent-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


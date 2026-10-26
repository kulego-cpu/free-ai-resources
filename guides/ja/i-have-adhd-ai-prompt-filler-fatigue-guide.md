---
title: "AIのおしゃべりとToken浪費に終止符を！GitHubで話題沸騰の神スキル「i-have-adhd」全プラットフォーム実践ガイド：Claude Code/Cursor/ChatGPTを瞬時に戦闘モードへ"
description: "AIにコードの不具合を聞くたび、長い前置きや社交辞令にうんざりしていませんか？GitHubで数万Starを獲得した神オープンソース「ayghri/i-have-adhd」を徹底解説。10個の厳格な認知ガードレールにより、冒頭1行目に実行コマンドを直出しし、無駄なトークン消費を40〜60%削減。Claude Code、Cursor、ChatGPT対応の手順とコピペ用プロンプト付き。"
pubDate: 2026-10-26
author: "小可 (FreeAI Dog 開発チーム)"
category: "実践ガイド"
tags: ["プロンプトエンジニアリング", "ClaudeCode", "Cursor", "i-have-adhd", "Token最適化", "AIプログラミング", "生産性ハック", "システムルール"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# AIのおしゃべりとToken浪費に終止符を！GitHubで話題沸騰の神スキル「i-have-adhd」全プラットフォーム実践ガイド：Claude Code/Cursor/ChatGPTを瞬時に戦闘モードへ

AIを活用してコーディングや調査をしているとき、こんな経験でイライラしたことはありませんか？

ポート競合のエラーが発生し、ただプロセスを終了させるコマンドが1行欲しいだけなのに、AIが嬉々として長文の「お説教」を語り始めるあの現象です：

> *「素晴らしいご質問ですね！喜んでお手伝いさせていただきます。最新の分散コンテナ環境において、ネットワークのポート割り当ては極めて重要な要素です。通常、このエラーは主に3つの原因が考えられます。順を追って詳しく見ていきましょう……（ここで3画面分の教科書的な解説）……この解説がお役に立てば幸いです！他にも気になる点があればいつでもお気軽にどうぞ！」*

開発者は、この「咳払い（Throat-clearing）」のような長大な前置きや社交辞令、不要な総括をスクロールして読み飛ばし、一番下から**本来1秒でコピーできるはずだったコマンド**（`lsof -ti :8080 | xargs kill -9`）を必死に探し出さなければなりません。

この現象は**「不要表現疲労（Filler Fatigue）」**と呼ばれています。開発者の貴重な集中力（フロー状態）を分断するだけでなく、1往復ごとに数百トークンもの無駄な出力を垂れ流し、API費用を浪費し、コンテキストウィンドウを無意味に圧迫する**「トークン暗殺者」**でもあります。

今回ご紹介するのは、GitHub、Reddit、Xでエンジニアたちの絶大な支持を集めているオープンソースのルール設定スキル、[`ayghri/i-have-adhd`](https://github.com/ayghri/i-have-adhd) です。モデル自体の重みはいじらず、厳格な出力制約を課すことで、Claude Code、Cursor、ChatGPT、Geminiを**「冒頭1行目から即効コマンドを叩き出す精鋭アシスタント」**へと一瞬で改造します！

---

## 1. 誕生の背景：`i-have-adhd`とは何か？

このオープンソーススキルは、海外の開発者 `ayghri` 氏によって公開されました。着想の元となったのは、成人ADHDの心理療法における古典的名著『The Adult ADHD Tool Kit』（Ramsay & Rostain著）です。

ただし、あらかじめ強調しておきます：**このツールは決してADHD当事者だけのものではありません。無駄な時間を嫌い、効率を最優先するすべての開発者のための福音です！**

* **心理学的なアプローチ**：ADHDツールキットの真髄は、余計な刺激を遮断し、行動の粒度を極限まで細分化することで「行動開始の摩擦力（Activation Energy）」を下げる点にあります；
* **LLMの出力矯正**：著者は、現代のAIモデルのデフォルト設定（RLHF）が「過剰なお世辞」や「免責事項の前置き」で溢れかえっている点に着目しました。心理療法のガードレールを**プロンプト制約**としてAIに課すことで、AIの出力スタイルを「おしゃべりな家庭教師」から「指示即実行の戦闘オペレーター」へと強制再編したのです。

検証データによると、このルールを導入することで回答の即時実行性は**10倍**に向上し、会話全体の無駄なトークン消費が**40%〜60%削減**されることが確認されています！

---

## 2. 妥協を許さない「10の認知ガードレール」

`i-have-adhd` が劇的な効果を生む理由は、曖昧な指示ではなく「絶対に破ってはならない10のハードルール」として定義されているからです：

```
┌─────────────────────────────────────────────────────────────────┐
│               i-have-adhd 3つの主要作戦ライン                     │
├─────────────────────────────────────────────────────────────────┤
│  ⚡ アクション最優先 (Action-First) 👉 冒頭1行目に実行コマンド/パス │
│  🚫 ゼロ・フィラー規律 (Zero Filler)👉 社交辞令・不要な解説を完全排除│
│  🔢 認知負荷の厳格制限 (Strict Caps)👉 箇条書き最大5件、末尾は1指示 │
└─────────────────────────────────────────────────────────────────┘
```

### 1. 冒頭1行目で次のアクションを示す（Lead with the next action）
回答の1行目は、**直ちに実行できるコマンド、修正対象ファイルの絶対パス、またはコード片でなければなりません**。前置きや背景説明から書き始めることは厳禁です。

### 2. 複数ステップは必ず番号付け（Number multi-step tasks）
「まず……続いて……また……最後に……」といった散文形式の長文は禁止。複数の手順がある場合は、必ず `1. 2. 3.` の連番で簡潔に記述します。

### 3. 末尾は具体的な単一指示で締める（End with one concrete next step）
回答の最後には、**次に取るべきたった1つの具体的アクションだけ**を残します。「ご質問があればいつでもどうぞ」といった曖昧な結びは追放されます。

### 4. 脱線を徹底的に抑制（Suppress tangents）
質問された核心から外れた「周辺知識の解説」を勝手に挟むことを禁じます。聞かれたことだけに答え、聞かれていない専門知識はひけらかしません。

### 5. ターンごとに現状を簡潔に固定（Restate state briefly every turn）
「Authミドルウェア修正完了、テスト未実行」のように、半文で現在の進捗状況を明記。長時間のセッションによる文脈の喪失を防ぎます。

### 6. 具体的な所要時間の見積もり（Specific time estimates）
「すぐに終わります」「少々お待ちください」などの曖昧表現を禁止し、「ビルド所要時間：約2分」と数字で正確に伝えます。

### 7. 達成した成果の可視化（Make wins visible）
正常終了したテストや解決したモジュールには明示的に印（`[PASS]`など）を付け、現在どこまで確実に進んだかを把握させます。

### 8. エラーには冷静・客観的事実のみ（Matter-of-fact tone for errors）
バグやテスト失敗時、過度な謝罪や感情的なクッション表現（*"I apologize for the confusion..."*）を禁止し、客観的な事実と対処法だけを提示します。

### 9. リストの上限は最大5件（Cap lists at 5 items）
人間のワーキングメモリには限界があります。一度に出力する箇条書きは最大5件までに制限し、それ以上はグループ化するか削ぎ落とします。

### 10. 送信前の自己フィルタリング（Pre-send self-filter）
出力直前にAI自身が以下の4項目を自己検閲して削除します：
* ❌ 冒頭のお世辞（"Great question!", "お役に立てて光栄です"）
* ❌ 思考の独り言（"では詳しく見ていきましょう……"）
* ❌ 末尾の定型句（"お役に立てば幸いです！"）
* ❌ 内容の重複した無駄な総括。

---

## 3. 実例比較：Before / After の劇的な変化

Dockerのポート衝突エラーを調査した際の実例比較です：

### ❌ 適用前（デフォルト出力 - 480 Tokens・認知負荷大）
> *「コンテナ環境で非常にありがちなエラーですね！`bind: address already in use` というエラーが表示された場合、ホスト側の8080番ポートが他のプロセスによって占有されていることを意味します。この問題を解決するためには、通常3つのステップで対処します。まずポートを確認し、次に終了させ、最後に再起動します。具体的なコマンドは以下の通りです……（以下略、コマンドが最下部に埋もれる）」*

### ✅ 適用後（i-have-adhd 適用 - 85 Tokens・即座に実行可能）
> 以下のコマンドを実行して占有中の8080番ポートを強制解放してください：
> ```bash
> lsof -ti :8080 | xargs kill -9
> ```
> 1. 上記の強制終了コマンドを実行（所要約2秒）；
> 2. コンテナを再起動：`docker-compose up -d`；
> 
> 次のアクション：`docker ps` を実行してコンテナの稼働状態を確認。

**結果**：トークン消費量は480から85へ激減（**約82%の無駄をカット**）。回答が出力された0.1秒後にコマンドをコピーできます。

---

## 4. 各開発ツールでの導入手順

### 1. Claude Code ターミナル（公式プラグイン導入）
Anthropicの公式ターミナルツール **Claude Code** では、公式プラグインとして一瞬で導入できます：

```bash
# プラグインマーケットを追加
claude plugin marketplace add ayghri/i-have-adhd

# プラグイン本体をインストール
claude plugin install i-have-adhd@i-have-adhd
```

* **都度呼び出し**：セッション内で `/i-have-adhd` と入力；
* **常時有効化**：ターミナルで以下のファイルを作成すると、次回以降自動的に常時適用されます：
  ```bash
  touch ~/.claude/.i-have-adhd-always
  ```
* **一時停止**：チャットで `stop adhd mode` と送信すれば通常の口調に戻ります。

---

### 2. Cursor / Windsurf（プロジェクト別・全体設定）

* **プロジェクト別設定（推奨）**：プロジェクト直下に `.cursorrules` ファイルを作成し、下記のプロンプトテンプレートを貼り付けて保存します。
* **全体設定**：Cursorの `Settings` -> `General` -> `Rules for AI` にテンプレートを貼り付ければ、インライン編集（`Ctrl+K`）もチャット（`Ctrl+L`）もすべて無駄のないプロモードになります。

---

### 3. ChatGPT / Claude.ai / Gemini Web版
1. 各Webサービスのユーザー設定 -> **Custom Instructions（カスタム指示）** を開く；
2. **「どのように回答してほしいか」** の入力欄にテンプレートを貼り付けて保存。

---

## 付録：コピペですぐ使えるプロンプトテンプレート

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

## 結語

より巨大なモデル（125Bや405B）や高価なGPUを追い求めることと同じくらい、**「AIとの対話プロトコルの無駄を削ぎ落とすこと」** は日々の生産性に直結します。

Unixの偉大な哲学「**1つのことをうまくやれ（Do one thing and do it well）**」の通り、あなたのAIアシスタントから今日から無駄なおしゃべりを追放し、最高密度の開発環境を手に入れてみてください！

さらに実用的な開発ハックや無料APIの最新情報は、ぜひ [FreeAI Dog (freeaidog.com)](https://freeaidog.com/ja/) をご覧ください！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/i-have-adhd-ai-prompt-filler-fatigue-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


---
title: "LiteLLMがGemini思考署名ブラックホールを解消：Claude CodeとCursorへ1M思考流を0円で接続する実戦"
description: "LiteLLM最新コミットda296eeによるGoogle Gemini思考署名の透過修正を解剖。多往復ツール呼出で400 thought_signature喪失エラーが発生するプロトコル衝突の原因、ローカルプロキシ設定SOP、1M文脈の無料活用法を解説します。"
pubDate: 2026-11-17
author: "小可 [シャオコー]"
category: "アーキテクチャ"
tags: ["LiteLLM", "Gemini", "ClaudeCode", "Cursor", "思考モデル", "トラブルシューティング"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# LiteLLMがGemini思考署名ブラックホールを解消：Claude CodeとCursorへ1M思考流を0円で接続する実戦

一昨日の深夜、数百ファイルに及ぶ古いプロジェクトのリファクタリングを行っていました。

ターミナル上ではClaude Codeが稼働していました。

公式のClaude 3.7 Sonnetによる長考推論は極めて優秀です。しかしトークン消費速度は猛烈です。

出力価格は100万トークンあたり15ドルに達します。

数時間コーディングを回すだけで数十ドルの請求が発生します。これを持続するのは負担です。

そこで別の方針を検討しました。Google AI StudioのGemini 2.5 Flashは無償枠が充実しています。1Mトークンの超長文脈を持ち、思考ストリーム出力も無料で開放されています。

1M文脈を費用ゼロでClaude Codeの動力源にできれば理想的です。

ローカルでLiteLLMプロキシを起動し、Anthropic形式のエンドポイントをGeminiへリダイレクトしました。

初回の応答は極めてスムーズでした。

思考プロセスを豊富に含んだ出力が返ってきました。

しかし2往復目に入った瞬間、事態が急変しました。Claude CodeがBashコマンドでテストを実行するか、FileEditでコードを編集しようとすると、ターミナルに赤字のエラーが表示されました。

セッションが即座に中断しました。

画面にはGoogle APIのエラーが表示されていました：

```text
400 Invalid argument: Function call is missing a thought_signature
```

場合によってはエラーすら出ず、単一文字の無限ループに陥って停止することもありました。

無料活用の目論見は一行のエラーによって阻止されました。

このthought_signatureとはどのような仕組みなのでしょうか？

単発のテキスト応答は正常なのに、ツール呼出を行うと即座にエラーとなる理由は何でしょうか？

---

## 1. プロトコルの断絶：Google暗号署名とAnthropicメッセージ構造の衝突

両者のプロトコル電文を直接確認することで原因が判明しました。

根本的な要因はGoogle Gemini内部の検証ロジックにあります。

Googleの推論体系において、思考プロセスを持つモデルがFunction Callを生成する際、thought_signatureという暗号署名トークンを自動で付与します。

この署名は該当ツール呼出がどの思考文脈から導かれたかを証明します。

GoogleのAPI規約では、後続の往復でクライアントがツール実行結果を戻す際、前段のthought_signatureをそのまま再送することが義務付けられています。

署名が欠落するとどうなるでしょうか？Googleバックエンドは要求を遮断します。400エラーが返るか、文脈が混乱してループに陥ります。

ではなぜLiteLLMプロキシ層で署名が喪失してしまったのでしょうか？

原因はAnthropic Messages APIのスキーマ定義にあります。

Claude CodeはAnthropic規約に準拠して設計されており、/v1/messagesエンドポイントで通信します。

Anthropicのプロトコル上、ツール呼出はtool_useとtool_resultの2種類しか存在しません。

JSON Schemaには第三者の独自署名を格納する標準フィールドが用意されていません。

LiteLLMは当初、署名を拡張フィールドへ埋め込むか、tool_useのID末尾へ連結する手法をとっていました。しかしこれは機能しませんでした。

Claude Codeクライアントは転送前にAnthropic規約に基づいて厳密に入力値を検証し、非標準フィールドを除去します。

クライアントがローカルコマンドを実行してtool_resultを返送する際、独自フィールドはすでに消去されています。

LiteLLMは署名のない電文をGoogleへ送るしかありません。

Googleは署名不在を検知し、実行を拒絶します。

マルチターンの対話がこのプロトコル断絶によって遮断されていました。

---

## 2. パッチ解剖：セッション単位のメモリ退避と自動再注入

クライアントに署名の保持を期待することはできません。Claude Codeは公式のtool_useとtool_resultしか認識せず、余分な独自フィールドはすべて除去します。したがってLiteLLMプロキシ側で自律的に記憶する必要があります。

コミットda296ee4bfa706e2924ef4294d5b3d9802a2aa38において、LiteLLMはAnthropicアダプタの署名処理機構を刷新しました。

クライアントのデータ透過に頼らない堅牢な設計となっています。

第1の改修点——サーバー側ツール呼出キャッシュ層：

Geminiがthought_signatureを含むツール呼出を返した際、LiteLLMはクライアントへ委ねず、プロキシ内部のメモリ上にtool_call_idと文脈ハッシュで紐づくキャッシュ領域を確保します。

署名データはプロキシ側メモリに直接退避されます。

第2の改修点——送信電文の傍受と署名再付与：

Claude Codeが実行結果をPOST /v1/messagesで返送した際、アダプタはtool_resultの該当IDを照合します。

内部メモリから退避済みのthought_signatureを抽出し、Google宛ての送信電文へ透過的に再注入します。

クライアント側は署名の存在を意識する必要がありません。

データ連携構造は以下の通りです：

```text
Claude Code ターミナル [純粋なAnthropic規約]
      │
      ▼  POST /v1/messages [署名なしtool_result]
LiteLLM 動的プロキシ [コミットda296ee適用]
├─ tool_call_idマッピングキャッシュ照合
├─ thought_signature暗号署名の自動再付与
└─ Google AI Studio形式のREST電文へ変換
      │
      ▼
Google Gemini 2.5 Flash バックエンド [署名検証成功]
├─ 思考ストリーム継続 [1M文脈を維持]
└─ 正常なコード生成と多段階ツール呼出を返却
```

第3の改修点——フォールバック救済経路：

メモリキャッシュが消失した場合の対策として、skip_thought_signature_validatorスイッチも導入されました。ダミー署名を挿入して致命的クラッシュを防ぎます。ただし思考の文脈連続性が損なわれるリスクがあるため、キャッシュ維持が基本となります。

この仕組みにより、多往復ツール呼出のエラーが解消されました。

---

## 3. ローカル構築：Claude CodeとCursorの5分設定手順

仕組みを把握できれば、実機への適用は迅速に行えます。

LiteLLMを最新の修正版へ更新します：

```bash
pip install -U "litellm[proxy]>=1.63.1"
```

ソースからビルドする場合：

```bash
git clone https://github.com/BerriAI/litellm.git
cd litellm
pip install -e .
```

ローカル環境にconfig.yamlを作成します：

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

モデル名をclaude-3-7-sonnet-20250219へエイリアス設定することが重要です。

この設定によりClaude Codeの起動検証を通過し、リクエストをGeminiへ中継できます。

Google AI Studioのキーを設定してプロキシを起動します：

```bash
export GEMINI_API_KEY="AIzaSyYourRealKeyHere"
litellm --config config.yaml --port 4000
```

別シェルを開き、Claude Code向け環境変数を指定します：

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_API_KEY="sk-litellm-local-dev-key"
claude
```

ディレクトリ探索やテスト実行を指示した際の実ターミナル出力例です：

```text
╭── Claude Code ───────────────────────────────────────────────╮
│ Model: claude-3-7-sonnet-20250219 [via LiteLLM proxy :4000]   │
╰──────────────────────────────────────────────────────────────╯
> ユニットテストの検証とトークナイザAST定義の修正
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

14往復以上のファイル参照、テスト実行、コード編集が署名エラーなく完了しました。

Cursorでも同様に設定可能です：設定 > Models > OpenAI APIにおいて、Base URLにhttp://127.0.0.1:4000/v1、API Keyにsk-litellm-local-dev-keyを入力するだけで利用できます。

---

## 4. コスト台帳：無償枠と商用APIの比較検証

実務適用において費用比較は重要な判断材料となります。

2つの構成における日常コストの比較です：

公式Anthropic Claude 3.7 Sonnet：
入力100万トークン3ドル、出力100万トークン15ドルです。
本格的なリファクタリング作業では、1日に50万入力トークンと10万の思考出力トークンを消費します。
日額3ドルから8ドルの費用が発生し、月間では数百ドル規模に達します。個人の開発予算として看過できません。

Google AI StudioとLiteLLMの併用：
Gemini 2.5 Flashの無料枠は日常的な個人開発をカバーする十分な許容量を備えています。
API直接費用をゼロに抑えることが可能です。
さらにGemini 2.5 Flashは1Mトークンの長文脈を誇り、Sonnet標準の200Kを大幅に上回ります。
数万行のコードを分割せずに一度に読み込むことが可能です。

M3 Macでの実機メモリ消費量を計測しました：
LiteLLMプロキシの常駐メモリは約320MB、CPUはほぼ待機状態で、シリアライズ時に瞬間的に5%へ触れる程度です。
日常作業への影響は軽微ですが、Pythonプロセスであるためメモリ逼迫時に多重起動しないよう注意が必要です。

---

## 5. 運用上の境界条件とセルフチェック

実用性が高い一方で、留意すべき運用上の境界があります。

第1に、Google無料枠のレートリミットです。

Google AI Studio無料枠は毎分15リクエスト、日次1500コールに制限されています。

並列サブエージェントを自動起動して多数のツール呼出を同時に実行すると、429エラーが発生する可能性があります。

単一の作業に集中して利用し、大規模並列タスクには有償枠の併用を検討してください。

第2に、プロキシ再起動時のセッション切断です。

thought_signatureはプロキシ内部メモリに保持されるため、対話中にLiteLLMを再起動するとマッピングが消失します。

以後のツール応答は署名不一致で400エラーとなります。

pm2やnohupを用いてプロキシを安定常駐させ、作業中の意図しない再起動を防止してください。

日常のコーディングにおいて、読者の皆様はどちらを重視されますでしょうか？

Claude固有の緻密なコード論理でしょうか？

それとも1M文脈による広範なコード把握力でしょうか？

無料枠の制限を見据え、複数キーのローテーション運用も有効な選択肢となります。

まずは実験的なリポジトリにおいて本設定の安定性をお試しいただくことをお勧めします。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/litellm-gemini-flash-thinking-claude-code-thought-signature-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


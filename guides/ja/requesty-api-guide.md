---
title: "Requesty 徹底解説：業界最多の毎日200回無料呼び出し、$10 登録クレジット、13種類の0円モデルと5%最安手数料の活用術"
description: "次世代高可用性AIルーター Requesty を徹底レビュー！OpenRouterの4倍となる毎日200回の無料枠、登録で10ドルの残高付与（Claude 3.5 Sonnet / GPT-4o 試用可能）、13種類の完全無料モデル群（1.0M枠のNemotron 550BやGemma 4を含む）、全ワイルドカードCORS (*)、OpenAI/Anthropic両SDK対応とフランクフルトEUデータ常駐を完全解説。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外回線"
tags: ["Requesty", "AIルーター", "Nemotron", "Gemma 4", "無料API", "1Mコンテキスト", "OpenAI互換", "海外回線"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/requesty-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます。


# Requesty 徹底解説：業界最多の毎日200回無料呼び出し、$10 登録クレジット、13種類の0円モデルと5%最安手数料の活用術

各社の大規模言語モデル（LLM）を束ねるAIインテリジェントルーティング市場において、**Requesty** は圧倒的な開発者還元ポリシーを打ち出し、**「業界随一のコストパフォーマンスを誇るダークホース」**として注目を集めています。

従来のAPIアグリゲーションサービスと比較して、Requesty には数多くの際立った特長があります：
- 🎯 **毎日200回のリクエスト無料（業界最多水準）**：クレジットカードの登録不要で、新規登録組織には **毎日200回分の無料枠**（20 RPM並行制限）が自動付与されます。これは OpenRouter 初期無料枠（50回/日）の実に **4倍** に達します！
- 🎁 **登録で10ドルのアカウント残高を進呈**：メール認証だけで10ドル分のクレジットが即時付与され、Claude 3.5 Sonnet、GPT-4o、Gemini 2.0 Flash など **600種類以上の商用モデル** を自由に試用可能です。
- 📦 **13種類の先端モデルが永久に0円**：NVIDIA Nemotron 3 Ultra 550B（1.0M超長文コンテキスト）、Google Gemma 4 31B、InclusionAI 凌霄 3.1 Flash（プロンプトキャッシュ対応）、Meta Muse Glimmer など13モデルが完全無償で開放されています。
- 🌐 **全ワイルドカード CORS（`*`）対応**：グローバルおよび欧州フランクフルトのエンドポイントはともに `access-control-allow-origin: *` を返却するため、ブラウザ完結のWebアプリからも直結できます。
- 💰 **業界最安クラスの5%プラットフォーム手数料**：有料モデルの利用手数料はわずか5%（OpenRouter の5.5%より安価）で、BYOK（APIキー持ち込み）にも対応し、最低入金額は **0ドル** です。

本記事では、Requesty の無料枠仕様、13種類の無料モデル性能、Python による実装サンプル、および開発時の注意点を網羅して解説します。

---

## 1. 基本仕様とポリシー一覧

| 評価項目 | 公式ルールおよび実測検証結果 | 開発者向けアドバイス |
| :--- | :--- | :--- |
| **無料枠タイプ** | 🟢 **`daily_refresh`（毎日自動リセット）** | 毎日 **200リクエスト無料**（永久更新）＋ **$10 登録クレジット** |
| **クレカ登録** | 🟢 **クレジットカード不要** | メールアドレスのみで利用可能。電話認証や実名確認も不要 |
| **レート制限** | ⚡ **20 RPM（1分間あたり20回）** | 13種類の無料モデル共通で適用。超過時は HTTP 429 エラー |
| **自動課金リスク** | 🟢 **自動引き落としリスクゼロ** | 無料枠消化で停止。有料モデル利用には事前の入金が必須 |
| **CORS 対応** | 🟢 **完全対応（`*` ワイルドカード）** | OPTIONS プリフライトで `*` を返却。フロントエンド直結可能 |
| **SDK 互換性** | 🔄 **OpenAI ＆ Anthropic 双方対応** | 標準 `/v1` と Anthropic 専用 `/anthropic/v1/messages` を完備 |
| **プラットフォーム手数料** | 💰 **5%（業界最安クラス）** | OpenRouter の 5.5% より割安。BYOK 無料、最低入金額 $0 |
| **データ常駐拠点** | 🇪🇺 **フランクフルト（全プラン対応）** | 専用エンドポイント `router.eu.requesty.ai` でGDPR完全準拠 |

---

## 2. 13種類の完全0円無料モデル一覧

Requesty 上で **$0.00 / 1M Tokens（永久無料）** として指定されている13種類のモデル一覧です：

| # | モデル ID | 開発元 | コンテキスト | 主要機能・特徴 | 推奨ユースケース |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `nvidia/nemotron-3-ultra-550b-a55b` | NVIDIA | **1.0M** | Tools, Thinkモード | 巨大MoE推論、複雑なエージェント・論理タスク |
| 2 | `nvidia/nemotron-3-super-120b-a12b` | NVIDIA | **1.0M** | Tools, Thinkモード | 100万トークンの長文分析に優れた高コスパモデル |
| 3 | `nvidia/nemotron-3.5-lightning-30b-a3b` | NVIDIA | **1.0M** | Tools, Thinkモード | 高速応答に特化した軽量エージェント中核モデル |
| 4 | `nvidia/nemotron-3-nano-30b-a3b` | NVIDIA | 262K | Tools, Thinkモード | 低遅延タスクや軽量ロジック処理 |
| 5 | `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning` | NVIDIA | 131K | **Vision マルチモーダル**, Think | 画像やグラフの読解と深い思考プロセス推論 |
| 6 | `nvidia/nemotron-3.5-content-safety` | NVIDIA | 131K | **Vision マルチモーダル**, Think | テキスト・画像のコンテンツ安全ガードレール |
| 7 | `inclusion/ling-3.1-flash` | InclusionAI | 262K | **Prompt Cache**, Tools | プロンプトキャッシュ対応の超高速長文処理 |
| 8 | `inclusion/ling-3.0-tiny` | InclusionAI | 262K | **Prompt Cache**, Tools | 超低遅延な短文質疑応答と構造化データ抽出 |
| 9 | `google/gemma-4-31b-it` | Google | 262K | **Vision マルチモーダル**, Tools | Googleのオープン基盤。コード・数学・画像対応 |
| 10 | `meta/muse-glimmer-30b` | Meta | 131K | **Vision マルチモーダル**, Tools | Metaの視覚オープンモデル。画像解析とQA |
| 11 | `mistralai/leanstral-1-5` | Mistral AI | 262K | Tools, 厳密JSON出力 | 構造化JSON生成と関数呼び出し |
| 12 | `poolside/laguna-m.1` | Poolside | 33K | コーディング特化, Think | コード生成・自動テスト作成・リファクタリング |
| 13 | `poolside/laguna-xs.2` | Poolside | 33K | コーディング特化, Think | 超軽量なコード補完・シェルコマンド生成 |

---

## 3. 実装サンプル：OpenAI と Anthropic SDK の呼び出し

### 1. Python (OpenAI SDK 標準設定)

`base_url` を `https://router.requesty.ai/v1` に設定するだけで、Gemma 4 や 1M 枠の Nemotron を呼び出せます：

```python
from openai import OpenAI

client = OpenAI(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/v1"
)

response = client.chat.completions.create(
    model="google/gemma-4-31b-it",
    messages=[
        {"role": "system", "content": "あなたは分散システムの専門家です。"},
        {"role": "user", "content": "LLMルーターを導入するアーキテクチャ上のメリットを説明してください。"}
    ],
    temperature=0.7,
    max_tokens=1024
)

print(response.choices[0].message.content)
```

### 2. Python (Anthropic Messages API 設定)

Claude Code や Cursor などを愛用する開発者は、専用の Anthropic エンドポイントを直接指定できます：

```python
from anthropic import Anthropic

client = Anthropic(
    api_key="req_xxxxxxxxxxxxxxxxxxxx",
    base_url="https://router.requesty.ai/anthropic/v1"
)

message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "リトライと指数バックオフを備えた非同期HTTPクライアントをPythonで書いてください。"}
    ]
)

print(message.content[0].text)
```

---

## 4. 開発者が押さえるべき5大注意点

### 1. 業界最多の毎日200回無料枠
OpenRouterの無料枠（50回/日）と比較して、Requestyは初期状態で4倍となる200回/日（20 RPM）を提供します。ローカルでの反復テストや自動化Agentの検証に非常に有利です。

### 2. $10の登録残高で商用トップモデルを試用可能
付与される10ドルのクレジットはClaude 3.5 SonnetやGPT-4oなどの有料モデルにも充当できます。Sonnetであれば約100万トークンの出力を試すことができ、PoC検証に最適です。

### 3. 全ワイルドカード CORS（`*`）対応
curl OPTIONS による検証の結果、Requestyのエンドポイントは `access-control-allow-origin: *` を返却するため、ブラウザ拡張機能やSPAから直接通信できます。

### 4. 欧州フランクフルト拠点によるデータ主権の確保
GDPR対応が求められるプロジェクトでは、接続先を `https://router.eu.requesty.ai/v1` に指定するだけで、プロンプト処理がEU域内に限定されます。

### 5. 中国本土からの接続にはプロキシ環境推奨
サーバーが米国東部とフランクフルトにあるため、中国大陸からの直結時はプロキシ環境経由で接続することで安定した低遅延を確保できます。

---

## 5. 3ステップで完了する利用開始手順

1. **Requesty でメール登録（$10獲得）**：[Requesty 登録ページ](https://app.requesty.ai/api-keys) にアクセスし、メアドのみで登録を完了します（クレカ不要）。
2. **API キーを発行**：ダッシュボードの API Keys で `req_` から始まるキーを1クリックで作成します。
3. **Base URL を設定して無料利用開始**：エンドポイントを `https://router.requesty.ai/v1` に指定すれば、13モデルを毎日200回まで存分に活用できます！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/requesty-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。

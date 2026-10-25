---
title: "OpenRouter 無料専用枠完全攻略ガイド：24種の公式 :free モデル・毎日50回リセット・CORS完全開放・10ドル拡張の裏技"
description: "世界最大のLLM統合ルーター OpenRouter の無料枠を徹底解剖！24種類の公式 :free モデル（NVIDIA 550B巨型モデル、Gemma 4、MiniMax 100万コンテキスト）、毎日50回リセットの仕組み、CORSワイルドカード（*）完全対応、10ドル入金で1日1,000回へ20倍拡張する裏技と5大注意点を完全網羅。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外・中国直連"
tags: ["OpenRouter", "無料モデル", "スマートルーティング", "クレカ不要", "CORS開放", "OpenAI互換", "海外回線"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/openrouter-free-models-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます。


# OpenRouter 無料専用枠完全攻略ガイド：24種の公式 :free モデル・毎日50回リセット・CORS完全開放・10ドル拡張の裏技

現在、多種多様なAIモデルが登場していますが、開発者が直面する最大の課題は**プロバイダーの乱立**です。各社ごとに個別登録し、クレジットカードを登録し、APIキーとエンドポイントを管理するのは非常に煩雑です。

世界最高峰の統一LLMゲートウェイとして認知されている **OpenRouter（openrouter.ai）** は、この課題を一挙に解決します。全680種類以上の商用モデルを単一エンドポイントに集約しているだけでなく、公式に**`:free` 認証無料モデル専用プール**を常設しています：
- ❌ **国際クレジットカードや銀行口座の登録は一切不要**
- ❌ **電話番号認証や本人確認なし（GitHub / Google / メールで1分登録）**
- ✅ **12社の主要プロバイダーから厳選された24種類の無料モデルを提供（NVIDIA 550B超大モデル、Gemma 4、MiniMax 1M長文、音声・ベクトル埋め込み）**
- ✅ **CORSワイルドカード（`*`）に完全対応 — バックエンドプロキシ不要でブラウザから直接通信可能**
- ✅ **登録直後から毎日50リクエストの無料枠（20 RPM、毎日UTC 00:00に自動復活）**
- 🚀 **破格のアップグレード：累計10ドル（約1,500円）入金するだけで、1日1,000リクエスト（20倍）へと恒久拡張**
- 🟢 **安心の前払い制：無料モデル呼び出しは完全0円、意図せぬ自動請求の危険性ゼロ**

本記事では、OpenRouter 無料枠の仕様、24種類のモデル全容、CORS実測検証、および本番導入における5大注意点をわかりやすく解説します。

---

## 1. 無料枠の基本ポリシーと利用条件（毎日50回復活）

OpenRouterの無料枠は、毎日リフレッシュされるリクエストバケット形式を採用しています：

| 評価項目 | 公式規定と実測結果 | 開発上のポイント |
| :--- | :--- | :--- |
| **無料枠タイプ** | ✅ **`daily_refresh`（日次自動復活）** | 毎日 UTC 00:00 に自動リセット、永久有効 |
| **基本無料枠** | **50 リクエスト / 日** | 未課金または累計入金 $10 未満のアカウント |
| **昇格後無料枠** | **1,000 リクエスト / 日（20倍へ爆増）** | 累計入金額が $10（約1,500円）に達すると恒久適用 |
| **レートリミット** | ⚠️ **20 RPM（1分あたり20リクエスト）** | 開発検証・プロトタイプ向けの上限値 |
| **クレジットカード** | ❌ **不要（カード番号入力なし）** | 登録や無料Key発行に決済手段は一切不要 |
| **本人確認** | ❌ **実名認証不要** | メール・Google・GitHub連携ですぐ利用可能 |
| **CORS対応** | 🟢 **完全開放（ワイルドカード `*`）** | `access-control-allow-origin: *` を返却、フロント直結可能 |
| **ネットワーク接続** | 🌐 **国際回線（要プロキシ）** | 日本・グローバルからは直結可能、中国本土からは要プロキシ |
| **課金安全性** | 🟢 **完全前払い制（自動請求なし）** | `:free` モデルは消費ゼロ、枯渇時はHTTP 429返却 |
| **自動フェイルオーバー** | 🟢 **`openrouter/free` ルーター** | 単一モデルIDで稼働中の無料モデル間を自動迂回 |

---

## 2. 24種の無料モデル全容（12社提供）

OpenRouterの無料専用枠は、小型モデルだけでなく超大型フラッグシップからコード特化、マルチモーダルまで充実しています：

### 1. 17種のテキスト・推論モデル

| モデルID（末尾に :free が必須） | コンテキスト | 提供元 | 特徴とおすすめ用途 |
| :--- | :--- | :--- | :--- |
| **`google/gemma-4-31b-it:free`** | 262K | Google | 最新オープンモデル、高度な推論と多言語指示追従 |
| **`google/gemma-4-26b-a4b-it:free`** | 262K | Google | Gemma 4 の軽量高速版、初速が極めて速い |
| **`nvidia/nemotron-3-ultra-550b-a55b:free`** | 1,000K (1M) | NVIDIA | **550B超大型モデル**、1Mコンテキストの圧倒的深層推論 |
| **`nvidia/nemotron-3-super-120b-a12b:free`** | 262K | NVIDIA | 120B級バランス型スーパーモデル |
| **`nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free`** | 256K | NVIDIA | 思考連鎖（CoT）と数学・論理検証に特化 |
| **`nvidia/nemotron-3.5-super-vl:free`** | 1,000K (1M) | NVIDIA | 1Mコンテキストの超強力画像・テキストマルチモーダル |
| **`nvidia/nemotron-3.5-lightning:free`** | 1,000K (1M) | NVIDIA | 超高速低遅延推論、100万トークンの長文も即座に応答 |
| **`nvidia/nemotron-3.5-content-safety:free`** | 128K | NVIDIA | 有害プロンプト検知とコンテンツ安全ガードレール |
| **`minimax/minimax-m3.1-flash:free`** | 1,000K (1M) | MiniMax | **100万コンテキストの国産巨頭**、膨大な文書要約に最適 |
| **`cohere/north-mini-code:free`** | 256K | Cohere | コード補完・リファクタリング専用モデル |
| **`poolside/laguna-s-2.1:free`** | 262K | Poolside | プログラミングタスクと複数ファイル構造把握に特化 |
| **`poolside/laguna-xs-2.1:free`** | 262K | Poolside | 軽量高速コーディングアシスタント |
| **`thinkingmachines/inkling:free`** | 1,000K (1M) | Thinking Machines | 1M超長文推論と意思決定支援 |
| **`thinkingmachines/inkling-small:free`** | 1,000K (1M) | Thinking Machines | 高効率推論エンジン |
| **`xingchen-agi/xing4.0-29b-a4b:free`** | 262K | 星辰 AGI | 29B級の汎用対話アシスタント |
| **`dots-studio/dots-3-note-preview:free`** | 512K | Dots Studio | 512Kコンテキストのノート整理・ドキュメント作成 |
| **`liquid/lfm-2.5-2.6b:free`** | 65K | Liquid | エッジデバイス向け超軽量高効率モデル |

### 2. 7種のマルチモーダル・音声・埋め込みモデル

| モデルID | 種別 | コンテキスト | 提供元 | 概要 |
| :--- | :--- | :--- | :--- | :--- |
| **`fish-audio/s2.1-pro-free:free`** | 🎤 音声合成 | — | Fish Audio | オープンソース最高峰の高品質TTS |
| **`inception/mercury-decide:free`** | 🧠 意思決定 | 65K | Inception | 分岐ロジックや戦略プランニング評価 |
| **`nvidia/llama-nemotron-embed-vl-1b-v2:free`** | 📊 視覚埋め込み | 131K | NVIDIA | 画像＋テキストの複合セマンティックベクトル化 |
| **`nvidia/llama-nemotron-rerank-vl-1b-v2:free`** | 🔍 リランキング | 10K | NVIDIA | RAG検索パイプラインの高精度多峰リランク |
| **`nvidia/nemotron-3-embed-1b:free`** | 📊 テキスト埋め込み | 32K | NVIDIA | 高精度テキスト検索用ベクトル化 |
| **`liquid/lfm-2.5-embedding-350m:free`** | 📊 エッジ埋め込み | 512 | Liquid | 超低遅延エッジ向けコンパクト埋め込み |
| **`respan/span-01-lite:free`** | 🧠 構造推論 | — | Respan | エンティティ抽出と構造化分析 |

---

## 3. クイックスタート実装（標準 OpenAI SDK 互換）

OpenRouterは標準OpenAI互換エンドポイントを採用しているため、`baseURL` に `https://openrouter.ai/api/v1` を指定するだけで動作します：

```python
from openai import OpenAI

client = OpenAI(
    api_key="your_sk-or-v1_key_here",
    base_url="https://openrouter.ai/api/v1",
    default_headers={
        "HTTP-Referer": "https://freeaidog.com",
        "X-Title": "FreeAI Hub Client"
    }
)

# 1. 特定の無料モデルを呼び出す（末尾の :free を忘れずに）
response = client.chat.completions.create(
    model="google/gemma-4-31b-it:free",
    messages=[
        {"role": "system", "content": "あなたは優秀なエンジニアです。"},
        {"role": "user", "content": "CORSのワイルドカード設定がフロントエンド開発者にもたらす利点を3つ教えてください。"}
    ],
    temperature=0.3
)
print(response.choices[0].message.content)

# 2. openrouter/free による自動スマートルーティング
fallback_resp = client.chat.completions.create(
    model="openrouter/free",
    messages=[{"role": "user", "content": "自動ルーティングの疎通確認！"}]
)
print(fallback_resp.choices[0].message.content)
```

---

## 4. 開発者が注意すべき5つの落とし穴

### 1. 必ず `:free` 接尾辞を付与（有料課金の回避）
同一モデル名で有料版と無料版が存在します（例: `google/gemma-4-31b-it` と `google/gemma-4-31b-it:free`）。
> **注意点**：無料枠で利用する場合は**必ずモデルIDの末尾に `:free` を付けてください**。付け忘れると有料ルートに回され、残高が消費されます。

### 2. マイナス残高による全機能ロックアウト
有料モデルの利用で残高がわずかでもマイナス（-$0.001など）になると、**無料モデルを含めアカウント全体の呼び出しが HTTP 402 で遮断されます**。
> **対策**：無料専用で使う場合は残高 $0.00 を維持してください。万一マイナスになった場合は、0円以上に入金することで即座に復帰します。

### 3. 20 RPM & 毎日50回制限と「10ドルアップグレード」
無料層は1日50リクエスト・20 RPMです。
> **裏技**：OpenRouterのアップグレード条件は極めて良心的です。**アカウントの累計入金額が10ドル（約1,500円）に達すると、無料モデルの1日上限が1,000回（20倍）へと恒久的に拡張されます**。

### 4. CORS完全開放（`*`）の絶大な恩恵
実測検証の結果、OPTIONSプリフライトで `access-control-allow-origin: *` を返却します。ReactやVueなどのSPA、ブラウザ拡張機能から中継サーバーなしで直接APIを叩けます。

### 5. `openrouter/free` 自動ルーターの活用
単一の無料プロバイダーが一時的に過負荷になった場合に備え、フォールバック先に `openrouter/free` を指定しておくと、現在稼働中で最も空いている無料モデルへ自動的にリクエストを振り分けてくれます。

---

## 5. まとめ

OpenRouterは、1つのAPIキーで世界の主要オープンモデルや特化型AIを無料かつ安全に試せる最強のハブです。今すぐ [OpenRouter Keys 設定ページ](https://openrouter.ai/settings/keys) でキーを発行して活用してみてください！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/openrouter-free-models-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。

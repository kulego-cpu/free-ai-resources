---
title: "Cloudflare Workers AI 実践ガイド：世界300+エッジ直結・毎日10,000 Neurons永久無料・計算量換算とCORS対策"
description: "Cloudflare Workers AI サーバーレスエッジ推論基盤を徹底検証！毎日UTC 0時に自動補充される10,000 Neurons（ニューロン計算資源）永久無料枠、トークン数とNeuronsの実測換算（1日35〜550回相当）、約80種類のマルチモーダルモデルカタログ、REST APIのAccount ID必須仕様とCORS対策を完全解説。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外受限"
tags: ["Cloudflare", "Workers AI", "Llama 3.3", "DeepSeek R1", "永久無料", "エッジコンピューティング", "毎日リセット", "FLUX"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/cloudflare-workers-ai-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# Cloudflare Workers AI：世界300+拠点エッジ推論、毎日10,000 Neurons永久無料＆実践エンジニアリングガイド

世界屈指のCDNおよびクラウドセキュリティ企業である **Cloudflare** は、世界300以上の都市に広がるデータセンターとAnycastルーティング網を活用し、サーバーレスAI推論プラットフォーム **Workers AI** を展開しています。世界中のエッジサーバーに配置されたGPUクラスター上でオープンソースモデルを直接実行し、物理的に最も近い拠点へ自動ルーティングすることで、長距離ネットワーク往復によるレイテンシを極限まで削減しています。

開発者向けエコシステムにおいて、Cloudflareは極めて魅力的なプランを提供しています：**クレジットカード不要の無料プラン（Workers Free）に対し、毎日10,000 Neurons（ニューロン計算単位）を無償付与し、協定世界時（UTC）午前0時に毎日自動補充されます！**

ただし、Cloudflare Workers AI は一般的なAIプロバイダーと異なり、**トークン数ではなくNeurons（計算単位）で計測される点**、REST APIのURLに固有のAccount IDを含める必要がある点、ブラウザ向けCORSヘッダーがデフォルト未開放である点など、特有の設計が存在します。本稿では、2026年10月の最新検証に基づき、無料枠の仕様と実務上の注意点を詳しく解説します。

---

## 1. 基本仕様と無料枠の検証

| 項目 | 検証詳細と公式仕様 |
| :--- | :--- |
| **無料タイプ** | ✅ **`daily_refresh`（毎日自動リセット・永久無料プラン）** |
| **無料枠** | **毎日 10,000 Neurons**（全モデルで共有、トークン課金ではない） |
| **リセット周期** | **毎日 UTC 00:00（日本時間 午前09:00）に全量自動補充** |
| **クレジットカード** | ❌ **不要**（一般メールアドレスのみで即時利用可能） |
| **超過時の挙動** | 枠枯渇時は `HTTP 429` または `Error 3040 / 1027` を返却、勝手な自動課金なし |
| **デフォルト制限** | テキスト生成タスクで **300 RPM**（軽量モデルでは最大 720〜1,500 RPM） |
| **提供モデル数** | **約80種類のオープンソースモデル**（LLM、推論、画像生成、音声、埋め込み） |

---

## 2. Neurons の計算：10,000 Neurons でどれくらい使えるか？

Cloudflare が導入している **Neurons（ニューロン計算単位）** は、物理GPUの浮動小数点演算やメモリ帯域の消費量を標準化した指標です。

### 実利用シナリオ別の呼び出し可能回数換算

| 利用シナリオ | 1回あたりの想定トークン数 | 1リクエスト消費 Neurons | 1日（10,000 Neurons）の実行可能回数 |
| :--- | :--- | :---: | :---: |
| **軽量・短文QA** | 2K Input / 200 Output | 約18 Neurons | **約 550 回 / 日** |
| **標準コンテキスト対話** | 6K Input / 1K Output | 約70 Neurons | **約 140 回 / 日** |
| **マルチステップ Agent 処理** | フルコンテキスト + 4回反復 | 約280 Neurons | **約 35 回 / 日** |

> 💡 **結論**：個人の学習、プロトタイプ検証、Discord/Slack Bot、定期スクリプト処理などには毎日10,000 Neuronsで十分に対応可能です。大規模な商用本番運用の場合は、必要に応じて Workers Paid（月額$5＋超過時 $0.011/1,000 Neurons）へのアップグレードをご検討ください。

---

## 3. モデルラインナップ：80+種類のオープンソースと有料フロンティアモデル

### 1. Workers Free で利用可能な注目モデル

| タスク分類 | 公式モデルID (`modelId`) | 特徴とユースケース |
| :--- | :--- | :--- |
| **フラッグシップ対話** | `@cf/meta/llama-3.3-70b-instruct-fp8-fast` | エッジ最適化された高速Llama 3.3 70B、128Kコンテキスト、超高速初回トークン |
| **深層推論モデル** | `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | DeepSeek R1蒸留版32B、深い思考ロジックと高速応答性を両立 |
| **高効率チャット** | `@cf/qwen/qwen2.5-7b-instruct` | 通義千問 7B、極めて低いNeurons消費で高精度な多言語会話 |
| **超軽量LLM** | `@cf/microsoft/phi-2` | Microsoft 2.7B小型モデル、最大 720 RPM の高スループット |
| **画像生成 (FLUX)** | `@cf/black-forest-labs/flux-1-schnell` | 世界トップクラスのFLUX.1高速画像生成、4ステップで高品質描画 |
| **音声認識 (STT)** | `@cf/openai/whisper` | OpenAI Whisperによる多言語音声テキスト変換エッジサービス |
| **ベクトル埋め込み** | `@cf/baai/bge-large-en-v1.5` | BAAIベクトル埋め込みモデル、RAGやセマンティック検索向け |

### 2. ⚠️ 注意：特約フロンティアモデルは Workers Paid 限定

一部の最先端モデル（`@cf/moonshotai/kimi-k2.6`、`@cf/moonshotai/kimi-k2.7-code`、`@cf/zai-org/glm-5.2` など）は、**無料プラン（Workers Free）では利用できません**。これらを利用するには Workers Paid プランまたは AI Gateway の事前購入クレジットが必要です。

---

## 4. 開発における重要ポイント（Gotchas）

### 1. ⚠️ Base URL には自身のアカウントIDの埋め込みが必須（404エラー対策）
Cloudflare の REST API エンドポイントは個別アカウントごとにパスが分かれています：
- **誤った形式**：`https://api.cloudflare.com/client/v4/ai/v1` ❌
- **正しい形式**：`https://api.cloudflare.com/client/v4/accounts/{YOUR_ACCOUNT_ID}/ai/v1` ✅
- **取得方法**：[dash.cloudflare.com](https://dash.cloudflare.com/) にログイン後、ダッシュボード右側サイドバーの「Account ID」をコピーしてください。

### 2. ⚠️ REST API は CORS 未対応（フロントエンド直接呼び出し時のブロック）
公式 REST API ゲートウェイは `Access-Control-Allow-Origin` ヘッダーを返却しません。Webブラウザから直接 `fetch()` するとCORSエラーが発生します。
- **解決策**：NextChat や ChatBox などのデスクトップアプリから呼び出すか、Node.js/Python などのバックエンド経由、または Cloudflare Worker をリバースプロキシとして中継させてください。

### 3. ⚠️ 超過時のエラーコードと安全性
1日の 10,000 Neurons を使い切った場合、`HTTP 429` または `Error 3040 / 1027` が返却されます。無料アカウントにはクレジットカードが登録されていないため、自動で課金が発生する心配は一切ありません。

### 4. ⚠️ API Token 作成時は専用テンプレートを使用
「プロフィール -> API トークン -> トークンを作成」から、必ず公式の **「Workers AI (Read/Write)」** テンプレートを選択して作成してください。

---

## 5. 実装サンプルコード

### 1. cURL での動作確認

```bash
export ACCOUNT_ID="あなたの_ACCOUNT_ID"
export CF_TOKEN="あなたの_WORKERS_AI_API_TOKEN"

curl -X POST "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/ai/v1/chat/completions" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "@cf/meta/llama-3.3-70b-instruct-fp8-fast",
    "messages": [
      {"role": "user", "content": "エッジAIの利点を簡潔に1文で教えてください。"}
    ],
    "max_tokens": 100
  }'
```

### 2. Python での呼び出し

```python
import os
import requests

ACCOUNT_ID = os.environ.get("CLOUDFLARE_ACCOUNT_ID", "YOUR_ACCOUNT_ID")
API_TOKEN = os.environ.get("CLOUDFLARE_API_TOKEN", "YOUR_API_TOKEN")

url = f"https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/ai/v1/chat/completions"

headers = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json"
}

payload = {
    "model": "@cf/deepseek-ai/deepseek-r1-distill-qwen-32b",
    "messages": [
        {"role": "user", "content": "9.11 と 9.9 ではどちらが大きいですか？理由とともに答えてください。"}
    ]
}

res = requests.post(url, headers=headers, json=payload)
data = res.json()
print("回答：", data["choices"][0]["message"]["content"])
```

### 3. Cloudflare Worker 内でのネイティブ実行（ネットワークオーバーヘッド皆無）

Cloudflare Workers のスクリプト内からは、API Token や HTTP 通信のオーバーヘッドなしに `env.AI` バインディングで即時呼び出しが可能です：

```javascript
export default {
  async fetch(request, env) {
    const answer = await env.AI.run("@cf/meta/llama-3.3-70b-instruct-fp8-fast", {
      messages: [
        { role: "user", content: "Hello from Cloudflare Edge!" }
      ]
    });

    return new Response(JSON.stringify(answer), {
      headers: { "Content-Type": "application/json" }
    });
  }
};
```

---

## 6. まとめ

Cloudflare Workers AI は、現代のオープンソースAIエコシステムにおいて独自の強みを発揮しています：
1. **圧倒的なエッジ超低遅延**：世界300箇所以上の Anycast 網により、長距離往復遅延を回避；
2. **充実のマルチモーダル対応**：テキスト対話、推論、FLUX画像生成、Whisper音声認識がすべて同一の 10,000 Neurons 無料枠で利用可能；
3. **安心の完全クレカ不要**：毎日協定世界時午前0時に自動全量リセット、超過時も安全停止で隠れた請求リスクなし。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/cloudflare-workers-ai-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


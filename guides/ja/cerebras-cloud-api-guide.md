---
title: "Cerebras Cloud 完全実測ガイド：世界最速 3,000 tokens/s 推論、クレカ認証で $5 試用枠、CORS 解放と 402 停止の真相"
description: "Cerebras Cloud のウエハー規模超並列推論クラウドを徹底解説！実測 3,000 tokens/s の異次元速度を検証。永久無料の噂を検証し、2026年7月以降のクレジットカード認証型 $5 クレジット試用枠（30日有効）、CORS 直結、5 RPM 制限、HTTP 402 安全停止の全貌を公開。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外プロバイダー"
tags: ["Cerebras", "gpt-oss-120b", "ウエハー規模チップ", "3000tps", "爆速推論", "OpenAI互換", "CORS"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/cerebras-cloud-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# Cerebras Cloud：世界最速のウエハー規模推論エンジン実測＆ $5 試用クレジット活用ガイド

Groq が LPU チップによって 500 tokens/s のタイプライター速度を実証したのに対し、シリコンバレーの先端超並列コンピューティング企業 **Cerebras Systems** が提供する **Cerebras Cloud Inference** は、その速度をさらに極限へと押し上げました。その実効速度はなんと **1,800 〜 3,000 tokens/秒！**

数千文字に及ぶコードのリファクタリングや長大な技術文献の要約が、画面上にわずか 1〜2 秒で一気に吐き出されます。

しかし、Cerebras の無料ポリシーや利用制限については誤った情報も多く見受けられます。本記事では、2026 年 10 月の最新エンジニアリング検証に基づき、無料枠の仕様とトラブル回避策を詳しく解説します。

---

## 1. 無料枠の真相：クレカ不要の「永久無料枠」は存在するのか？

過去のコミュニティ情報では「Cerebras はクレカ不要で毎日 100 万トークンが永久無料」と紹介されることがありました。**しかし、この旧制度は 2026 年 7 月 16 日をもって正式に終了しています。**

公式ドキュメントおよび利用規約の調査から判明した最新状況は以下の通りです：

1. **旧来の永久無料枠は廃止**：
   - 公式 FAQ には *"Is there a permanently free tier? No."*（永久無料枠はありますか？いいえ）と明記されています。
   - クレカ登録不要の 1M tokens/日 プランは 2026 年 7 月 16 日に終了しました。
2. **現在の仕様：$5 クレジットの試用プラン（クレジットカード認証必須）**：
   - 新規登録後、**有効な海外利用可能クレジットカード（Visa/MasterCard/Amex）を登録**することで、**$5 分の体験クレジット**が有効化されます。
   - カード登録は**不正利用防止の本人確認（KYC）**目的であり、試用期間中に無断で請求されることはありません。
3. **有効期間は 30 日間、枯渇時は HTTP 402 で安全停止**：
   - 付与された $5 クレジットは**カード認証から 30 日間有効**です。
   - トークンを使い切るか 30 日が経過すると、API は `HTTP 402 PaymentRequired` を返して自動停止します。
   - **自動課金リスクなし**：コンソールで手動で有料枠を購入しない限り、登録カードから従量課金が引き落とされることは一切ありません。

---

## 2. ウエハー規模超並列計算：なぜ ~3,000 tokens/s が可能なのか？

Cerebras の圧倒的な生成速度は、独自の **Wafer-Scale Engine 3 (WSE-3)** ハードウェアに起因します：

- **300mm シリコンウエハー全体が 1 つの巨大チップ**：従来の GPU（NVIDIA H100 等）は露光装置の制約からウエハーを数百枚のチップに切断しますが、Cerebras はウエハー全体を単一プロセッサとして製造。90 万個の AI コアと 44GB の超巨大オンチップ SRAM を搭載しています。
- **高遅延な外部 HBM メモリ通信が不要**：重みパラメータとキャッシュがすべてオンチップ SRAM 上に保持されるため、毎秒数ペタバイトの超広帯域で遅延が完全に排除されています。

### 現在利用可能な対象モデル

モデルラインナップは定期的に改編されており、旧来の Llama 3.1 8B/70B や Qwen-3-235B はすでに提供を終了しています。**現在の Free Trial では以下の 2 モデルが稼働中です：**

| モデル ID | モデル特徴 | コンテキスト | 推論速度 | 価格 ($/M tokens) |
| :--- | :--- | :---: | :---: | :---: |
| **`gpt-oss-120b`** | OpenAI 構造のオープン主力モデル、最高峰の推論能力 | 128K | **~3,000 tokens/s** | 入力 $0.35 / 出力 $0.75 |
| **`qwen-3.8-27b`** | 通義千問 27B 高効率モデル | 128K | **~1,800 tokens/s** | 入力 $0.99 / 出力 $1.49 |

---

## 3. 開発時の 5 大注意点（Gotchas）

Cerebras Cloud をシステムに組み込む際は、以下のポイントにご注意ください：

### 1. ⚠️ クレジットカード登録が必須
大手無料推論プロバイダーの中で、Cerebras は現在唯一クレジットカード認証を必須としています。カードがない場合は API Key の発行ができません。

### 2. ⚠️ 無料枠のレート制限は極めて厳格 (5 RPM)
Free Trial は非常に強い流量制限がかけられています：
- **RPM 制限**：1 モデルあたり **毎分 5 リクエスト** のみ；
- **Uncached TPM**：プロンプト新規計算は **毎分 30,000 トークン** まで；
- **Total TPM**：合計スループットは **毎分 90,000 トークン**；
- **推奨用途**：単発の長文生成、ドキュメント解析、速度ベンチマークに最適。**連続会話やループ型 Agent には不向き**です。

### 3. 🚦 明確な HTTP エラーコード：429 と 402
- **レート制限超過**：`HTTP 429 Too Many Requests` が返され、メッセージ内で超過理由（`uncached_tpm` または `requests_per_minute`）が通知されます。
- **残高枯渇または期限切れ**：`HTTP 402 PaymentRequired` で遮断され、勝手な課金は発生しません。

### 4. 🌐 CORS は完全開放（ブラウザ直結可能）
嬉しい特徴として、Cerebras API はレスポンスヘッダーに以下を出力します：
```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: DELETE, GET, HEAD, OPTIONS, PATCH, POST, PUT
Access-Control-Allow-Headers: authorization, content-type
```
これにより、フロントエンド JavaScript からバックエンドのプロキシなしで直接 `fetch` 呼び出しが可能です。

### 5. 🌍 海外ネットワーク要件
API エンドポイント（`https://api.cerebras.ai/v1`）は北米インフラ上で動作しています。日本国内からは安定した通信が可能です。

---

## 4. 接続コード例（OpenAI SDK 完全互換）

Cerebras は標準的な OpenAI 規格に準拠しているため、エンドポイントと API Key を書き換えるだけでそのまま利用できます：

### Python (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.cerebras.ai/v1",
    api_key="csk-xxxxxxxxxxxxxxxxxxxxxxxx"
)

response = client.chat.completions.create(
    model="gpt-oss-120b",
    messages=[
        {"role": "user", "content": "ウエハー規模スーパーチップについてわかりやすく解説してください。"}
    ],
    max_tokens=512,
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### ブラウザ JavaScript（CORS 直結）

```javascript
async function callCerebras(prompt) {
  const res = await fetch("https://api.cerebras.ai/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer csk-xxxxxxxxxxxxxxxxxxxxxxxx"
    },
    body: JSON.stringify({
      model: "gpt-oss-120b",
      messages: [{ role: "user", content: prompt }],
      max_tokens: 300
    })
  });
  const data = await res.json();
  console.log(data.choices[0].message.content);
}
```

---

## 5. まとめと推奨シナリオ

| 比較軸 | 評価 | 活用アドバイス |
| :--- | :---: | :--- |
| **出力速度** | 👑 **世界最高速** | 3,000 tokens/s、文字通り一瞬で長文が出力される圧倒的な体験 |
| **参加ハードル** | ⚠️ **クレカ必須** | カードをお持ちでない場合は Groq や SiliconFlow が代替候補 |
| **提供期間** | ⚠️ **初回30日限定** | 本番日常利用ではなく、速度検証やプロトタイプ開発向け |
| **並行性能** | ⚠️ **5 RPM** | リクエストキューによる流量制御が必須 |
| **請求リスク** | 🟢 **安全遮断** | 402 エラーで確実に停止し、自動請求は発生しません |

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/cerebras-cloud-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


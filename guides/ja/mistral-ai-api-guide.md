---
title: "Mistral AI 完全実践ガイド：欧州オープンソース最高峰・クレカ不要Experiment枠・2026年10月改定の真相とCORS直結対策"
description: "欧州トップAIユニコーン Mistral AI の開発者向け無料環境を徹底解説！2026年10月に価格表から消えた「$10枠」の真相、クレジットカード不要のExperiment無料実験枠、ブラウザから直接FetchできるネイティブCORS開放の実測検証、Codestralコード特化モデルと上位モデル課金制限を完全網羅。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外受限"
tags: ["Mistral AI", "Codestral", "Mistral Small", "欧州オープンソース", "永久無料", "CORS開放", "OpenAI互換"]
featured: false
---

# Mistral AI：欧州オープンソース最高峰、クレカ不要Experiment実験枠、2026年10月改定の真相＆実践エンジニアリングガイド

元Google DeepMindおよびMeta AIの第一線研究者たちによってフランス・パリで設立された **Mistral AI** は、欧州および世界を牽引する最高峰のAIユニコーン企業です。オープンソース界に衝撃を与えた Mistral 7B、Mixtral 8x7B から、プログラマー必携のコード特化モデル Codestral、最新のエッジ向け小型モデル Ministral に至るまで、高いパラメータ効率と厳格な論理的推論力で世界中のエンジニアから熱烈な支持を集めています。

開発者向け提供基盤（La Plateforme）において、Mistral はクレジットカード登録不要の **「Experiment Plan（実験プラン）」** をデフォルト提供しています。メールアドレスと携帯電話番号のSMS認証を済ませるだけで、即座にAPI Keyを発行して無料利用を開始できます。

しかし、2026年10月7日、Mistral は価格体系の静かな改訂を実施しました。**価格表から「毎月$10クレジット付与」の表記が削除され**、非公開のレート制限付き無料実験モードへと移行し、`mistral-large-4` などの最上位モデルは有料契約限定となりました。一方で、Mistral には他社にない強力な武器があります：**APIゲートウェイがネイティブにCORSクロスオリジンを開放している点です！** 本稿では、最新の仕様と実装上の注意点を詳しく解説します。

---

## 1. 2026年10月7日改定の真相と無料枠の仕様

インターネット上には「毎月$10の無料クレジットがもらえる」という古い情報が残っていますが、現在の公式仕様は以下の通りです：

### 1. 価格表の改定内容
- **旧ポリシー（2026年10月7日以前）**：無料枠に「$10/mo in API credits」と明記されていました。
- **現行ポリシー（2026年10月7日以降）**：該当表記が削除され、**「Experiment Free Mode」** へと再定義されました。残高表示ではなく、管理画面の「Limits」ページに基づき、約1〜2 RPM（秒間約1リクエスト）の実験的レート制限下で動作します。
- **無料は継続されているか？**：**はい、完全に無料です！** 有料のProプラン（月額$14.99）に自らアップグレードしない限り、クレジットカードの登録は不要であり、勝手な自動引き落としは絶対に発生しません。

### 2. 開発者無料実験枠の主要仕様

| 項目 | 実測状況と公式ルール | 開発者向けアドバイス |
| :--- | :--- | :--- |
| **無料タイプ** | ✅ **`rate_limit`（期限なし実験的レート制限無料枠）** | 期限なし、アクセス頻度を守れば恒久的に利用可能 |
| **レート制限 (RPM)** | **約1〜2 RPM**（約1 req/s） | 高頻度ポーリングには不向き、単発タスクやBot処理に最適 |
| **クレジットカード** | ❌ **不要** | メールアドレスとSMS認証のみで即利用可能 |
| **電話番号認証** | ✅ **必須（携帯SMS認証）** | 1つの電話番号につき1つのアカウントのみ有効 |
| **CORSヘッダー** | ✅ **ネイティブ開放 (`*`)** | 業界でも極めて希少！ブラウザJavaScriptから直接Fetch可能 |
| **OpenAI互換** | ✅ **完全準拠 (`/v1`)** | Base URL は `https://api.mistral.ai/v1` を指定 |
| **超過時の挙動** | `HTTP 429 Too Many Requests` を返却 | クレジットカード未登録のため追加課金リスク皆無 |

---

## 2. 提供モデル一覧：無料枠で動くもの・弾かれるもの

全モデルのAPI呼び出し実測により、無料枠における明確な階層分離が確認されています：

| モデル ID (`modelId`) | 用途と特徴 | 無料枠の利用可否 | 特徴とおすすめシナリオ |
| :--- | :---: | :---: | :--- |
| **`mistral-small-latest`** | 汎用対話 | 🟢 **利用可能 (主推)** | 知能密度が高く応答が迅速。無料枠の基本モデルとして最適 |
| **`codestral-latest`** | プログラミング特化 | 🟢 **利用可能 (主力)** | 80以上のプログラミング言語に対応。コード補完の世界的標準 |
| **`open-mistral-nemo`** | 共同開発モデル | 🟢 **利用可能** | NVIDIAと共同開発された12Bモデル。バランスが良く出力が安定 |
| **`ministral-8b-latest`** | エッジ超小型 | 🟢 **利用可能** | 最新のエッジ向け小型モデル。初回トークン生成が極めて高速 |
| **`mistral-large-4`** | 商用フラッグシップ | 🔴 **利用不可 (有料限定)** | **返却エラー**：`"This model is not available in your subscription tier"` |

> ⚠️ **注意**：無料Keyで `mistral-large-4` などの上位モデルを呼び出すと、即座にサブスクリプションエラーが発生します。無料枠では必ず `mistral-small-latest` または `codestral-latest` を指定してください。

---

## 3. 実践エンジニアリング注意点（Gotchas）

### 1. ⚠️ ネイティブCORS開放（フロントエンド開発者への最大の恩恵）
OpenAIやAnthropicなどのAPIはCORSヘッダーを返さないため、ブラウザから直接呼び出すとセキュリティエラーになります。
- **実測結果**：Mistral APIはOPTIONSプリフライトリクエストに以下を返却します：
  ```http
  access-control-allow-origin: *
  access-control-allow-headers: Authorization,Content-Type,User-Agent,Cache-Control,Keep-Alive,X-Mistral-User-Agent
  access-control-allow-methods: GET,HEAD,PUT,PATCH,POST,OPTIONS,DELETE
  ```
- **メリット**：WebフロントエンドやChrome拡張機能、ユーザースクリプトから、**Node.js/Pythonなどの中継サーバーを一切立てずに直接 `fetch()` できます！**

### 2. ⚠️ 携帯電話番号のSMS認証が必須
La Plateformeへの登録時、SMSコードによる本人確認が必須です。一部の国際番号や回線では認証コードの受信に遅延が発生する場合があります。

### 3. ⚠️ 無料実験枠のリクエストデータはモデル学習に使用される可能性あり
利用規約には以下の記述があります：
> *"API requests made under the Experiment plan may be used to train Mistral's models."*
- 機密コードや個人情報を扱う場合は、無料キーでの送信を避け、有料Proプラン（Zero Data Retention保証）を利用するか、ローカル環境でオープンソース版をセルフホストしてください。

---

## 4. 実装サンプルコード

### 1. ブラウザJavaScriptでの直接Fetch（中継サーバー不要）

```html
<script>
async function askMistral(apiKey, prompt) {
  const res = await fetch("https://api.mistral.ai/v1/chat/completions", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      model: "mistral-small-latest",
      messages: [{ role: "user", content: prompt }],
      max_tokens: 150
    })
  });
  const data = await res.json();
  console.log("Mistralの回答:", data.choices[0].message.content);
}
</script>
```

### 2. Python OpenAI SDK での標準利用

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ.get("MISTRAL_API_KEY", "YOUR_API_KEY"),
    base_url="https://api.mistral.ai/v1"
)

response = client.chat.completions.create(
    model="codestral-latest",
    messages=[
        {"role": "system", "content": "あなたは優秀なPythonエンジニアです。"},
        {"role": "user", "content": "型ヒント付きのフィボナッチ数列ジェネレータ関数を書いてください。"}
    ],
    temperature=0.2
)

print(response.choices[0].message.content)
```

---

## 5. まとめ

Mistral AI は、以下の強みを持つ極めてユニークな選択肢です：
1. **フロントエンド直結対応**：ネイティブCORS開放により、サーバーレスWebアプリや拡張機能の開発に最適；
2. **コード特化の圧倒的実力**：`codestral-latest` はIDEプラグイン環境で最高のコード補完を提供；
3. **完全クレカ不要**：制限超過時もHTTP 429が返るのみで、想定外の課金リスクは皆無。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/mistral-ai-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。

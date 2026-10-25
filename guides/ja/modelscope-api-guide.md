---
title: "ModelScope（魔搭）完全実測ガイド：毎日250魔粒無料枠、900+オープンモデル直結・クレカ不要＆CORS対応"
description: "アリババ傘下の中国最大オープンソースコミュニティ「ModelScope（魔搭）」無料Serverless推論APIを徹底検証！毎日250魔粒（約125〜500回呼出）、Qwen通義千問・DeepSeek-V3/R1・智譜GLMなど900+モデル対応、ネイティブCORS直接呼出、実名認証条件と5大落とし穴を解説。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "国内直连"
tags: ["ModelScope", "アリババ", "Qwen", "DeepSeek", "オープンソース", "CORS対応", "毎日更新", "クレカ不要"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/modelscope-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます。


# ModelScope（魔搭）完全実測ガイド：毎日250魔粒無料枠、900+オープンモデル直結・クレカ不要＆CORS対応

中国のオープンソースAI生態系において、アリババクラウドが主導する **ModelScope（魔搭コミュニティ）** は、国内最大規模を誇るオープンモデルプラットフォームです。アリババ自社開発の通義千問（Qwen）全シリーズはもちろん、世界的話題を呼んだ DeepSeek-V3 / R1、智譜 GLM、百川、MiniMax など、数千の優れたモデルウェイトが結集しています。

開発者にとって特に魅力的なのが、コミュニティが提供する **Serverless API 推論の恒久無料枠** です：
- ❌ **クレジットカードや海外決済の登録は一切不要**
- ❌ **前払いチャージやデポジットも不要**
- ✅ **アリババ CDN 直結による極低遅延ストリーミングレスポンス**
- ✅ **毎日 250 魔粒を無料配布（約 125 〜 500 回の推論呼び出しが可能）**
- ✅ **API ゲートウェイがネイティブで CORS プリフライトに対応（ブラウザから直接 fetch 可能）**

本記事では、ModelScope 無料枠の算出ルール、魔粒の消費モデル、対応ラインナップ、および 5 大重要注意点を詳細に解説します。

---

## 1. コアポリシーと水桶容量：毎日循環する「魔粒」システム

ModelScope は **「魔粒（MoLi）」** と呼ばれるプラットフォーム共通通貨を採用しています：

| 評価項目 | 実測ステータスと公式規約 | 要点と推奨事項 |
| :--- | :--- | :--- |
| **無料枠タイプ** | ✅ **`daily_refresh`（毎日自動リセット）** | 毎日日本時間 01:00（北京時間 00:00）に自動更新 |
| **日次無料枠** | **250 魔粒 / 日** | 日次ログインで 200 魔粒 ＋ 実名認証済みアリババ口座連携で ＋50 魔粒 |
| **推定呼出回数** | **約 125 〜 500 回 / 日** | 利用するモデルの規模ティアに応じて変動 |
| **有効期限ルール** | **短期魔粒は 24 時間で失効** | 毎日付与される 250 魔粒は当日限り有効（翌日繰越不可）。新規登録時に別途 50 長期魔粒（90日有効）が付与 |
| **クレジットカード** | ❌ **完全不要** | 意図しない自動引き落としリスクは完全ゼロ |
| **実名認証条件** | ⚠️ **必須条件** | 中国身分証明書で実名認証を完了したアリババクラウド口座（Alipay認証）の連携が必要 |
| **CORS 跨域** | 🟢 **ネイティブ開放 (`*`)** | OPTIONS プリフライトで `access-control-allow-origin: *` を返却、SPAから直結可能 |
| **OpenAI 互換性** | 🟢 **完全互換** | 標準エンドポイント `https://api-inference.modelscope.cn/v1` |
| **超過時レスポンス** | `HTTP 429 Too Many Requests` | 枠終了時は安全に 429 が返却され、請求は発生しない |

---

## 2. モデル規模別の魔粒消費ティア

魔粒の消費レートは、モデルのパラメータ規模と計算コストに応じて 3 つのティアに分かれています：

### 1. 軽量モデルティア：0.5 魔粒 / 回（1日約 500 回呼出可能）
- **代表モデル**：`Qwen/Qwen2.5-7B-Instruct`、`Qwen/Qwen2.5-0.5B`、`Qwen/Qwen2.5-1.5B`
- **推奨用途**：日常的な分類・データ構造化、軽量コード補完、意図判定、シンプルなマルチターン会話。

### 2. 主流モデルティア：1.0 魔粒 / 回（1日約 250 回呼出可能）
- **代表モデル**：`deepseek-ai/DeepSeek-V3`、`deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`、`ZhipuAI/GLM-4-9B-Chat`
- **推奨用途**：深い論理的思考、複雑な長文要約、高精度コード生成、社内ナレッジベース RAG。

### 3. 旗艦フルスペックティア：2.0 魔粒 / 回（1日約 125 回呼出可能）
- **代表モデル**：`Qwen/Qwen2.5-72B-Instruct` などの 70B+ 級大型モデル
- **推奨用途**：高度なエージェント推論、複雑な多言語翻訳、難解な学術・ベンチマーク評価。

---

## 3. 実装における 5 大落とし穴 (Gotchas)

### 1. ⚠️ アリババクラウド実名認証（海外ユーザー最大の壁）
ModelScope API 推論は中国国内の法規制に準拠して運用されています。アリババクラウド口座の連携が必須であり、その口座は中国本土の身分証明書（Alipay経由）で実名認証されている必要があります。**中国本土の身分証を持たない海外ユーザーは利用できません。**

### 2. ⚠️ 短期魔粒は 24 時間で失効（繰り越し不可）
毎日付与される 250 魔粒は「短期魔粒」であり、毎日 00:00 (UTC+8) に失効・再付与されます。翌日に貯めることはできないため、当日のうちに積極的に消費して検証するのがベストです。

### 3. ⚠️ ネイティブ CORS 対応（フロントエンド直結が可能）
国内クラウドサービスの多くがブラウザからの CORS を遮断しているのに対し、ModelScope の推論ゲートウェイは OPTIONS プリフライトに `access-control-allow-origin: *` を返却します。バックエンドの中継サーバーを立てることなく、Vue/React SPA やブラウザ拡張機能からネイティブな `fetch()` で直接通信できます。

### 4. ⚠️ 商用本番環境への投入禁止＆動的負荷制限
公式利用規約において、API 推論無料枠は学習・プロトタイピング・評価専用と明記されており、**SLA 保証は提供されません**。コミュニティ全体の負荷状況に応じて動的にレート制限が課されるため、ピーク時には `HTTP 429` が返る頻度が高くなります。

### 5. ⚠️ モデル指定時は組織プレフィックスを含む完全IDが必須
OpenAI クライアント等から呼び出す際は、`model` パラメータに組織プレフィックスを必ず含める必要があります：
- 誤り：`model: "Qwen2.5-72B-Instruct"` ❌
- 正しい指定：`model: "Qwen/Qwen2.5-72B-Instruct"` ✅
- 正しい指定：`model: "deepseek-ai/DeepSeek-V3"` ✅

---

## 4. サンプルコード

### 1. Python (OpenAI SDK 完全互換)

```python
import os
from openai import OpenAI

# modelscope.cn/apikey で取得した Access Token を指定
client = OpenAI(
    api_key=os.environ.get("MODELSCOPE_API_KEY", "your_modelscope_access_token"),
    base_url="https://api-inference.modelscope.cn/v1"
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-72B-Instruct",
    messages=[
        {"role": "system", "content": "あなたはModelScopeで提供されているQwen 2.5 72Bモデルです。"},
        {"role": "user", "content": "ModelScope API推論の主なメリットを箇条書きで3点教えてください。"}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

### 2. cURL コマンドライン実行

```bash
curl -X POST "https://api-inference.modelscope.cn/v1/chat/completions" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1-Distill-Qwen-32B",
    "messages": [{"role": "user", "content": "9.11 と 9.8 はどちらが大きいですか？"}],
    "temperature": 0.6
  }'
```

---

## 5. 3 ステップ導入手順

1. **登録と口座連携**：[modelscope.cn](https://modelscope.cn/) で会員登録し、設定画面で実名認証済みアリババ口座を連携；
2. **トークンの取得**：[modelscope.cn/apikey](https://modelscope.cn/apikey) を開き、自動生成された Access Token をコピー；
3. **設定して即時利用**：Base URL を `https://api-inference.modelscope.cn/v1`、キー、モデルに `Qwen/Qwen2.5-72B-Instruct` を設定して毎日 250 魔粒の無料枠をお楽しみください！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/modelscope-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。

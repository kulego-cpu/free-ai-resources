---
title: "階躍星辰 StepFun API 完全実測ガイド：stepaudio-3 音声マルチモーダル期間限定無料、ワイルドカードCORS完全開放＆完全前払いゼロリスク解説"
description: "元マイクロソフト幹部が率いるAIユニコーン「階躍星辰（StepFun）」開発者プラットフォームを徹底解説！stepaudio-3 音声・音楽マルチモーダル4モデルの期間限定無料体験、希少なCORS完全開放（*）、圧倒的低価格 step-3.5-flash（$0.10/1M）、中国直結低遅延と5大注意点を網羅。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "国内直连"
tags: ["階躍星辰", "StepFun", "マルチモーダル", "期間限定無料", "CORS完全開放", "完全前払い", "中国直結"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/stepfun-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます。


# 階躍星辰 StepFun API 完全実測ガイド：stepaudio-3 音声マルチモーダル期間限定無料、ワイルドカードCORS完全開放＆完全前払いゼロリスク解説

元マイクロソフトグローバルパートナーおよび副社長を務めた姜大昕博士が率いる **階躍星辰（StepFun / 躍問）** は、自社開発の千億および万億パラメータ規模のMoE（Mixture-of-Experts）基盤モデルを武器に、中国のトップAIユニコーンの一角を占めています。

高度なテキスト推論や視覚的マルチモーダル理解に加え、StepFunは音声領域（リアルタイム音声対話、統一オーディオ生成、作詞作曲音楽生成）において際立った技術優位性を確立しています。

2026年10月の最新実測において、StepFun開発者プラットフォームを「5段階深層検証法」に基づき徹底検証したところ、開発者にとって極めて魅力的な事実が判明しました：
- 🎙️ **stepaudio-3 音声マルチモーダル全4モデルが期間限定無料**：`stepaudio-3-realtime-preview`、`stepaudio-3-chat-preview`、`stepaudio-3-gen-preview`、`stepaudio-3-music-preview` が、実名認証完了後に完全無償で呼び出し可能！
- 🌐 **極めて希少なCORSワイルドカード完全開放**：OPTIONSプリフライトで `Access-Control-Allow-Origin: *` が直接返却され、ブラウザフロントエンド（SPA）やWeb拡張機能からのダイレクトfetchが完全サポートされています（主要プロバイダー全20数社中わずか5社のみ）。
- 💰 **破壊的な推論コストパフォーマンス**：最新フラッグシップ `step-3.5-flash` の入力価格はわずか $0.10 / 1M tokens（キャッシュ時 $0.02 / 1M）と、同等クラスモデルの10分の1以下の超低価格を実現。
- 🔒 **厳格な完全前払い（Prepaid）設計**：残高や無料枠が枯渇すると `HTTP 402 Payment Required` で安全停止するため、**クレジットカードの勝手な自動課金や請求超過のリスクはゼロ**！

本記事では、StepFunの無料枠仕様、CORS設定、レート制限、そして5大注意事項を分かりやすく解説します。

---

## 1. コア仕様と5段階実測検証

| 評価項目 | 実測結果と公式ポリシー | 開発上の要点と推奨事項 |
| :--- | :--- | :--- |
| **無料枠タイプ** | 🟢 **`voucher`（期間限定無料モデル体験）** | stepaudio-3 音声・音楽4モデルが現在公式無料開放中 |
| **利用開始の要件** | 電話番号登録 ＋ **中国身分証による実名認証** | 個人アカウントは身分確認完了後にAPIキーの発行・利用が可能 |
| **無料枠の容量** | 期間限定無料のため期間中の呼び出し上限なし | ⚠️ 公式に「期間限定」と明記されており、早期の検証を推奨 |
| **CORSクロスドメイン**| 🟢 **完全開放（ワイルドカード `*`）** | `access-control-allow-origin: *`、SPAやブラウザ直結可能 |
| **ネットワークと接続**| 🇨🇳 **中国本土直結（火山引擎 WAF）** | `api.stepfun.com` は低遅延（約140ms）；国際版 `.ai` は独立運営 |
| **課金・資金安全性** | 🔒 **完全プリペイド（自動課金リスクなし）** | 残高枯渇時はクリーンに `HTTP 402` 返却、勝手な引き落としなし |
| **合規フィルター** | ⚠️ **`HTTP 451`（コンテンツ規約遮断）** | センシティブな入力・出力に対して451を返却。クライアント側で例外処理が必要 |

### CORS 実測エビデンス (cURL OPTIONS)

```bash
$ curl -sI -X OPTIONS 'https://api.stepfun.com/v1/chat/completions' \
    -H 'Origin: https://example.com' \
    -H 'Access-Control-Request-Method: POST' \
    -H 'Access-Control-Request-Headers: authorization,content-type'

HTTP/2 204
access-control-allow-origin: *
access-control-allow-credentials: true
access-control-allow-methods: GET, PUT, POST, DELETE, PATCH, OPTIONS
access-control-max-age: 86400
```

---

## 2. モデル一覧と料金体系

### 1. 期間限定無料モデル群（音声・音楽マルチモーダル）
実名認証完了後、以下の先端音声モデルを事前チャージなしで無償利用できます：
- **`stepaudio-3-realtime-preview`**：双方向リアルタイム対話を可能にする超低遅延エンドツーエンド音声推論モデル；
- **`stepaudio-3-chat-preview`**：自然な抑揚と豊かな感情表現を備えたマルチターン音声対話モデル；
- **`stepaudio-3-gen-preview`**：高品質な音声合成（TTS）と声質適応に対応した統一オーディオ生成モデル；
- **`stepaudio-3-music-preview`**：プロンプトからポップス、ジャズ、クラシックなどの完成楽曲を生成する最先端音楽AI。

### 2. 圧倒的コスパの商用モデル（前払い残高利用）
- **`step-3.5-flash`**（入力 $0.10 / 出力 $0.30 per 1M tokens、キャッシュ時 $0.02）：主流モデルの10分の1以下の価格で提供される高性能推論モデル。プログラミングや複雑推論で高い評価を獲得；
- **`step-3.7-flash` / `step-5-preview`**：長大な思考プロセス（CoT）を備えた階躍星辰の最新推論特化型モデル。

---

## 3. 5大エンジニアリング注意点 (Gotchas)

### 1. ⚠️ 「期間限定無料」は予告なく終了する可能性あり
stepaudio-3 シリーズの4モデルは、公式ドキュメントで「Free (limited time)」と指定されています。終了時期は公表されていないため、本番運用時は将来の有償化に備えたフォールバック処理を設計しておくことを推奨します。

### 2. ⚠️ テキスト・推論モデルはチャージが必要
「登録すれば全モデルが無料」と誤解して `step-3.5-flash` を呼び出すと `HTTP 402` が返却されます。無料枠は stepaudio-3 シリーズ限定であり、テキストモデルの利用にはコンソールからのチャージ（最低約$1）または Step Plan の購読が必要です。

### 3. ⚠️ 国内版ポータルは中国身分証の実名認証が必須
`platform.stepfun.com` でAPIキーを発行して利用するには、中国本土の個人身分証による認証が必須となります。

### 4. ⚠️ 国内ポータル (.com) と国際ポータル (.ai) は別システム
StepFunは2系統の独立したインフラを運用しています：
- 国内版：`https://api.stepfun.com/v1`（火山引擎 WAF 直結）
- 国際版：`https://api.stepfun.ai/v1`（Aliyun 海外 ALB 経由）
両ポータルはアカウントおよびAPIキーが完全に独立しており、相互利用はできません。

### 5. ⚠️ コンテンツ監査フィルターによる HTTP 451
StepFunは厳格な合規フィルタリングを導入しています。入力プロンプトやモデル生成内容が規約に抵触した場合、API は `HTTP 451 (Unavailable For Legal Reasons)` を返却します。UI上で適切なメッセージを表示できるようにハンドリングを実装してください。

---

## 4. クイックスタート実装コード

### 1. Python (OpenAI SDK準拠)

```python
import os
from openai import OpenAI

# platform.stepfun.com で発行した sk- キーを指定
client = OpenAI(
    api_key=os.environ.get("STEP_API_KEY", "your_stepfun_api_key"),
    base_url="https://api.stepfun.com/v1"
)

response = client.chat.completions.create(
    model="stepaudio-3-chat-preview",
    messages=[
        {"role": "system", "content": "あなたは階躍星辰の音声対話AIです。親しみやすい口調で回答してください。"},
        {"role": "user", "content": "あなたの音声マルチモーダル機能の強みを2行で教えてください。"}
    ],
    temperature=0.7
)

print(response.choices[0].message.content)
```

### 2. cURL コマンドによる疎通テスト

```bash
curl -X POST "https://api.stepfun.com/v1/chat/completions" \
  -H "Authorization: Bearer $STEP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "stepaudio-3-chat-preview",
    "messages": [{"role": "user", "content": "9.11 と 9.8 はどちらが大きいですか？"}],
    "temperature": 0.3
  }'
```

---

## 5. 簡単3ステップ利用手順

1. **登録と実名認証**：[platform.stepfun.com](https://platform.stepfun.com/) にアクセスして電話番号で登録し、個人センターで実名認証を完了します；
2. **API キーの発行**：「API Keys」画面で `sk-` から始まるキーを生成し、安全に保管します；
3. **設定して呼び出し**：Base URLを `https://api.stepfun.com/v1` に設定し、`stepaudio-3-chat-preview` を選択すれば、完全無料で最先端の音声マルチモーダルAIが即座に体験できます！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/stepfun-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。

---
title: "Pollinations AI 徹底解説：100% オープンソース多モーダル、登録不要のURL生図、毎週1.5 Pollen付与＆CORS完全開放"
description: "世界で最もオープンな多モーダルAIプラットフォーム Pollinations AI を徹底レビュー！登録不要で使える Free Classic 生図（1回/45秒）・テキスト生成（1回/30秒）、HTTP 402 Retry-After 待機設計、登録で毎週もらえる 1.5 Pollen、16種以上の 0-Pollen 無料モデル、CORS開放（*）と実装手順を解説。"
pubDate: 2026-10-19
author: "FreeAI Hub 編集部"
category: "海外回線"
tags: ["Pollinations AI", "登録不要", "FLUX生図", "オープンソース", "Pollenクレジット", "多モーダル", "CORS開放", "海外回線"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/pollinations-api-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# Pollinations AI 徹底解説：100% オープンソース多モーダル、登録不要のURL生図、毎週1.5 Pollen付与＆CORS完全開放

数多くの商用大模型や推論APIプロバイダーが存在する中で、**Pollinations AI（pollinations.ai）** は極めて異彩を放つオープンソース精神を貫いています。コードリポジトリ、インフラ構成、開発ロードマップ、さらには開発チームの日常的な技術ディスカッションに至るまで、そのすべてが GitHub と Discord 上で 100% 公開されています。

現在、世界中で 500 以上の独立系アプリケーションが Pollinations をバックエンドに採用しており、開発者から絶大な支持を集める理由は以下の通りです：
- 🎨 **Free Classic API による完全登録不要・キー不要の生図**：アカウント登録も API Key も一切不要。ブラウザや HTML の `<img>` タグから直接 URL に GET リクエストを送るだけで、FLUX による高精細画像を数秒で取得可能
- ⏱️ **Retry-After 付きの明瞭なレートリミット**：IP ごとに画像 1回/45秒、テキスト 1回/30秒に制限。連打時は HTTP 402 とともに再試行までの待機秒数を返却
- 🎁 **登録で毎週 1.5 Pollen の無料クレジット付与**：`enter.pollinations.ai` でメール登録するだけで毎週 1.5 ポイントが自動補充され、16 種類以上の 0-Pollen 完全無料モデル（gpt-4o、gemini-3.5-flash-lite、deepseek-v3.2 など）を呼び出し可能
- 🏆 **コミュニティ貢献で計算資源を稼ぐ Pollen Quests**：公式 Discord 参加で 1 Pollen、GitHub PR マージで 5 Pollen が付与される開発者還元システム
- 🌐 **CORS 完全開放（ワイルドカード `*`）**：全エンドポイントで `access-control-allow-origin: *` を返却するため、フロントエンド完結の Web アプリからプロキシなしで直接利用可能

本記事では、Pollinations AI のアーキテクチャ構造、登録不要の生図手法、OpenAI 規格での実装例、および 5 つの重要な注意点を詳しく解説します。

---

## 1. スペックと主要ポリシー一覧

| 評価項目 | 公式仕様と実測データ | 開発上の要点とアドバイス |
| :--- | :--- | :--- |
| **無料枠アーキテクチャ** | 🟢 **2段階構成（Classic 登録不要 + 登録 Pollen 枠）** | Classic は登録不要、登録アカウントは毎週 1.5 Pollen 付与 |
| **画像生成制限** | ⏱️ **1回 / 45秒（IP ごと）** | Classic 登録不要生図（FLUX モデル）に適用 |
| **テキスト生成制限** | ⏱️ **1回 / 30秒（IP ごと）** | Classic 登録不要テキスト推論に適用 |
| **上限超過時の挙動** | 🛡️ **HTTP 402 + `Retry-After: 45` ヘッダー** | 次回可能までの秒数を明示、**自動課金リスクはゼロ** |
| **クレジットカード** | 🟢 **完全不要（0円利用可能）** | メールアドレスのみで登録可能、クレカ登録なし |
| **CORS 跨域** | 🟢 **完全開放（ワイルドカード `*`）** | `access-control-allow-origin: *`、ブラウザ直結対応 |
| **OpenAI 互換規格** | `https://gen.pollinations.ai/v1` | `/v1/chat/completions` および `/v1/models` に対応 |
| **回線状況** | 🌐 **海外回線（Cloudflare）** | 日本国内から直接高速アクセス可能 |

---

## 2. Classic API：登録不要・URL だけで呼び出せる画像生成

Pollinations AI 最大の魅力は、トークン不要の URL コールです：

### 1. ブラウザ URL から直接 FLUX 画像生成

URL のパスにプロンプトを入力するだけで、バイナリ画像（JPEG/PNG）が即座に返却されます：

```bash
# 1. デフォルト FLUX 高精細レンダリング
https://image.pollinations.ai/prompt/a%20cute%20cyberpunk%20cat%20in%20neon%20city?model=flux&width=1024&height=1024

# 2. アートスタイル（zimage）
https://image.pollinations.ai/prompt/sunset%20over%20the%20ocean?model=zimage

# 3. 軽量アイデア出し（klein）
https://image.pollinations.ai/prompt/futuristic%20sports%20car?model=klein
```

HTML の画像タグにそのまま埋め込むことも可能です：
```html
<img src="https://image.pollinations.ai/prompt/a%20golden%20retriever%20running%20on%20grass?width=800&height=600" alt="Generated Image" />
```

### 2. 登録不要のテキスト生成

```bash
curl -s "https://text.pollinations.ai/Hello!%20Introduce%20Pollinations%20AI"
```

---

## 3. 登録アカウントと統一ゲートウェイ（Pollen 体系）

本格的なチャットボットや並行処理を行う場合は、`https://enter.pollinations.ai` で登録して専用キーを発行します：
- **`pk_...`（Publishable Key）**：フロントエンド公開用（1 pollen/IP/hour の保護付き）
- **`sk_...`（Secret Key）**：バックエンドサーバー用（並行リクエスト対応）

### 1. 0 Pollen で使える無料モデル群

毎週自動付与される 1.5 Pollen を使わずに、以下のモデルは **0 Pollen（完全無料）** で利用可能です：
- **テキストモデル**：`openai/gpt-4o`、`openai/gpt-4o-mini`、`google/gemini-3.5-flash-lite`、`deepseek/deepseek-v3.2`、`qwen/qwen3-coder`、`x-ai/grok-4.7`
- **画像モデル**：`black-forest-labs/flux.1-schnell`、`seedream`、`kontext`
- **音声・3D**：`tts`（テキスト読み上げ）、`microsoft/trellis-2`（3D メッシュ生成）

### 2. Python による実装例

```python
import os
from openai import OpenAI

# 統一ゲートウェイを指定してクライアント初期化
client = OpenAI(
    base_url="https://gen.pollinations.ai/v1",
    api_key=os.environ.get("POLLINATIONS_API_KEY")  # enter.pollinations.ai で取得した pk_ または sk_
)

# 0-Pollen の無料フラッグシップモデルを呼び出し
response = client.chat.completions.create(
    model="openai/gpt-4o",
    messages=[
        {"role": "system", "content": "あなたは優秀なフロントエンドエンジニアです。"},
        {"role": "user", "content": "Web Componentsのメリットを3点で教えてください。"}
    ],
    temperature=0.7,
    max_tokens=512,
    stream=True
)

for chunk in response:
    content = chunk.choices[0].delta.content or ""
    print(content, end="", flush=True)
```

---

## 4. 開発における 5 つの重要な注意点（Gotchas）

1. **HTTP 402 と `Retry-After` ヘッダーの制御**：
   Classic 生図は 1回/45秒の制限があります。連打した場合は HTTP 402 が返却されるため、レスポンスヘッダーの `Retry-After: 45` を読み取って待機時間を制御してください。
2. **ウォーターマークの解除**：
   Classic 登録不要モードの画像には薄いロゴが入る場合があります。無料登録して Token を付与してリクエストすることで無透かしの画像が得られます。
3. **`pk_` と `sk_` キーの適切な使い分け**：
   React や Vue などのクライアント側には必ず `pk_` キーを配置し、サーバーサイドでのみ `sk_` を使用してください。
4. **Pollen Quests による無料算力の獲得**：
   Discord への参加（1 Pollen）や GitHub への PR マージ（5 Pollen）により、開発貢献を通じて計算クレジットを追加獲得できます。
5. **CORS 完全開放のメリット**：
   すべてのエンドポイントで `access-control-allow-origin: *` が有効化されているため、フロントエンド完結の SPA から直接 `fetch()` できます。

---

## 5. まとめ

Pollinations AI は、オープンソース愛好者にとって最高のツールキットです：
* **プロトタイプやドキュメント用画像**：登録不要の URL 直打ちで即座に FLUX 生図；
* **本格的な多モーダルアプリ**：無料登録で毎週 1.5 Pollen と GPT-4o や DeepSeek を完全無料で活用しましょう！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/pollinations-api-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


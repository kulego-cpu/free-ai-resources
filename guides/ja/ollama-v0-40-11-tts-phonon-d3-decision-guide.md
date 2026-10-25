---
title: "1つのCLIで対話・意図決定・音声合成が完結！Ollama v0.40.11 実践レビュー：内蔵 Phonon TTS と d3 ルーティングモデル活用ガイド"
description: "Ollama が歴史的な進化を遂げました！v0.40.11 で macOS ネイティブ対応の Phonon 音声合成（OpenAI /v1/audio/speech 互換）と、vLLM Semantic Router 由来の d3 決定モデルファミリー（2B/4B/9B/27B）が同時リリース。複数サービスを組み合わせる手間を解消し、完全ローカルで動く音声エージェントの構築手順と Python 実装コードを徹底解説します。"
pubDate: 2026-10-25
author: "Kole (FreeAI Dog 編集部)"
category: "実践・応用ガイド"
tags: ["Ollama", "ローカルLLM", "TTS音声合成", "d3決定モデル", "AIエージェント", "Phonon", "OpenAI互換API"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/ollama-v0-40-11-tts-phonon-d3-decision-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# 1つのCLIで対話・意図決定・音声合成が完結！Ollama v0.40.11 実践レビュー：内蔵 Phonon TTS と d3 ルーティングモデル活用ガイド

> **はじめに**：ローカル環境で AI エージェントや音声アシスタントを構築した経験がある方なら、これまでの複雑なシステム構成に頭を悩ませたことがあるはずです。  
> - **音声を出力したい（TTS）**：Python 仮想環境を立ち上げ、Kokoro、ChatTTS、Kitten-TTS などを別途動かす必要があり、PyTorch のバージョン競合や 2GB〜4GB の追加 VRAM 消費が避けられませんでした。  
> - **意図分類やツール呼び出しの振り分け（ルーティング）**：「天気を調べる質問か？」を判定するためだけに 32B や 70B の巨大モデルを呼び出すと、応答に 1〜2 秒かかり GPU リソースも激しく浪費します。かといって正規表現では柔軟性がありません。  
> - **API 連携**：別々のコンテナを Fast API 等で束ねるコードを書く手間がかかっていました。
>
> 2026年10月24日、ローカル LLM のデファクトスタンダードである **Ollama の v0.40.11** が正式リリースされました。今回のアップデートは、単なる「対話モデル実行ツール」の枠を超え、**【LLM 思考】＋【d3 によるミリ秒単位の意思決定】＋【Phonon によるネイティブ音声合成】が1つに統合されたローカル AI プラットフォーム**へと大きく進化しました。
>
> 本記事では、環境のアップデート手順からコマンドライン操作、OpenAI 互換エンドポイントの呼び出し、Python による音声エージェント実装まで、実践に直結する手順を詳しく解説します。

---

## 1. Ollama v0.40.11 がもたらすローカル AI の統合

従来のローカル音声アシスタント構成は、極めて分断されていました：

```
[ユーザーの入力] 
   │
   ├─► 手順 1 (意図判定): 重い LLM またはルールベースのスクリプト
   ├─► 手順 2 (回答生成): Ollama (Qwen / Llama)
   └─► 手順 3 (音声合成): 独立した TTS サーバー (Kokoro / Edge-TTS)
```

3 つの独立プロセスを管理するため、ポートの競合やメモリの断片化、応答遅延が大きな課題でした。

**Ollama v0.40.11 は、このアーキテクチャを単一バイナリで完結させます**：
1. **内蔵 Phonon 音声合成エンジン（macOS ネイティブ）**：外部依存ゼロで、`ollama run phonon "こんにちは"` と打つだけでスピーカーから自然な音声が流れます。バックグラウンドサービスは OpenAI 規格の `/v1/audio/speech` をネイティブ公開します。
2. **公式 d3 決定モデルファミリー（2B / 4B / 9B / 27B）**：vLLM の Semantic Router 開発チームが手がけた超軽量の意思決定特化モデルです。長文生成ではなく**意図分類、True/False 判定、安全ガードレール判定**に特化しており、わずか **20〜30 ミリ秒**、約 1.5GB の VRAM で高速動作します。

`ollama serve` を 1 つ立ち上げるだけで、自律エージェントの基盤がすべて揃います。

---

## 2. アップデート手順と環境確認

現在の Ollama バージョンを確認します：

```bash
ollama --version
```

`v0.40.10` 以前の場合は、以下の方法ですぐにアップデートできます：

- **macOS（Homebrew）**：
  ```bash
  brew update && brew upgrade ollama
  ```
- **macOS（公式アプリ版）**：
  メニューバーの Ollama アイコン -> `Restart to update` をクリック、または公式サイトから最新版を再ダウンロードして上書きします。
- **Linux**：
  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ```

アップデート後、バージョンを確認します：
```bash
ollama version is 0.40.11
```

---

## 3. 実践①：Phonon 音声合成で直接喋らせる

### 1. コマンドラインから直接発声

ターミナルで `phonon` モデルを取得して直接実行します：

```bash
ollama run phonon "こんにちは。Ollama のネイティブ音声合成へようこそ！"
```

追加の Python 環境やブラウザを開くことなく、Mac のスピーカーから低遅延で自然な音声が出力されます。

### 2. パイプ連携：LLM の回答をその場で読み上げる

Unix パイプを利用して、対話モデルの回答をリアルタイムに `phonon` に渡すことができます：

```bash
ollama run deepseek-r1:8b "プログラマーのコーヒーに関するジョークを2行で教えて" | ollama run phonon
```

DeepSeek-R1 がテキストを生成し終わった瞬間に Phonon が音声ストリーミングを開始します。全体の遅延は 1 秒未満に収まります。

### 3. アプリ連携：OpenAI 互換 `/v1/audio/speech` エンドポイント

Ollama の HTTP サービス（デフォルト `http://localhost:11434`）は、OpenAI 規格の TTS リクエストをそのまま受け付けます：

```bash
curl http://localhost:11434/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "phonon",
    "input": "本日の探活により、新たに59個の先端モデルが探索プールに登録されました。",
    "voice": "alloy",
    "response_format": "mp3"
  }' \
  --output ./test_voice.mp3
```

生成された `test_voice.mp3` は、Web アプリやデスクトップアプリからすぐに再生可能です。

---

## 4. 実践②：d3 決定モデルによる高速意図ルーティング

### 1. d3:2b の取得

マシンのスペックに合わせてモデルサイズを選択できます：
- **d3:2b**（約 1.6GB：一般的なノートPCや省メモリ環境に最適）
- **d3:4b**（約 2.8GB：複雑な複数カテゴリ分類向け）
- **d3:9b / d3:27b**（厳格な企業向けコンプライアンス判定向け）

今回は実用性の高い 2B モデルを取得します：

```bash
ollama pull d3:2b
```

### 2. 活用例 1：Yes/No 判定（Boolean Guardrail）

サポート対応での意図判定テスト：

```bash
ollama run d3:2b "A customer says their credit card was charged twice for order 8848." \
  --ask "Is this a billing issue?"
```

**約 30 ミリ秒で出力される結果：**
```json
{
  "result": true,
  "confidence": 0.982,
  "reasoning": "The query explicitly describes duplicate charging on a credit card, which is a financial/billing transaction dispute."
}
```

マークダウンの構文解析を行わなくても、`response["result"]` でそのままコードの条件分岐が記述できます。

### 3. 活用例 2：複数カテゴリへの意図振り分け

```bash
ollama run d3:2b "ユーザーから、Mac版クライアントでログイン時に403エラーが出るとの報告がありました。" \
  --ask "Classify into: [technical_bug, billing, sales_inquiry, general_feedback]"
```

**出力結果：**
```text
Classification: technical_bug
Confidence: 0.96
```

### 4. 活用例 3：ツール呼び出しの安全ガードレール

```bash
ollama run d3:2b "rm -rf /Users/test/temp_cache/*" \
  --ask "Is this command potentially destructive to user data?"
```

破壊的なファイル削除コマンドを瞬時に検知し、エージェントの暴走を防ぐことができます。

---

## 5. 10分で構築：完全ローカルで動くスマート音声アシスタント（Python）

```python
import requests
import json
import subprocess

OLLAMA_BASE = "http://localhost:11434"

def classify_intent(user_prompt: str) -> str:
    """ステップ1: d3:2b を使い 30ms 以内で意図を分類"""
    payload = {
        "model": "d3:2b",
        "prompt": f"Classify the following text into one of [weather, system_status, general_chat]: '{user_prompt}'",
        "stream": False
    }
    res = requests.post(f"{OLLAMA_BASE}/api/generate", json=payload).json()
    return res.get("response", "").strip().lower()

def speak_text(text: str):
    """ステップ2: Ollama の /v1/audio/speech で音声を合成・再生"""
    tts_payload = {
        "model": "phonon",
        "input": text,
        "voice": "alloy",
        "response_format": "wav"
    }
    audio_res = requests.post(f"{OLLAMA_BASE}/v1/audio/speech", json=tts_payload)
    
    with open("/tmp/assistant_reply.wav", "wb") as f:
        f.write(audio_res.content)
    
    subprocess.run(["afplay", "/tmp/assistant_reply.wav"])

def main():
    test_queries = [
        "明日の東京は傘が必要ですか？",
        "本日レーダーで検出された新着モデルを教えてください",
        "キーボードに関する面白い豆知識を教えて"
    ]

    for q in test_queries:
        print(f"\n👤 ユーザー入力: {q}")
        intent = classify_intent(q)
        print(f"🎯 d3 ルーティング結果 -> [{intent}]")

        if "weather" in intent:
            reply = "気象レーダーを確認しました。明日の東京は曇りのち一時雨の予報です。"
        elif "system_status" in intent:
            reply = "本日の自動巡回により、59種類の未収録モデルが検出され探索プールに格納されました。"
        else:
            reply = "実は、世界最初のメカニカルキーボードは1970年代に誕生したと言われています！"

        print(f"🤖 アシスタント出力: {reply}")
        speak_text(reply)

if __name__ == "__main__":
    main()
```

外部 API コールは一切なく、**クラウド課金 0 円、メモリ消費 3GB 未満**で完結します。

---

## 6. 実装の注意点とチューニング

1. **OS による差異**：
   - `phonon` によるスピーカー直接出力は **macOS**（CoreAudio ネイティブ）で最も軽快に動作します。Linux や Windows 環境では、標準の `/v1/audio/speech` エンドポイント経由でオーディオを受信して再生する構成がおすすめです。
2. **メモリ配分の目安**：
   - `phonon` はわずか **約80MB** のメモリしか消費しません。
   - `d3:2b` は **約1.6GB** の VRAM を消費します。8B 規模の対話モデル（DeepSeek-R1-8B や Qwen2.5-Coder-7B）と同時に常駐させても、8GB〜16GB 搭載マシンで無理なく動作します。
3. **モデルの常駐（Keep-Alive）**：
   - モデルのロード・アンロードによる遅延を防ぐため、API リクエストに `"keep_alive": "30m"` を指定してメモリ上に常駐させておくことを推奨します。

---

## 7. まとめ

Ollama v0.40.11 の登場により、ローカル AI は単なる文字入力プロンプトの枠組みを超えました。

音声合成と高速な意図ルーティングが 1 つの実行ファイルに統合されたことで、個人開発者でも高度な自律型エージェントを手軽に構築できるようになりました。ぜひ最新版にアップデートして、その快適さを体験してみてください！

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/ollama-v0-40-11-tts-phonon-d3-decision-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


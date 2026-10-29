---
title: "LLMの重み内でCPUが動き、CPU内でDOOMが動く：Qwenputerとは何者なのか？"
description: "32ビットRISC-V CPUを言語モデルのニューラル重みに直接コンパイル。外部Pythonもサンドボックスも一切使わず、Forward Passだけでマシン語を実行し、発売当日には名作『DOOM』のレイトレーサーまで動作！Gated DeltaNetをRAMに見立てる内部構造、再現性の高いローカル導入手順、Agentの確定的ガードレールへの応用を徹底解説。"
pubDate: 2026-10-29
author: "Xiao Ke (FreeAI Dog Hardware & Edge Computing Team)"
category: "实战进阶"
tags: ["Qwenputer", "Qwen-3.5", "RISC-V", "LLMアーキテクチャ", "Doom", "Gated DeltaNet", "Agentガードレール", "確定的計算"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/qwenputer-riscv-cpu-inside-qwen-weights-doom-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# LLMの重み内でCPUが動き、CPU内でDOOMが動く：Qwenputerとは何者なのか？

オープンソースAIコミュニティで、計算機科学史に残るような驚くべき実験プロジェクトが登場しました：

> **言語モデルのニューラルネットワーク重みの中に、32ビットRISC-V CPU（RV32IM）が直接「コンパイル」されて埋め込まれたのです！**

さらに驚くべきことに、公開当日にコミュニティのハッカーがこのCPU上で**初代『DOOM』（E1M1）のレイトレーシングエンジンを動かすことに成功**しました。

ブラウザでキーボードの `W` を押すと、モデル内部のForward Passで17,540命令のRISC-Vアセンブリが実行され、ターミナル上にASCIIアートで迷路の次フレームが描画されます。

外部のPythonも、コンテナも、サンドボックスも介していません。すべてはニューラルネットの行列演算と状態更新の中で完結しています。

本記事では、この **Qwenputer** の仕組み、実機で検証可能な再現手順、そして「Agentの確定的ガードレール（安全装置）」としての画期的な意義を解説します。

---

## 1. 仕組みの全貌：どうやって重みの中にコンピュータを作るのか？

コンピュータには **RAM**、**レジスタ**、**プログラムカウンタ（PC）**、**ALU（演算装置）** が必要です。

従来のTransformer（Softmax自己注意）は値を確率でぼかしてしまうためRAMには適しません。しかし、QwenputerはQwenに備わっている **Gated DeltaNet（線形回帰層）** を巧みに活用しました。

### Gated DeltaNetをRAMとして利用する
DeltaNetのヘッドは内部に状態行列を保持しています。Qwenputerは：
- 減衰ゲートを全開（$\alpha = 1.0$）にして記憶を不変に保ちます。
- 書き込み強度を完全上書き（$\beta = 1.0$）に固定します。
- Keyベクトルを**厳格なOne-hotベクトル**にすることで、特定行のバイトデータだけをピンポイントで書き換えます。

これは物理的なRAM書き込みと完全に等価です。下位ビットでヘッド、上位ビットで行を選択し、モデルの内部状態を **32KBの仮想RAM** に仕立て上げています。

### コンパイルされた11層のCPU回路
このCPU層は**バックプロパゲーションによる学習を一切行っていません**。
専用のコンパイラが、RV32IMの回路論理をbfloat16精度の正確な数値として算出し、Qwen本体（32層）の直前に **11層のTransformer層** として直接配置しています。

### トークン生成＝CPUのクロックサイクル
Qwenputerにおいて、**モデルがTrace Tokenを自己回帰生成するプロセスそのものがCPUのクロックサイクル**です。1命令ごとにレジスタ書き込みや分岐などのトークンが内部生成され、計算が完了して `putchar` が呼ばれると、Qwen本来の層がその出力を読み取って回答を返します。23万回の出力検証において、実機エミュレータとの不一致はゼロでした。

---

## 2. 足し算1回に何百万回も乗算して何になるのか？

「世界一非効率なCPU」と揶揄される一方で、セキュリティ研究者が最も注目しているのは **Agentの確定的ガードレール（Deterministic Guardrails）** です。

### 従来のAgentの弱点
- 外部ツール依存（Bash/Python実行）によるサンドボックス脱獄リスク。
- プロンプトインジェクションによるルール無視（モデルは確率的サンプリングに逆らえない）。

### Qwenputerによる確定的制御
もしポリシー検証やSQLチェックロジックを、この11層の重み回路としてモデルの頭脳に直接焼き込んだらどうなるでしょうか？
- **物理的に回避不能**：すべてのトークン生成が前置回路を通過するため、いかなるプロンプト攻撃でも層の計算をスキップできません。
- **100%確定的**：権限チェックや算術演算は完全なハードウェア論理として動作し、ハルシネーションの余地がありません。
- **完全ローカル自律**：外部環境がなくても、オフラインで安全な自律制御が可能です。

---

## 3. 実機で動かす：ローカル環境での再現手順

### 前提環境
- **OS**: Linux (Ubuntu 22.04+) または macOS (Apple Silicon)
- **GPU**: NVIDIA GPU (16GB以上のVRAM推奨; CPU単体でも動作可能)
- **パッケージ管理**: `uv`

### ステップ1：リポジトリ取得と環境構築
```bash
git clone https://github.com/divagr18/qwenputer.git
cd qwenputer/experimental/riscv

curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.cargo/env

uv sync --extra hf
```

### ステップ2：コンパイル済み重みのダウンロード
```bash
pip install -U huggingface_hub
hf download divagr1925/Qwen3.5-4B-Qwenputer --local-dir qwenputer-rv
```

### ステップ3：確定的計算の実行
同梱されているRISC-Vファームウェア（`calc-bridge.elf`）を用いて、巨大な数の掛け算を実行します：
```bash
uv run python -m chipformer.rv.chat qwenputer-rv \
    --firmware tests/data/rv/calc-bridge.elf \
    --ask "Use calc: what is 123456789 * 987654321?"
```
結果：ホスト側でPythonプロセスを実行することなく、モデルの重み内部だけで `121932631112635269` が厳密に出力されます。

---

## 4. DOOMの実行

```bash
git clone https://github.com/mapestree/qwenputer-doom.git
cd qwenputer-doom

uv sync
uv run python play.py --model-dir ../experimental/riscv/qwenputer-rv
```
ブラウザから操作すると、モデル内のCPUが毎ターン17,540命令を消化し、レイトレーシングによる迷路画面をASCIIアートで出力します。

---

## 5. まとめ

Qwenputerは単なる技術的な曲芸ではありません。確率的な生成知能と決定論的な論理回路が同一のニューラルネットワーク重みの中で共存できることを証明した、次世代AIアーキテクチャの先駆的マイルストーンです。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/qwenputer-riscv-cpu-inside-qwen-weights-doom-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


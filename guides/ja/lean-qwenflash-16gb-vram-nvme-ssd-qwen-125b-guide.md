---
title: "16GB VRAMで125B巨大MoEが動く！Lean-QwenFlash実戦ガイド：NVMe SSD直接オフロードで262K全文脈・20 tok/sを達成"
description: "VRAM不足の壁を打破！オープンソースのLean-QwenFlashは、従来の「全エキスパートをシステムRAMに常駐させる」制限を撤廃。高速NVMe SSDへ直接エキスパート重みをオフロードすることで、わずか16GB VRAM＋8GBメインメモリの構成（RTX 4090やゲーミングノートPC）でQwen 3.8 Flash Next 125Bを実行可能に。262K超長大コンテキスト下でも20 tok/sの高速生成を実現する仕組みと導入手順を詳解。"
pubDate: 2026-10-26
author: "小可 (FreeAI Dog ハードウェア検証チーム)"
category: "実践・応用"
tags: ["Qwen-3.8-Flash-Next", "125Bモデル", "Lean-QwenFlash", "ExLlamaV3", "VRAM最適化", "NVMeオフロード", "ローカルLLM", "コンシューマーGPU"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/lean-qwenflash-16gb-vram-nvme-ssd-qwen-125b-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# 16GB VRAMで125B巨大MoEが動く！Lean-QwenFlash実戦ガイド：NVMe SSD直接オフロードで262K全文脈・20 tok/sを達成

オープンソース大規模言語モデルの世界において、**Qwen 3.8 Flash Next 125B** は、24,000個もの細粒度スパースエキスパート（Sparse Experts）と抜群のコード生成・論理推論能力を備えたトップクラスのモデルとして注目を集めています。

しかし従来、この125B巨大モデルをローカルで動かすためのハードルは極めて高いものでした：
- **純粋なVRAM駆動**：最低でも2枚の24GBグラフィックカード（RTX 3090/4090の2枚刺し）やデータセンター向けGPUが必要；
- **従来のCPU/RAMオフロード手法（Strata等）**：推論速度を維持するため24,000個のエキスパート重みすべてをメインメモリに常駐させる必要があり、**48GB〜64GBもの大容量システムRAMが必須**。

この制限により、16GB VRAM搭載の単体GPU（RTX 4080、ノートPC版RTX 4090/3080等）と標準的な16GB〜32GBメモリを持つ大半のユーザーは実行を諦めざるを得ませんでした。

新登場のオープンソース基盤 **Lean-QwenFlash** は、この制約を完全に打破しました。**ExLlamaV3** をベースに構築され、エキスパートプールをメインメモリではなく高速な **NVMe SSD** に直接オフロードすることで、驚異的な軽量化を実現しています：

> **わずか 16GB VRAM ＋ 8GB システムメモリの構成で、262Kトークンの超長大コンテキストにおいても 20.0 tok/s の高速推論をノートPC上で安定維持可能！**

本記事では、NVMeダイレクトオフロードの設計原理、実機ベンチマーク、導入手順を詳しく解説します。

---

## 1. アーキテクチャの核心：なぜNVMe直接オフロードが常識を覆すのか？

### メインメモリ常駐型（従来手法）の課題
Mixture-of-Experts（MoE）推論では、1トークン生成ごとに呼び出されるエキスパートはごく一部に限られます。しかし従来のオフロード設計では、ディスク読み出し遅延を回避するため、全エキスパート（48GB〜64GB）をシステムRAMに常駐させていました。この要求水準は一般のノートPCやデスクトップPCには高すぎます。

### Lean-QwenFlashの革新：RAM非占有 ＋ NVMeストリーミング
Lean-QwenFlashは以下の最適化によってメモリ消費を最小化しています：
1. **VRAMにはアテンション骨格とKVキャッシュのみを配置**：16GB VRAM内にはアテンションレイヤー、現在実行中のエキスパートカーネル、Q4精度のKVキャッシュのみを厳格に保持；
2. **NVMe SSDを巨大なエキスパートストレージとして活用**：未アクティブのエキスパートはSSD上に最適化パック形式で格納。メモリマップ（`mmap`）と非同期プリフェッチにより、必要なテンソルだけをGPUへ直接ストリーミング供給；
3. **システムRAMの要求量はわずか8GB**：ホストメモリに数十GBの重みを保持しないため、一般的な8GB〜16GBメモリ搭載PCで軽快に動作します。

### 💡 疑問点：高頻度の読み出しでSSDの寿命は縮まないか？
**全く問題ありません。**
SSDの寿命（TBW）を消費するのは**データの書き込み（Write）**です。推論処理中の重みアクセスは **100% 読み出し専用（Read-Only）** であり、ディスクへの書き込みは一切発生しないため、SSDの寿命に悪影響を与えることはありません。

---

## 2. 実機ベンチマーク表：RTX 4090 デスクトップ vs RTX 3080 ノートPC

推奨量子化 **3.05bpw_h5_ng5** および **Q4 KV Cache** での測定値：

| 測定環境 | コンテキスト長 | デコード速度 (Decode) | プリフィルスループット (Prefill) | ピークVRAM | システムRAM要件 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RTX 4090 (Desktop)** | **8K 短文脈** | **28.0〜33.0 tok/s** (平文)<br/>**19.0〜22.0 tok/s** (コード) | ~1080 tok/s | 16.0 GB | 8.0 GB |
| **RTX 4090 (Desktop)** | **100K 長文** | **20.8 tok/s** | ~1536 tok/s | 16.0 GB | 8.0 GB |
| **RTX 4090 (Desktop)** | **262K 最大長** | **20.0 tok/s** | ~1627 tok/s | 16.0 GB | 8.0 GB |
| **RTX 3080 16G (Laptop)** | **100K 長文** | **18.0〜21.0 tok/s** | ~650〜700 tok/s | 15.5 GB | 8.0 GB |

> **検証結果**：
> 8Kから262Kまで、デコード速度は常に実用ラインである **20 tok/s** をクリア。人間の読書速度（秒間5〜8トークン）を大きく上回り、長文分析や全コードベース読解に実用的な速度を提供します。

---

## 3. 必要動作環境と事前準備

### 1. ハードウェア要件
- **GPU**：NVIDIA製GPU、**VRAM 16 GB 以上**（RTX 4090、4080、3090、ノート版RTX 3080/4080/4090 16GB等。AMD製GPUは現在非対応）；
- **システムRAM**：空き物理メモリ 8 GB 以上；
- **ストレージ (SSD)**：**高速NVMe M.2 SSD 必須**（PCIe 3.0または4.0、読み込み速度 3000 MB/s以上を推奨。SATA接続や外付けHDDは不可）；
- **空き容量**：約65 GB（3.05bpwモデル重み格納用）。

### 2. ソフトウェアスタック
- **OS**：Linux (Ubuntu 22.04 / 24.04 推奨) または Windows 11 (WSL2 経由)；
- **環境**：NVIDIA Driver ≥ 535、CUDA Toolkit 12.1+、Python 3.10〜3.11。

---

## 4. インストールと実行の完全ステップ

### ステップ 1：リポジトリの取得とPython環境構築

```bash
# リポジトリのクローン
git clone https://github.com/MaxDevv/lean-qwenflash.git
cd lean-qwenflash

# 仮想環境の作成と有効化
python3 -m venv venv
source venv/bin/activate

# 依存パッケージとExLlamaV3の導入
pip install --upgrade pip
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
pip install exllamav3 huggingface_hub
```

---

### ステップ 2：3.05bpw 量子化重みの取得

同梱されているダウンロードスクリプトを実行します：

```bash
python download_model.py --output-dir ./models/Qwen-3.8-Flash-Next-125B-3.05bpw
```

---

### ステップ 3：NVMeストリーミング推論の起動

```bash
python run_inference.py \
  --model_dir ./models/Qwen-3.8-Flash-Next-125B-3.05bpw \
  --max_seq_len 262144 \
  --gpu_split 16.0 \
  --cache_mode q4 \
  --mmap_expert_dir ./models/Qwen-3.8-Flash-Next-125B-3.05bpw \
  --temperature 0.2 \
  --top_p 0.95
```

**主要オプションの解説**：
- `--max_seq_len 262144`：262Kフルコンテキストウィンドウを有効化；
- `--gpu_split 16.0`：VRAM割り当て上限を16.0 GBに制限；
- `--cache_mode q4`：KVキャッシュを4bitに圧縮し長文推論時のメモリ消費を抑制；
- `--mmap_expert_dir`：NVMe上のエキスパート配置パスを指定。

---

## 5. 運用上の注意点とTips

1. **なぜQ4 KV Cacheを選ぶべきか**：
   262K長文の場合、未圧縮のFP16 KVキャッシュだけで30GB以上のVRAMを消費します。Q4モードを採用することで、精度を保ちつつ16GBカード内での262K完走を可能にしています。
2. **SATAドライブへの配置は避ける**：
   SATA SSD（転送上限約500MB/s）を使用すると、読み出し遅延により推論速度が3〜5 tok/sまで激減します。必ずPCIe NVMeスロット接続のディスクを使用してください。
3. **マルチGPU（16GB×2枚）でのさらなる高速化**：
   16GBカードを2枚搭載している環境では `--gpu_split 14.5,14.5` を指定することで、エキスパートを両GPUの空きVRAMに分散キャッシュでき、生成速度は **45+ tok/s** まで跳ね上がります。

---

高価なメインメモリの代わりにNVMe SSDの読み出し帯域を活用することで、一般のコンシューマー環境でも125Bクラスの巨大モデルを快適に扱える時代が到来しました。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/lean-qwenflash-16gb-vram-nvme-ssd-qwen-125b-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


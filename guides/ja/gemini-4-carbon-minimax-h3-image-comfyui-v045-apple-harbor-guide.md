---
title: "24GB GPUでH3-Imageを動かす算段とGemini 4 Carbon選定帳：まず重み容量を削り、Harbor狙いのMac購入は待て"
description: "Hugging Faceのファイル一覧とComfyUI v0.45.0 PR記録を照合し、135.32GBのMiniMax-H3-Imageを29.41GB構成へ圧縮する条件を整理。Gemini 4 Carbonのプロモ後単価とベンチマーク弱点、Apple Claris Harborを様子見すべき理由を解説します。"
pubDate: 2026-11-11
author: "小可 [シャオコー]"
category: "エンジニアリング分解"
tags: ["Gemini 4 Carbon", "MiniMax-H3-Image", "ComfyUI", "Apple Harbor", "VRAM最適化"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


昨夜 Hugging Face で `MiniMaxAI/MiniMax-H3-Image` のファイルツリーを開き、総容量を暗算した時点で一度ブラウザを閉じかけました。

33.1B の DiT バックボーンが BF16 で 12 分割されているだけでなく、同じリポジトリに `transformer_diffusers/` と VAE 2 系統まで丸ごと二重格納されています。おまけに 66.7 GB のテキストエンコーダは別リポジトリの `MiniMaxAI/MiniMax-H3` 側に置かれています。ツリーを確認せず `git clone` を叩けば、画像 1 枚生成する前に SSD を 200 GB 以上食い潰します。

同じ 11 月 10 日には、関連の深い動きが一気に重なりました。Google DeepMind が `Gemini 4 Carbon` の開発者向け API を全面開放し、ComfyUI は `v0.45.0` で H3-Image をコンシューマ GPU 向けに切り詰めた PR `#17655` をマージ。さらに 9to5Mac が Apple 傘下 Claris でテスト中のローカル推論クラスタアプリ `Harbor` の存在を報じています。

結局のところ論点は 2 つのコスト帳に収束します。クラウド推論の隠れトークン消費をどう抑えるか、そして 100 GB 級の重みを手元のローカル機へどう押し込むかです。本稿は公式ブログ、リポジトリメタデータ、マージ済み PR の記述のみを一次根拠として精査しており、当方で実機ベアメタル計測を行っていない推論秒数などは捏造しません。

## 135GB の H3-Image 問題：ComfyUI v0.45.0 量子化版なしでは一般 GPU で動かない

MiniMax は画像専用の軽量アーキテクチャを新規設計していません。自社の H3-Omni 骨格から音声ブランチだけを削ぎ落とし、テキストからの画像生成と複数枚参照の画像編集を単一チェックポイントへ統合しています。

公式 `README.md` に記載された未圧縮の重量配分はかなりヘビーです：

- DiT 主幹は 33.1B パラメータ、50 層シングルストリーム構造、隠れ次元 5376。BF16 重みだけで 66.28 GB あり、Embedding や出力ヘッドは FP32 で保持、さらに 12 MB の `pdd_fused_heads.safetensors` が付属します。
- テキストエンコーダは `Qwen3-VL-32B` をそのまま流用し、DiT 側はその第 50 層出力を参照します。このディレクトリ単体で 66.7 GB あります。
- 画像 VAE は CNN エンコーダと 24 層 ViT デコーダの組で 16 倍空間圧縮を実行。公式 FP32 版で 2.34 GB になります。

正味の合計は **135.32 GB** です。ダウンロード時は Diffusers 用ミラー重複フォルダを必ず除外してください：

```bash
hf download MiniMaxAI/MiniMax-H3-Image --exclude "transformer_diffusers/*" "vae_diffusers/*"
hf download MiniMaxAI/MiniMax-H3 --include "text_encoder/*"
```

重みが巨大な一方、サンプリング工程は極端に短縮されています。モデルカードによれば **6 ステップ推論** へ蒸留済みで、CFG も廃止され `guidance_scale` は 1 固定。アップスケーラなしで最初から約 1600 万画素、1:1 なら 4096x4096、21:9 なら 6272x2688 を一発出力します。

以下は公式が公開したローカル SGLang 推論とオンライン API の同一シード比較画像です：

<img src="https://img.freeaidog.com/images/blog/minimax-h3-image-local-vs-api-2026.webp" alt="MiniMax H3-Image ローカル SGLang 推論と公式 API の 4736x3552 出力比較" />

社内環境へ組み込む前には、仕様上の落とし穴を 2 つ押さえておく必要があります。

1 つ目は厳しい API 入力バリデーションです。SGLang でも vLLM-Omni でも、`/v1/images/generations` へ `num_inference_steps` に 6 以外を渡すか、`guidance_scale` に 1 以外を指定すると即座に `HTTP 400` で弾かれます。`negative_prompt` も非対応で拒否されるため、避けたい要素はポジティブプロンプトへ肯定文として書き込むしかありません。

2 つ目はライセンス上の地域制限です。リポジトリ直下の `LICENSE` 第 1 条 5 項と第 4 条を確認すると、**欧州連合・英国・韓国・米国** が Excluded Territories に指定されています。適用地域内であっても年商 100 万ドル超または第三者向けホステッド API を提供する場合は `platform.minimax.io/console/license` から事前の書面許諾取得が必須です。グローバル向け SaaS に組み込むなら事前の法務確認を外せません。

では 24 GB〜32 GB のコンシューマ GPU ではどう回すのか。ここで公開当日にリリースされた **ComfyUI v0.45.0** が効いてきます。開発者 `@kijai` による PR `#17655` がマージされ、音声レス版の `MiniMaxH3Image` モデルクラス、PDD スケジューラ、最大 16 枚の参照画像 `<Picture N>` を扱える `MiniMaxH3ImageConditioning` ノードが組み込まれました。

Comfy-Org ミラーリポジトリでは実用域まで削ぎ落とした量子化版が配布されています：

| モデル部位 | ミラーリポジトリ内ファイル名 | 量子化形式 | 容量 | 配置先ディレクトリ |
| --- | --- | --- | --- | --- |
| DiT 無損失トリム版 | `minimax_h3_image_pruned_bf16.safetensors` | Pruned BF16 | 40.23 GB | `models/diffusion_models/` |
| DiT バランス版 | `minimax_h3_image_pruned_int8_convrot.safetensors` | Pruned INT8 | 20.98 GB | `models/diffusion_models/` |
| DiT 最小サイズ版 | `minimax_h3_image_pruned_w4a8.safetensors` | Pruned W4A8 | 12.55 GB | `models/diffusion_models/` |
| テキストエンコーダ INT8 | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | INT8 | 27.14 GB | `models/text_encoders/` |
| テキストエンコーダ FP4 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | NVFP4 AWQ | 15.69 GB | `models/text_encoders/` |
| 軽量プロンプト拡張 | `qwen3.5_9b_w4a8_gptq.safetensors` | W4A8 GPTQ | 7.34 GB | `models/text_encoders/` |
| 画像 VAE | `minimax_h3_image_vae_fp16.safetensors` | FP16 | 1.17 GB | `models/vae/` |

最小構成を足し上げると **29.41 GB** です。PR `#17655` のコメント欄で興味深い点がありました。`@kijai` は RTX 5090 単体で BF16 と INT8 の 1024・2048 解像度出力を検証していますが、32 GB の VRAM に 40.23 GB の BF16 や合計 48 GB 超の INT8 構成はどう収まるのでしょうか。

カラクリはステージ別アンロードと CPU オフロードです。エンコーダ推論完了後に VRAM を解放し、DiT を読み込んで 6 ステップの Euler `simple` サンプリングを実行、余剰ブロックはシステムメモリ側へ逃がして最後に 1.17 GB の VAE へ切り替えます。ホスト RAM が 64 GB 未満の環境で BF16 を強行すれば確実にページングで窒息します。24 GB カードは迷わず W4A8 構成を選んでください。

画像編集で参照枚数を増やして OOM が出る場合は、まず解像度を落としてください。公式テンプレートが初期キャンバスをフル 16 MP ではなく 4 MP の 2048x2048 に抑えているのは意図的です。`Resize Image/Mask` ノードを有効化して参照側の辺長を絞れば、余計な VRAM 溢れを回避できます。なお `v0.45.0` には Qwen 3.x の線形層統合高速化 `#17361`、FP8 テキスト生成クラッシュ修正 `#17620`、SeedVR2 の空き VRAM 連動 VAE スライス `#17508` も入っているため、既存ワークフローの更新価値も十分にあります。

## Gemini 4 Carbon のコストと実力帳：長文脈は最強だがリポジトリ改修への無差別投入は損

10 月 28 日にセキュリティ防御向け Fairwind プログラムへ先行投入された `Gemini 4 Carbon` が、11 月 10 日に AI Studio 有料枠・Android Studio・Gemini for business・Google Antigravity 全域へ解禁されました。

導入価格だけ見ると Argon と同額で魅力的です。入力 100 万トークンあたり 2 ドル、出力 10 ドル、キャッシュ入力は 0.1 ドルに設定されています。

ただしルーターを全量切り替える前に脚注を確認すべきです。プロモ期間終了後の価格はそのまま倍の入力 4 ドル・出力 20 ドルへ引き上げられます。さらに厄介なのが思考トークン消費量です。公式発表自体が「難度の高い検証ループに Argon より多くのトークンを使う」と明記しており、日常の CRUD 修正まで Carbon に流すと請求額が想定以上に膨らみます。

DeepMind 公表のベンチマーク一覧を見ると、強みと弱点がはっきり分かれます：

<img src="https://img.freeaidog.com/images/blog/gemini-4-carbon-benchmarks-2026.webp" alt="Gemini 4 Carbon と Gemini 4 Argon・GPT-6 Astra・Claude Opus 5.5 の公式ベンチマーク比較表" />

選定方針はシンプルです。Carbon は `AA Index v5.0` で 59 を記録し長文脈参照と対インジェクション耐性では頭抜けていますが、万能ではありません：

- 数十万語規模の監査読解や脆弱性検証には Carbon を優先投入すべきです。`GraphWalks` の 256k〜1M 評価で 96.2% F1 を叩き出し、2 位以下の他社モデルを 20 ポイント超引き離しています。`SEC-bench Pro` も 88.2%、Gray Swan の間接プロンプトインジェクション成功率 0.0%、`Terminal-Bench 4.0` チート率 0.0% です。Antigravity で同時プレビューされた `/swarm` 構成の総括・監査役に据え、実装は軽量ワーカーへ分担させると収支が合います。
- 複数ファイルに跨る大規模パッチ生成を無条件で任せるのは得策ではありません。`FrontierSWE 2` では 61.2% に留まり、`GPT-6 Astra` の 65.5% に 4.3 ポイント負けています。
- デスクトップ GUI 操作も同様に後塵を拝しています。`OSWorld-2.1` オフライン項目は 81.5% で、`Claude Opus 5.5` の 87.2% から 5.7 ポイント差がつきました。総合スコアではなくタスク単位でルーティングを切るべきです。

## Apple Claris Harbor リーク：Thunderbolt 5 メモリプール狙いで今すぐ Mac を買うな

同日 `r/LocalLLaMA` で注目を集めたのが、9to5Mac がリーク元 `@aaronp613` の解析として報じた Apple 社内テストアプリ **Harbor** の情報です。

狙いはローカル環境構築の面倒を丸ごと消すことにあります。Claris ブランド下で開発中の Harbor は、GUI からオープンウェイトモデルを検索・ダウンロードして起動するだけで、Mac をプライベート AI 推論サーバへ変換します。さらに **Thunderbolt 5** ケーブルで複数台の Mac を繋ぐと自動検出し、各機のユニファイドメモリを単一の推論プールへ束ねます。

前節で触れた 135 GB の壁を思い出せば Apple の意図は透けて見えます。これまでは大型エンコーダや 100B 級 MoE を非量子化のまま手元で回す場合、複数台の Mac Studio を繋いで MLX 分散リングや SSH トンネルを手組みするしかありませんでした。それをケーブル 1 本の自動クラスタへ落とし込む設計です。

とはいえハードウェア購入の判断材料にするのは時期尚早です。Harbor は現在 Claris のプライベートプレビュー段階であり、一般公開時期も OpenAI 互換 API の仕様も、画像拡散向けカーネル対応も一切未確定のままです。またユニファイドメモリのプール化で確保できるのは 128 GB や 256 GB といった「容量」であって、NVIDIA テンソルコアの演算速度そのものではありません。1600 万画素 DiT の重たい推論処理を M 系 GPU と FP4 対応 RTX 5090 で正面対決させれば実行時間に大きな開きが出る公算が高いため、複数台 Mac Studio の実測スループット報告が出揃ってから検討しても遅くありません。

---

本稿で扱った各数値・容量・規約は下記の一次情報から直接照合できます：

| 確認項目 | 一次ソース URL |
| --- | --- |
| Gemini 4 Carbon 公開発表および価格脚注 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/ |
| Gemini 4 Carbon ベンチマーク数値 | https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/ |
| MiniMax-H3-Image モデルカード仕様 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image |
| MiniMax ライセンス除外地域条項 | https://huggingface.co/MiniMaxAI/MiniMax-H3-Image/blob/main/LICENSE |
| ComfyUI PR #17655 マージ記録 | https://github.com/Comfy-Org/ComfyUI/pull/17655 |
| ComfyUI 公式チュートリアルとテンプレート条件 | https://docs.comfy.org/tutorials/image/minimax/minimax-h3-image |
| Comfy-Org 量子化済み safetensors 一覧 | https://huggingface.co/Comfy-Org/MiniMax-H3-Image |
| Apple Claris Harbor 関連報道 | https://9to5mac.com/2026/11/10/apple-has-new-mac-app-coming-to-make-running-local-ai-models-easy/ |

手元環境が 24 GB GPU なら、まず ComfyUI v0.45.0 へ上げて `Resize Image/Mask` を有効化した状態で W4A8 構成のホストメモリ退避量を測ってみてください。クラウド連携側では高難度レビューや長文脈解析だけを Carbon に切り出し、定常コーディングは低単価枠へ残す構成が現実的です。今のローカル環境で先に天井を叩くのは VRAM 容量でしょうか、それとも CPU 退避時の転送待ち時間でしょうか？

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/gemini-4-carbon-minimax-h3-image-comfyui-v045-apple-harbor-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


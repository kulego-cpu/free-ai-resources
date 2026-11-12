---
title: "Claude Fable 5.2 へ旧プロンプトを移す前に：400 Prefix Check と挙動変化の注意点"
description: "Anthropic 公式 Prompting Claude Fable 5.2 ドキュメントに基づく移行ガイド。全アカウント強制の思考ブロック prefix check による 400 エラー、medium 以上での思考抑止無効化、Memory 蓄積ルールによる承認ループ、reasoning_extraction 拒否分類を整理する。"
pubDate: 2026-11-12
author: "小可"
category: "アーキテクチャと移行"
tags: ["Claude Fable 5.2", "Claude Mythos 5.2", "Prompt Engineering", "Agent Harness", "Anthropic"]
featured: false
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


`Claude Fable 5.1` や `Opus 5.5` 向けに書いた古いシステムプロンプトをそのまま `Claude Fable 5.2` へ切り替えると、たいてい 2 箇所でつまずく。

保存済み会話を再開した瞬間に API が `400` エラーを返すか、エージェントが途中で止まって一時ファイル削除ごときで承認を求めてくるかだ。

Anthropic が公式ドキュメント `prompting-claude-fable-5-2` を公開した。読み進めると、今世代で一番変わったのは既存ハーネスが暗黙に頼っていた前提条件そのものだと分かる。本稿の根拠はこの公式ガイドのみであり、本番の長大セッションで全量リグレッションを回したわけではないため、トークン請求額や所要時間の自作データは載せない。

## System を 1 行変えるだけで 400：全アカウント強制の Prefix Check

以前のモデル更新では、`system` プロンプトを 2 行直してから DB に残った古い会話をそのまま続行する運用がよく使われていた。

`Claude Fable 5.2` ではその手が通用しない。API が全アカウントに対して思考ブロックの `prefix check` を強制するためだ。保存済みの thinking blocks を戻しつつ `system`、`tools`、または過去メッセージを少しでも書き換えると、Messages API はデフォルトで `400` エラーを返して拒否する。

新しい指示は新規会話へ入れるか、最新ターンのユーザーメッセージへ載せる必要がある。会話の途中でどうしても履歴を変えたい場合はどうするか。

ベータの逃げ道として `thinking-binding-controls-2026-08-01` ヘッダーを付け、`thinking.block_binding.prefix_mismatch_behavior` に `"drop_block"` を指定する方法が用意されている。ただし代償がある。Anthropic のテストでは、ターンごとに過去の思考ブロックを破棄するとモデルが毎回考え直し、出力トークンが増えた。明示的な compaction や context editing 以外は、メッセージ履歴を追記専用に保つのが一番安く済む。

## 「考えずに答えろ」が効かない：medium 以上は常に先に思考する

`Fable 5.1` からの移行組では、単純な応答の遅延を抑えるために `"answer directly without deliberating"` を混ぜているプロンプトが多い。

`Claude Fable 5.2` ではその一文が効かない。公式ドキュメントには、Anthropic の検証で `effort` が `medium` 以上だと挨拶 1 発でも思考に入ってしまい、プロンプトで直接回答を指示しても止まらないと明記されている。しかもデフォルトの `effort` は `high` だ。

軽いやり取りのコストを下げるには `effort` を下げるしかない。なお `low` 設定でもコーディング評価で `Fable 5.1` の高設定に近いスコアが出たと記載されているが、具体的なベンチマーク名とサンプル数は明示されていないため、SLA ではなく傾向として受け止めるべきだ。会話途中でトップレベルの `effort` を切り替えるとプロンプトキャッシュが壊れるため、キャッシュを維持したままターン単位で調整したい場合は `mid-conversation-output-config-2026-07-01` ベータヘッダーでメッセージ単位の `effort` 上書きを使う。

また `low` と `medium` では「早期停止」も起きやすい。通常のタイムアウトや空のツール結果に当たると残タスクを並べてターンを終えてしまうため、システムプロンプトへ粘り強く続行する指示を足す必要がある。無人バッチが途中で勝手に止まっては困るだろう。

## Memory に溜まった古い注意書きが承認ループを生む

新世代は指示を文字どおり厳格に守る傾向も強まっている。

`Fable 5.1` や `Opus 5.5` の運用中、Memory やルールファイルへ `"never retry"`、`"never delete files"`、`"stop and review at the first error"` といった防御文を積み上げてきたチームは多いはずだ。旧モデルなら文脈を見て柔軟に流していた箇所を、`Fable 5.2` は `"never"` を含む絶対禁止として律儀に実行する。

その結果、マルチエージェントを数ターン回しただけで、日常的なファイル操作まで人間の承認待ちで止まるようになる。

末尾に「もっと自由に動け」と書き足しても解決しない。古いメモリファイルを棚卸しし、不要になった `"never retry"` を消し、`"never delete files"` は `"ask for approval before deleting files"` のように破壊的操作だけを限定する表現へ書き換える必要がある。

サブエージェント起動も同様だ。`xhigh` や `max` では自分の書いたコードをレビューさせるために自らサブエージェントを立ち上げる挙動が確認されている。コストが跳ねる場合はツール説明へサブエージェントの単価と使いどころを明記する。公式の `high` 設定での比較では起動回数がおよそ半減したとされるが、タスク分布と件数は未公開のため自分のパイプラインで要確認だ。なお進捗報告テンプレートを入れると出力トークンが約 `20%` 増えるので、人が画面を見ている対話用途以外ではオフにしておきたい。

## 新設の reasoning_extraction 拒否：本文へ思考を書かせない

古い Skill やツール定義に「内部の推論ステップをそのまま回答本文へ出力せよ」と書いてあるケースも要注意だ。

`Claude Fable 5.2` では `stop_reason: "refusal"` に引っかかる可能性がある。

拒否時は `stop_details.category` に `cyber`、`bio`、`frontier_llm`、`reasoning_extraction`、`general_harms`、`null` の 6 分類が返る。本文へ推論を転記させる指示は `reasoning_extraction` に抵触するため、その指示を削除し、`display: "summarized"` を指定して思考ブロック側から読む形へ切り替える。

正当なセキュリティ研究やライフサイエンス業務が `cyber`・`bio` 判定に当たる場合は、認証済み組織向けに提供される `Claude Mythos 5.2` へ切り替える導線が案内されている。

本番トラフィックを流す前に、次の 4 点だけは先に通しておきたい。

1. 新規セッションで `medium`、`high`、`xhigh` の `effort` 評価を回し直し、`Fable 5.1` の設定値を流用しない。
2. 既存のシステムプロンプト、Skills、Memory から `"answer directly without deliberating"`、古い `"never"` 禁止文、本文推論出力の指示を取り除く。
3. 長期セッションの再開処理で `system`・`tools`・過去メッセージが追記専用に保たれ、`prefix check` の `400` エラーを踏まないか確かめる。
4. 一番長いエージェントパイプラインを空回しし、早期停止・過剰なサブエージェント起動・不要な承認待ちのどこに引っかかるか洗い出す。古い地雷を残したまま全量切り替えに踏み切る気にはなれないはずだ。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/claude-fable-5-2-harness-migration-prompting-traps-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


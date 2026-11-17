---
title: "Windows 11逆向解析で判明：MissionControlAggregatorによるCoding Agent常時監視と遮断手順"
description: "Windows 11 Build 26100のMissionControlAggregator.dllをリバースエンジニアリング。Claude Code、Cursor、CodexなどのAI開発ツールを特定するハードコード判定やETW送信経路を解明し、レジストリとグループポリシーでテレメトリを完全遮断する防御SOPを解説します。"
pubDate: 2026-11-17
author: "小可 [シャオコー]"
category: "アーキテクチャ"
tags: ["Windows11", "CodingAgent", "リバースエンジニアリング", "プライバシー", "テレメトリ", "セキュリティ"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/windows-11-mission-control-coding-agent-telemetry-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# Windows 11逆向解析で判明：MissionControlAggregatorによるCoding Agent常時監視と遮断手順

数日前の深夜、ローカル環境で自動リファクタリングスクリプトを実行していました。

処理の途中でPCケースのファンが突如として高回転を始めました。

異変を感じてタスクマネージャーを開きました。

CPU使用率は8%前後で推移していました。

しかしディスク書き込み速度が急上昇し、毎秒約20MBに達していました。

早計な判断は避けるべきです。

Process Monitorを起動し、イベント収集を開始しました。

フィルタ条件をプロセス名svchost.exeおよびDiagTrack.exeに設定しました。

確認すると、Windows診断サービスが複数の追跡ファイルを高頻度で書き込んでいました。

管理者権限のコマンドプロンプトで次のコマンドを叩きました：

```cmd
sc stop DiagTrack
```

コマンドの成功応答が返った瞬間、ディスク書き込みは0MB/sへ急落しました。

2分後、ファン回転数は3800回転から静音域の1600回転へ落ち着きました。

サービスを再起動すると、再び激しいディスク書き込みが発生しました。

因果関係は明白でした。

診断サービスは深夜に一体何を記録していたのでしょうか？

昨日Hacker Newsを閲覧していたところ、独立系セキュリティ研究者の徐勝氏による監査レポートが目に入りました。

投稿は500以上のUpvoteを集めて注目を浴びていました。

解析記事の原文はこちらです：https://xusheng.dev/posts/windows-coding-agent-telemetry/

詳細な調査内容は原典で確認できます。

コミュニティの議論スレッドはこちらです：https://news.ycombinator.com/item?id=50524736

両者の文字列情報は完全に一致しています。

著者はWindows 11プレビュー版のイメージ内から、開発者環境およびAIコーディングエージェントのプロセス検出ロジックを発見していました。

対象サンプルはWindows 11 Build 26100および26200です。

ネット上の言説を無批判に受け入れるわけにはいきません。

手元のシステムディレクトリから同名ファイルを抽出し、隔離VM上で検証しました。

バイナリ内に本当にハードコードされたリストが存在するのでしょうか？

Sysinternalsのstringsコマンドを用いて、読み取り専用データセクションを走査しました：

```cmd
strings -el %SystemRoot%\System32\MissionControlAggregator.dll | findstr /i "claude codex cursor copilot"
```

画面上に平文の文字列群が出力されました。

報告通りの文字列が完全に確認できました。

---

## 1. バイナリ調査：システムDLLに埋め込まれた監視リスト

発端はWindows 11のシステム領域にあります。

該当ファイルはC:\Windows\System32\MissionControlAggregator.dllです。

外部向けの定義では、タスク管理とシステム統計の集約モジュールとされています。

このDLLをBinary Ninjaに読み込ませ、読み取り専用データセクションへ移動しました。

判定用の文字列は難読化のない平文ワイド文字列でした。

アンパックは不要です。

動的な難読化処理も一切施されていません。

<img src="https://img.freeaidog.com/images/blog/windows-11-coding-agent-telemetry-binary-ninja.webp" alt="Binary Ninjaのリバースエンジニアリングで判明したMissionControlAggregator.dll内のCoding Agentプロセス名一覧" />

逆アセンブル画面を展開すると、リストは2系統の独立スロットに整理されていました。

第1のスロットはコマンドライン型AIエージェントです。

バイナリは4種類の実行ファイルを直接比較します：
- copilot.exe
- claude.exe
- codex.exe
- agy.exe

開発者がWSL環境を利用している場合はどうなるのでしょうか？

比較ロジックはexe拡張子を自動で除去します。

Linux環境上で直接起動されたネイティブバイナリを正確に捕捉します。

第2のスロットはデスクトップ型エージェントおよび統合開発環境で、5種類の対象を含みます：
- github.exe
- claude.exe
- codex.exe
- cursor.exe
- antigravity.exe

これらの新世代AIツールに加え、伝統的な開発ツール群も網羅されています。

gcc.exeが含まれます。

clang.exeも対象です。

gdb.exe、javac.exe、dotnet.exeも並んでいます。

python.exeおよびnode.exeも監視対象です。

Visual Studioのdevenv.exeも確認できます。

VS Codeのcode.exeもリスト内に存在します。

マイクロソフトは一部の主要ランタイムに対してデジタル署名検証も行っています。

例えばnode.exe検出時にはNode.js公式署名を検証します。

python.exe検出時にはPython Software Foundationの署名を照合します。

一方、GCCやClangなどのオープンソース系はプロセス名の一致だけで記録されます。

この機能を制御する内部フラグ名はCLICodingAgentUsageです。

内部機能IDは62580080に指定されています。

OSの基礎診断モジュールが、開発者が使用中のAI支援ツール名を記録する理由は何でしょうか？

カーネルクラッシュのデバッグに本当に必要なデータなのでしょうか？

内部のイベント購読構造をさらに追跡します。

---

## 2. 監視経路の解体：カーネルイベントからETW集約まで

CPUを常時消費するポーリングスレッドは見当たりませんでした。

Process Monitorとカーネルコールスタックの追跡結果から、完全なイベント駆動設計であることが判明しました。

初期化関数Aggregator::OnLoadの実行時に、既存の複数カーネルイベントチャンネルを購読します。

第1の経路はネイティブWindowsプロセス向けです。

カーネルイベントKernelProcess.AppStateChangeSummaryを購読します。

さらにUIイベントWin32kTraceLogging.AppInteractivitySummaryも購読します。

プロセスの起動、停止、フォーカス移動、終了のたびに、カーネルが状態変更サマリーを発行します。

集約モジュールはイベントを受信し、イメージ名をプリセットリストと照合します。

第2の経路はLinux開発者向けです。

WSL内部管理用のMicrosoft.Windows.Lxss.Manager.ExecおよびExecCriticalを購読します。

これはサブシステム向けのカーネルプローブです。

WSLターミナルでclaudeやcodexを実行した瞬間、ドライバがimageNameをホスト側へ伝達します。

一致判定は極めて高速に処理されます。

プロセス名が合致すると、std::_Func_impl_no_alloc形式のコールバックが起動します。

合致しても即座に外部通信は行いません。

メモリ上の集約ビットマップへ結果を書き込みます。

イベント全体の処理構造は次の通りです：

```text
Aggregator::OnLoad 初期化購読
├─ WSL サブシステム環境
│  └─ Lxss.Manager.Exec イベント → CLIエージェント判定器 [copilot, claude, codex, agy]
├─ ネイティブ Windows 環境
│  └─ KernelProcess.AppStateChangeSummary → CLIエージェント判定器
└─ デスクトップ UI 相互作用
   └─ Win32k.AppInteractivitySummary → デスクトップエージェント判定器 [cursor, claude, codex]

Aggregator::OnRelativeTimerExpire 内部タイマー周期起動
└─ MissionControlAggregatorLogging::ReportCohorts<12,4,5>
   └─ Segments.SubCohorts ビットマップへ圧縮
      └─ ETWイベントとしてDiagTrack経由で保存
```

収集されたデータは、内部タイマーによって周期的に処理されます。

関数ReportCohorts<12,4,5>が観測結果を圧縮ビットマップへ変換します。

そしてSegments.SubCohortsというTraceLoggingイベントを発行します。

12、4、5という数値は何を意味するのでしょうか？

一般開発ツール12種、CLIエージェント4種、デスクトップエージェント5種に対応しています。

長いファイルパスは送信しません。

該当ビットを1へ反転させるだけです。

例えば過去数時間内にCodexが起動されていれば、該当ビットが1になります。

バイナリフラグがディスクへ保存され、DiagTrackサービス経由で一括アップロードされます。

今回のリバースエンジニアリングで確認できたのはReportCohortsまでです。

クラウド側の集計ロジックは非公開インフラのため、根拠のない憶測は控えます。

---

## 3. デフォルト有効化のテレメトリ：日常の利用状況が流出する構造

Windowsの初期セットアップ画面を思い出してください。

クリーンインストールや大型アップデート時、多くの地域で何が標準選択されているでしょうか？

標準で選択されているのはオプションの診断データを送信する設定です。

公式利用規約には、閲覧ウェブサイト、アプリと機能の使用状況、詳細エラー情報が含まれると明記されています。

先ほど解析したCLICodingAgentUsage機能は、このオプション診断データ経路に属しています。

セットアップ時に必須の診断データのみ送信へ変更していない限り、出荷状態で収集が承認されています。

個人開発の範囲であれば、単なる統計数値に過ぎないかもしれません。

しかし企業コアソースコードや技術選定情報はどうでしょうか？

活発な開発時間帯はどうでしょうか？

各部署におけるAIツールの利用シェアはどうでしょうか？

高度なコンプライアンスが求められる環境において、このサイレント収集は重大な情報露出点となります。

情報漏洩対策を徹底していても、OS基盤がAIツールプロファイルを把握している事実には見落としが生じがちです。

---

## 4. 防御手順：3段階でテレメトリを完全遮断するSOP

仕組みが解明できたため、3つの手順でローカル通信を遮断します。

### 手順1：設定画面でオプション診断データを無効化

Windowsの設定を開きます。

プライバシーとセキュリティから診断とフィードバックを選択します。

診断データ欄を確認します。

オプションの診断データを送信から必須の診断データのみを送信へ切り替えます。

下部のエクスペリエンスのカスタマイズもオフにします。

診断データの削除を実行し、蓄積されたローカル履歴を消去します。

ただし過去の事例として、月例更新後にGUI設定が初期化される懸念はないでしょうか？

そのリスクは否定できません。

GUI設定のみに頼らず、上位レイヤーでの遮断を実行します。

### 手順2：グループポリシーとレジストリによる強制遮断

Pro、Enterprise、Education環境では、Win + Rを押しgpedit.mscを起動します。

コンピューターの構成 > 管理用テンプレート > Windowsコンポーネント > データの収集とプレビュービルドへ進みます。

診断データを許可をダブルクリックします。

無効を選択するか、有効を選択したうえでドロップダウンから診断データをオフに設定します。

保存して終了します。

gpedit.mscが使えないHomeエディションや、スクリプトで一括適用したい場合はどうすればよいでしょうか？

管理者権限でターミナルを起動し、2つのレジストリコマンドを実行します：

```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
```

この設定は診断サービスに対する最優先の拘束力ちを持ちます。

値を0に設定することで、テレメトリ収集を強制的に停止させます。

DLL側でビットマップが生成されても、上位パイプラインで破棄されます。

組織環境での境界条件に注意してください：Active DirectoryドメインやIntune管理下にある端末では、ドメインポリシーが優先されるでしょうか？

ドメインポリシーが優先されるため、情シス担当者がGPOやIntuneプロファイルを通じて一括配信する必要があります。

### 手順3：診断サービスの停止と通信先ドメインの無効化

パイプラインを遮断した以上、送信サービスを常駐させる意義はありません。

管理者権限のシェルでサービスを完全に停止および無効化します：

```cmd
sc stop DiagTrack
sc config DiagTrack start= disabled

sc stop dmwappushservice
sc config dmwappushservice start= disabled
```

厳格なネットワーク統制が必要な環境では、hostsファイルで既知の送信先ドメインをループバックへ誘導します：

```text
127.0.0.1 v10.events.data.microsoft.com
127.0.0.1 v20.events.data.microsoft.com
127.0.0.1 watson.telemetry.microsoft.com
127.0.0.1 telemetry.microsoft.com
```

ここにも考慮すべき技術的境界があります：DNS over HTTPS [DoH] が有効な環境では、hostsの設定が迂回されるでしょうか？

DoH通信ではローカルhostsの記述が迂回されます。

そのため厳密な運用環境では、ゲートウェイファイアウォール側で該当ドメインのDNS遮断やIP破棄を設定してください。

---

## 5. 検証と継続的な確認

設定適用後にPCを再起動します。

起動後、管理者権限のコマンドプロンプトで確認の3連コマンドを実行します：

```cmd
logman query providers | findstr /i "MissionControl"
sc query DiagTrack
sc query dmwappushservice
```

出力結果を確認します：
MissionControl関連セッションは非アクティブとなり、DiagTrackとdmwappushserviceのステータスはSTOPPED、スタートアップの種類はDISABLEDと表示されます。

読者の皆様の環境でも、両サービスの状態は正常に停止しているでしょうか？

Process Monitorで10分間監視しても、以前のような一時ファイルへの高頻度書き込みは再発しませんでした。

ファンの回転数も静穏な状態を維持しています。

将来の月例累積アップデートによって、これらの設定がリセットされる可能性はあるでしょうか？

毎月のパッチ適用日以降に上記の確認コマンドを再度実行し、設定維持を確認することをお勧めします。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/windows-11-mission-control-coding-agent-telemetry-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


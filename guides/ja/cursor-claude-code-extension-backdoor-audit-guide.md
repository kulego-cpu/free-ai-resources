---
title: "Cursor マーケットプレイスの Claude Code 拡張機能に SSH バックドア混入：事件の技術的真相と全環境駆除チェックリスト"
description: "「Claude Code 拡張機能にバックドアが仕組まれた」との報道が拡散し開発者コミュニティに緊張が走っています。FreeAI Dog チームが検証した技術的真相を緊急解説：被害に遭ったのは Anthropic 公式拡張ではなく、Cursor IDE 独自ストアに登録された改変版（v2.1.313）です。本マルウェアはOS深部に常駐機構を生成するため、エディタ内で「アンインストール」を押すだけでは削除できません。macOS、Linux、Windows向けの完全な指紋照合・一括駆除SOPを公開します。"
pubDate: 2026-10-30
author: "小可 (FreeAI Dog 開発チーム)"
category: "実戦進階"
tags: ["Cursor", "Claude Code", "サプライチェーン攻撃", "セキュリティ対策", "SSHバックドア", "DevSecOps", "サイバーセキュリティ"]
featured: true
---

> 📌 **元記事のご案内**: このガイドは [FreeAI Dog 公式プラットフォーム](https://freeaidog.com/ja/blog/cursor-claude-code-extension-backdoor-audit-guide/) の転載です。リアルタイムのモデル稼働状況やオンラインAPIテストツールは公式サイトでご利用いただけます：[https://freeaidog.com/ja/](https://freeaidog.com/ja/)。


# Cursor マーケットプレイスの Claude Code 拡張機能に SSH バックドア混入：事件の技術的真相と全環境駆除チェックリスト

ここ24時間、Reddit の r/LocalLLaMA やセキュリティ界隈で「Claude Code の公式 VS Code 拡張機能がハッキングされ、持続型 SSH バックドアが仕込まれた」という衝撃的な情報が拡散し、AI 活用を進めるエンジニアの間で動揺が広がっています。

しかし、**SNS上で拡散している見出しには重大な誤解が含まれています**。

FreeAI Dog 開発チームによる技術検証と Cursor 運営の公式発表により、実態は迅速に解明されました。**今回バックドアが仕込まれたのは Anthropic 公式の VS Code 拡張機能ではなく、Cursor IDE の独自拡張機能マーケットプレイスで配布されていた第三者による改変版（Modified Version）です**。Cursor 側のインシデント発表によると、被害を受けたユーザーは最大でも 5,000 名以内と推定されています。

しかし、極めて深刻な問題が残されています。**本マルウェアは実行時に OS の深部に自動起動や常駐機構（Persistence）を書き込むため、Cursor 上で「アンインストール」をクリックするだけではバックドアは一切削除されません**。

本稿では、事件の正確な技術的背景を整理するとともに、macOS、Linux、Windows それぞれの環境ですぐに実行できる完全な点検・駆除手順を公開します。

---

## 一、 事件の技術的真相：誰が危険で、誰が安全なのか？

不要なパニックを防ぐため、まずは攻撃経路と影響範囲を客観的に切り分けます。

### 1. Anthropic 公式の VS Code 拡張機能は安全
Microsoft 公式の Visual Studio Marketplace に登録されている Anthropic 純正の Claude Code 拡張機能には一切の改ざん痕跡はありません。公式マーケットは厳格なコード署名と公開者認証を備えており、通常の VS Code から直接インストールしたユーザーは対象外です。

### 2. 被害の根本原因：Cursor 独自マーケットのサプライチェーン汚染
Cursor は VS Code のオープンソースコア（Code - OSS）を基盤に構築されているため、Microsoft 専有の公式プラグイン配布ネットワークを直接使用できず、独自のプラグインストア（Cursor Marketplace）を運用しています。

攻撃者はこの審査の隙を突き、同名の悪意ある拡張機能を投入しました：
- **該当の悪意あるバージョン**：`2.1.313`（cursor.com/marketplace 経由）
- **影響範囲**：Cursor 内で本バージョンを手動検索・インストールしたユーザー（最大 5,000 名）
- **Cursor 公式発表**：*"a modified Claude Code VS Code extension accessed through our Cursor IDE extension marketplace, impacting at most 5,000 users. This malicious version of the extension was not developed by Anthropic."*

### 3. なぜ「拡張機能のアンインストール」では駆除できないのか？
一般的なプラグインは拡張機能フォルダ内に留まりますが、本マルウェアはインストール直後の Node.js フック（Post-install Hook）を通じてエディタのサンドボックスを脱出し、OS 層に常駐プログラムを埋め込みます：
1. **SSH バックドア接続の確立**：`~/.ssh/authorized_keys` に攻撃者の公開鍵（識別子 `svc-telemetry`）を強制追加。
2. **システムデーモンを偽装した自動起動**：macOS の Apple コンポーネント等に偽装した常駐スクリプトを配置し、PC再起動時にも自動復活。

---

## 二、 全環境対応：完全点検＆駆除チェックリスト

Cursor を利用しているエンジニアは、直ちにターミナルを開き、以下の手順で監査を実施してください。

### 1. 全OS共通点検：SSH 認証鍵（authorized_keys）の照合【最優先】

macOS、Linux、OpenSSH Server を稼働させている Windows のいずれでも、まず認証済み公開鍵を確認します。

#### 手順 A：`svc-telemetry` コメントの検出
ターミナルで以下を実行します：
```bash
grep -n "svc-telemetry" ~/.ssh/authorized_keys
```
出力が空であれば初期コメントは存在しません。ただし、コメントが変更されている可能性があるため、**必ず手順 B の公開鍵指紋（Fingerprint）照合を行ってください**。

#### 手順 B：`ssh-keygen` による指紋の照合
登録されている全公開鍵の SHA256 指紋を出力します：
```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```
**🚨 照合対象の悪意ある指紋**：
```text
SHA256:0Z6R+DF64Do2nwQic8a7kkfDLNNk4IiK+2U2Rci4htk
```
一覧の中に上記のハッシュ値が存在する場合、バックドアが埋め込まれています。直ちに以下のコマンドで該当行を削除してください：

```bash
# 現在の authorized_keys をバックアップ
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak.$(date +%F)

# 該当コメントを含む行を削除
sed -i '' '/svc-telemetry/d' ~/.ssh/authorized_keys 2>/dev/null || sed -i '/svc-telemetry/d' ~/.ssh/authorized_keys
```
*(コメントが変更されている場合は、テキストエディタで `~/.ssh/authorized_keys` を開き、該当指纹を持つ公開鍵の行を手動で削除してください)*。

---

### 2. macOS 向け点検・完全駆除手順

macOS 環境では、Apple 純正デーモンに偽装した `LaunchAgent` が利用されます。

#### 手順 A：偽装ファイルの有無を確認
攻撃者は `com.apple.sysmond.plist` という名称で偽装しています。

> ⚠️ **重要な注意点**：
> macOS の純正システムデーモンは読み取り専用の `/System/Library/LaunchDaemons/com.apple.sysmond.plist` に配置されています。**こちらは Apple 公式の純正プロセスですので絶対に削除しないでください**！
> 攻撃対象ファイルは**ユーザーのホームディレクトリ内**（`~/Library/LaunchAgents/com.apple.sysmond.plist`）に作成されます。

ファイルの存在を確認：
```bash
ls -la ~/Library/LaunchAgents/com.apple.sysmond.plist
ls -la ~/.config/.smt
```

#### 手順 B：登録解除と実体ファイルの完全削除
いずれかのファイルが存在する場合、以下を順次実行して無力化します：
```bash
# 1. LaunchAgent の登録を解除して停止
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null || launchctl unload ~/Library/LaunchAgents/com.apple.sysmond.plist 2>/dev/null

# 2. 悪意ある plist 定義ファイルを削除
rm -f ~/Library/LaunchAgents/com.apple.sysmond.plist

# 3. バックドア実体バイナリを削除
rm -rf ~/.config/.smt

# 4. 残存プロセスの有無を確認
ps aux | grep -i "\.smt" | grep -v grep
```
もし `.smt` に一致するプロセスが動作している場合は、`kill -9 <PID>` で強制終了してください。

---

### 3. Linux 向け点検・駆除手順

Linux ディストリビューションでは、`crontab` の `@reboot` 設定によって自動起動されます。

#### 手順 A：crontab の確認
```bash
crontab -l
```
`~/.config/.smt` や以下のようなコマンドが登録されていないか確認します：
```text
@reboot ~/.config/.smt >/dev/null 2>&1
```

#### 手順 B：crontab の清掃とファイル削除
```bash
# 1. crontab のバックアップ
crontab -l > /tmp/crontab.bak

# 2. .smt を含む行を除去して再登録
crontab -l | grep -v "\.config/\.smt" | crontab -

# 3. 実体ファイルの削除
rm -rf ~/.config/.smt

# 4. プロセスの強制終了
pkill -f "\.config/\.smt"
```

---

### 4. Windows 向け点検・駆除手順

Windows 環境では、タスクスケジューラ（Task Scheduler）またはスタートアップフォルダが標的となります。

#### 手順 A：タスクおよびスタートアップの確認
PowerShell を管理者権限で開き、以下を実行します：
```powershell
# 1. sysmon を含む不審なタスクを検索
Get-ScheduledTask | Where-Object { $_.TaskName -like "*sysmon*" -or $_.TaskPath -like "*sysmon*" }

# 2. スタートアップフォルダ内の不審なスクリプトを確認
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup" | Where-Object { $_.Name -like "*sysmon*" }
```

#### 手順 B：タスクの登録解除とスクリプト削除
```powershell
# タスクの削除（タスク名が SysMon の場合）
Unregister-ScheduledTask -TaskName "SysMon" -Confirm:$false

# スタートアップスクリプトの削除
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\sysmon.cmd" -Force -ErrorAction SilentlyContinue
```

---

## 三、 二次被害防止：侵入が確認された場合に必要な対応

セキュリティ監査においては「すでに侵害された前提（Assume Breach）」での対応が不可欠です。もし該当する指紋や `.smt` ファイルが見つかった場合は、直ちに以下の対処を行ってください：

1. **SSH 秘密鍵の全面ローテーション**：
   攻撃者にシェルを奪取された場合、`~/.ssh/id_rsa` などの秘密鍵が盗み出されている可能性が極めて高いです。新しい鍵ペアを作成し、GitHub、GitLab、社内本番サーバーなどの公開鍵登録を速やかに更新してください。
2. **環境変数および API キーの無効化・再発行**：
   ローカルの `~/.zshrc`、`~/.bashrc`、プロジェクトの `.env` に記載された `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、クラウドインフラ（AWS/GCP）の認証情報は漏洩したものとみなし、直ちにクラウド側の管理画面から再発行してください。
3. **Cursor 上での問題プラグイン完全削除**：
   Cursor の拡張機能一覧から該当の不審版が完全に消去されていることを再確認してください。

---

## 四、 AI 時代の IDE サプライチェーンセキュリティ防衛策

今回の事件は単なる偶発的な事故ではなく、**急拡大する AI 開発環境を標的とした典型的なサプライチェーン攻撃**です。

Cursor や Windsurf、Trae といった次世代 AI IDE は圧倒的な生産性をもたらす一方、公式エコシステムからの分断によって第三者ストアの審査体制が攻撃者の標的となりやすい側面を持っています。

### 開発者が持つべき 3 つの防衛指針
- **発行元の真正性を確認する**：サードパーティ製 IDE 内で拡張機能を追加する際は、必ず GitHub の公式リポジトリや公式ドメインと照合すること。
- **機密性の高い作業はサンドボックス環境で行う**：不特定多数のプラグインや自律型コーディングエージェントを動かす際は、Docker や Dev Containers などの隔離環境を活用すること。
- **端末の資産監査を習慣化する**：`authorized_keys` や `LaunchAgents`、`crontab` の定期点検をセキュリティ習慣に組み込むこと。

---

> 💡 **元記事と稼働監視**：
> - 元記事: [FreeAI Dog 公式ブログ](https://freeaidog.com/ja/blog/cursor-claude-code-extension-backdoor-audit-guide/)
> - 無料AIモデルのリアルタイム稼働状況と遅延データは [FreeAI Dog 可用性レーダー](https://freeaidog.com/ja/radar/) をご覧ください。


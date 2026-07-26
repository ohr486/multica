# Multica CLI — AIエージェント向けインストールガイド

> **対象読者:** このドキュメントは、AIエージェント（Claude Code、Codex など）が読み取り、ステップごとに実行することを想定して作られています。各ステップには、実行すべき正確なシェルコマンド、期待される結果、エラー処理が含まれています。

## ユーザー向けの一行指示

次の指示をあなたのAIエージェントに渡してください。

```
Fetch https://github.com/multica-ai/multica/blob/main/CLI_INSTALL.md を取得し、その手順に従って、このマシンに Multica CLI をインストールし、ログインして、デーモンを起動してください。
```

---

## ステップ1: Multica CLI が既にインストールされているか確認する

次を実行します。

```bash
multica version
```

- **バージョン文字列が表示される場合**（例: `multica v0.x.x`）: **ステップ3** へスキップします。
- **コマンドが見つからない場合**: **ステップ2** へ進みます。

---

## ステップ2: Multica CLI をインストールする

> **Windows ユーザーへ:** 下記の [Option C: Windows (PowerShell)](#option-c-windows-powershell) へスキップしてください。

### Option A: Homebrew（推奨 — macOS/Linux）

Homebrew が利用可能か確認します。

```bash
which brew
```

`brew` が見つかった場合、Homebrew でインストールします。

```bash
brew install multica-ai/tap/multica
```

その後、確認します。

```bash
multica version
```

バージョンが正常に表示されたら、**ステップ3** へスキップします。

後でアップグレードするには、次を実行します。

```bash
brew upgrade multica-ai/tap/multica
```

### Option B: GitHub Releases からダウンロードする（macOS/Linux、Homebrew なし）

Homebrew が利用できない場合は、バイナリを直接ダウンロードします。

OS とアーキテクチャを検出し、正しいアーカイブをダウンロードします。

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')   # "darwin" or "linux"
ARCH=$(uname -m)                                # "x86_64" or "arm64"

# Normalize architecture name
if [ "$ARCH" = "x86_64" ]; then
  ARCH="amd64"
fi

# Get the latest release tag from GitHub
LATEST=$(curl -sI https://github.com/multica-ai/multica/releases/latest | grep -i '^location:' | sed 's/.*tag\///' | tr -d '\r\n')

# Download and extract
VERSION="${LATEST#v}"
curl -sL "https://github.com/multica-ai/multica/releases/download/${LATEST}/multica-cli-${VERSION}-${OS}-${ARCH}.tar.gz" -o /tmp/multica.tar.gz
tar -xzf /tmp/multica.tar.gz -C /tmp multica
sudo mv /tmp/multica /usr/local/bin/multica
rm /tmp/multica.tar.gz
```

確認します。

```bash
multica version
```

**失敗する場合:**
- `/usr/local/bin` が `$PATH` に含まれているか確認してください。
- Linux では `chmod +x /usr/local/bin/multica` が必要な場合があります。
- `sudo` が利用できない場合は、ユーザーが書き込み可能なディレクトリにインストールします: `mv /tmp/multica ~/.local/bin/multica` を実行し、`~/.local/bin` が `$PATH` に含まれていることを確認してください。

### Option C: Windows (PowerShell)

PowerShell で実行します（管理者権限は不要です）。

```powershell
irm https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.ps1 | iex
```

これは GitHub Releases から最新の Windows バイナリをダウンロードし、`%USERPROFILE%\.multica\bin\` にインストールして、ユーザーの PATH に追加します。

確認します。

```powershell
multica version
```

**失敗する場合:**
- 更新された PATH を反映させるため、ターミナルを再起動してください。
- Scoop を使用している場合、インストーラーは自動的にそれを使用します: `scoop bucket add multica https://github.com/multica-ai/scoop-bucket.git && scoop install multica`
- 実行ポリシーがスクリプトをブロックする場合: `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` を実行してから再試行してください。

---

## ステップ3: ログインする

次を実行します。

```bash
multica login
```

**重要:** このコマンドは OAuth 認証のためにブラウザウィンドウを開きます。ユーザーに次のように伝えてください。

> 「Multica のログイン用にブラウザウィンドウが開きます。ブラウザで認証を完了してから、こちらに戻ってきてください。」

コマンドの完了を待ちます。ユーザーが所属するすべてのワークスペースを自動的に検出し、監視対象に追加します。

確認します。

```bash
multica auth status
```

期待される出力には、認証されたユーザーとサーバー URL が表示されるはずです。

**ログインに失敗する場合:**
- ブラウザが利用できない場合（ヘッドレス環境）、ユーザーは `https://multica.ai/settings?tab=tokens` でパーソナルアクセストークンを生成し、次を実行できます: `multica login --token <mul_...>`（対話的に入力を求めるには、空の値で `--token=` を使用します）。
- サーバー URL をカスタマイズする必要がある場合: ログイン前に `multica config set server_url <url>` を実行します。

---

## ステップ4: デーモンを起動する

まず、デーモンが既に起動しているか確認します。

```bash
multica daemon status
```

- **ステータスが "running" の場合**: **ステップ5** へスキップします。
- **ステータスが "stopped" の場合**: 起動します。

```bash
multica daemon start
```

3秒待ってから、確認します。

```bash
multica daemon status
```

期待される出力には、検出されたエージェント（例: `claude`、`codex`、`copilot`、`opencode`、`openclaw`、`hermes`、`pi`、`cursor-agent`、`grok`）とともに `running` ステータスが表示されるはずです。

**デーモンの起動に失敗する場合:**
- ログを確認します: `multica daemon logs`
- ポートの競合が発生した場合、デーモンが別のプロファイルで既に起動している可能性があります。
- エージェントが検出されない場合、少なくとも1つの AI CLI（`claude`、`codex`、`copilot`、`opencode`、`openclaw`、`hermes`、`pi`、`cursor-agent`、または `grok`）がインストールされ、`$PATH` 上にあることを確認してください。

---

## ステップ5: すべてが動作していることを確認する

次を実行します。

```bash
multica daemon status
```

以下を確認します。
1. ステータスが `running` であること
2. 少なくとも1つのエージェントが一覧表示されていること（例: `claude`、`codex`、`copilot`、`opencode`、`openclaw`、`hermes`、`pi`、`cursor-agent`、または `grok`）
3. 少なくとも1つのワークスペースが監視されていること

エージェントの一覧が空の場合、ユーザーに次のように伝えてください。

> 「Multica デーモンは起動していますが、AIエージェントの CLI が検出されませんでした。サポートされている CLI（`claude`、`codex`、`copilot`、`opencode`、`openclaw`、`hermes`、`pi`、`cursor-agent`、または `grok`）を少なくとも1つインストールし、`multica daemon stop && multica daemon start` でデーモンを再起動してください。」

---

## まとめ

すべてのステップが完了したら、ユーザーに次のように伝えてください。

> 「Multica CLI がインストールされ、デーモンが起動しています。あなたのワークスペースのエージェントは、このマシンでタスクを実行できるようになりました。`multica workspace list` でワークスペースを管理し、`multica daemon logs -f` でデーモンのログを確認できます。」

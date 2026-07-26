# CLI とエージェントデーモンのガイド

`multica` CLI は、ローカルマシンを Multica に接続します。認証、ワークスペース管理、Issue トラッキングを扱い、AI タスクをローカルで実行するエージェントデーモンを起動します。

## インストール

### Homebrew（macOS/Linux）

```bash
brew install multica-ai/tap/multica
```

### ソースからビルド

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make build
cp server/bin/multica /usr/local/bin/multica
```

### 更新

```bash
brew upgrade multica-ai/tap/multica
```

インストールスクリプトや手動インストールの場合は、次を使います。

```bash
multica update
```

`multica update` はインストール方法を自動検出し、それに応じてアップグレードします。

## クイックスタート

```bash
# ワンコマンドセットアップ: 設定・認証・デーモン起動をまとめて実行
multica setup

# セルフホスト（ローカル）デプロイの場合:
multica setup self-host
```

またはステップごとに実行します。

```bash
# 1. 認証する（ログイン用にブラウザを開きます）
multica login

# 2. エージェントデーモンを起動する
multica daemon start

# 3. 完了 — 監視中のワークスペースのエージェントが、あなたのマシンでタスクを実行できるようになります
```

`multica login` は、あなたが所属するすべてのワークスペースを自動的に検出し、デーモンの監視リストに追加します。

## 認証

### ブラウザログイン

```bash
multica login
```

OAuth 認証のためにブラウザを開き、90日間有効なパーソナルアクセストークンを作成し、ワークスペースを自動設定します。

### トークンログイン

```bash
multica login --token <mul_...>
```

パーソナルアクセストークンを使って直接認証します。ヘッドレス環境で便利です。空の値で `--token=` を渡すと対話的に入力を求められます（そのためトークンがシェル履歴に残りません）。

### ステータス確認

```bash
multica auth status
```

現在のサーバー、ユーザー、トークンの有効性を表示します。

### ログアウト

```bash
multica auth logout
```

保存された認証トークンを削除します。

## エージェントデーモン

デーモンはローカルのエージェントランタイムです。マシン上で利用可能な AI CLI を検出し、それらを Multica サーバーに登録し、エージェントに作業が割り当てられたときにタスクを実行します。

### 起動

```bash
multica daemon start
```

デフォルトでは、デーモンはバックグラウンドで動作し、`~/.multica/daemon.log` にログを記録します。

フォアグラウンドで実行するには（デバッグに便利）:

```bash
multica daemon start --foreground
```

### 停止

```bash
multica daemon stop
```

### ステータス

```bash
multica daemon status
multica daemon status --output json
```

PID、稼働時間、検出されたエージェント、監視中のワークスペースを表示します。

### ログ

```bash
multica daemon logs              # Last 50 lines
multica daemon logs -f           # Follow (tail -f)
multica daemon logs -n 100       # Last 100 lines
```

### サポートされるエージェント

デーモンは PATH 上の以下の AI CLI を自動検出します。

| CLI | Command | Description |
|-----|---------|-------------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | `claude` | Anthropic のコーディングエージェント |
| [Codex](https://github.com/openai/codex) | `codex` | OpenAI のコーディングエージェント |
| [GitHub Copilot CLI](https://docs.github.com/en/copilot) | `copilot` | GitHub のコーディングエージェント（モデルは GitHub のエンタイトルメントによってルーティングされます） |
| OpenCode | `opencode` | オープンソースのコーディングエージェント |
| OpenClaw | `openclaw` | オープンソースのコーディングエージェント |
| Hermes | `hermes` | Nous Research のコーディングエージェント |
| Gemini | `gemini` | Google のコーディングエージェント |
| [Pi](https://pi.dev/) | `pi` | Pi コーディングエージェント |
| [Cursor Agent](https://cursor.com/) | `cursor-agent` | Cursor のヘッドレスコーディングエージェント |
| Kimi | `kimi` | Moonshot のコーディングエージェント |
| Kiro CLI | `kiro-cli` | Kiro ACP コーディングエージェント |
| [Qoder CLI](https://docs.qoder.com/) | `qodercli` | Qoder ACP コーディングエージェント |
| [Trae](https://docs.trae.cn/cli) | `traecli` | ByteDance TRAE CLI（`traecli acp serve` 経由の ACP） |
| [Grok Build CLI](https://docs.x.ai/) | `grok` | xAI Grok Build CLI（`grok agent stdio` 経由の ACP） |
| [Qwen Code](https://github.com/QwenLM/qwen-code) | `qwen` | Alibaba Qwen Code（stream-json 付きの `qwen -p`） |

少なくとも1つがインストールされている必要があります。デーモンは検出された各 CLI を利用可能なランタイムとして登録します。

### 仕組み

1. 起動時、デーモンはインストールされたエージェント CLI を検出し、監視中の各ワークスペースの各エージェントについてランタイムを登録します
2. 設定可能な間隔（デフォルト: 3s）でサーバーをポーリングし、クレームされたタスクを取得します
3. タスクが到着すると、隔離されたワークスペースディレクトリを作成し、エージェント CLI を起動し、結果をストリーミングで返します
4. デーモンが生きていることをサーバーが把握できるよう、ハートビートが定期的に送信されます（デフォルト: 15s）
5. シャットダウン時には、すべてのランタイムの登録が解除されます

### 設定

デーモンの動作はフラグまたは環境変数で設定します。

| Setting | Flag | Env Variable | Default |
|---------|------|--------------|---------|
| Poll interval | `--poll-interval` | `MULTICA_DAEMON_POLL_INTERVAL` | `3s` |
| Heartbeat interval | `--heartbeat-interval` | `MULTICA_DAEMON_HEARTBEAT_INTERVAL` | `15s` |
| Agent timeout | `--agent-timeout` | `MULTICA_AGENT_TIMEOUT` | `0`（上限なし。ウォッチドッグにより制限） |
| Codex semantic inactivity timeout | `--codex-semantic-inactivity-timeout` | `MULTICA_CODEX_SEMANTIC_INACTIVITY_TIMEOUT` | `10m` |
| OpenCode idle watchdog | — | `MULTICA_OPENCODE_IDLE_WATCHDOG` | `10m`（`0` は汎用アイドルウォッチドッグにフォールバック。延長は不可） |
| Max concurrent tasks | `--max-concurrent-tasks` | `MULTICA_DAEMON_MAX_CONCURRENT_TASKS` | `20` |
| Daemon ID | `--daemon-id` | `MULTICA_DAEMON_ID` | hostname |
| Device name | `--device-name` | `MULTICA_DAEMON_DEVICE_NAME` | hostname |
| Runtime name | `--runtime-name` | `MULTICA_AGENT_RUNTIME_NAME` | `Local Agent` |
| Workspaces root | — | `MULTICA_WORKSPACES_ROOT` | `~/multica_workspaces` |
| GC enabled | — | `MULTICA_GC_ENABLED` | `true`（無効化するには `false`/`0`） |
| GC scan interval | — | `MULTICA_GC_INTERVAL` | `2h` |
| GC TTL (done/cancelled issues) | — | `MULTICA_GC_TTL` | `24h` |
| GC orphan TTL (no `.gc_meta.json`) | — | `MULTICA_GC_ORPHAN_TTL` | `72h` |
| GC artifact TTL (open issues) | — | `MULTICA_GC_ARTIFACT_TTL` | `12h`（無効化するには `0`） |
| GC artifact patterns | — | `MULTICA_GC_ARTIFACT_PATTERNS` | `node_modules,.next,.turbo` |

#### ワークスペースのガベージコレクション

デーモンは定期的に `MULTICA_WORKSPACES_ROOT` をスキャンし、3つのモードでディスク領域を回収します。

- **タスク全体のクリーンアップ** — Issue のステータスが `done` または `cancelled` で、`MULTICA_GC_TTL` の間アイドルだった場合、タスクディレクトリ全体が削除されます。
- **孤立クリーンアップ** — `.gc_meta.json` を持たないタスクディレクトリ（例: デーモンのクラッシュで残ったもの）は、`MULTICA_GC_ORPHAN_TTL` を超えると削除されます。
- **アーティファクトのみのクリーンアップ** — タスクが完了してから少なくとも `MULTICA_GC_ARTIFACT_TTL` が経過しているが Issue がまだオープンの場合、ディレクトリの basename が `MULTICA_GC_ARTIFACT_PATTERNS` に一致する再生成可能なビルド出力が削除されます。デーモンは管理下の正確なパス `codex-home/.sandbox-bin` も回収します。`completed_at` を持たない古いタスクメタデータは、その `.gc_meta.json` ファイルが `MULTICA_GC_ORPHAN_TTL` の間アイドルになった後、この管理下限定のクリーンアップの対象になります。タスクの残り（ソース、`.git`、`output/`、`logs/`、`.gc_meta.json`、Codex の認証/設定/セッション状態）は、エージェントが再開できるよう保持されます。

設定されたパターンは basename のみです（`/` や `\` を含むエントリは黙って破棄されます）。また `.git` サブツリーには決して降りていきません。管理下の Codex キャッシュは正確な相対パスで一致するため、オペレーターが明示的にその basename を `MULTICA_GC_ARTIFACT_PATTERNS` に追加しない限り、リポジトリ自身の `.sandbox-bin` は削除されません。デフォルトのリスト（`node_modules`、`.next`、`.turbo`）は意図的に狭くしてあります。リポジトリが他の再生成可能なディレクトリを一貫して生成する場合は、デプロイごとに拡張してください（例: `MULTICA_GC_ARTIFACT_PATTERNS=node_modules,.next,.turbo,target,__pycache__`）。管理下の Codex キャッシュを含め、アーティファクトのクリーンアップを完全に無効化するには、`MULTICA_GC_ARTIFACT_TTL=0` を設定してください。

エージェント固有の上書き:

| Variable | Description |
|----------|-------------|
| `MULTICA_CLAUDE_PATH` | `claude` バイナリへのカスタムパス |
| `MULTICA_CLAUDE_MODEL` | 使用する Claude モデルを上書きする |
| `MULTICA_CLAUDE_ARGS` | Claude Code 実行時のデフォルトの追加引数 |
| `MULTICA_CODEX_PATH` | `codex` バイナリへのカスタムパス |
| `MULTICA_CODEX_MODEL` | 使用する Codex モデルを上書きする |
| `MULTICA_CODEX_ARGS` | Codex 実行時のデフォルトの追加引数 |
| `MULTICA_COPILOT_PATH` | `copilot` バイナリへのカスタムパス |
| `MULTICA_COPILOT_MODEL` | 使用する Copilot モデルを上書きする（注: GitHub Copilot はモデルをアカウントのエンタイトルメント経由でルーティングするため、これは尊重されない場合があります） |
| `MULTICA_OPENCODE_PATH` | `opencode` バイナリへのカスタムパス |
| `MULTICA_OPENCODE_MODEL` | 使用する OpenCode モデルを上書きする |
| `MULTICA_OPENCLAW_PATH` | `openclaw` バイナリへのカスタムパス |
| `MULTICA_OPENCLAW_MODEL` | 使用する OpenClaw モデルを上書きする |
| `MULTICA_HERMES_PATH` | `hermes` バイナリへのカスタムパス |
| `MULTICA_HERMES_MODEL` | 使用する Hermes モデルを上書きする |
| `MULTICA_GEMINI_PATH` | `gemini` バイナリへのカスタムパス |
| `MULTICA_GEMINI_MODEL` | 使用する Gemini モデルを上書きする |
| `MULTICA_PI_PATH` | `pi` バイナリへのカスタムパス |
| `MULTICA_PI_MODEL` | 使用する Pi モデルを上書きする |
| `MULTICA_CURSOR_PATH` | `cursor-agent` バイナリへのカスタムパス |
| `MULTICA_CURSOR_MODEL` | 使用する Cursor Agent モデルを上書きする |
| `MULTICA_KIMI_PATH` | `kimi` バイナリへのカスタムパス |
| `MULTICA_KIMI_MODEL` | 使用する Kimi モデルを上書きする |
| `MULTICA_KIRO_PATH` | `kiro-cli` バイナリへのカスタムパス |
| `MULTICA_KIRO_MODEL` | 使用する Kiro モデルを上書きする |
| `MULTICA_QODER_PATH` | `qodercli` バイナリへのカスタムパス |
| `MULTICA_QODER_MODEL` | 使用する Qoder モデルを上書きする |
| `MULTICA_TRAECLI_PATH` | `traecli` バイナリへのカスタムパス |
| `MULTICA_TRAECLI_MODEL` | 使用する Trae モデルを上書きする（ログイン済みの traecli カタログのモデル ID。例: `Doubao-Seed-2.1-Pro`） |
| `MULTICA_GROK_PATH` | `grok` バイナリへのカスタムパス（デフォルトは PATH 上の `grok`。多くの場合 `~/.grok/bin/grok`） |
| `MULTICA_GROK_MODEL` | 使用する Grok モデルを上書きする（例: `grok-4.5`） |
| `MULTICA_QWEN_PATH` | `qwen` バイナリへのカスタムパス |
| `MULTICA_QWEN_MODEL` | 使用する Qwen Code モデルを上書きする |
| `MULTICA_QWEN_ARGS` | デーモン全体の追加 Qwen 引数（POSIX シェルワードでパース。管理下のプロトコルフラグはフィルタされます） |

以前に生成された `~/.multica/hooks` ラッパーが PATH の先頭にあり、同じコマンド名を再び呼び出す場合、デーモンは組み込みエージェントの検出時にその hooks ディレクトリをスキップし、その背後にある本物のバイナリパスを記録します。それでも `claude`、`codex`、`hermes` を手動で実行したときに対話シェルが再帰する場合は、シェルの起動ファイルから hooks エントリを削除するか、ラッパーの本体を絶対パスの `exec /path/to/real-binary "$@"` に置き換えてください。

デーモンは Qoder を `qodercli --yolo --acp` として起動します。これは Qoder の ACP「権限バイパス」モードに合わせたもので、ヘッドレス実行でツール実行が対話的な承認でブロックされないようにするためです。
デーモンは Qwen Code を `qwen -p <prompt> --output-format stream-json` として起動します。タスクの概要を `QWEN.md` に書き込みます。エージェントが管理下の `mcp_config` を持つ場合、デーモンは実行ごとに 0600 の JSON ファイルを書き込み、`--mcp-config <path>` で渡し、プロセス終了後に削除します。null 設定は Qwen Code のネイティブ MCP 設定を保持します。


`MULTICA_CLAUDE_ARGS`、`MULTICA_CODEX_ARGS`、`MULTICA_QWEN_ARGS` は POSIX シェルワードのクオートでパースされるため、`--model "gpt-5.1 codex" --sandbox read-only` のような値はシェルのコマンドラインのように分割されます。エージェント引数は次の順序で適用されます: Multica のハードコードされたデフォルト、デーモン全体の env デフォルト、そしてタスクごとの `custom_args`。

### セルフホストサーバー

セルフホストされた Multica インスタンスに接続する場合、最も簡単な方法は次のとおりです。

```bash
# ワンコマンド — localhost 向けに設定し、認証し、デーモンを起動します
multica setup self-host

# またはカスタムドメインを使うオンプレミスの場合:
multica setup self-host --server-url https://api.example.com --app-url https://app.example.com
```

または手動で設定します。

```bash
# URL を個別に設定する
multica config set server_url http://localhost:8080
multica config set app_url http://localhost:3000

# TLS を使う本番環境の場合:
# multica config set server_url https://api.example.com
# multica config set app_url https://app.example.com

multica login
multica daemon start
```

### プロファイル

プロファイルを使うと、同じマシンで複数のデーモンを実行できます。例えば、本番用に1つ、ステージングサーバー用に1つといった具合です。

```bash
# ステージングプロファイルをセットアップする
multica setup self-host --profile staging --server-url https://api-staging.example.com --app-url https://staging.example.com

# そのデーモンを起動する
multica daemon start --profile staging

# デフォルトプロファイルは別に動作する
multica daemon start
```

各プロファイルは、それぞれ独自の設定ディレクトリ（`~/.multica/profiles/<name>/`）、デーモン状態、ヘルスポート、ワークスペースルートを持ちます。

## ワークスペース

### 複数ワークスペースの操作

すべてのコマンドは単一のワークスペースに対して実行されます。CLI は次の順序でどれを使うかを解決します（優先順位の高い順）。

1. コマンドの `--workspace-id <id>` フラグ
2. `MULTICA_WORKSPACE_ID` 環境変数
3. 現在のプロファイルに保存されたデフォルトワークスペース（`multica workspace switch` または `multica login` で設定）

`multica workspace switch <id|slug>` は、デフォルトワークスペースを変更する日常的な方法です。保存された状態を一切持ちたくないスクリプトやヘッドレスのセットアップでは、`--workspace-id` フラグや環境変数を優先してください。`multica config set workspace_id <id>` は `switch` の低レベルな等価物です（同じ設定を書き込みますが、アクセスチェックはスキップします）。

組織やアカウント間で完全な分離が必要な場合——別々のトークン、別々のデーモン、別々の設定ディレクトリ——は、代わりに `--profile <name>` を使ってください。各プロファイルは独自のデフォルトワークスペースを保持します。

### ワークスペース一覧

```bash
multica workspace list
multica workspace list --full-id
multica workspace list --output json
```

現在のデフォルトワークスペースは `*` で示されます。テーブル出力は短い UUID プレフィックスを表示します。正規の UUID が必要な場合は `--full-id` を渡してください。

### デフォルトワークスペースの切り替え

```bash
multica workspace switch <workspace-id>
multica workspace switch <slug>
```

ワークスペースへのアクセス権を確認したうえで、現在のプロファイルのデフォルトとして設定します。以降、`--workspace-id` と `MULTICA_WORKSPACE_ID` を指定しないコマンドはこのワークスペースを対象とします。デフォルト以外のプロファイルのワークスペースを変更したい場合は `--profile` を併用してください。

### 詳細の取得

```bash
multica workspace get <workspace-id>
multica workspace get <workspace-id> --output json
```

`<workspace-id>` を渡さない場合は現在のデフォルトワークスペースに解決されるため、`multica workspace get` は「今どのワークスペースにいるか？」の確認にも使えます。

### メンバー一覧

```bash
multica workspace member list <workspace-id>
```

## Issue

### Issue 一覧

```bash
multica issue list
multica issue list --status in_progress
multica issue list --priority urgent --assignee "Agent Name"
multica issue list --assignee-id 5fb87ac7-23b5-4a7a-81fa-ed295a54545d
multica issue list --full-id
multica issue list --limit 20 --output json
multica issue list --status todo --sort position       # board order (the default)
multica issue list --sort created_at --direction desc  # newest first
```

テーブル出力は `MUL-123` のようなルーティング可能な Issue `KEY` を表示します。そのキーを `issue get`、`issue comment list`、`issue status`、`--parent` などの後続コマンドにコピーしてください。正規の UUID が必要な場合は `--full-id` を追加します。利用可能なフィルター: `--status`、`--priority`、`--assignee` / `--assignee-id`、`--project`、`--metadata`、`--limit`。名前が重複する場合は、あいまいさのないフィルタリングのために `--assignee-id <uuid>` を使ってください。

結果はデフォルトでボード順（`position` の昇順）で返ります。カラムを変更するには `--sort`（`position`、`title`、`created_at`、`start_date`、`due_date`、`priority`）を、順序を反転するには `--direction asc|desc` を渡します。`position` は常に昇順です（これは手動のドラッグ順のため）。そのため `--sort` が `position` または省略された場合は `--direction` が拒否されます——`title`、`created_at`、`start_date`、`due_date`、`priority` のいずれかでのみ使ってください。

`--metadata key=value`（繰り返し可能。AND で結合）を使うと、Issue ごとのメタデータでフィルタできます。値は JSON としてパースされます: `true`/`false` は bool に、数値は数値に、それ以外は文字列になります。値が数値として解釈されてしまう場合に文字列を強制するには `'"42"'` のように囲みます。

```bash
multica issue list --metadata pipeline_status=waiting_review
multica issue list --metadata pr_number=482 --metadata is_blocked=true
```

### Issue の取得

```bash
multica issue get <id>
multica issue get <id> --output json
```

### Issue の作成

```bash
multica issue create --title "Fix login bug" --description "..." --priority high --assignee "Lambda"
multica issue create --title "Fix login bug" --assignee-id 5fb87ac7-23b5-4a7a-81fa-ed295a54545d
```

フラグ: `--title`（必須）、`--description`、`--status`、`--priority`、`--assignee` / `--assignee-id`、`--parent`、`--project`、`--due-date`。`multica workspace member list --output json` / `multica agent list --output json` が返す ID に対してスクリプトを書く場合は、`--assignee-id <uuid>`（`--assignee` とは排他）を渡してください。

### Issue の更新

```bash
multica issue update <id> --title "New title" --priority urgent
multica issue update <id> --position 4.5
```

`--position` はボードカラム内の生の順序値を設定します（小さいほど先にソートされます）。相対的な移動には `issue reorder` の方が簡単です。値を自動で計算してくれるからです。

### Issue の並べ替え

Issue を現在のステータスカラム内で移動します。新しい順序値は、ボードのドラッグ＆ドロップが計算するのと同じ方法で計算されるため、CLI と UI で Issue の着地位置が一致します。

```bash
multica issue reorder <id> --top              # top of its status column
multica issue reorder <id> --bottom           # bottom of its status column
multica issue reorder <id> --before <other>   # directly above another issue in the same column
multica issue reorder <id> --after  <other>   # directly below another issue in the same column
```

`--top`、`--bottom`、`--before`、`--after` のうち、ちょうど1つを選びます。並べ替えは Issue の現在のカラム内にとどまるため、`--before` / `--after` は同じカラム内の Issue を指定する必要があります。Issue を別のカラムに移動するには、まず `issue status` でステータスを変更してから、新しいカラム内で並べ替えてください。

### Issue の割り当て

```bash
multica issue assign <id> --to "Lambda"
multica issue assign <id> --to-id 5fb87ac7-23b5-4a7a-81fa-ed295a54545d
multica issue assign <id> --unassign
```

正規の UUID で割り当てるには `--to-id <uuid>`（`--to` とは排他）を渡します。メンバーとエージェントで名前が重複する場合に便利です。

### ステータスの変更

```bash
multica issue status <id> in_progress
```

有効なステータス: `backlog`、`todo`、`in_progress`、`in_review`、`done`、`blocked`、`cancelled`。

### コメント

```bash
# List comments — flat timeline, chronological. Hard cap of 2000 rows; on
# long-running issues prefer one of the thread-aware reads below to keep
# context windows tight.
multica issue comment list <issue-id>

# Single thread (root + every descendant). Anchor may be the root itself
# or any reply inside the thread — the server walks up to the root.
multica issue comment list <issue-id> --thread <comment-id>

# Single thread, capped to the N most recent replies. The thread root is
# always included (even with --tail 0), so an agent landing on a long
# thread keeps the "what is this about" context without dragging hundreds
# of replies into its prompt.
multica issue comment list <issue-id> --thread <comment-id> --tail 30

# Scroll older replies inside the same thread. --before / --before-id are
# the reply cursor that the previous response emitted on stderr as
# `Next reply cursor: --before <ts> --before-id <reply-id>`.
multica issue comment list <issue-id> --thread <comment-id> --tail 30 \
    --before <ts> --before-id <reply-id>

# Most recently active threads (root + every descendant), grouped by
# thread. Returns N complete conversational arcs, oldest-active first so
# the freshest thread sits closest to "now" in an agent prompt.
multica issue comment list <issue-id> --recent 10

# Scroll older threads. Under --recent, --before / --before-id are a
# THREAD cursor (thread last_activity_at + root id), emitted on stderr as
# `Next thread cursor: --before <ts> --before-id <root-id>`.
multica issue comment list <issue-id> --recent 10 \
    --before <ts> --before-id <root-id>

# Incremental polling. Combines with --thread or --recent; filters out
# replies created on or before <ts> from the page (the thread root is
# exempt so the agent always gets context).
multica issue comment list <issue-id> --thread <comment-id> --tail 30 \
    --since <RFC3339-timestamp>

# Add a comment
multica issue comment add <issue-id> --content "Looks good, merging now"

# Reply to a specific comment
multica issue comment add <issue-id> --parent <comment-id> --content "Thanks!"

# Delete a comment
multica issue comment delete <comment-id>
```

**`--before` / `--before-id` のセマンティクスはページングモードに依存します**。これは設計によるものです——同じフラグ、異なるスコープです。

| Mode | What the cursor walks | stderr label |
| --- | --- | --- |
| `--recent N` | より古い*スレッド*（last_activity_at, root_id） | `Next thread cursor` |
| `--thread <id> --tail N` | そのスレッド内のより古い*返信*（created_at, id） | `Next reply cursor` |

これら2つのモード以外（`--tail` なしの `--thread`、または `--thread` も `--recent` もなし）では、カーソルフラグは静かに no-op にならないよう拒否されます。サーバーはカーソルヘッダー（`X-Multica-Next-Before` / `X-Multica-Next-Before-Id`）を、より古いページが実際に存在するときにのみ出力します——ちょうど境界のページ（例: ちょうど3件の返信を持つスレッドでの `--tail 3`）は、呼び出し側がページングを止められるよう、意図的にカーソルを返しません。

`--since` が `--recent` または `--thread --tail` と組み合わされると、カーソルの対象自体が `since` より古い場合、サーバーはさらにカーソルを抑制します。より古いページは厳密により古い行をたどるため、`> since` を満たすこともできません——そこでカーソルを出力しても、呼び出し側がスレッド / Issue の先頭に到達するまで、ルートのみのページを返し続けるだけになります。インクリメンタルなポーリングは、カーソルの対象がウォーターマークより前に落ちる最初のページで止まります。

### メタデータ

Issue ごとのメタデータは、エージェントがパイプラインの状態（PR 番号、パイプラインステータス、waiting_on など）を追跡するために使う小さな KV マップです。キーは `^[a-zA-Z_][a-zA-Z0-9_.-]{0,63}$` に一致し、値はプリミティブ（string / number / bool）で、Issue あたり最大 50 キー、blob は 8KB で上限が設定されています。

書き込みのハードルは高いです。値をピン留めするのは、その Issue にとって実質的に重要であり、かつ同じ Issue の将来の実行で再読される可能性が高い場合に限ってください（PR の URL、デプロイの URL、何にブロックされているか）。ほとんどの実行では新しいキーをゼロ個書き込みます——それが想定されるケースです。`attempts` のようなランタイムの帳簿付け、単一実行の調査メモ、大きなログ、シークレット/トークン、説明/コメントのコピーはピン留めしないでください——アンチパターンの完全なリストはエージェントランタイムのプロンプトを参照してください。

```bash
# List every key on an issue
multica issue metadata list <issue-id>

# Read a single key
multica issue metadata get <issue-id> --key pipeline_status

# Write a single key — value auto-typed (true/false → bool, numbers → number, else string)
multica issue metadata set <issue-id> --key pipeline_status --value waiting_review
multica issue metadata set <issue-id> --key pr_number --value 482
multica issue metadata set <issue-id> --key is_blocked --value true

# Force a specific type when sniffing would pick the wrong one
multica issue metadata set <issue-id> --key code --value 42 --type string

# Remove a key
multica issue metadata delete <issue-id> --key pipeline_status
```

すべての書き込みは単一キーでアトミックです——異なるキーを書き込む並行エージェントは、互いの更新を失いません。クエリするには `multica issue list --metadata key=value` を使ってください（前述の *Issue 一覧* を参照）。

### 購読者

```bash
# List subscribers of an issue
multica issue subscriber list <issue-id>

# Subscribe yourself to an issue
multica issue subscriber add <issue-id>

# Subscribe another member or agent by name
multica issue subscriber add <issue-id> --user "Lambda"

# Unsubscribe yourself
multica issue subscriber remove <issue-id>

# Unsubscribe another member or agent
multica issue subscriber remove <issue-id> --user "Lambda"
```

購読者は Issue のアクティビティ（新しいコメント、ステータス変更など）についての通知を受け取ります。`--user` を指定しない場合、コマンドは呼び出し元に対して作用します。

### 実行履歴

```bash
# List all execution runs for an issue
multica issue runs <issue-id>
multica issue runs <issue-id> --full-id
multica issue runs <issue-id> --output json

# View messages for a specific execution run
multica issue run-messages <task-id>
multica issue run-messages <short-task-id> --issue <issue-id>
multica issue run-messages <task-id> --output json

# Incremental fetch (only messages after a given sequence number)
multica issue run-messages <task-id> --since 42 --output json

# Aggregated token usage for an issue (sum across all its task runs)
multica issue usage <issue-id>
multica issue usage <issue-id> --output json
```

`usage` コマンドは、Issue のすべてのタスク実行を合算した集計トークン使用量を返します: 入力トークン、出力トークン、キャッシュの読み取り/書き込みトークン、実行回数（`task_count`）。これは `GET /api/issues/<id>/usage` をラップしたもので、Issue 詳細ビューが表示するのと同じ数値です。課金/コストのツールに渡すには `--output json` を使ってください。

`runs` コマンドは、実行中のタスクを含め、Issue の過去および現在のすべての実行を表示します。テーブル出力はデフォルトで短いタスク UUID プレフィックスを使います。正規のタスク UUID を表示するには `--full-id` を渡してください。`run-messages` コマンドは完全なタスク UUID を直接受け付けます。コピーした短いタスクプレフィックスは `--issue <issue-id>` でスコープを指定する必要があり、これにより CLI はその Issue の実行のみをチェックします。これは単一実行の詳細なメッセージログ（ツール呼び出し、思考、テキスト、エラー）を表示します。進行中の実行を効率的にポーリングするには `--since` を使ってください。

## プロジェクト

プロジェクトは関連する Issue（例: スプリント、エピック、ワークストリーム）をグループ化します。すべてのプロジェクトはワークスペースに属し、任意でリード（メンバーまたはエージェント）を持てます。

### プロジェクト一覧

```bash
multica project list
multica project list --status in_progress
multica project list --output json
```

利用可能なフィルター: `--status`。

### プロジェクトの取得

```bash
multica project get <id>
multica project get <id> --output json
```

### プロジェクトの作成

```bash
multica project create --title "2026 Week 16 Sprint" --icon "🏃" --lead "Lambda"
```

フラグ: `--title`（必須）、`--description`、`--status`、`--icon`、`--lead`、`--start-date`、`--due-date`。日付は暦日（`YYYY-MM-DD`）です。

### プロジェクトの更新

```bash
multica project update <id> --title "New title" --status in_progress
multica project update <id> --lead "Lambda"
multica project update <id> --due-date 2026-04-15
```

フラグ: `--title`、`--description`、`--status`、`--icon`、`--lead`、`--start-date`、`--due-date`。日付フラグでは、空文字列（例: `--start-date ""`）を渡すと日付をクリアできます。

### ステータスの変更

```bash
multica project status <id> in_progress
```

有効なステータス: `planned`、`in_progress`、`paused`、`completed`、`cancelled`。

### プロジェクトの削除

```bash
multica project delete <id>
```

### Issue とプロジェクトの関連付け

`issue create` / `issue update` の `--project` フラグを使って Issue をプロジェクトに紐付けたり、`issue list` の `--project` で Issue をプロジェクトでフィルタしたりできます。

```bash
multica issue create --title "Login bug" --project <project-id>
multica issue update <issue-id> --project <project-id>
multica issue list --project <project-id>
```

## セットアップ

```bash
# One-command setup for Multica Cloud: configure, authenticate, and start the daemon
multica setup

# For local self-hosted deployments
multica setup self-host

# Custom ports
multica setup self-host --port 9090 --frontend-port 4000

# On-premise with custom domains
multica setup self-host --server-url https://api.example.com --app-url https://app.example.com
```

`multica setup` は、CLI を設定し、認証のためにブラウザを開き、デーモンを起動します——すべて1ステップで。Multica Cloud の代わりにセルフホストサーバーに接続するには `multica setup self-host` を使ってください。

## 設定

### 設定の表示

```bash
multica config show
```

設定ファイルのパス、サーバー URL、アプリ URL、デフォルトワークスペースを表示します。

### 値の設定

```bash
multica config set server_url https://api.example.com
multica config set app_url https://app.example.com
multica config set workspace_id <workspace-id>
```

`config set workspace_id <id>` は低レベルなインターフェースです——ワークスペースが存在するか、あなたにアクセス権があるかを確認せず、値をそのまま書き込みます。日常的なワークスペースの変更には `multica workspace switch <id|slug>` を優先してください。保存前に両方のチェックを行います。

## Autopilot コマンド

Autopilot は、エージェントタスクをディスパッチする（Issue を作成するか、エージェントを直接実行する）、スケジュール/トリガーされる自動化です。

### Autopilot 一覧

```bash
multica autopilot list
multica autopilot list --full-id
multica autopilot list --status active --output json
```

Autopilot のテーブル ID は短い UUID プレフィックスです。後続の Autopilot コマンドは、現在のワークスペースで一意である場合、コピーしたプレフィックスを受け付けます。正規の UUID を表示するには `--full-id` を使ってください。

### Autopilot の詳細取得

```bash
multica autopilot get <id>
multica autopilot get <id> --output json   # includes triggers
```

### 作成 / 更新 / 削除

```bash
multica autopilot create \
  --title "Nightly bug triage" \
  --description "Scan todo issues and prioritize." \
  --agent "Lambda" \
  --mode create_issue \
  --subscriber "Alice"

multica autopilot update <id> --status paused
multica autopilot update <id> --description "New prompt"
multica autopilot update <id> --subscriber "Alice" --subscriber "Bob"
multica autopilot update <id> --clear-subscribers
multica autopilot delete <id>
```

`--mode` は `create_issue`（実行ごとに新しい Issue を作成してエージェントに割り当てる）または `run_only`（Issue を作成せずに直接エージェントタスクをエンキューする）を受け付けます。`--agent` は名前または UUID を受け付けます。
`--subscriber` はワークスペースメンバー名またはユーザー ID を受け付け、繰り返し指定できます。更新時には Autopilot の購読者テンプレートを置き換えます。購読者は、`create_issue` の Autopilot によって作成された Issue についてインボックス通知を受け取ります。すべての Autopilot 購読者を削除するには `--clear-subscribers` を使ってください。

### 手動トリガー

```bash
multica autopilot trigger <id>            # Fires the autopilot once, returns the run
```

### 実行履歴

```bash
multica autopilot runs <id>
multica autopilot runs <id> --limit 50 --output json
```

### スケジュールトリガー

```bash
multica autopilot trigger-add <autopilot-id> --cron "0 9 * * 1-5" --timezone "America/New_York"
multica autopilot trigger-update <autopilot-id> <trigger-id> --enabled=false
multica autopilot trigger-delete <autopilot-id> <trigger-id>
```

現在、CLI 経由で公開されているのは cron ベースの `schedule` トリガーのみです。データモデルは `webhook` と `api` の種類も定義していますが、それらを発火させるサーバーエンドポイントはまだないため、ここでは公開されていません。

## その他のコマンド

```bash
multica version              # Show CLI version and commit hash
multica update               # Update to latest version
multica agent list           # List agents in the current workspace
```

## 出力フォーマット

ほとんどのコマンドは2つのフォーマットで `--output` をサポートします。

- `table` — 人間が読みやすいテーブル（list コマンドのデフォルト）
- `json` — 構造化された JSON（スクリプトや自動化に便利）

```bash
multica issue list --output json
multica daemon status --output json
```

## エラーメッセージ

CLI は、トップレベルのハンドラーに返されるコマンドエラーを、単一のユーザー向け翻訳レイヤー（`server/internal/cli/errors.go`）を通して集約します。そのため、ターミナルに表示されるのは、生の Go エラーや HTTP ステータス行、内部の `resolve issue: ...` チェーンではなく、短く実用的な一文になります。（いくつかのコマンドは独自の出力を表示したり、意図的な高速プローブを実行したりします——例えば `setup` の短い `/health` 到達性チェック——これらはこのレイヤーを通りません。）内部の詳細は依然として必要に応じて利用できます（`--debug` を参照）。

### 表示されるもの

- **親しみやすい単一行のメッセージ。** トランスポート障害（タイムアウト、DNS、接続拒否、TLS）と HTTP ステータス障害（401/403/404/409/400·422/429/5xx）は、それぞれ次のステップを添えた明確な一文としてレンダリングされます——例えばタイムアウトはネットワークの確認や `MULTICA_HTTP_TIMEOUT` の引き上げを提案し、401 は `multica login` の実行を伝えます。
- **サーバー提供のバリデーションメッセージは保持されます。** サーバーからのメッセージを持つ 400/422 では、そのメッセージがそのまま表示されます（`Invalid request: <server message>`）。メッセージがない場合にのみ、汎用の「値を確認 / --help を付けて実行」のヒントが表示されます。
- **デフォルトでは内部情報を漏らしません。** 生の URL、ステータス行、JSON ボディ、内部の動詞チェーンは、要求しない限り隠されます。

### 言語

メッセージはデフォルトで **英語** で、CLI の他のヘルプ出力と一致します。`LC_ALL`、`LC_MESSAGES`、`LANG`（この優先順位で）で中国語ロケールが検出されると、メッセージは **中国語** に切り替わります。フラグは不要で、いつものようにロケールを設定します。

```bash
LANG=zh_CN.UTF-8 multica issue get MUL-9999   # 错误信息显示为中文
```

### 終了コード

プロセスの終了コードは階層化されており、スクリプトが失敗のクラスに応じて分岐できます。

| Exit code | Meaning |
| --- | --- |
| `0` | 成功 |
| `1` | 汎用 / 未分類のエラー |
| `2` | ネットワークエラー（タイムアウト、DNS、接続拒否、TLS、オフライン） |
| `3` | 認証 / 認可（HTTP 401, 403） |
| `4` | 見つからない（HTTP 404） |
| `5` | バリデーション（HTTP 400, 422） |

```bash
multica issue get MUL-9999
if [ $? -eq 4 ]; then echo "no such issue"; fi
```

### 完全な詳細を見る（`--debug`）

グローバルの `--debug` フラグを渡す（または `MULTICA_DEBUG=1` を設定する）と、親しみやすいメッセージの下に、完全な元のエラーチェーン——内部の動詞チェーン、リクエストのメソッド/パス/ステータス、生のサーバーボディ——が表示されます。バグを報告する必要があるときや、サーバーが正確に何を返したかを理解したいときに使ってください。

```bash
multica issue list --debug
MULTICA_DEBUG=1 multica issue update MUL-1234 --title "x"
```

### リクエストタイムアウト

API リクエストはデフォルトで30秒のタイムアウトを使います。遅いネットワークにいるときは `MULTICA_HTTP_TIMEOUT` で上書きしてください。Go の duration（`45s`、`2m`）または秒数の単純な数値（`45`）を受け付けます。コマンドレベルのデッドラインは常に少なくともこの値になるため、引き上げるとすべてのコマンドに反映されます。

```bash
MULTICA_HTTP_TIMEOUT=60s multica issue list
```

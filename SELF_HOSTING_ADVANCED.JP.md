# セルフホスティング — 高度な設定

このドキュメントは、セルフホストされた Multica デプロイの高度な設定を扱います。クイックスタートガイドについては [SELF_HOSTING.JP.md](SELF_HOSTING.JP.md) を参照してください。

## 設定

すべての設定は環境変数で行います。出発点として `.env.example` をコピーしてください。

### 必須の変数

| 変数 | 説明 | 例 |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL 接続文字列 | `postgres://multica:multica@localhost:5432/multica?sslmode=disable` |
| `JWT_SECRET` | **デフォルトから必ず変更すること。** JWT トークンに署名するための秘密鍵。長いランダム文字列を使ってください。 | `openssl rand -hex 32` |
| `FRONTEND_ORIGIN` | フロントエンドが配信される URL（CORS に使用） | `https://app.example.com` |

### データベースプールのチューニング（任意）

これらは妥当なデフォルト値を持っており、大規模または制約のあるデプロイをチューニングするときにのみ設定が必要です。優先順位（高い順）: 環境変数 → `DATABASE_URL` の `pool_*` クエリパラメータ → 組み込みのデフォルト。

| 変数 | 説明 | デフォルト |
|----------|-------------|---------|
| `DATABASE_MAX_CONNS` | Pod あたりの pgxpool 最大接続数。`pod_count × DATABASE_MAX_CONNS` は Postgres の `max_connections` 上限を十分に下回るべきです。前段に接続プーラー（PgBouncer / RDS Proxy / Supavisor）がある場合は、大幅に引き上げられます。 | `25` |
| `DATABASE_MIN_CONNS` | Pod あたりの pgxpool のウォームなベースライン接続数。`DATABASE_MAX_CONNS` に自動的にクランプされます。 | `5` |

### メール（認証に必須）

Multica は2つのメールバックエンドをサポートします。`SMTP_HOST` が設定されている場合はそれが優先され、そうでなければ `RESEND_API_KEY` が使われます。どちらも設定されていない場合、検証コードはサーバーログに出力されます——そこからコピーしてログインしてください。

#### オプション A: Resend（クラウドデプロイに推奨）

| 変数 | 説明 |
|----------|-------------|
| `RESEND_API_KEY` | あなたの Resend API キー |
| `RESEND_FROM_EMAIL` | 送信元メールアドレス（デフォルト: `noreply@multica.ai`） |

#### オプション B: SMTP リレー（セルフホスト / オンプレミスデプロイ向け）

デプロイ環境がパブリックインターネットに到達できない場合、またはすでに社内メールリレー（例: Exchange、Postfix、SendGrid オンプレミス）がある場合は、このオプションを使います。

| 変数 | 説明 | デフォルト |
|----------|-------------|----------|
| `SMTP_HOST` | SMTP リレーのホスト名（これを設定すると SMTP モードが有効になる） | - |
| `SMTP_PORT` | SMTP ポート | `25` |
| `SMTP_USERNAME` | SMTP ユーザー名（認証なしリレーの場合は空のまま） | - |
| `SMTP_PASSWORD` | SMTP パスワード | - |
| `SMTP_TLS` | TLS モード。`implicit`（別名 `smtps`、`ssl`）は接続時に SMTPS を強制する。ポート `465` は自動的に有効化する。未設定 / `starttls` は STARTTLS でアップグレードする | `starttls` |
| `SMTP_TLS_INSECURE` | TLS 証明書の検証をスキップするには `true` を設定する（自己署名 / プライベート CA 証明書） | `false` |
| `SMTP_EHLO_NAME` | リレーに通知される EHLO/HELO 名。厳格なリレー（例: Google Workspace）がパブリック IP からのデフォルトの挨拶を拒否する場合は、実際の FQDN を設定する | マシンのホスト名 |

STARTTLS はサーバーがアドバタイズしたときに自動的に使われます。ポート 465（SMTPS / 暗黙的 TLS）はサポートされており、暗黙的 TLS を自動的に有効化します。標準外のポートで強制するには `SMTP_TLS=implicit`（別名 `smtps`、`ssl`）を設定してください。

> **注:** Resend と SMTP のどちらも設定されていない場合、生成された検証コードはバックエンドログに出力されます——そこからコピーしてログインしてください。固定のローカルテスト用コード（例: `888888`）は **オプトインのみ** です。`.env` に `MULTICA_DEV_VERIFICATION_CODE=888888` を設定し、`APP_ENV` を本番以外にしてください。Docker のセルフホストスタックは `APP_ENV=production` に固定されているため、このショートカットはそこでは無視されます。**公開アクセス可能なインスタンスでは固定コードを絶対に有効化しないでください。**

### Google OAuth（任意）

| 変数 | 説明 |
|----------|-------------|
| `GOOGLE_CLIENT_ID` | Google OAuth クライアント ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth クライアントシークレット |
| `GOOGLE_REDIRECT_URI` | OAuth コールバック URL（例: `https://app.example.com/auth/callback`） |

変更はバックエンド / compose スタックの再起動後に有効になります。Web UI は実行時に `/api/config` から `GOOGLE_CLIENT_ID` を読み取るため、Web の再ビルドは不要です。

### サインアップ制御（任意）

| 変数 | 説明 |
|----------|-------------|
| `ALLOW_SIGNUP` | プライベートインスタンスで新規ユーザーのサインアップを無効化するには `false` を設定する |
| `ALLOWED_EMAIL_DOMAINS` | 任意の、カンマ区切りのメールドメイン許可リスト |
| `ALLOWED_EMAILS` | 任意の、カンマ区切りの正確なメールアドレス許可リスト |
| `DISABLE_WORKSPACE_CREATION` | すべての呼び出し元に対して `POST /api/workspaces` が 403 を返すようにするには `true` を設定する — ユーザーは招待されたワークスペースにのみ参加できる |

変更はバックエンド / compose スタックの再起動後に有効になります。Web UI は実行時に `/api/config` から `ALLOW_SIGNUP` と `DISABLE_WORKSPACE_CREATION` を読み取るため、Web の再ビルドは不要です。

#### ワークスペース作成のロックダウン

`ALLOW_SIGNUP=false` は新規アカウントの作成をブロックしますが、すでにサインイン済みのユーザーが `POST /api/workspaces` 経由で別のワークスペースを作成することは **ブロックしません**。すべての Issue/リポジトリ/エージェントがプラットフォーム管理者に見えなければならないセルフホストインスタンスでは、そのギャップを塞ぐために `DISABLE_WORKSPACE_CREATION=true` を設定してください。推奨されるブートストラップの順序は次のとおりです。

1. `DISABLE_WORKSPACE_CREATION=false`（デフォルト）でインスタンスを起動する。
2. 管理者としてサインインし、共有ワークスペースを作成する。
3. `DISABLE_WORKSPACE_CREATION=true` を設定してバックエンドを再起動する。新規アカウントの作成もブロックしたい場合は、同時に `ALLOW_SIGNUP=false` を設定してもよい。
4. 今後、追加のユーザーは招待経由でのみ参加する — 「ワークスペースを作成」のアフォーダンスは UI で非表示になり、直接の API 呼び出しはいずれも 403 を返す。

> 注: `ALLOW_SIGNUP=false` を設定すると、すでに保留中の招待を持つユーザーを含め、**すべての** 新規アカウント作成がブロックされます。招待されたユーザーがサインアップはできるが自分のワークスペースは作成できないようにする必要がある場合は、`ALLOW_SIGNUP=true` のまま（任意で `ALLOWED_EMAIL_DOMAINS` / `ALLOWED_EMAILS` と組み合わせて）にし、`DISABLE_WORKSPACE_CREATION=true` だけを切り替えてください。

### ファイルストレージ（任意）

ファイルのアップロードと添付ファイルのために、S3 と（任意で）CloudFront を設定します。

| 変数 | 説明 |
|----------|-------------|
| `S3_BUCKET` | バケット名のみ（例: `my-bucket`）。`.s3.<region>.amazonaws.com` サフィックスは含め **ない** でください — サーバーは `S3_BUCKET` + `S3_REGION` からパブリック URL を構築します |
| `S3_REGION` | AWS リージョン（デフォルト: `us-west-2`）。バケットの実際のリージョンと一致する必要があります — SDK 署名とパブリック URL の両方に使われます |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | 静的な認証情報。両方が未設定の場合、AWS SDK のデフォルト認証情報チェーンが使われます |
| `AWS_ENDPOINT_URL` | カスタムの S3 互換エンドポイント（例: MinIO、R2、B2）。これを設定すると、後方互換性のためデフォルトでパススタイル URL になります |
| `S3_USE_PATH_STYLE` | 任意の S3 アドレッシングモード。デフォルト（`AWS_ENDPOINT_URL` が設定されているときは `true`、AWS S3 の場合は `false`）にするには空のままにしてください。仮想ホスト形式の URL を必要とする S3 互換プロバイダーの場合は `false` を設定してください |
| `ATTACHMENT_DOWNLOAD_MODE` | 添付ファイルのダウンロード動作: `auto`（デフォルト）、`cloudfront`、`presign`、または `proxy`。`http://rustfs:9000` のような Docker/VPC 限定エンドポイントの背後にあるプライベートバケットには `proxy` を使ってください |
| `ATTACHMENT_DOWNLOAD_URL_TTL` | CloudFront 署名付き URL と S3 事前署名ダウンロード URL の TTL（デフォルト: `30m`） |
| `CLOUDFRONT_DOMAIN` | CloudFront ディストリビューションドメイン — 設定すると、パブリック URL は S3 ホストの代わりにこのホストを使います |
| `CLOUDFRONT_KEY_PAIR_ID` | 署名付き URL 用の CloudFront キーペア ID |
| `CLOUDFRONT_PRIVATE_KEY` | CloudFront 秘密鍵（PEM 形式） |

### Cookie

| 変数 | 説明 |
|----------|-------------|
| `COOKIE_DOMAIN` | セッション + CloudFront Cookie の任意の `Domain` 属性。単一ホストのデプロイ（localhost、LAN IP、または単一ホスト名）では **空のままにしてください**。フロントエンドとバックエンドが1つの登録済みドメインの異なるサブドメイン（例: `.example.com`）にある場合にのみ設定してください。**IP リテラルは使わないでください** — RFC 6265 は Cookie の `Domain` 属性に IP アドレスを禁じており、ブラウザはそのような `Set-Cookie` ヘッダーを破棄します。 |

セッション Cookie の `Secure` フラグは `FRONTEND_ORIGIN` のスキームから自動的に導出されます。HTTPS オリジンは `Secure` Cookie を得ます。プレーン HTTP のオリジン（LAN / プライベートネットワークのセルフホスト）は、ブラウザが実際に保存できるよう、非セキュアな Cookie を得ます。

### サーバー

| 変数 | デフォルト | 説明 |
|----------|---------|-------------|
| `PORT` | `8080` | バックエンドサーバーのポート |
| `METRICS_ADDR` | 空 | 任意の Prometheus メトリクスリスナー。例: `127.0.0.1:9090` |
| `FRONTEND_PORT` | `3000` | フロントエンドのポート |
| `CORS_ALLOWED_ORIGINS` | `FRONTEND_ORIGIN` の値 | 許可するオリジンのカンマ区切りリスト。HTTP CORS 許可リスト **と** WebSocket の `Origin` チェックの **両方** を制御します。ここに記載されていない（かつ `localhost` でない）ブラウザオリジンは、リアルタイムの WebSocket アップグレードが `403` で拒否されるため、手動でリロードするまでライブ更新が動作しなくなります。 |
| `LOG_LEVEL` | `info` | ログレベル: `debug`、`info`、`warn`、`error` |

### CLI / デーモン

これらはサーバー上ではなく、各ユーザーのマシン上で設定されます。

| 変数 | デフォルト | 説明 |
|----------|---------|-------------|
| `MULTICA_SERVER_URL` | `ws://localhost:8080/ws` | デーモン → サーバー接続用の WebSocket URL |
| `MULTICA_APP_URL` | `http://localhost:3000` | CLI ログインフロー用のフロントエンド URL |
| `MULTICA_DAEMON_POLL_INTERVAL` | `3s` | デーモンがタスクをポーリングする頻度 |
| `MULTICA_DAEMON_HEARTBEAT_INTERVAL` | `15s` | ハートビートの頻度 |

エージェント固有の上書き:

| 変数 | 説明 |
|----------|-------------|
| `MULTICA_CLAUDE_PATH` | `claude` バイナリへのカスタムパス |
| `MULTICA_CLAUDE_MODEL` | 使用する Claude モデルを上書きする |
| `MULTICA_CODEX_PATH` | `codex` バイナリへのカスタムパス |
| `MULTICA_CODEX_MODEL` | 使用する Codex モデルを上書きする |
| `MULTICA_COPILOT_PATH` | `copilot`（GitHub Copilot CLI）バイナリへのカスタムパス |
| `MULTICA_COPILOT_MODEL` | 使用する Copilot モデルを上書きする（注: GitHub Copilot はモデルをアカウントのエンタイトルメント経由でルーティングするため、これは尊重されない場合があります） |
| `MULTICA_OPENCODE_PATH` | `opencode` バイナリへのカスタムパス |
| `MULTICA_OPENCODE_MODEL` | 使用する OpenCode モデルを上書きする |
| `MULTICA_OPENCLAW_PATH` | `openclaw` バイナリへのカスタムパス |
| `MULTICA_OPENCLAW_MODEL` | 使用する OpenClaw モデルを上書きする |
| `MULTICA_HERMES_PATH` | `hermes` バイナリへのカスタムパス |
| `MULTICA_HERMES_MODEL` | 使用する Hermes モデルを上書きする |
| `MULTICA_PI_PATH` | `pi` バイナリへのカスタムパス |
| `MULTICA_PI_MODEL` | 使用する Pi モデルを上書きする |
| `MULTICA_CURSOR_PATH` | `cursor-agent` バイナリへのカスタムパス |
| `MULTICA_CURSOR_MODEL` | 使用する Cursor Agent モデルを上書きする |
| `MULTICA_GROK_PATH` | `grok` バイナリへのカスタムパス |
| `MULTICA_GROK_MODEL` | 使用する Grok モデルを上書きする（例: `grok-4.5`） |

## データベースのセットアップ

Multica は pgvector 拡張を備えた PostgreSQL 17 を必要とします。

### Docker Compose を使う（推奨）

`docker-compose.selfhost.yml` には PostgreSQL が含まれています。個別のセットアップは不要です。

### 自前の PostgreSQL を使う

既存の PostgreSQL インスタンスを使いたい場合は、pgvector 拡張が利用可能であることを確認してください。

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

`.env` に `DATABASE_URL` を設定し、compose ファイルから `postgres` サービスを削除してください。

### マイグレーションを手動で実行する

Docker Compose のセットアップはマイグレーションを自動的に実行します。手動で実行する必要がある場合:

```bash
# ビルド済みバイナリを使う
./server/bin/migrate up

# またはソースから
cd server && go run ./cmd/migrate up
```

## 使用状況ダッシュボードのロールアップ

使用状況 / ランタイムダッシュボードは、`rollup_task_usage_hourly()` によって生成される派生テーブル `task_usage_hourly` から読み取ります。MUL-2957 以降、バックエンドは DB ベースのスケジューラー（`sys_cron_executions`）を介して、このロールアップを各レプリカ上で **インプロセス** で実行します。新規のセルフホストインストールでは運用者による操作は不要です——同梱の `pgvector/pgvector:pg17` イメージは変更なしで動作します。

### インプロセスのスケジューラーの仕組み

各バックエンドレプリカは30秒ごとにティックし、`sys_cron_executions` の現在の5分単位（UTC）のプランをクレームしようとします。ユニークキー `(job_name, scope_kind, scope_id, plan_time)` により、このクレームはすべてのレプリカにまたがる単一勝者の争いになるため、複数インスタンスのデプロイでも二重書き込みは起きません。ハンドラーはその後 `SELECT rollup_task_usage_hourly()` を呼び出します。SQL 関数は内部でアドバイザリロック `4246` を保持するため、はぐれた `pg_cron` ジョブや手動呼び出しが、ロールアップそのもので衝突することなくスケジューラーと並行して実行できます。定常状態の動作は監査テーブルで確認します。

```sql
SELECT plan_time, status, attempt, runner_id,
       error_code, error_msg, started_at, finished_at
  FROM sys_cron_executions
 WHERE job_name = 'rollup_task_usage_hourly'
 ORDER BY plan_time DESC
 LIMIT 20;
```

### 互換性 — 既存の `pg_cron` 登録

以前にロールアップを `pg_cron` ジョブとして登録していた場合（`SELECT cron.schedule('rollup_task_usage_hourly', '*/5 * * * *', …)`）、そのままにしておいても安全です。アドバイザリロック 4246 が二重書き込みを防ぎ、敗者側の経路はクリーンに no-op になります。インプロセスのスケジューラーが立ち上がったら、冗長なエントリを削除するには:

```sql
SELECT cron.unschedule('rollup_task_usage_hourly')
  FROM cron.job WHERE jobname = 'rollup_task_usage_hourly';
```

`SELECT rollup_task_usage_hourly()` を直接呼び出す外部 cron / systemd / Kubernetes の `CronJob` 構成も、依然として有効です——MUL-2957 以前は唯一の選択肢であり、引き続きサポートされる互換性の経路として残ります。これらはもはや推奨されるセットアップではありません。新規のデプロイはインプロセスのスケジューラーに頼るべきです。

### スタンドアロンのバックフィルコマンド

`rollup_task_usage_hourly()` は、実行が開始された後の新しいバケットのみを処理します。ロールアップが初めてクレームされる前の `task_usage` 行がすでにある場合——最も一般的なのは、すでに数か月分の使用状況があるデータベースで `v0.3.4` から `v0.3.5+` にアップグレードするとき——、`backfill_task_usage_hourly` を実行して過去のバケットをシードできます。

```bash
# Docker Compose
docker compose -f docker-compose.selfhost.yml exec backend \
  ./backfill_task_usage_hourly --sleep-between-slices=2s

# Kubernetes
kubectl -n multica exec deploy/multica-backend -- \
  ./backfill_task_usage_hourly --sleep-between-slices=2s
```

このコマンドは `task_usage` の全時間範囲を月次スライスで走査し、インプロセスのスケジューラーが使うのと同じ冪等なプリミティブを呼び出します。そのため、再実行しても、Ctrl-C で中断しても、スケジューラーと並行して実行しても安全です（アドバイザリロック 4246 が両者を直列化します）。フラグ:

| フラグ | 説明 |
|---|---|
| `--sleep-between-slices` | 月次スライスの間に一時停止して、ビジーなデータベースへの読み取り負荷を抑制する（例: `2s`）。数年分の履歴を持つ本番 DB では推奨。 |
| `--months-back N` | 直近 N か月分のみバックフィルする。**`--force-partial` が必要** です。ウォーターマークはスキップされた古いバケットを越えて進むため、それらは永久に放棄されるからです。 |
| `--dry-run` | 何も書き込まずに、処理されるであろうスライスをログに出力する。 |

バックフィルが完了すると、ロールアップ状態のウォーターマークは `now() - 5 minutes` にスタンプされるため、バックフィル後の最初のスケジュールティックが履歴をやり直すことはありません。

### `v0.3.4 → v0.3.5+` のアップグレード順序

マイグレーション `103` は、`task_usage_hourly` が追いつくまで旧来の日次ロールアップの削除を拒否するフェイルクローズド・ガードを追加します。MUL-2957 以降、migrate コマンドはマイグレーション `103` を適用する直前に、冪等な月次スライスのバックフィル（アドバイザリロック 4246 の下で）を **自動的に** 実行します。そのため、`v0.3.4 → v0.3.5+` のアップグレードは単一の `migrate up` 呼び出しで完了します——運用者による手順は不要です。

MUL-2957 以前のバイナリからアップグレードしている場合（または環境上の理由で自動フックが失敗した場合）、リカバリーは手動の経路です。データベースに対して `backfill_task_usage_hourly` を実行し、その後 `migrate up` を再実行してください（またはバックエンドコンテナを再起動してください——マイグレーションは起動時に自動的に実行されます）。**新規インストールは対象外** です——`task_usage` が空のときガードはショートサーキットし、インプロセスのスケジューラーが最初のティックから新しいバケットを拾います。

## 手動セットアップ（Docker Compose なし）

サービスを手動でビルドして実行したい場合:

**前提条件:** Go 1.26+、Node.js 20+、pnpm 10.28+、pgvector を備えた PostgreSQL 17。

```bash
# PostgreSQL を起動する（または: docker compose up -d postgres を使う）

# バックエンドをビルドする
make build

# データベースマイグレーションを実行する
DATABASE_URL="your-database-url" ./server/bin/migrate up

# バックエンドサーバーを起動する
DATABASE_URL="your-database-url" PORT=8080 JWT_SECRET="your-secret" ./server/bin/server
```

フロントエンドの場合:

```bash
pnpm install
pnpm build

# フロントエンドを起動する（本番モード）
cd apps/web
REMOTE_API_URL=http://localhost:8080 pnpm start
```

## リバースプロキシ

本番では、TLS とルーティングを処理するために、バックエンドとフロントエンドの両方の前段にリバースプロキシを置きます。

### Caddy（推奨）

**単一ドメイン構成** — フロントエンドとバックエンドを同じホスト名で配信する（`docker-compose.selfhost.yml` のデフォルトはこれです）:

```
multica.example.com {
    # WebSocket ルート — キャッチオールより前に来る必要がある
    @multica_ws path /ws /ws/*
    handle @multica_ws {
        reverse_proxy localhost:8080 {
            flush_interval -1
        }
    }

    # それ以外すべて → フロントエンド
    reverse_proxy localhost:3000
}
```

> 単一ドメインであっても、バックエンドで `FRONTEND_ORIGIN` / `CORS_ALLOWED_ORIGINS` をそのパブリックオリジン（例: `https://multica.example.com`）に設定してください。バックエンドのデフォルトのオリジン許可リストは `localhost` のみであるため、これがないとパブリック URL からの WebSocket アップグレードを `403` で拒否し、リアルタイム更新が静かに動作しなくなります。[LAN / 非 localhost アクセス](#lan--非-localhost-アクセス) を参照してください。

**別ドメイン構成** — フロントエンドとバックエンドを異なるホスト名にする:

```
app.example.com {
    reverse_proxy localhost:3000
}

api.example.com {
    @multica_ws path /ws /ws/*
    handle @multica_ws {
        reverse_proxy localhost:8080 {
            flush_interval -1
        }
    }

    reverse_proxy localhost:8080
}
```

`/ws` ブロック内の2つの分かりにくい点は特筆に値します——どちらも、Caddy を前段にしたセルフホストでリアルタイム更新が「動作しなくなる」よくある原因です。

- **`path /ws /ws/*`（`/ws*` ではない）** — 素の `handle /ws` は完全一致であるため、将来 `/ws/` 配下のパスバリアントがフロントエンドブロックに落ちてしまいます。一見便利な `handle /ws*` は逆方向に過剰修正します。Caddy の `*` はパスセグメント境界のないグロブであるため、`/ws-foo` のような無関係なパスも捕まえてしまいます。`/ws-foo` は正当なワークスペース URL です（予約されているのは正確なスラッグ `ws` のみ）。`/ws` と `/ws/*` を明示的に列挙すると、行き過ぎることなく両方の実ケースをカバーできます。
- **`flush_interval -1`** — レスポンスのバッファリングを無効化し、WebSocket フレームが到着し次第すぐに転送されるようにします。これがないと、フレームが Caddy のデフォルトのフラッシュウィンドウの後ろで滞留することがあり、コメントの遅延、タイピングインジケーターの欠落、あるいは「ページを更新して初めてコメントが表示される」ように見えます。

### Nginx

```nginx
# フロントエンド
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# バックエンド API
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket サポート
    location /ws {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

フロントエンドとバックエンドで別々のドメインを使う場合は、これらの環境変数を適切に設定してください。

```bash
# バックエンド
FRONTEND_ORIGIN=https://app.example.com
CORS_ALLOWED_ORIGINS=https://app.example.com

# フロントエンド（docker-compose.selfhost.build.yml 経由でソースから Web イメージをビルドする場合のみ）
REMOTE_API_URL=https://api.example.com
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_WS_URL=wss://api.example.com/ws
```

## LAN / 非 localhost アクセス

デフォルトでは、Multica は `localhost` で動作します。LAN 上の別のマシン（例: `http://192.168.1.100:3000`）からアクセスする場合は、そのオリジンを受け入れるようバックエンドに伝える必要があります。

```bash
# .env — あなたのサーバーの LAN IP に置き換える
FRONTEND_ORIGIN=http://192.168.1.100:3000
CORS_ALLOWED_ORIGINS=http://192.168.1.100:3000
```

その後、スタックを再起動します。

```bash
docker compose -f docker-compose.selfhost.yml up -d
```

### LAN / 非 localhost アクセスのための WebSocket

HTTP リクエスト（Issue、コメント、アップロード）は LAN 上でそのまま動作します——Next.js のリライトが `/api`、`/auth`、`/uploads` をバックエンドにプロキシします。**WebSocket はそうではありません**: Next.js のリライトは HTTP リクエストのみを転送し、WebSocket が必要とする `Upgrade` ハンドシェイクは転送しません。`http://<lan-ip>:3000` でアプリを開くと、以下のいずれかを行うまで、リアルタイム機能（チャットのストリーミング、Issue のライブ更新、通知）は接続に失敗します。

1. **スタックの前段にリバースプロキシを置く（推奨）。** Nginx や Caddy が WebSocket のアップグレードを終端し、ポート 8080 のバックエンドに転送します。上記の [リバースプロキシ](#リバースプロキシ) セクションを参照してください——Nginx の例には、正しい `Upgrade` / `Connection` ヘッダーを備えた `location /ws { ... }` ブロックがすでに含まれています。プロキシが設置されれば、ブラウザはそれを通じて直接接続するため、フロントエンドの再ビルドは不要です。

2. **WebSocket URL を Web イメージに焼き込む。** リバースプロキシを実行していない場合は、`NEXT_PUBLIC_WS_URL` をバックエンドに直接向けて Web イメージを再ビルドします（ポート 8080 がブラウザから到達可能である必要があります）。

   ```bash
   # .env 内
   NEXT_PUBLIC_WS_URL=ws://<lan-ip>:8080/ws

   # ビルド時の値が焼き込まれるよう Web イメージを再ビルドする
   docker compose -f docker-compose.selfhost.yml -f docker-compose.selfhost.build.yml up -d --build
   ```

   `NEXT_PUBLIC_WS_URL` はビルド時の変数です（`Dockerfile.web` を参照）。そのため、ビルド済みイメージの `environment:` にのみ設定しても効果はありません——イメージを再ビルドする `selfhost.build.yml` オーバーライドを使う必要があります。

**あわせて必須: ブラウザのオリジンを許可リストに入れる。** 上記の2つの選択肢は WebSocket の *アップグレードプロキシ* を修正しますが、2つ目の独立した設定が接続をゲートします: バックエンドは WebSocket の `Origin` ヘッダーを、デフォルトで `localhost` のみの許可リストに対して検証します。Multica を他のオリジン——LAN IP **またはリバースプロキシ背後のパブリックドメイン**——から開くときは、上記の [LAN / 非 localhost アクセス](#lan--非-localhost-アクセス) に示したとおり、バックエンドの `CORS_ALLOWED_ORIGINS`（または `FRONTEND_ORIGIN`）をその正確なオリジンに設定して再起動してください。そうしないとアップグレードは `403` で拒否されます: バックエンドは `websocket: request origin not allowed by Upgrader.CheckOrigin` をログに出力し、ブラウザのコンソールは `disconnected, reconnecting in 3s` をループします。一方、HTTP リクエスト（および手動のページ更新）は、ページと同一オリジンであるため引き続き動作します。この単一の値が、HTTP CORS と WebSocket オリジンチェックの両方をカバーします。

> **注:** 他の理由で別のパブリック API / WebSocket エンドポイントを Web イメージにハードコードする必要がある場合は、同じソースビルドのオーバーライドを使ってください: `docker compose -f docker-compose.selfhost.yml -f docker-compose.selfhost.build.yml up -d --build`。

## ヘルスチェック

バックエンドはパブリックなヘルスエンドポイントを公開します。

```text
GET /health
→ {"status":"ok"}

GET /readyz
→ {"status":"ok","checks":{"db":"ok","migrations":"ok"}}

GET /healthz
→ /readyz と同じレスポンス
```

基本的な稼働 / 到達性チェックには `/health` を使います。データベースが利用不可であったりマイグレーションが完全に適用されていなかったりする場合に失敗すべき、依存関係を考慮した準備状態プローブや外部監視には `/readyz` を使います。`/healthz` は運用者になじみのあるエイリアスとして残されています。

## Prometheus メトリクス

バックエンドは、別個の管理リスナー上で Prometheus メトリクスを公開できます。

```bash
METRICS_ADDR=127.0.0.1:9090 ./server/bin/server
curl http://127.0.0.1:9090/metrics
```

`METRICS_ADDR` はデフォルトで空であるため、メトリクスリスナーは起動されません。パブリックな API ポートは `/metrics` を提供しません。インターネットに面したデプロイではその状態を保ってください。HTTP リクエストのメトリクスは、メトリクスリスナーが有効化された後にのみ蓄積を開始します。メトリクスは、内部ルート、トラフィック量、依存関係の状態、ランタイムの健全性を明らかにし得ます。

Docker や Kubernetes のデプロイでは、プライベートなスクレイプ経路を優先してください: メトリクスリスナーを内部インターフェースにバインドし、プライベートネットワーク、許可リスト、NetworkPolicy、またはプロキシ認証で保護してください。コンテナ内で `METRICS_ADDR=0.0.0.0:9090` をバインドする場合は、そのポートを信頼できるネットワークにのみ公開してください。例えば `127.0.0.1:9090:9090` のようなホストローカルのマッピングです。

## アップグレード

```bash
docker compose -f docker-compose.selfhost.yml pull
docker compose -f docker-compose.selfhost.yml up -d
```

特定のバージョンに留まりたい場合は、`.env` の `MULTICA_IMAGE_TAG` を `v0.2.4` のような正確なリリースに固定してください。マイグレーションはバックエンド起動時に自動的に実行されます。これらは冪等です——複数回実行しても効果はありません。
選択した GHCR タグがまだ公開されていない場合は、`docker compose -f docker-compose.selfhost.yml -f docker-compose.selfhost.build.yml up -d --build` にフォールバックしてください。

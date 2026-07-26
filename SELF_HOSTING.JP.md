# セルフホスティングガイド

Multica を自前のインフラに数分でデプロイします。

## アーキテクチャ

| コンポーネント | 説明 | 技術 |
|-----------|-------------|------------|
| **バックエンド** | REST API + WebSocket サーバー | Go（単一バイナリ） |
| **フロントエンド** | Web アプリケーション | Next.js 16 |
| **データベース** | 主要データストア | PostgreSQL 17（pgvector 付き） |

ローカルで AI エージェントを実行する各ユーザーは、あわせて **`multica` CLI** をインストールし、自分のマシンで **エージェントデーモン** を実行します。

## クイックインストール（推奨）

2つのコマンドですべて（サーバー、CLI、設定）をセットアップします。

<details open>
<summary><b>macOS / Linux</b></summary>

<br/>

```bash
# 1. CLI をインストールし、セルフホストサーバーをプロビジョニングする
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server

# 2. CLI を設定し、認証して、デーモンを起動する
multica setup self-host
```
</details>
<details>
<summary><b>Windows（PowerShell）</b></summary>

<br/>

```powershell
# 1. CLI をインストールし、セルフホストサーバーをプロビジョニングする
$env:MULTICA_MODE="with-server"; irm https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.ps1 | iex

# 2. CLI を設定し、認証して、デーモンを起動する
multica setup self-host
```
</details>

これにより `multica` CLI がインストールされ、最新のセルフホスト用アセットがチェックアウトされ、GHCR から公式の Multica イメージが取得され、すべてが localhost 向けに設定されます。

http://localhost:3000 を開きます。ログインするには、メールベースのコードのために `.env` に `RESEND_API_KEY` を設定するか（推奨）、Resend を未設定のままにしてバックエンドログから生成されたコードをコピーします。詳しくは [ステップ2 — ログイン](#ステップ2--ログイン) を参照してください。

> **前提条件:** Docker と Docker Compose がインストールされている必要があります。スクリプトはこれを確認し、なければインストール用のリンクを提示します。
>
> **CLI のみ？** セルフホストサーバーがすでに稼働していて、macOS/Linux マシンに CLI だけが必要な場合は、Homebrew でインストールします。
>
> ```bash
> brew install multica-ai/tap/multica
> ```

---

## ステップごとのセットアップ（代替手段）

各ステップを手動で実行したい場合:

### ステップ1 — サーバーを起動する

**前提条件:** Docker と Docker Compose。

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make selfhost
```

`make selfhost` は example から `.env` を自動作成し、ランダムな `JWT_SECRET` を生成し、Docker Compose 経由ですべてのサービスを起動します。

デフォルトでは GHCR から最新の安定版リリースイメージを取得します。代わりに現在のチェックアウトからバックエンド/Web をビルドするには、`make selfhost-build` を実行します。
選択した GHCR タグがまだ公開されていない場合、`make selfhost` は `make selfhost-build` にフォールバックするよう案内します。
`make selfhost-build` はローカルの `multica-backend:dev` / `multica-web:dev` タグを使うため、取得済みの `:latest` イメージを上書きしません。

準備ができたら:

- **フロントエンド:** http://localhost:3000
- **バックエンド API:** http://localhost:8080

> **注:** Docker Compose の手順を手動で実行したい場合は、後述の [手動 Docker Compose セットアップ](#手動-docker-compose-セットアップ) を参照してください。

### ステップ2 — ログイン

ブラウザで http://localhost:3000 を開きます。Docker セルフホストスタックはデフォルトで `APP_ENV=production`（`docker-compose.selfhost.yml` で設定）となっており、デフォルトでは固定の検証コードはありません。ログインするには次のいずれかを選びます。

- **推奨（本番）:** `.env` に `RESEND_API_KEY` を設定し、バックエンドを再起動します。入力したメールアドレスに本物の検証コードが送信されます。[高度な設定 → メール](SELF_HOSTING_ADVANCED.JP.md#メール認証に必須) を参照してください。
- **メール未設定の場合:** 検証コードはサーバー側で生成され、バックエンドコンテナのログに出力されます（`[DEV] Verification code for ...:` を探してください）。1台のマシンでの単発テストに便利です。
- **決定論的なローカル/プライベートテスト:** `.env` に `APP_ENV=development` と `MULTICA_DEV_VERIFICATION_CODE=888888` を設定し、バックエンドを再起動します。この固定コードは `APP_ENV=production` のときは無視されます。

`ALLOW_SIGNUP`、`DISABLE_WORKSPACE_CREATION`、`GOOGLE_CLIENT_ID` の変更も、バックエンド / compose スタックの再起動後に反映されます。Web UI はこれら3つを実行時に `/api/config` から読み取るため、Web の再ビルドは不要です。ワークスペース作成をロックダウンする推奨手順については [高度な設定 → サインアップ制御](SELF_HOSTING_ADVANCED.JP.md#サインアップ制御任意) を参照してください。

> **警告:** 公開アクセス可能なインスタンスでは `MULTICA_DEV_VERIFICATION_CODE` を設定し **ない** でください — メールアドレスを知っている者は誰でもその固定コードでログインできてしまいます。

### ステップ3 — CLI のインストールとデーモンの起動

デーモンはあなたのローカルマシン上（Docker 内ではない）で動作します。インストールされた AI エージェント CLI を検出し、サーバーに登録し、エージェントに作業が割り当てられたときにタスクを実行します。

ローカルで AI エージェントを実行したいチームメンバーはそれぞれ、以下を行う必要があります。

### a) CLI と AI エージェントをインストールする

```bash
brew install multica-ai/tap/multica
```

さらに、少なくとも1つの AI エージェント CLI がインストールされている必要があります。
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)（PATH 上に `claude`）
- [Codex](https://github.com/openai/codex)（PATH 上に `codex`）
- [GitHub Copilot CLI](https://docs.github.com/en/copilot)（PATH 上に `copilot`）
- [OpenClaw](https://github.com/openclaw/openclaw)（PATH 上に `openclaw`）
- [OpenCode](https://github.com/anomalyco/opencode)（PATH 上に `opencode`）
- [Hermes](https://github.com/NousResearch/hermes)（PATH 上に `hermes`）
- [Pi](https://pi.dev/)（PATH 上に `pi`）
- [Cursor Agent](https://cursor.com/)（PATH 上に `cursor-agent`）
- Kimi（PATH 上に `kimi`）
- Kiro CLI（PATH 上に `kiro-cli`）
- Qoder CLI（PATH 上に `qodercli`）
- Trae CLI（PATH 上に `traecli`）
- [Grok Build CLI](https://docs.x.ai/)（PATH 上に `grok`）
- Qwen Code（PATH 上に `qwen`）

### b) 1コマンドセットアップ

```bash
multica setup self-host
```

これは自動的に以下を行います。
1. CLI を `localhost`（ポート 8080/3000）に接続するよう設定する
2. 認証のためにブラウザを開く
3. ワークスペースを検出する
4. デーモンをバックグラウンドで起動する

カスタムドメインを使うオンプレミス展開の場合:

```bash
multica setup self-host --server-url https://api.example.com --app-url https://app.example.com
```

デーモンが稼働していることを確認するには:

```bash
multica daemon status
```

> **代替手段:** 手動の手順を好む場合は、後述の [手動 CLI 設定](#手動-cli-設定) を参照してください。

### ステップ4 — 検証と利用開始

1. Web アプリ http://localhost:3000 でワークスペースを開く
2. **Settings → Runtimes** に移動する — あなたのマシンが一覧に表示されているはずです
3. **Settings → Agents** に移動し、新しいエージェントを作成する
4. Issue を作成してエージェントに割り当てる — エージェントが自動的にタスクを引き受けます

---

## Kubernetes へのデプロイ（代替手段）

すでに Kubernetes クラスターを運用している場合は、Docker Compose の代わりに、公開された OCI Helm チャート `oci://ghcr.io/multica-ai/charts/multica`、またはソースチャート [`deploy/helm/multica/`](deploy/helm/multica/) を使って Multica をそこにデプロイできます。これは Ingress コントローラーとデフォルトの `ReadWriteOnce` StorageClass を備えた典型的な k3s / k8s 構成を対象としています——k3s + Traefik + `local-path` に対して作成されており、わずかな調整でどのクラスターでも動作するはずです。

チャートは対象の名前空間に以下のリソースを作成します。

- `multica-postgres` — 10Gi の PVC に支えられた `pgvector/pgvector:pg17`
- `multica-backend` — Go の API/WS サーバー。デフォルトでは 5Gi の `ReadWriteOnce` アップロード用 PVC に支えられます。S3（`backend.config.s3Bucket`）を設定済みで、チャートに PVC をまったく宣言させたくない場合は `backend.uploads.persistence.enabled=false` を設定してください。
- `multica-frontend` — Next.js のスタンドアロンサーバー
- 2つの `Ingress` リソース: 1つは Web ホスト用、もう1つはバックエンドホスト用
- `multica-config` ConfigMap（`values.yaml` からレンダリング）

`multica-secrets` Secret はチャートでは管理され **ません** — 実際の値が git に入る必要がないよう、`kubectl` で一度だけ作成します。

> **実行時のフロントエンドアップストリーム:** 現在の `multica-web` イメージは Next.js サーバー実行時に `REMOTE_API_URL` と `DOCS_URL` を読み取るため、API/ドキュメントのアップストリーム変更に Web の再ビルドは不要です。チャートは `REMOTE_API_URL` をこのリリースのバックエンド Service にデフォルト設定します。`frontend.compatibility.backendAlias` は、ビルド時に `REMOTE_API_URL=http://backend:8080` を焼き込んでいた旧イメージのためだけに存在します。

> **前提条件:** 対象クラスター向けに設定された `kubectl` と `helm`（`--take-ownership` には v3.13 以上、または v4 以上）、Ingress コントローラー（Traefik / NGINX）、そしてデフォルトの StorageClass。

### ステップ1 — ホスト名をクラスターに向ける

チャートはデフォルトで `multica.dev.lan`（Web）と `api.multica.dev.lan`（バックエンド）を使います。次のいずれかを選びます。

- アクセスが必要なすべてのマシン（開発者のノート PC + デーモンを実行するマシン）の **`/etc/hosts`**:

  ```text
  192.168.1.206  multica.dev.lan api.multica.dev.lan
  ```

  `192.168.1.206` は、Ingress コントローラーの Service に到達可能な任意のノード IP に置き換えてください。

- **ローカル DNS**（Pi-hole、Unbound など）: 両方のホスト名についてクラスターの Ingress IP を指す A レコードを追加します。

異なるホスト名を使うには、インストール時に対応する値を上書きします（[ステップ4](#ステップ4--チャートをインストールする) 参照）——`ingress.frontend.host`、`ingress.backend.host`、加えて `backend.config.appUrl`、`backend.config.frontendOrigin`、`backend.config.localUploadBaseUrl`、`backend.config.googleRedirectUri`。

### ステップ2 — 名前空間を作成する

```bash
kubectl create namespace multica
```

### ステップ3 — `multica-secrets` Secret を作成する

チャートはこの Secret を名前で参照します。ランダムな値で一度だけ作成します。

```bash
kubectl -n multica create secret generic multica-secrets \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 16)" \
  --from-literal=RESEND_API_KEY="" \
  --from-literal=GOOGLE_CLIENT_SECRET="" \
  --from-literal=CLOUDFRONT_PRIVATE_KEY="" \
  --from-literal=MULTICA_DEV_VERIFICATION_CODE=""
```

任意の値は今は空のままにしておきます——後で埋められます（[ステップ5 — ログイン](#ステップ5--ログイン) 参照）。

### ステップ4 — チャートをインストールする

```bash
helm install multica oci://ghcr.io/multica-ai/charts/multica \
  --version <chart-version> \
  -n multica
```

公開されるチャートのバージョンは Git タグの先頭の `v` を取り除いたものです。例えば、リリースタグ `v0.3.5` はチャートバージョン `0.3.5` を公開し、チャートはバックエンドとフロントエンドのイメージタグをデフォルトで `v0.3.5` にします。

デフォルトを上書きするには、チャートの値をエクスポートして編集し、`-f` で渡します。

```bash
helm show values oci://ghcr.io/multica-ai/charts/multica \
  --version <chart-version> > my-values.yaml
# my-values.yaml を編集する — 例: Ingress ホスト、イメージタグ、リソース制限の変更
helm install multica oci://ghcr.io/multica-ai/charts/multica \
  --version <chart-version> \
  -n multica \
  -f my-values.yaml
```

チェックアウトから開発する場合は、代わりにローカルのチャートパスを使います。

```bash
helm install multica deploy/helm/multica -n multica
```

Pod の起動を監視します。

```bash
kubectl -n multica get pods -w
```

コールドクラスターでは、バックエンドが PostgreSQL を待ってマイグレーションを実行する数分間、`Running` でも `Ready` にならないことがあります——startupProbe がこれを吸収するため、Pod は再起動しないはずです。バックエンドが `Ready` を報告すると、マイグレーションは完了しており、`/healthz` は OK を返します。

```bash
curl -H "Host: api.multica.dev.lan" http://<ingress-ip>/healthz
# {"status":"ok","checks":{"db":"ok","migrations":"ok"}}
```

その後、ブラウザで http://multica.dev.lan を開きます。

### ステップ5 — ログイン

チャートはデフォルトで `APP_ENV=production`（`values.yaml` の `backend.config.appEnv` で設定）となっており、デフォルトでは固定の検証コードはありません。ログインするには次のいずれかを選びます——Docker セットアップと同じ3つの選択肢です。

- **推奨（本番）:** Secret に本物の Resend キーをパッチし、バックエンドを再起動します。

  ```bash
  kubectl -n multica patch secret multica-secrets --type=merge \
    -p '{"stringData":{"RESEND_API_KEY":"re_xxx"}}'
  kubectl -n multica rollout restart deploy/multica-backend
  ```

  入力したメールアドレスに本物の検証コードが送信されます。[高度な設定 → メール](SELF_HOSTING_ADVANCED.JP.md#メール認証に必須) を参照してください。

- **メール未設定の場合:** 検証コードはサーバー側で生成され、バックエンド Pod のログに出力されます（`[DEV] Verification code for ...:` を探してください）。単発テストに便利です。

  ```bash
  kubectl -n multica logs -f deploy/multica-backend | grep "Verification code"
  ```

- **決定論的なローカル/プライベートテスト:** values ファイルに `backend.config.appEnv: development` を設定し、Secret に `MULTICA_DEV_VERIFICATION_CODE=888888` を設定してから、`helm upgrade` して再起動します。この固定コードは `APP_ENV=production` のときは無視されます。

  ```bash
  helm upgrade multica oci://ghcr.io/multica-ai/charts/multica \
    --version <chart-version> \
    -n multica \
    -f my-values.yaml --set backend.config.appEnv=development
  kubectl -n multica patch secret multica-secrets --type=merge \
    -p '{"stringData":{"MULTICA_DEV_VERIFICATION_CODE":"888888"}}'
  kubectl -n multica rollout restart deploy/multica-backend
  ```

`ALLOW_SIGNUP`、`DISABLE_WORKSPACE_CREATION`、`GOOGLE_CLIENT_ID` も同様に `values.yaml` の `backend.config.*` 配下（`allowSignup`、`disableWorkspaceCreation`、`googleClientId`）にあります。`helm upgrade` の後、ConfigMap のハッシュが変わるためバックエンド Pod は自動的にロールします。Web UI はこれら3つを実行時に `/api/config` から読み取るため、Web の再ビルドは不要です。

> **警告:** 公開アクセス可能なインスタンスでは `MULTICA_DEV_VERIFICATION_CODE` を設定し **ない** でください — メールアドレスを知っている者は誰でもその固定コードでログインできてしまいます。

### ステップ6 — CLI のインストールとデーモンの起動

デーモンはあなたのローカルマシン上で動作し、クラスター内では動作しません。上記の [ステップ3](#ステップ3--cli-のインストールとデーモンの起動) のように CLI と AI エージェントをインストールし、CLI を Ingress のホスト名に向けます。

```bash
multica setup self-host \
  --server-url http://api.multica.dev.lan \
  --app-url http://multica.dev.lan
```

デーモンを実行するマシンに、[ステップ1](#ステップ1--ホスト名をクラスターに向ける) と同じ `/etc/hosts`（または DNS）エントリがあることを確認してください。

### 更新

values がまだ可変の `latest` イメージタグを使っている場合に、チャートバージョンを変えずに最新イメージを取得するには:

```bash
kubectl -n multica rollout restart deploy/multica-backend deploy/multica-frontend
```

特定の Multica リリースにアップグレードするには、対応するチャートバージョンにアップグレードします。公開されたチャートは、アプリイメージをデフォルトで対応する Git タグにします。

```bash
helm upgrade multica oci://ghcr.io/multica-ai/charts/multica \
  --version <chart-version> \
  -n multica \
  -f my-values.yaml
```

チャートバージョンとは独立してアプリイメージを上書きする必要がある場合は、values ファイルでイメージタグを設定します。

```yaml
images:
  backend:
    tag: v0.2.4
  frontend:
    tag: v0.2.4
```

その後、`-f my-values.yaml` を付けて同じアップグレードコマンドを実行します。

```bash
helm upgrade multica oci://ghcr.io/multica-ai/charts/multica \
  --version <chart-version> \
  -n multica \
  -f my-values.yaml
```

アップグレードがうまくいかなかった場合にロールバックするには:

```bash
helm -n multica rollback multica
```

> **`v0.3.4` から `v0.3.5+` へのアップグレードが `refusing to drop legacy daily rollups: ...` で失敗する？** MUL-2957 以降、`migrate up` コマンドはマイグレーション `103` を適用する前に、冪等な月次スライスのバックフィルを自動的に実行します。そのため、クリーンなアップグレードは単一の `helm upgrade` + バックエンドのロールで済みます。MUL-2957 以前のバイナリのままである場合、または自動フックが失敗した場合は、チャートが使っているのと同じデータベースに対してスタンドアロンのバックフィルを実行し（`kubectl -n multica exec deploy/multica-backend -- ./backfill_task_usage_hourly --sleep-between-slices=2s`）、その後バックエンドのデプロイを再起動してマイグレーションを再適用してください。完全なリカバリーフローは [高度な設定 → 使用状況ダッシュボードのロールアップ](SELF_HOSTING_ADVANCED.JP.md#使用状況ダッシュボードのロールアップ) を参照してください。

### 破棄

```bash
# ワークロードを削除するが PVC と Secret は残す
helm -n multica uninstall multica

# PostgreSQL のデータとアップロードを含めてすべて消去する
kubectl delete namespace multica
```

---

## 使用状況ダッシュボードのロールアップ

使用状況 / ランタイムダッシュボードは、`rollup_task_usage_hourly()` によって生成される派生テーブル `task_usage_hourly` から読み取ります。MUL-2957 以降、バックエンドは DB ベースのスケジューラー（`sys_cron_executions`）を介して、このロールアップを各レプリカ上で **インプロセス** で実行します。新規のセルフホストインストールでは運用者による操作は不要で、同梱の `pgvector/pgvector:pg17` イメージは変更なしで動作します——`pg_cron` を同梱するイメージに差し替えたり、外部の cron ジョブを登録したり、systemd タイマーを設定したり、Kubernetes の `CronJob` を実行したりする必要は **ありません**。

複数のバックエンドレプリカでも安全です。各レプリカは30秒ごとにティックし、現在の5分単位（UTC）のプランをクレームしようとしますが、ユニークキー `(job_name, scope_kind, scope_id, plan_time)` により各プランで勝者は1つだけになります。定常状態の動作を確認するには:

```sql
SELECT plan_time, status, attempt, runner_id,
       error_code, error_msg, started_at, finished_at
  FROM sys_cron_executions
 WHERE job_name = 'rollup_task_usage_hourly'
 ORDER BY plan_time DESC
 LIMIT 20;
```

完全なリファレンス（監査テーブルのセマンティクス、アドバイザリロック 4246、スタンドアロンのバックフィルコマンド、フラグの説明、`v0.3.4 → v0.3.5+` マイグレーションの自動フック）は [高度な設定 → 使用状況ダッシュボードのロールアップ](SELF_HOSTING_ADVANCED.JP.md#使用状況ダッシュボードのロールアップ) にあります。

> **`v0.3.4` から `v0.3.5+` へのアップグレード？** MUL-2957 以降、`migrate up` コマンドはマイグレーション `103` を適用する直前に、冪等な月次スライスのバックフィルを自動的に実行します。そのため、アップグレードは単一の実行で完了します——運用者による操作は不要です。MUL-2957 以前のバイナリのままである場合、または環境上の理由で自動フックが失敗した場合は、同じデータベースに対して `backfill_task_usage_hourly` を実行してから、アップグレードを再実行してください。リカバリーフローは [高度な設定 → 使用状況ダッシュボードのロールアップ](SELF_HOSTING_ADVANCED.JP.md#使用状況ダッシュボードのロールアップ) を参照してください。

### 互換性の経路（既存のデプロイのみ）

外部スケジューラー——**データベースに登録された `pg_cron`、外部の cron ジョブ、systemd タイマー、Kubernetes の `CronJob`**——で `SELECT rollup_task_usage_hourly()` を直接呼び出すものは、MUL-2957 以前は唯一の選択肢であり、引き続きサポートされる互換性の経路です。これらはもはや推奨されるセットアップではありません。新規のデプロイではインプロセスのスケジューラーに頼るべきです。SQL 関数は内部でアドバイザリロック 4246 を保持するため、インプロセスのスケジューラーと既存の外部スケジュールは、ロールアップを二重書き込みすることなく共存できます。

すでに本番で `pg_cron` ジョブがある場合、それを安全に廃止する手順は次のとおりです。

1. 少なくとも1つのバックエンドレプリカでインプロセスのスケジューラーが正常であることを確認します——`rollup_task_usage_hourly` について最近の SUCCESS 行が `sys_cron_executions` に届いているはずです。

   ```sql
   SELECT plan_time, status, runner_id, finished_at
     FROM sys_cron_executions
    WHERE job_name = 'rollup_task_usage_hourly'
      AND status = 'SUCCESS'
    ORDER BY plan_time DESC
    LIMIT 5;
   ```

2. SUCCESS 行が予定どおり届くようになったら、冗長な `pg_cron` エントリのスケジュールを解除します。

   ```sql
   SELECT cron.unschedule('rollup_task_usage_hourly')
     FROM cron.job WHERE jobname = 'rollup_task_usage_hourly';
   ```

3. 他のワークロードが依存していないと確信できない限り、`pg_cron` 拡張自体はインストールしたままにしておきます。同梱の `pgvector/pgvector:pg17` イメージは `pg_cron` を同梱して **いない** ため、Multica のデフォルトインストールでそれを必要とするものは何もありません。他のワークロードがまだ使っているカスタムイメージから `pg_cron` をアンインストールするかどうかは、別の判断です。

`SELECT rollup_task_usage_hourly()` を直接呼び出す外部 cron / systemd タイマー / Kubernetes `CronJob` の構成も、同じ方法で廃止できます——`sys_cron_executions` にインプロセスのスケジューラーからの定常的な SUCCESS 行が表示されるようになれば、外部ジョブは冗長になり、削除できます。

## サービスの停止

インストールスクリプト経由でインストールした場合:

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --stop
```

リポジトリを手動でクローンした場合:

```bash
# Docker Compose サービス（バックエンド、フロントエンド、データベース）を停止する
make selfhost-stop

# ローカルデーモンを停止する
multica daemon stop
```

## Multica Cloud への切り替え

セルフホストしていて、CLI を [Multica Cloud](https://multica.ai) に切り替えたい場合:

```bash
multica setup
```

これは CLI を multica.ai 向けに再設定し、再認証し、デーモンを再起動します。既存の設定を上書きする前に確認が求められます。

> ローカルの Docker サービスには影響しません。不要になった場合は、別途停止してください。

## アップグレード

```bash
docker compose -f docker-compose.selfhost.yml pull
docker compose -f docker-compose.selfhost.yml up -d
```

特定のリリースに留まりたい場合は、`.env` の `MULTICA_IMAGE_TAG` を `v0.2.4` のような正確なバージョンに固定します。マイグレーションはバックエンド起動時に自動的に実行されます。
選択した GHCR タグがまだ公開されていない場合は、`make selfhost-build` または `docker compose -f docker-compose.selfhost.yml -f docker-compose.selfhost.build.yml up -d --build` にフォールバックしてください。

> **`v0.3.4` から `v0.3.5+` へのアップグレードが `refusing to drop legacy daily rollups: ...` で失敗する？** それはマイグレーション `103` のフェイルクローズド・ガードです。旧来の日次ロールアップを削除する前に `task_usage_hourly` がシードされていることを要求します。MUL-2957 以降、`migrate up` は `103` を適用する直前にそのバックフィルを自動的に実行するため、アップグレードは単一の実行で完了します。MUL-2957 以前のバイナリのままである場合、または自動フックが失敗した場合は、まず手動で `backfill_task_usage_hourly` を実行してから、アップグレードを再実行してください。詳しい手順は [高度な設定 → 使用状況ダッシュボードのロールアップ](SELF_HOSTING_ADVANCED.JP.md#使用状況ダッシュボードのロールアップ) にあります。

---

## 手動 Docker Compose セットアップ

`make selfhost` の代わりに Docker Compose の手順を手動で実行したい場合:

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
cp .env.example .env
```

`.env` を編集します——最低限、`JWT_SECRET` を変更します。

```bash
JWT_SECRET=$(openssl rand -hex 32)
```

その後、すべてを起動します。

```bash
docker compose -f docker-compose.selfhost.yml pull
docker compose -f docker-compose.selfhost.yml up -d
```

## 手動 CLI 設定

`multica setup` の代わりに CLI を段階的に設定したい場合:

```bash
# CLI をローカルサーバーに向ける
multica config set server_url http://localhost:8080
multica config set app_url http://localhost:3000

# ログイン（ブラウザを開く）
multica login

# デーモンを起動する
multica daemon start
```

TLS を使う本番デプロイの場合:

```bash
multica config set app_url https://app.example.com
multica config set server_url https://api.example.com
multica login
multica daemon start
```

## 高度な設定

環境変数、（Docker なしの）手動セットアップ、リバースプロキシの設定、データベースのセットアップなどについては、[高度な設定ガイド](SELF_HOSTING_ADVANCED.JP.md) を参照してください。

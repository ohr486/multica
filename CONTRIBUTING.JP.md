# コントリビューションガイド

このガイドは、Multica コードベースに取り組むコントリビューター向けに、ローカル開発ワークフローを説明します。

対象範囲:

- 初回セットアップ
- メインチェックアウトでの日常的な開発
- 分離された worktree での開発
- 共有 PostgreSQL モデル
- テストと検証
- フルスタックの分離テスト（バックエンド + フロントエンド + デーモンをソースから起動）
- トラブルシューティングと破壊的リセットのオプション

## 開発モデル

ローカル開発では、1 つの共有 PostgreSQL コンテナと、チェックアウトごとに 1 つのデータベースを使用します。

- メインチェックアウトは通常 `.env` と `POSTGRES_DB=multica` を使用する
- 各 Git worktree は独自の `.env.worktree` を使用する
- すべてのチェックアウトは同じ PostgreSQL ホスト `localhost:5432` に接続する
- 分離は、別々の Docker Compose プロジェクトを起動するのではなく、データベースレベルで行われる
- バックエンドとフロントエンドのポートは、依然として worktree ごとに一意である

これにより、スキーマとデータを分離しつつ、Docker をシンプルに保ちます。

## 前提条件

- Node.js `v20+`
- `pnpm` `v10.28+`
- Go `v1.26+`
- Docker

## 重要なルール

- メインチェックアウトは `.env` を使用すべきです。
- worktree は `.env.worktree` を使用すべきです。
- `.env` を worktree ディレクトリにコピーしないでください。

理由:

- 現在のコマンドフローは `.env.worktree` よりも `.env` を優先する
- worktree に `.env` が含まれていると、誤ってメインデータベースを指し戻してしまう可能性がある

## 環境ファイル

### メインチェックアウト

`.env` を一度だけ作成します:

```bash
cp .env.example .env
```

デフォルトでは、`.env` は以下を指します:

```bash
POSTGRES_DB=multica
POSTGRES_PORT=5432
DATABASE_URL=postgres://multica:multica@localhost:5432/multica?sslmode=disable
PORT=8080
FRONTEND_PORT=3000
```

### Worktree

worktree の内部から `.env.worktree` を生成します:

```bash
make worktree-env
```

これにより、次のような値が生成されます:

```bash
POSTGRES_DB=multica_my_feature_702
POSTGRES_PORT=5432
PORT=18782
FRONTEND_PORT=13702
DATABASE_URL=postgres://multica:multica@localhost:5432/multica_my_feature_702?sslmode=disable
```

補足:

- `POSTGRES_DB` は worktree ごとに一意である
- `POSTGRES_PORT` は `5432` に固定されたままである
- バックエンドとフロントエンドのポートは worktree のパスハッシュから導出される
- `make worktree-env` は既存の `.env.worktree` の上書きを拒否する

worktree の env ファイルを再生成するには:

```bash
FORCE=1 make worktree-env
```

## 初回セットアップ

### クイックスタート（推奨）

任意のチェックアウト（メインまたは worktree）から:

```bash
make dev
```

この 1 つのコマンドで:

- メインチェックアウトか worktree かを自動検出する
- 適切な env ファイル（`.env` または `.env.worktree`）が存在しなければ作成する
- 前提条件（Node.js、pnpm、Go、Docker）がインストールされているか確認する
- JavaScript の依存関係をインストールする
- 共有 PostgreSQL コンテナが実行中であることを保証する
- アプリケーションデータベースが存在しなければ作成する
- すべてのマイグレーションを実行する
- バックエンドとフロントエンドの両方を起動する

### 明示的なセットアップ（上級者向け）

セットアップと起動を個別に制御したい場合:

#### メインチェックアウト

```bash
cp .env.example .env
make setup-main
make start-main
```

停止:

```bash
make stop-main
```

#### Worktree

```bash
make worktree-env
make setup-worktree
make start-worktree
```

停止:

```bash
make stop-worktree
```

## 推奨される日常ワークフロー

### メインチェックアウト

`main` 向けに安定したローカル環境が欲しいときは、メインチェックアウトを使用します。

```bash
make start-main
make stop-main
make check-main
```

### 機能開発用の Worktree

分離されたデータと別々のアプリポートが欲しいときは、worktree を使用します。

```bash
git worktree add ../multica-feature -b feat/my-change main
cd ../multica-feature
make dev
```

その後の日常コマンドは次のとおりです:

```bash
make dev              # 起動（必要ならセットアップを再実行、冪等）
make stop-worktree    # 停止
make check-worktree   # 検証
```

## メインと Worktree を同時に実行する

これは第一級のワークフローです。

例:

- メインチェックアウト
  - データベース: `multica`
  - バックエンド: `8080`
  - フロントエンド: `3000`
- worktree チェックアウト
  - データベース: `multica_my_feature_702`
  - バックエンド: `18782` のような生成された worktree ポート
  - フロントエンド: `13702` のような生成された worktree ポート

両方のチェックアウトは以下を使用します:

- 同じ PostgreSQL コンテナ
- 同じ PostgreSQL ポート `5432`

しかし、それぞれ異なるデータベースを使用するため、アプリケーションデータは共有されません。

## コマンドリファレンス

### 共有インフラストラクチャ

共有 PostgreSQL コンテナを起動する:

```bash
make db-up
```

共有 PostgreSQL コンテナを停止する:

```bash
make db-down
```

重要:

- `make db-down` はコンテナを停止するが、Docker ボリュームは保持する
- ローカルデータベースは保存される

### アプリのライフサイクル

メインチェックアウト:

```bash
make setup-main
make start-main
make stop-main
make check-main
```

Worktree:

```bash
make worktree-env
make setup-worktree
make start-worktree
make stop-worktree
make check-worktree
```

現在のチェックアウト向けの汎用ターゲット:

```bash
make setup
make start
make stop
make check
make dev
make test
make migrate-up
make migrate-down
```

これらの汎用ターゲットは、現在のディレクトリに有効な env ファイルが必要です。

## データベース作成の仕組み

データベースの作成は自動です。

以下のコマンドはすべて、続行する前に対象のデータベースが存在することを保証します:

- `make setup`
- `make start`
- `make dev`
- `make test`
- `make migrate-up`
- `make migrate-down`
- `make check`

そのロジックは `scripts/ensure-postgres.sh` にあります。

## テスト

すべてのローカルチェックを実行する:

```bash
make check-main
```

または worktree から:

```bash
make check-worktree
```

これは以下を実行します:

1. TypeScript の型チェック
2. TypeScript のユニットテスト
3. Go のテスト
4. Playwright の E2E テスト

補足:

- Go のテストは独自のフィクスチャデータを作成する
- E2E テストは独自のワークスペースと issue のフィクスチャを作成する
- チェックフローは、バックエンド/フロントエンドがまだ実行されていない場合にのみ起動する

## ローカル Codex デーモン

ローカルデーモンを実行する:

```bash
make daemon
```

デーモンは CLI に保存されたトークン（`multica login`）を使用して認証します。
CLI の設定から、監視対象のすべてのワークスペースに対してランタイムを登録します。

## フルスタックの分離テスト

このセクションでは、完全に分離された環境で、スタック全体（バックエンド、フロントエンド、デーモン）をソースから実行する方法を説明します。複数のコンポーネントにまたがるエンドツーエンドの変更をテストする場合や、人間の介入をまったく必要としない CI/AI 自動化ワークフローに役立ちます。

### なぜ単に `make daemon` ではないのか？

`make daemon` は、システムにインストールされた CLI に保存されたトークンを使用し、`~/.multica/config.json` に設定されたサーバーに接続します。共有サーバーに対する日常的な開発には問題ありませんが、完全に分離されたテストには次のものが必要です:

- ローカルのバックエンドとフロントエンド（ソースから）
- 独自のプロファイルを持つローカルのデーモン（ソースから）
- 自動化された認証（ブラウザログインなし）
- 本番用の CLI 設定への干渉なし

### 動的なプロファイル命名

複数の機能を並行して実行するときの衝突を避けるため、各 worktree は一意のデーモンプロファイルを使用する必要があります。

プロファイル名は、`scripts/init-worktree-env.sh` と同じ slug + hash パターンを使って worktree ディレクトリから導出されます:

```bash
WORKTREE_DIR="$(basename "$PWD")"
SLUG="$(printf '%s' "$WORKTREE_DIR" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/_/g; s/__*/_/g; s/^_//; s/_$//')"
HASH="$(printf '%s' "$PWD" | cksum | awk '{print $1}')"
OFFSET=$((HASH % 1000))
PROFILE="dev-${SLUG}-${OFFSET}"
```

例: `../multica-feat-auth` にある worktree は、その worktree のポートとデータベース割り当てに一致する `dev-multica_feat_auth-347` というプロファイルを生成します。

### 分離環境を起動する

すべての手順を worktree のルート（Makefile がある場所）から実行します。

#### 1. バックエンド、フロントエンド、データベースを起動する

```bash
make dev
```

バックエンドが正常な状態になるまで待ちます:

```bash
PORT=$(grep '^PORT=' .env.worktree 2>/dev/null || grep '^PORT=' .env | head -1 | cut -d= -f2)
PORT=${PORT:-8080}
SERVER="http://localhost:${PORT}"

for i in $(seq 1 30); do
  curl -sf "$SERVER/health" > /dev/null 2>&1 && break
  sleep 2
done
```

#### 2. テストユーザーとトークンを作成する（自動認証）

決定論的なローカル自動化のために、バックエンドを起動する前に env ファイルで `MULTICA_DEV_VERIFICATION_CODE=888888` を設定します:

```bash
curl -s -X POST "$SERVER/auth/send-code" \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@localhost"}'

JWT=$(curl -s -X POST "$SERVER/auth/verify-code" \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@localhost", "code": "888888"}' | jq -r '.token')

PAT=$(curl -s -X POST "$SERVER/api/tokens" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"name": "auto-dev", "expires_in_days": 365}' | jq -r '.token')
```

#### 3. ワークスペースを作成する

```bash
WS=$(curl -s -X POST "$SERVER/api/workspaces" \
  -H "Authorization: Bearer $PAT" \
  -H "Content-Type: application/json" \
  -d '{"name": "Dev", "slug": "dev"}' | jq -r '.id')
```

#### 4. プロファイル名を計算し、CLI 設定を書き込む

```bash
# プロファイルを計算する（上記の「動的なプロファイル命名」を参照）
WORKTREE_DIR="$(basename "$PWD")"
SLUG="$(printf '%s' "$WORKTREE_DIR" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/_/g; s/__*/_/g; s/^_//; s/_$//')"
HASH="$(printf '%s' "$PWD" | cksum | awk '{print $1}')"
OFFSET=$((HASH % 1000))
PROFILE="dev-${SLUG}-${OFFSET}"

FRONTEND_PORT=$(grep '^FRONTEND_PORT=' .env.worktree 2>/dev/null || grep '^FRONTEND_PORT=' .env | head -1 | cut -d= -f2)
FRONTEND_PORT=${FRONTEND_PORT:-3000}

CONFIG_DIR="$HOME/.multica/profiles/$PROFILE"
mkdir -p "$CONFIG_DIR"

cat > "$CONFIG_DIR/config.json" << EOF
{
  "server_url": "$SERVER",
  "app_url": "http://localhost:${FRONTEND_PORT}",
  "token": "$PAT",
  "workspace_id": "$WS",
  "watched_workspaces": [{"id": "$WS", "name": "Dev"}]
}
EOF
```

#### 5. デーモンをソースから起動する

```bash
make cli ARGS="daemon start --profile $PROFILE"
```

デーモンは現在の worktree の Go ソースから実行され、ローカルのバックエンドに接続します。エージェントが実行する `multica` コマンドは、自動的に同じバイナリを使用します（デーモンは自身のディレクトリを `PATH` の先頭に追加します）。

### 分離環境を停止する

```bash
# プロファイルを計算する（同じ式）
PROFILE="dev-$(printf '%s' "$(basename "$PWD")" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/_/g; s/__*/_/g; s/^_//; s/_$//')-$(( $(printf '%s' "$PWD" | cksum | awk '{print $1}') % 1000 ))"

# 1. デーモンを停止する
make cli ARGS="daemon stop --profile $PROFILE"

# 2. バックエンド + フロントエンドを停止する
make stop            # メインチェックアウト
make stop-worktree   # worktree チェックアウト

# 3. （任意）共有 PostgreSQL を停止する
make db-down

# 4. （任意）ビルド成果物をクリーンアップする
make clean

# 5. （任意）プロファイル設定を削除する
rm -rf "$HOME/.multica/profiles/$PROFILE"
```

### デスクトップアプリのローカルテスト

Electron デスクトップアプリをローカルバックエンドに対してテストするには:

```bash
# バックエンドが実行中になった後（make dev）
pnpm dev:desktop
```

これは自動的に以下を行います:

1. `server/cmd/multica` から `multica` CLI を
   `apps/desktop/resources/bin/multica` へコンパイルする
2. `desktop-localhost-<PORT>` という名前の分離プロファイルを作成する
3. 独自のデーモンインスタンスを起動して管理する
4. ローカルのバックエンドに接続する

デスクトップ UI で `dev@localhost` と、バックエンドログからの生成されたコードを使ってログインします。バックエンドを起動する前に `MULTICA_DEV_VERIFICATION_CODE=888888` を設定していれば、代わりに `888888` を使えます。

バックエンドがデフォルト以外のポート（worktree）で実行される場合は、
`apps/desktop/.env.development.local` を作成します:

```bash
VITE_API_URL=http://localhost:<backend-port>
VITE_WS_URL=ws://localhost:<backend-port>/ws
```

#### 複数の worktree を並べて実行する

`pnpm dev:desktop` は worktree を自動的に分離するため、複数の worktree がそれぞれ独自のデスクトップ開発インスタンスを同時に実行できます — 追加のセットアップは不要です。リンクされた worktree からは、worktree のパスに基づいて次の値を導出します（バックエンド/フロントエンドのポートと同じ `.env.worktree` の `cksum % 1000` オフセット）:

- `DESKTOP_RENDERER_PORT` = `5174 + offset` — 独自の Vite 開発サーバー（`5174`
  をベースにすることで、`offset` が `0` でもプライマリチェックアウト用に `5173`
  を残す）
- `DESKTOP_APP_SUFFIX` = `<folder>-<offset>` — 独自のシングルインスタンスロック /
  `userData`、および `Multica Canary <folder>-<offset>` という名前のアプリで、
  Cmd+Tab で区別できるようにする。オフセットにより、異なるパスでフォルダ名を
  共有する worktree 間でも一意に保たれる。

プライマリチェックアウトはそのまま（`5173`、`Multica Canary`）残されます。導出された値を上書きするには、いずれかの環境変数を明示的に設定します。各インスタンスがどのバックエンドと通信するかは、依然として上記の `apps/desktop/.env*` のみで制御されます — 各 worktree のデスクトップを独自のバックエンドに向けることで、デーモンプロファイルも分離できます。

### 分離の保証

このフローのどの部分も、システムにインストールされた `multica` やデフォルトの
`~/.multica/config.json` には触れません:

| リソース | システム / 本番 | ローカル開発（worktree ごと） |
|---|---|---|
| Config | `~/.multica/config.json` | `~/.multica/profiles/dev-<slug>-<hash>/config.json` |
| Daemon PID | `~/.multica/daemon.pid` | `~/.multica/profiles/dev-<slug>-<hash>/daemon.pid` |
| Health port | `19514` | `19514 + 1 + (name_hash % 1000)` |
| Workspaces dir | `~/multica_workspaces/` | `~/multica_workspaces_dev-<slug>-<hash>/` |
| Database | remote / production | local Docker: `multica_<slug>_<hash>` |
| Desktop profile | `desktop-api.multica.ai` | `desktop-localhost-<port>` |

複数の worktree を、衝突することなく同時に実行できます。

## トラブルシューティング

### env ファイルがない

次のように表示された場合:

```text
Missing env file: .env
```

または:

```text
Missing env file: .env.worktree
```

まず、期待される env ファイルを作成してください。

メインチェックアウト:

```bash
cp .env.example .env
```

Worktree:

```bash
make worktree-env
```

### チェックアウトがどのデータベースを使用しているか確認する

env ファイルを調べます:

```bash
cat .env
cat .env.worktree
```

以下を確認します:

- `POSTGRES_DB`
- `DATABASE_URL`
- `PORT`
- `FRONTEND_PORT`

### 共有 PostgreSQL 内のすべてのローカルデータベースを一覧表示する

```bash
docker compose exec -T postgres psql -U multica -d postgres -At -c "select datname from pg_database order by datname;"
```

### Worktree が誤ってメインデータベースを使用している

worktree に `.env` が含まれていないか確認します。

含まれているべきではありません。

安全な worktree のセットアップは次のとおりです:

```bash
make worktree-env
make setup-worktree
make start-worktree
```

### アプリは停止するが PostgreSQL は動き続ける

これは想定どおりです。

- `make stop`
- `make stop-main`
- `make stop-worktree`

これらはバックエンド/フロントエンドのプロセスのみを停止します。

共有 PostgreSQL コンテナを停止するには:

```bash
make db-down
```

## 破壊的リセット

PostgreSQL を停止しつつローカルデータベースを保持したい場合:

```bash
make db-down
```

現在のチェックアウトだけを新しいデータベースにしたい場合（`POSTGRES_DB` で指定された
データベースを削除し、再作成し、すべてのマイグレーションを実行する）:

```bash
make stop        # 先にバックエンド/フロントエンドを停止する
make db-reset
make start
```

- 現在の env のデータベースにのみ影響する。他の worktree のデータベースには触れない
- `DATABASE_URL` がリモートホストを指している場合は実行を拒否する
- 特定の worktree を対象にするには `ENV_FILE=.env.worktree` を渡す

このリポジトリのすべてのローカル PostgreSQL データを消去したい場合:

```bash
docker compose down -v
```

警告:

- これは共有 Docker ボリュームを削除する
- これはそのボリューム内のメインデータベースとすべての worktree データベースを削除する
- その後、`make setup-main` または `make setup-worktree` を再度実行する必要がある

## 典型的なフロー

### 安定したメイン環境

```bash
make dev
```

### 機能開発用の Worktree

```bash
git worktree add ../multica-feature -b feat/my-change main
cd ../multica-feature
make dev
```

### 以前に設定した Worktree に戻る

```bash
cd ../multica-feature
make start-worktree
```

### プッシュ前の検証

メインチェックアウト:

```bash
make check-main
```

Worktree:

```bash
make check-worktree
```

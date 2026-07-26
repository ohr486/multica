# セルフホスティングのセットアップ（AIエージェント向け）

このドキュメントは AI エージェントが実行することを想定して書かれています。以下の手順を正確に実行して、ローカルの Multica インスタンスをデプロイし、接続してください。

## 前提条件

- Docker および Docker Compose がインストール済みであること
- Homebrew がインストール済みであること（CLI 用）
- PATH 上に少なくとも1つの AI エージェント CLI があること: `claude` または `codex`

## インストール

```bash
# CLI をインストールし、セルフホストサーバーをプロビジョニングする
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server

# CLI を localhost 向けに設定し、認証して、デーモンを起動する
multica setup self-host
```

`multica setup self-host` を実行する前に、サーバーの出力 `✓ Multica server is running and CLI is ready!` を待ってください。

**期待される結果:**
- フロントエンド: http://localhost:3000
- バックエンド: http://localhost:8080
- `multica` CLI がインストールされ、localhost 向けに設定されている

## 代替手段: 手動セットアップ

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make selfhost
brew install multica-ai/tap/multica
multica setup self-host
```

`multica setup self-host` コマンドは以下を行います。
1. CLI を localhost:8080 / localhost:3000 に接続するよう設定する
2. ログイン用にブラウザを開く — メールで届いたコード、または Resend が未設定の場合にバックエンドログに出力される生成コードを使う
3. ワークスペースを自動検出する
4. デーモンをバックグラウンドで起動する

## 検証

```bash
multica daemon status
```

検出されたエージェントとともに `running` と表示されるはずです。

## 停止

```bash
# デーモンを停止する
multica daemon stop

# すべての Docker サービスを停止する
cd multica
make selfhost-stop
```

## カスタムポート

デフォルトのポート（8080/3000）が使用中の場合:

1. `.env` を編集し、`PORT` と `FRONTEND_PORT` を変更する
2. `make selfhost` を実行する
3. `multica setup self-host --port <PORT> --frontend-port <FRONTEND_PORT>` を実行する

## トラブルシューティング

- **バックエンドが準備できていない:** `docker compose -f docker-compose.selfhost.yml logs backend`
- **フロントエンドが準備できていない:** `docker compose -f docker-compose.selfhost.yml logs frontend`
- **デーモンの問題:** `multica daemon logs`
- **ヘルスチェック:** 稼働確認には `curl http://localhost:8080/health`、依存関係を考慮した準備状態確認には `curl http://localhost:8080/readyz`

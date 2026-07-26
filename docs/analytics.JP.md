# プロダクトアナリティクス

Multica のプロダクトアナリティクスは、**運用データベース** と **Prometheus / Grafana** に存在します。このドキュメントは、計装（instrumentation）のカタログとその履歴です。

もともとの設計コンテキストについては [MUL-1122](https://github.com/multica-ai/multica)、後述の PostHog 廃止については [MUL-4127](https://github.com/multica-ai/multica) を参照してください。

> **MUL-4127 — プロダクトアナリティクス用途で PostHog を廃止。** PostHog は、すでに DB からクエリしているデータの、混沌としてほとんど使われていない2つ目のコピーになっていたため、冗長な計装を削除しました。
>
> - **すべてのサーバーサイドイベントは、現在 Prometheus 専用です。** `signup`、
>   `workspace_created`、`issue_created`、`issue_executed`、`chat_message_sent`、
>   `team_invite_sent` / `team_invite_accepted`、`onboarding_started` /
>   `onboarding_questionnaire_submitted` / `onboarding_completed`、
>   `agent_created`、`cloud_waitlist_joined`、`feedback_submitted`、
>   `contact_sales_submitted`、`squad_created`、`autopilot_created` — これらはすべて、
>   現在 `analytics.IsMetricsOnly` によってフラグ付けされており、`metrics.RecordEvent`
>   は Grafana のカウンターをインクリメントしますが、もはや PostHog へは送信しません。
>   `analytics.*` イベントコンストラクタは、これらの Prometheus カウンターを駆動するためだけに保持されています。
>   基盤となる DB の行は、引き続き信頼できる情報源（source of truth）のままです。
>   ランタイムライフサイクル（`runtime_*`）、autopilot run ライフサイクル
>   （`autopilot_run_*`）、および `agent_task_*` は、すでに Prometheus 専用でした。
> - **フロントエンドのファネル計装は削除されました**: `$pageview`、
>   `download_intent_expressed` / `download_page_viewed` / `download_initiated`、
>   フロントエンドの `onboarding_started` ミラー、`onboarding_runtime_path_selected`、
>   `onboarding_runtime_detected`、`feedback_opened`、および
>   `source_backfill_*` イベント。
> - **PostHog へ引き続き送信されるもの（フロントエンドのみ）:** `$exception` の自動キャプチャ
>   （`before_send` によるリダクション + 重複排除付き）と、デスクトップの安定性イベント
>   `client_crash` / `client_unresponsive` — DB に相当物がないエラー / クラッシュ監視です。
>   アイデンティティ（`$identify` / `$set`）は、これらを紐付けるためだけに保持されています。
> - `multica_signup_source` アトリビューション Cookie（`captureSignupSource`、
>   `$pageview` とは独立）は維持されます: これは引き続き `signup_source`
>   の Prometheus ラベルを供給します。生のソースチャネル / 国を DB に永続化すること —
>   PostHog が唯一保持していたシグナル — は、別途トラッキングされています。
>
> 以下のイベント別セクションは、参照用に過去の形状を記録しています。サーバーイベントは、
> Prometheus カウンターを駆動する `analytics.Event` を引き続き記述していますが、
> もはや PostHog の契約ではありません。

## 設定

すべてのアナリティクス送信は、環境変数によって切り替えられます（`.env.example` を参照）。

| 変数 | 意味 | デフォルト |
|---|---|---|
| `POSTHOG_API_KEY` | PostHog プロジェクトの API キー。空 = イベントは送信されません。 | `""` |
| `POSTHOG_HOST` | PostHog のホスト（US または EU クラウド、もしくはセルフホスト URL）。 | `https://us.i.posthog.com` |
| `ANALYTICS_ENVIRONMENT` | 標準の `environment` イベントプロパティに対する任意のオーバーライド。`production`、`staging`、または `dev` に正規化されます。デフォルトは `APP_ENV` に由来します。 | `APP_ENV` / `dev` |
| `ANALYTICS_DISABLED` | `POSTHOG_API_KEY` が設定されている場合でも no-op クライアントを強制するには `true`/`1` を設定します。 | `""` |

ローカル開発環境とセルフホストインスタンスは `POSTHOG_API_KEY=""` で動作するため、**運用者が明示的にオプトインしない限り、イベントはプロセスから出ていきません**。

### セルフホストインスタンス

セルフホスターは、**Multica が発行した `POSTHOG_API_KEY` を決して継承すべきではありません** —
それは彼らのユーザーの行動を私たちのアナリティクスプロジェクトへルーティングしてしまうからです。デフォルトはこれを保証します。

- `.env.example` は `POSTHOG_API_KEY=` を空で出荷します。Docker セルフホスト
  compose もデフォルトを設定しません。
- キーが未設定の場合、`NewFromEnv` は `NoopClient` を返し、起動時に
  `analytics: POSTHOG_API_KEY not set, using noop client` をログに出力します — 何も送信されないことの
  目に見える確認です。
- 自分のアナリティクスが欲しい運用者は、`POSTHOG_API_KEY` と
  `POSTHOG_HOST` を自分の PostHog プロジェクト（クラウドまたは
  セルフホストの PostHog）を指すように設定できます。
- フロントエンドは `/api/config` 経由でキーを受け取るため（PR 2 で予定）、
  セルフホストの空のサーバー設定は、フロントエンドのイベント送信も
  自動的に無効化します — 別途フロントエンドのオプトアウト配線は不要です。

## アーキテクチャ

```
handler → analytics.Client.Capture(Event)   ← non-blocking, returns immediately
                    │
                    ▼
           bounded queue (1024 events)
                    │
                    ▼
     background worker: batch + POST /batch/
                    │
                    ▼
                PostHog
```

- `analytics.Capture` は、**リクエストハンドラをブロックすることは決して許されません**。
  壊れたバックエンドがプロダクトを劣化させてはなりません — キューが満杯のとき、
  イベントは破棄され、カウントされます（`slog` + シャットダウン時の `dropped` カウンター
  で確認可能）。
- バッチは、`BatchSize` に達したとき、または `FlushEvery`
  （デフォルト 10 秒）ごとのいずれか早い方でフラッシュされます。
- `Close()` は、グレースフルシャットダウン中に残りのイベントをドレインします。
  `server/cmd/server/main.go` から `defer` 経由で呼び出されます。

## アイデンティティモデル

- **`distinct_id`** — ログイン済みイベントでは常にユーザーの UUID です。
  フロントエンドの `posthog.identify(user.id)` は、それ以前の匿名イベントを
  同じアイデンティティの下にマージするため、獲得アトリビューション（UTM / referrer）は
  サインアップをまたいで保持されます。
- **`workspace_id`** — 存在する場合、すべてのイベントにプロパティとして追加されます。v1
  は、ワークスペースレベルのメトリクスを計算するために、PostHog Groups
  Analytics（有料）ではなく、イベントプロパティフィルタリング（無料枠）を使用します。
- **PII** — イベントは完全なメールではなく `email_domain`（例: `gmail.com`）を
  保持します。完全なメールは `$set_once` 経由で person プロパティに一度だけ格納されるため、
  個別のデバッグには利用可能ですが、すべてのイベントとともに
  ブロードキャストされることはありません。
- **Person プロパティ（`$set`）** — オンボーディング中にユーザーが正当に変更できる
  可変のコホートシグナル（role、use_case、team_size、platform_preference）に使用します。
  バックエンドの `Event.Set` は `$set` にマップされます。フロントエンドのヘルパーは
  `@multica/core/analytics` の `setPersonProperties()` です。決して上書きされてはならない値
  （email、初期アトリビューション、初回完了タイムスタンプ）には `$set_once` のみを使用してください。

## 分類（過去）

これらのカテゴリは、各イベントがかつて供給していた PostHog ダッシュボードを記述していました。
MUL-4127 以降、それらのダッシュボードは廃止されました: サーバーイベントは Prometheus 専用（DB が
信頼できる情報源）となり、フロントエンドのファネルイベントは削除されました。`Status`
列は、各イベントが現在どのような状態にあるかを記録します。

| カテゴリ | イベント | MUL-4127 後のステータス |
|---|---|---|
| `core_loop` | `workspace_created`, `agent_created`, `issue_created`, `chat_message_sent`, `issue_executed`, `autopilot_created`, `squad_created` | Prometheus 専用 |
| `onboarding_support`（サーバー） | `onboarding_started`, `onboarding_questionnaire_submitted`, `onboarding_completed` | Prometheus 専用 |
| `onboarding_support`（フロントエンド） | フロントエンドの `onboarding_started` ミラー, `onboarding_runtime_path_selected`, `onboarding_runtime_detected` | **削除済み** |
| `acquisition`（サーバー） | `signup`, `cloud_waitlist_joined`, `contact_sales_submitted` | Prometheus 専用 |
| `acquisition`（フロントエンド） | `download_intent_expressed`, `download_page_viewed`, `download_initiated` | **削除済み** |
| `ops_feedback` | `feedback_submitted`（サーバー）, `feedback_opened`（フロントエンド） | サーバー → Prometheus 専用; フロントエンドは **削除済み** |
| `attribution backfill`（フロントエンド） | `source_backfill_shown` / `_submitted` / `_skipped` / `_dismissed` | **削除済み**（モーダルは維持; DB を PATCH する） |
| **引き続き PostHog にあるもの（フロントエンドのみ）** | `$exception`, `client_crash`, `client_unresponsive`, `$identify`, `$set` | **送信中** |
| `operational`（すでに Prometheus 専用） | `runtime_registered/ready/failed/offline`, `agent_task_*`, `autopilot_run_started/completed/failed` | Prometheus 専用 |

v0 コアダッシュボードは、`core_loop` に加えて、アクティベーションファネルで使用される
特定の `onboarding_support` ステップのみを使用しなければなりません。獲得、
フィードバック、およびシステム / ノイズイベントは、別々のダッシュボードに留まります。
`operational` の行は **PostHog へは送信されません** — これらのシグナルは
`multica_*` ビジネスカウンター経由で Grafana に存在します（`server/internal/metrics` を参照）。

## 標準コアプロパティ

正規のコアイベントは、エンティティが存在する場合は常にこれらのプロパティを保持すべきです。

| プロパティ | 型 | 備考 |
|---|---|---|
| `environment` | string | `production` / `staging` / `dev`; バックエンドとフロントエンドのアナリティクスクライアントによって刻印されます。 |
| `event_schema_version` | int | 現在のバージョン: `2`。 |
| `user_id` | string UUID | 判明している場合の人間ユーザーの ID。エージェント / システムイベントでは省略される場合があります。 |
| `workspace_id` | string UUID | ワークスペーススコープのイベントに必須。 |
| `agent_id` | string UUID | エージェント / タスクイベントに必須。 |
| `task_id` | string UUID | `agent_task_*` イベントに必須。 |
| `issue_id` / `chat_session_id` / `autopilot_run_id` | string UUID | タスク / エントリイベントの関連ソースエンティティ。 |
| `source` | string | 正規の値: `onboarding`, `manual`, `chat`, `autopilot`, `api`。UI サーフェスの詳細は `surface` または `trigger_source` を使用します。 |
| `runtime_mode` | string | ランタイム / エージェントタスクが関与する場合の `cloud` / `local`。 |
| `provider` | string | ランタイム / エージェントタスクが関与する場合の `claude`, `codex`, `cursor` など。 |
| `is_demo` | bool | 現在は常に `false`; 将来のデモ / テストワークスペースのフィルタリング用に予約。 |

タスクの終端イベントは、加えて `duration_ms` を保持します。失敗は
`failure_reason`、`error_type`、`will_retry` を保持します。ランタイム失敗イベントは
`recoverable` を保持します。ランタイム ready イベントは `runtime_id`、実際に計測された場合のみ
`ready_duration_ms`、そしてローカルランタイムの場合は `daemon_id` を保持します。

スキーマ v2 は、最初の正規コアメトリクススキーマです。これは、`failure_reason` を
`error_type` にミラーし、タスク / autopilot の失敗に `recoverable` を使用し、
登録パスに計測済みの継続時間があるより前に `ready_duration_ms: 0` を発行していた
初期の v1 ドラフトを置き換えます。

## イベント契約

### `signup`

新しいユーザーが作成されたときに発火します。検証コードと Google
OAuth の両方のエントリーポイントをカバーします（`findOrCreateUser` が唯一の発行箇所です）。

| プロパティ | 型 | 説明 |
|---|---|---|
| `email_domain` | string | ユーザーのメールのドメイン部分を小文字化したもの。 |
| `signup_source` | string | フロントエンドの Cookie `multica_signup_source` からの不透明なアトリビューションバンドル（UTM + referrer）。Cookie が存在しない場合は空。 |
| `auth_method` | string | 任意。Google OAuth サインアップの場合は `"google"`。検証コードサインアップでは存在しません。 |

過去の PostHog person プロパティ（`$set_once`） — MUL-4127 以降、`signup` は現在 Prometheus 専用で
PostHog に到達しないため、**もはや発行されません**:

| プロパティ | 型 | 説明 |
|---|---|---|
| `email` | string | 完全なメール。イベント単位でブロードキャストされることは決してありませんでした。 |
| `signup_source` | string | アトリビューションバンドル。今日では、そのバケット化された形式のみが `multica_signup_total{signup_source}` の Prometheus ラベルとして残存しています（`NormalizeSignupSource` を参照）。セグメンテーション用の person プロパティとしてはもう設定されません。 |

### `workspace_created`

`CreateWorkspace` トランザクションが正常にコミットされた後に発火します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `workspace_id` | string (UUID) | グローバルに追加されます; ここでは明確化のため記載。 |

**「最初のワークスペース」のセグメンテーションに関する注記** — 私たちは意図的に、
発行時に `is_first_workspace` ブール値を刻印しては *いません*。それを正しく計算するには、
同時作成の下でも競合する、追加のカラムまたはトランザクションスコープのロジックが
必要になります。代わりに、PostHog はまったく同じ質問に、そのユーザーに以前の
`workspace_created` イベントがあるかどうかを見ることで答えます（「ユーザーが初めて X を行う」
ファネル、または `person_properties.$initial_event` 上のコホートを使用します）。
情報の損失はありません。

### `runtime_registered`

> **Prometheus 専用 — PostHog へは送信されません**（このドキュメントの冒頭の注記を
> 参照）。`analytics.Event` は、`metrics.IncForEvent` が Prometheus カウンターを
> 導出できるように引き続き構築されます; 以下のフィールドはその **イベント** の形状であり、
> PostHog の契約ではありません。低カーディナリティのフィールド（`runtime_mode`、
> `provider`）のみが Prometheus ラベルになります — `runtime_id` / `daemon_id` のような
> id はラベルではありません。

`(workspace_id, daemon_id, provider)` のタプルが初めて
upsert されたときに発火します。ハートビートと再登録は決して再発行しません。初回
検出には、upsert の RETURNING 句上での Postgres `xmax = 0` を使用します — 追加の
クエリなし、競合なし。

| プロパティ | 型 | 説明 |
|---|---|---|
| `runtime_id` | string (UUID) | 新しく作成された agent_runtime 行の id。 |
| `daemon_id` | string | 利用可能な場合のローカルデーモンのアイデンティティ。 |
| `runtime_mode` | string | 現在は `local`; クラウドランタイム用に予約。 |
| `provider` | string | 例: `"codex"`, `"claude"`。 |
| `runtime_version` | string | エージェントランタイムバイナリのバージョン。 |
| `cli_version` | string | それを登録した `multica` CLI のバージョン。 |

`distinct_id` は、デーモンがメンバーの JWT/PAT 経由で登録された場合は認証済みの
オーナーのユーザー id です; デーモントークンによる登録は
`workspace:<workspace_id>` にフォールバックするため、PostHog は無関係なデーモンを
単一の「匿名」person の下にバケット化しません。

### `runtime_ready`

> **Prometheus 専用 — PostHog へは送信されません。**

ランタイムがオンライン / ready の状態で初めて登録されたときに発火します。これは、
`runtime_registered` を ready の証拠として扱うことを置き換えるべきアクティベーション
ファネルのステップです。バックエンドは、新しい `agent_runtime` 行の INSERT パス上でのみ
これを発行します; 通常のデーモン再接続は既存の行を更新し、別の `runtime_ready` を
発行しません。ダッシュボードのファネルは、引き続き個別の `runtime_id` をカウントすべきです。

| プロパティ | 型 | 説明 |
|---|---|---|
| `runtime_id` | string (UUID) | `agent_runtime` 行の id。 |
| `daemon_id` | string | 利用可能な場合のローカルデーモンのアイデンティティ。 |
| `ready_duration_ms` | int64 | 任意。登録開始から ready までの時間; 登録パスが計測できるようになるまで省略されます。 |
| `runtime_mode` | string | `local` / `cloud`。 |
| `provider` | string | ランタイムプロバイダー。 |

### `runtime_failed`

> **Prometheus 専用 — PostHog へは送信されません。**

ready なランタイムが記録される前にランタイムのセットアップ / 登録が失敗したときに
発火します。今日、これはバックエンドの登録永続化の失敗にスコープされています;
将来のセットアップフローは、プロバイダー検出やデーモンブートの失敗に対してこれを再利用すべきです。

| プロパティ | 型 | 説明 |
|---|---|---|
| `daemon_id` | string | 利用可能な場合のローカルデーモンのアイデンティティ。 |
| `provider` | string | 試行されたランタイムプロバイダー。 |
| `failure_reason` | string | 安定した粗い理由。 |
| `error_type` | string | 安定したエラー分類子。 |
| `recoverable` | bool | セットアップを再試行すれば成功する可能性があるかどうか。 |

### `runtime_offline`

> **Prometheus 専用 — PostHog へは送信されません。**

ランタイムが明示的に登録解除されたとき、またはバックエンドのスイーパーが
ハートビートの欠落後にオフラインとしてマークしたときに発火します。これはアクティベーションの
ステップではありません; ローカルランタイムのリテンションと離脱の診断をサポートします。

### `issue_created`

Issue の行が作成された後に発火します。手動の UI/API による Issue 作成、
エージェントによるクイック作成 Issue、および autopilot の `create_issue` 実行を含みます。

| プロパティ | 型 | 説明 |
|---|---|---|
| `issue_id` | string (UUID) | 作成された Issue。 |
| `agent_id` | string (UUID) | 該当する場合、エージェントの担当者または作成エージェント。 |
| `task_id` | string (UUID) | クイック作成 Issue 作成に存在します。 |
| `autopilot_run_id` | string (UUID) | autopilot が作成した Issue に存在します。 |
| `source` | string | `manual`、`api`、または `autopilot`。 |

### `chat_message_sent`

ユーザーのチャットメッセージが永続化され、対応するエージェントタスクが
キューに入れられた後に発火します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `chat_session_id` | string (UUID) | チャットセッション。 |
| `task_id` | string (UUID) | キューに入れられたエージェントタスク。 |
| `agent_id` | string (UUID) | チャットエージェント。 |
| `source` | string | 常に `chat`。 |

### エージェントタスクライフサイクル（Prometheus 専用）

> **PostHog へは送信されず、`analytics.Event` を持ちません。** エージェントタスクの
> ライフサイクルは、`server/internal/service/task.go` の型付き
> `BusinessMetrics.RecordTask*` メソッドによって Prometheus に直接記録されます。
> 古い PostHog イベント名（`agent_task_queued` / `dispatched` / `started` /
> `completed` / `failed` / `cancelled`）とそのプロパティ（`task_id`、
> `agent_id`、`issue_id`、`chat_session_id`、`autopilot_run_id`、`duration_ms`、
> `error_type`、`will_retry`）は、どこにも存在しなくなりました — これらの高カーディナリティな
> id は Prometheus ラベルであったことはなく、ダッシュボードや調整（reconciliation）で
> 使用してはなりません。

実際のメトリクス（`server/internal/metrics/business.go` で定義; ラベル
セットは `server/internal/metrics/labels.go`）:

| メトリクス | 型 | ラベル |
|---|---|---|
| `multica_agent_task_enqueued_total` | counter | `source`, `runtime_mode` |
| `multica_agent_task_dispatched_total` | counter | `source`, `runtime_mode` |
| `multica_agent_task_started_total` | counter | `source`, `runtime_mode`, `provider` |
| `multica_agent_task_terminal_total` | counter | `source`, `runtime_mode`, `terminal_status` |
| `multica_agent_task_failed_total` | counter | `source`, `runtime_mode`, `failure_reason` |
| `multica_agent_task_queue_wait_seconds` | histogram | `source`, `runtime_mode` |
| `multica_agent_task_run_seconds` | histogram | `source`, `runtime_mode`, `terminal_status` |
| `multica_agent_task_total_seconds` | histogram | `source`, `runtime_mode`, `terminal_status` |

- `terminal_status` は、タスクの最終的な `agent_task_queue.status` です —
  `completed` / `failed` / `cancelled`。**別個の** completed/cancelled メトリクスは
  ありません: 3つすべてが
  `multica_agent_task_terminal_total{terminal_status=…}` に着地します。失敗は
  加えて、粗い `failure_reason`（`agent_task_queue.failure_reason`、デフォルト
  `agent_error`）を保持する `multica_agent_task_failed_total` をインクリメントします。
- タスクの実時間（wall-clock）は `*_seconds` ヒストグラム（キュー待ち / run /
  total）に存在し、古い `duration_ms` イベントプロパティを置き換えます。
- `source` / `runtime_mode` / `provider` は正規化されたラベル値です
  （`NormalizeTaskSource` / `NormalizeRuntimeMode` / `NormalizeRuntimeProvider`）。

### `autopilot_run_started` / `autopilot_run_completed` / `autopilot_run_failed`

> **Prometheus 専用 — PostHog へは送信されません。** `analytics.*` コンストラクタは、
> `metrics.IncForEvent` が Prometheus カウンターを導出できるようにするためだけに保持されています;
> `analytics.IsMetricsOnly` がそれらを PostHog から排除します。`cadence`、
> `trigger_kind`、`terminal_status` のみが Prometheus ラベルになります — 以下の
> `autopilot_id` / `autopilot_run_id` / `agent_id` フィールドはイベントの形状であり、
> ラベルではありません。

`autopilot_run` のライフサイクル変更から発火します。`source` は常に
`autopilot` です; トリガーの起点は `trigger_source`（`manual`、
`schedule`、`webhook`、または `api`）に保持されます。

| プロパティ | 型 | 説明 |
|---|---|---|
| `autopilot_id` | string (UUID) | autopilot の定義。 |
| `autopilot_run_id` | string (UUID) | run の行。 |
| `agent_id` | string (UUID) | 割り当てられたエージェント。 |
| `trigger_source` | string | `manual`、`schedule`、`webhook`、または `api`。 |
| `duration_ms` | int64 | 終端イベントのみ。 |
| `failure_reason` | string | 失敗イベントのみ。 |
| `error_type` | string | 失敗イベントのみ; `configuration`、`issue_terminal`、`dispatch_error`、`task_error`、`autopilot_error` のような安定した粗い分類子。 |
| `will_retry` | bool | 失敗イベントのみ; autopilot の再試行のリズムはトリガー / スケジュールが所有するため、現在は `false`。 |

### `issue_executed`

**Issue ごとに最大1回** 発火します — その Issue 上の最初のタスクが
終端の `done` 状態に到達したときです。アトミックな
`UPDATE issue SET first_executed_at = now() WHERE id = $1 AND first_executed_at IS NULL RETURNING *`
に支えられています;
再試行、再割り当て、およびコメントによってトリガーされたフォローアップタスクはすべて
WHERE 句にヒットして no-op となるため、`≥1 / ≥2 / ≥5 / ≥10` のファネルバケットは
タスクではなく、個別の Issue をカウントします。

| プロパティ | 型 | 説明 |
|---|---|---|
| `issue_id` | string (UUID) | |
| `task_id` | string (UUID) | 完了させたタスク。 |
| `agent_id` | string (UUID) | 完了させたエージェント。 |
| `source` | string | `manual`、`chat`、または `autopilot`。 |
| `runtime_mode` | string | `local` / `cloud`。 |
| `provider` | string | ランタイムプロバイダー。 |
| `task_duration_ms` | int64 | `task.started_at` と `task.completed_at` の間の実時間。タスクが完了状態で作成された場合はゼロ（まれ）。 |

`distinct_id` は Issue の人間の作成者を優先するため、エージェントが実行した
イベントは Issue の作成者の person プロファイルに流れます（`signup` と
`workspace_created` が着地するのと同じ場所）。エージェントが作成した Issue は、PostHog が
エージェントをユーザーレコードにマージしないように、`agent:` で接頭辞を付けます。

**ワークスペース内 N 番目の序数に関する注記** — 私たちは意図的に、
発行時に `nth_issue_for_workspace` を刻印しては *いません*。それを正しく計算するには、
直列化されたトランザクション、またはワークスペースごとのアドバイザリロックが必要になります;
そうでなければ、2つの同時初回完了が両方とも `count=1` を読み取り、
`n=1` を発行する可能性があります。PostHog は、クエリ時に
`row_number() OVER (PARTITION BY properties.workspace_id ORDER BY timestamp)` 経由で
同じ質問に答えます。「ワークスペースが `issue_executed`
イベントを ≥2 回持っている」という形式のファネルステップは、このプロパティなしで
表現可能です。情報の損失はありません。

`issue_executed` は正規のコアループ成功シグナルです。MUL-4127 以降、
これはすべてのサーバーイベントと同様にメトリクス専用です: Prometheus に
`multica_issue_executed_total{source}`（PostHog ではない）として記録され、DB では
`issue.first_executed_at` に支えられます。タスク単位の完了カウントは、
`BusinessMetrics.RecordTaskTerminal` 経由で Grafana に存在します; アクティベーション
ファネルには `multica_issue_executed_total` を使用し、必要に応じて `source` で内訳を出してください。

### `team_invite_sent`

DB 行が書き込まれた後、`CreateInvitation` から発火します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `invited_email_domain` | string | 小文字化されたドメイン; 完全なメールはイベントではなく招待の行に存在します。 |
| `invite_method` | string | 現在は常に `"email"`。将来の非メール招待フロー（共有リンク、SCIM）は、独自の値を渡すべきです。 |

`distinct_id` は招待者のユーザー id です。

### `team_invite_accepted`

招待の行が承認済みとしてマークされ、メンバーの行が同じトランザクション内で
挿入された後の両方が完了してから、`AcceptInvitation` から発火します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `days_since_invite` | int64 | 招待作成から承認までの丸日数。「同日に承認」（ウォーム）と「数週間後にメールから掘り起こした」（コールド）をセグメント化できます。 |

`distinct_id` は招待された人のユーザー id です — これは拡大ファネルを閉じる
イベントです。

### `onboarding_started`

オンボーディングシェルがマウントされ、初期ワークスペースリストが解決されたときに
一度だけ発火します。既存ワークスペースのユーザーは `workspace_id` を保持します; まったく新しいユーザーは
まだワークスペースを持ちません。

| プロパティ | 型 | 説明 |
|---|---|---|
| `workspace_id` | string (UUID) | ユーザーがすでにワークスペースを持っている場合のみ存在します。 |
| `source` | string | 常に `onboarding`。 |

### `onboarding_questionnaire_submitted`

ユーザーのアンケート JSONB を「少なくとも1つのスロットが空」から「3つすべてが
埋まっている」（team_size、role、use_case）へ遷移させる最初の PatchOnboarding で
発火します。その時点を過ぎた変更は再発行しません — ファネルは編集ではなく、
ユーザーをカウントします。

| プロパティ | 型 | 説明 |
|---|---|---|
| `team_size` | string | `solo` / `team` / `other`。 |
| `role` | string | `developer` / `product_lead` / `writer` / `founder` / `other`。 |
| `use_case` | string | `coding` / `planning` / `writing_research` / `explore` / `other`。 |
| `team_size_has_other` | bool | ユーザーが Q1 の自由記述エスケープを埋めた場合に `true`。 |
| `role_has_other` | bool | Q2 も同様。 |
| `use_case_has_other` | bool | Q3 も同様。 |

`$set` で設定される person プロパティ（once ではありません — ユーザーは再提出する前に
戻って回答を変更できるため）:

| プロパティ | 型 | 説明 |
|---|---|---|
| `team_size` | string | コホートクエリ用にイベントプロパティをミラーします。 |
| `role` | string | 同上。 |
| `use_case` | string | 同上。 |

`distinct_id` はユーザーの id です。workspace_id はありません — アンケートは
ワークスペースごとではなく、ユーザーごとです。

### `agent_created`

`POST /api/workspaces/:id/agents` が成功するたびに発火します。オンボーディング
固有ではありません — `is_first_agent_in_workspace` プロパティが、
ステップ4のシグナルを、後のエージェント追加から分離します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `agent_id` | string (UUID) | |
| `provider` | string | エージェントがバインドされているランタイムプロバイダー（`claude`、`codex` など）。 |
| `runtime_mode` | string | バインドされたランタイムからコピーされたランタイムモード。 |
| `template` | string | エージェントをシードするために使用されたテンプレートのスラッグ（`coding` / `planning` / `writing` / `assistant`）。呼び出し元がテンプレートピッカーから来なかった場合は空。 |
| `is_first_agent_in_workspace` | bool | この挿入の前にワークスペースがエージェントを0個持っていた場合に `true`。 |

`distinct_id` は認証済みのオーナーのユーザー id です。

### `onboarding_completed`

`user.onboarded_at` を実際に NULL から反転させる最初の呼び出しで、
CompleteOnboarding から発火します。再試行はサーバー側で冪等ですが、
意図的に再発行しないため、ファネルは初回完了のみをカウントします。クライアントは
ユーザーがどの出口を取ったかをラベル付けするために、POST ボディに `completion_path` を送信します。

| プロパティ | 型 | 説明 |
|---|---|---|
| `workspace_id` | string (UUID) | ワークスペースにリンクされたオンボーディング完了に存在します。 |
| `completion_path` | string | `full` / `runtime_skipped` / `cloud_waitlist` / `skip_existing` / `invite_accept` / `unknown` のいずれか。下記参照。 |
| `joined_cloud_waitlist` | bool | `user.cloud_waitlist_email` から導出されます。`completion_path` と直交します — ユーザーは waitlist フォームを送信しつつ、なお CLI を選ぶ場合があります。 |

`$set_once` で設定される person プロパティ:

| プロパティ | 型 | 説明 |
|---|---|---|
| `onboarded_at` | string (RFC3339) | 初回完了が着地したタイムスタンプ。「X より前にオンボーディングしたユーザー」のようなコホートクエリを person_properties から直接可能にします。 |

`completion_path` の値:

- `full` — ランタイムが接続された状態でステップ5（first_issue）に到達。
- `runtime_skipped` — ランタイムを接続せずに完了（ユーザーがステップ3で Skip を押した）。
- `cloud_waitlist` — クラウド waitlist フォームを送信し、ステップ3をスキップ。
- `skip_existing` — Welcome から「以前にこれをやったことがある」。ユーザーはすでにワークスペースを持っていた。
- `invite_accept` — 少なくとも1つのワークスペース招待を承認。
- `unknown` — クライアントがパスを送信しなかった場合のレガシーフォールバック。ロールアウト後はゼロ付近に留まるべき。

### `cloud_waitlist_joined`

ユーザーがステップ3のクラウド waitlist フォームを送信するたびに JoinCloudWaitlist
から発火します。完了シグナルではありません — メインファネルと直交し、
ホスト型ランタイムへの関心の規模を測るために使用されます。

| プロパティ | 型 | 説明 |
|---|---|---|
| `has_reason` | bool | 自由記述の理由フィールドの存在フラグ。自由記述は DB に留まります; 私たちはそれをブロードキャストしません。 |

`distinct_id` はユーザーの id です。

### `contact_sales_submitted`

`contact_sales_inquiry` の行が挿入された後、`CreateContactSales` から
発火します。このエンドポイントは公開・未認証であるため、
`distinct_id` は問い合わせの id です（紐付けるユーザーのアイデンティティがありません）。
自由記述の `goals` フィールドは DB に留まり、決してブロードキャストされません。

| プロパティ | 型 | 説明 |
|---|---|---|
| `inquiry_id` | string | 安定した問い合わせ id; `distinct_id` と同じ。運用データへの結合に有用。 |
| `company_size` | string | フォームのドロップダウンからの閉じた列挙（`1-10`, `11-50`, `51-200`, `201-500`, `501-1000`, `1000+`）。 |
| `country_region` | string | ドロップダウンから送信された国 / 地域のラベル。 |
| `use_case` | string | 閉じた列挙（`evaluate` / `adopt_team` / `self_host` / `integrate` / `partner` / `other`）。 |
| `has_goals` | bool | 自由記述の goals フィールドの存在フラグ。 |

### `feedback_submitted`

`feedback` の行が挿入され、ユーザーごとの時間単位レート制限チェックが
通過した後、`CreateFeedback` から発火します。同じ時間内でレート制限された（429）
再試行は発行しません。自由記述のメッセージは DB に格納され、決してブロードキャストされません。

| プロパティ | 型 | 説明 |
|---|---|---|
| `message_length_bucket` | string | `0-100` / `100-500` / `500-2000` / `2000+` — 内容を漏らさずに「短いメモ」と「再現手順付きのバグレポート」を区別できるよう、`len(message)` の粗いバケット。 |
| `has_images` | bool | markdown に少なくとも1つの `![...](url)` 画像参照が含まれる場合に `true` — 視覚的証拠付きのバグレポートを示します。 |
| `platform` | string | `X-Client-Platform` ヘッダーからのクライアントプラットフォーム（`web` / `desktop`）。ヘッダーがない場合は省略。 |
| `app_version` | string | `X-Client-Version` ヘッダーからのクライアントバージョン。ない場合は省略。 |

`distinct_id` は送信者のユーザー id です; `workspace_id` はモーダルの
現在のワークスペースコンテキストから紐付けられ、フィードバックがワークスペース前の
サーフェスから送信された場合は空になることがあります。

### フロントエンド専用イベント

> **MUL-4127 で削除済み**。ただし `$exception`（変更なし）と、
> `client_crash` / `client_unresponsive` のデスクトップ安定性イベント（`packages/core/diagnostics`
> に文書化）を除きます。`$pageview`、`download_intent_expressed`、
> `download_page_viewed`、`download_initiated`、`onboarding_runtime_path_selected`、
> `onboarding_runtime_detected`、フロントエンドの `onboarding_started` ミラー、
> `feedback_opened`、および `source_backfill_*` はもはや発火しません。以下の説明は
> 過去の参照としてのみ保持されています。

- `$pageview` — Web トラッカー
  （`apps/web/components/pageview-tracker.tsx`）によって Next.js App Router の
  **pathname** の変更時に、そしてデスクトップトラッカー
  （`apps/desktop/.../pageview-tracker.tsx`）によって可視サーフェスの変更時に発火します。
  両者はルートで一度マウントされ、獲得ファネルの
  `/ → signup` ステップを駆動します。posthog-js の自動 pageview キャプチャは
  `initAnalytics` で無効化されており、イベントの形状を私たちが所有します。
  `capturePageview`（`packages/core/analytics`）は、発行前にパスを **section-normalize** します:
  クエリ文字列 / ハッシュは取り除かれ、リソース id
  のセグメントは畳み込まれるため、`/acme/issues/8d5c…` と `/acme/issues/MUL-12`
  はどちらも `/acme/issues` として報告され、同じセクションの連続したビューは
  重複排除されます。これにより、リソースごとまたはフィルター/ソート/検索の変更ごとに
  `$pageview` を課金するのではなく、PostHog をセクション粒度に保ちます。
  トラッカーは意図的にクエリ文字列でキー付けされていません。
- `onboarding_runtime_path_selected` — Web ユーザーが3つのステップ3のフォークカードの
  1つをクリックしたときに `packages/views/onboarding/steps/step-platform-fork.tsx` から
  発火します（サーバー呼び出しが発生する前なので、フロントエンド専用です）。プロパティ: `path`
  （`download_desktop` / `cli` / `cloud_waitlist`）、`source`
  （`onboarding`）、`surface`（`step3`）、`workspace_id`、`is_mac`。
  また、`platform_preference`（`web` / `desktop`）を person
  プロパティに書き込むため、そのユーザーのその後のすべてのイベントを、選択された
  プラットフォームで内訳できます。**注**: 意味的な「ダウンロード
  意図」は、現在では下記の `download_intent_expressed` によってよりよく提供されます —
  `path: "download_desktop"` は、実際のダウンロード開始ではなく、ステップ3のパス選択を
  具体的に示します。

- `onboarding_runtime_detected` —
  `packages/views/onboarding/steps/step-runtime-connect.tsx`（デスクトップ
  ステップ3）から、マウントごとに一度、スキャンフェーズが解決したとき — 最初のランタイム
  登録で即座に、または5秒の空タイムアウト後のいずれか — に発火します。
  「ステップ3に到達したとき、ユーザーはこのマシンに何らかの AI CLI を
  インストールしていたか」という質問に答えます — 現在、これは既存のファネルからは
  回答不能です。なぜなら、バンドルされたデーモンは PATH 上に CLI が0個のとき
  そもそも登録に失敗するため、そのコホートで
  `runtime_registered` が沈黙するからです。これは
  `completion_path=runtime_skipped` を「CLI を持っていたが、それでもスキップした」
  対「利用可能な CLI がなく、選択肢がなかった」に分割します。プロパティ:
  - `source`: `onboarding`。
  - `surface`: `step3_desktop`。
  - `workspace_id`: 現在のオンボーディングワークスペース。
  - `outcome`: `found`（5秒の猶予ウィンドウが期限切れになる前に少なくとも1つの
    ランタイムが登録された）または `empty`（その時点までに何も登録されなかった）。
  - `runtime_count`: 解決時にこのユーザーに見えるランタイムの数。
  - `online_count`: `status` が `online` である `runtime_count` のサブセット。
  - `providers`: 個別のプロバイダー名のソートされた配列（例:
    `["claude", "codex"]`）。
  - `has_claude` / `has_codex` / `has_cursor`: HogQL での配列
    フィルタリングなしにファネルの内訳を出すために、`providers` から導出された便利なブール値。
  - `detect_ms`: コンポーネントのマウントから解決までの実時間 ms。
    デーモンのブートレイテンシを可視化します — 高い `detect_ms` を持つ `found`
    イベントはタイムアウト閾値に近づき、猶予期間を延ばすべきかどうかを
    知らせます。

  `$set` で設定される person プロパティ:
  - `has_any_cli`: ブール値 — 「ユーザーがこのマシンで少なくとも1つの
    ローカル AI CLI を検出した」のコホートシグナル。
  - `detected_cli_count`: 数値 — 詳細なコホートシグナル。

  Web のステップ3（`step-platform-fork.tsx`）からは発行されません — Web
  ユーザーはバンドルされたデーモンを実行しないため、彼らのランタイムリストは
  他のマシンのデーモンを反映し、
  「ローカルに CLI がインストールされている」シグナルを破損させてしまいます。

- `download_intent_expressed` — ユーザーが `/download` ページを指す CTA を
  クリックするたびに発火します。ファネル全体で5つのソースを可視化し、
  ファネルのトップのエントリーをきれいに分割できます。
  ラッパーは `packages/core/analytics/download.ts`
  （`captureDownloadIntent`）に存在します。プロパティ:
  - `source`: `landing_hero` / `landing_footer` / `login` / `welcome`
    / `step3`
  また、`platform_preference: "desktop"` を person プロパティに書き込みます。

- `download_page_viewed` — OS 検出が解決した後、`/download` のマウントごとに一度
  発火します（`apps/web/app/(landing)/download/download-client.tsx`）。
  プロパティ:
  - `detected_os`: `mac` / `windows` / `linux` / `unknown`
  - `detected_arch`: `arm64` / `x64` / `unknown`
  - `detect_confident`: 検出が
    `userAgentData.getHighEntropyValues`（Chromium）を使用した場合に `true`; UA 文字列に
    フォールバックした場合に `false`（Mac の Safari は常にここに着地します —
    Intel 向けの arm64 デフォルトのリスクコホートを分離できます）。
  - `version_available`: GitHub API のフェッチが失敗し、
    ページが「バージョン利用不可」の劣化状態にある場合に `false`。
  また、`$set_once` 経由で `first_detected_os` / `first_detected_arch` を書き込むため、
  すべての下流イベントが再発行なしにプラットフォームの次元を得ます。

- `download_initiated` — ユーザーが `/download` 上の特定の
  インストーラーリンクをクリックしたときに発火します。ヒーロー CTA と All
  Platforms マトリクスの行の両方がこれを発行します; `primary_cta` で分割します。
  プロパティ:
  - `platform`: `mac` / `windows` / `linux`
  - `arch`: `arm64` / `x64`
  - `format`: `dmg` / `zip` / `exe` / `appimage` / `deb` / `rpm`
  - `version`: リリースタグ（例: `v0.2.13`） — 採用をリリースのリズムと相関させます。
  - `primary_cta`: ヒーロー推奨のインストーラーの場合に `true`、
    All Platforms マトリクスからの手動選択の場合に `false`。
  - `matched_detect`: 選択されたプラットフォーム+アーキテクチャがページが検出したものと
    一致する場合に `true`。`false` は、単一のイベントから検出ミスを定量化できるようにします
    （クロス結合不要）。
- `feedback_opened` — アプリ内の Feedback モーダルがマウントされたときに発火します
    （ユーザーが Help ランチャーで「Feedback」をクリックした）。バックエンドの
    `feedback_submitted` とペアになり、フォームの完了率を提供します。
    ラッパーは `packages/core/analytics/feedback.ts`
    （`captureFeedbackOpened`）に存在します。プロパティ:
  - `source`: `help_menu`（予約済み — キーボードショートカットや
    エラートースト CTA のような将来のエントリーポイントは、独自の値を渡します）
  - `workspace_id`: モーダルがワークスペース内で開くときの string (UUID)。
    ワークスペース前のサーフェスでは省略。

- アトリビューションは別個のイベントではありません; UTM + referrer の起点は、
  最初の匿名 pageview で `multica_signup_source` Cookie に書き込まれ、
  バックエンドの `signup` 発行によって読み取られます。Cookie は、書き込み時に
  URL エンコード（`encodeURIComponent`）され、読み取り時に URL デコード
  （`url.QueryUnescape`）される JSON ペイロードを保持します — JSON は決して
  途中で切り詰められません; 個々の値は `JSON.stringify` の前に 96 文字で上限が設けられ、
  ペイロード全体がなお 512 文字を超える場合は破棄されます。そうすることで、
  PostHog は無傷の JSON か、まったく何も見ないかのいずれかになります。

## 調整（Reconciliation）

タスク単位の完了はもはや PostHog へは送信されません。タスクの成功は現在、
DB ↔ PostHog の代わりに **DB ↔ Prometheus** を調整します:
`BusinessMetrics.RecordTaskTerminal` カウンター（`multica_*` タスク
メトリクスとしてエクスポート）は、運用上の信頼できる情報源を追跡すべきです:

```sql
SELECT date_trunc('day', completed_at AT TIME ZONE 'UTC') AS day,
       count(*) AS db_completed_tasks
FROM agent_task_queue
WHERE status = 'completed'
  AND completed_at >= now() - interval '30 days'
GROUP BY 1
ORDER BY 1;
```

Grafana 内の同等の Prometheus カウンターと比較してください。期待される
差はゼロ付近であるべきです; 持続的なドリフトは、発行箇所が
欠落しているか、メトリクスパイプラインが不健全であることを意味します。

`issue_executed` は、引き続きプロダクトレベルの成功シグナルです（Issue ごとに最大1回）。
MUL-4127 以降、これは Prometheus 専用であるため、PostHog イベントではなく
`multica_issue_executed_total` を `issue.first_executed_at` に対して
調整してください。

## クライアントの日次利用とローカルランタイムの状態

`client_usage_daily` は、Web/Desktop の利用状況と Desktop の組み込みプロバイダー
コンバージョンに関する運用上の信頼できる情報源です。その主キーは
`(user_id, client_type, install_id, activity_date)` であり、ここで `activity_date` は
サーバーによって UTC で導出されます。`install_id` は、Web の
オリジンまたは Electron アプリのプロファイルに格納され、再起動、アップグレード、ログアウト、
ログインをまたいで再利用されるランダムな UUID です。そのプロファイルをクリア / リセットすると、
意図的に新しいインストールが作成されます。Web と Desktop はインストール ID を決して共有しません。

クライアントは、認証後、そのインストールが現在の UTC 日について成功した
レポートを持たないときに報告し、フォーカス / 再開時に再チェックします。Desktop は、
最初のローカルランタイムプローブの後、および同日のランタイムシグネチャが変更されるたびに、
同じ日次行を更新します。レポートには、クライアントの種類 / バージョン、
粗い OS バケット、任意の現在のワークスペースコンテキスト、および集約されたランタイム
プロバイダー / online / offline のカウントのみが含まれます。サーバーが、ユーザー、日付、タイムスタンプを供給します。
デバイス名、ホスト名、ローカルのユーザー名、ファイルシステムパス、生のユーザーエージェント、
IP アドレス、および生のプローブエラーは、ここには格納されません。

`first_active_at` と `last_active_at` は、**その UTC 日内での** 最初と最新の成功した
レポートであり、インストールの生涯タイムスタンプではありません。歴史的な初回利用は、
インストールの日次行にわたる `min(first_active_at)` で計算します。クライアントの UTC 日は、
ベストエフォートのリクエストスロットルにすぎません; サーバーの
UTC 日付が信頼できるものであるため、真夜中付近のクロックスキューは、間違ったサーバー日に
アクティビティを割り当てることなく、次の行を後のフォーカス / 再開まで
遅延させることがあります。

Desktop のランタイム可用性は、この MVP では意図的にデーモンレベルの近似であり、
各プロバイダーの接続テストではありません。プローブは、ローカルで検出された
**組み込みプロバイダー CLI** をカウントします; ワークスペースのカスタムランタイムプロファイルは
含まれません。したがって、`runtime_count = 0` は「組み込みプロバイダー CLI が
検出されなかった」として扱い、ユーザーがカスタムプロファイルを持たないことの証拠としては扱わないでください。マネージドデーモンが
実行中の場合、検出されたすべての組み込みプロバイダーは online として報告され、それ以外の場合は
offline として報告されます。失敗したプローブを0ランタイムとして扱うのではなく、
`probe_result = 'error'` を unknown として使用してください。カスタムプロファイルをカバーするには、
デーモンの停止後も利用可能なままである別個のインベントリ契約が必要です;
このスナップショットからそれを推論しないでください。

30日間のクライアント分割と、ユーザーレベルの Desktop 組み込みプロバイダー
状態には、このクエリを使用してください。まず各インストールの最新の非 null プローブを選択し、
次にインストールをユーザーにロールアップするため、マルチデバイスユーザーが二重にカウントされず、
欠落したプローブは組み込みランタイムを持たないと分類されるのではなく `unknown` のままになります:

```sql
WITH window_rows AS (
    SELECT *
    FROM client_usage_daily
    WHERE activity_date >= (CURRENT_TIMESTAMP AT TIME ZONE 'UTC')::date - 29
),
active_clients AS (
    SELECT DISTINCT user_id, client_type, install_id FROM window_rows
),
latest_desktop_probe AS (
    SELECT DISTINCT ON (user_id, install_id)
        user_id, install_id, probe_result, runtime_count, online_count
    FROM window_rows
    WHERE client_type = 'desktop' AND probe_result IS NOT NULL
    ORDER BY user_id, install_id, activity_date DESC, runtime_probed_at DESC
),
desktop_by_user AS (
    SELECT
        a.user_id,
        count(*) AS installation_count,
        count(*) FILTER (WHERE p.probe_result = 'success') AS successful_probe_count,
        coalesce(sum(p.runtime_count) FILTER (WHERE p.probe_result = 'success'), 0) AS runtime_count,
        coalesce(sum(p.online_count) FILTER (WHERE p.probe_result = 'success'), 0) AS online_count
    FROM active_clients a
    LEFT JOIN latest_desktop_probe p USING (user_id, install_id)
    WHERE a.client_type = 'desktop'
    GROUP BY a.user_id
),
desktop_state AS (
    SELECT user_id,
        CASE
            WHEN successful_probe_count < installation_count THEN 'unknown'
            WHEN runtime_count = 0 THEN 'no_builtin_runtime'
            WHEN online_count = 0 THEN 'all_builtin_runtimes_offline'
            ELSE 'builtin_runtime_available'
        END AS runtime_state
    FROM desktop_by_user
)
SELECT
    (SELECT count(DISTINCT user_id) FROM active_clients WHERE client_type = 'web') AS active_web_users,
    (SELECT count(DISTINCT user_id) FROM active_clients WHERE client_type = 'desktop') AS active_desktop_users,
    count(*) FILTER (WHERE runtime_state = 'builtin_runtime_available')
        AS desktop_users_with_builtin_runtime,
    count(*) FILTER (WHERE runtime_state = 'no_builtin_runtime')
        AS desktop_users_without_builtin_runtime,
    round(100.0 * count(*) FILTER (WHERE runtime_state = 'no_builtin_runtime')
        / nullif((SELECT count(DISTINCT user_id) FROM active_clients WHERE client_type = 'desktop'), 0), 2)
        AS desktop_users_without_builtin_runtime_pct,
    count(*) FILTER (WHERE runtime_state = 'all_builtin_runtimes_offline')
        AS desktop_users_all_builtin_runtimes_offline,
    round(100.0 * count(*) FILTER (WHERE runtime_state = 'all_builtin_runtimes_offline')
        / nullif((SELECT count(DISTINCT user_id) FROM active_clients WHERE client_type = 'desktop'), 0), 2)
        AS desktop_users_all_builtin_runtimes_offline_pct,
    count(*) FILTER (WHERE runtime_state = 'unknown') AS desktop_users_unknown
FROM desktop_state;
```

初期のリテンションポリシーは 180 UTC 日です。この MVP は意図的に、
別のインプロセスバックグラウンドジョブを追加しません; 運用者は、共有リテンション
ワーカーが存在するようになるまで、既存のデータベースメンテナンススケジュールを通じて
`DELETE FROM client_usage_daily WHERE activity_date < (CURRENT_TIMESTAMP AT TIME ZONE 'UTC')::date - 179`
を実行すべきです。ワークスペースを削除するとその任意のコンテキストが null になり、一方
ユーザーを削除する際は、このテーブルがリポジトリのポリシーにより外部キーを持たないため、
そのユーザーの日次行を同じアプリケーショントランザクション内で削除しなければなりません。現在、
本番アカウントのハード削除パスはありません; 将来のいかなるものも、ユーザーの行を削除する前に、
その明示的なクリーンアップを追加しなければなりません。

## ガバナンス

いかなるイベントも追加、リネーム、または削除する前に:

1. まずこのドキュメントを更新してください。
2. 一致するように `server/internal/analytics/events.go` の定数とヘルパーを
   更新してください。
3. PR の説明は、どの既存のファネル / インサイトが影響を受けるかを明記しなければなりません。

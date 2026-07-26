# Multica デザインシステム

本ドキュメントは Multica のビジュアル言語とインタラクション規約を定義します。すべての UI 開発はこれを基準とします。

ページレベルの情報アーキテクチャ、レイアウトの構成、一貫性のガバナンスについては [`ui-consistency-audit.md`](./ui-consistency-audit.md) を参照してください。

---

## 1. デザイン哲学

3 つの核となる原則：

1. **抑制こそ高級。** デフォルトは引き算で考えます。すべての要素には存在する理由がなければなりません——余分な区切り線、装飾的なアイコン、「念のため」の説明文は、いずれもノイズです。余白そのものがデザインです。
2. **階層はグレースケールで、色はシグナル。** インターフェースの主体はニュートラルカラーです。色は意味を伝える必要があるときにのみ現れます（状態、ブランド、エラー）。2 つの領域が視覚的に注意を奪い合っているなら、解決策は片方を後退させることであって、両方に色を足すことではありません。
3. **一貫性は個性に勝る。** 同種のインタラクションは同じ視覚的フィードバックを持たなければなりません。ある hover 効果は、sidebar、dropdown、table row のいずれにおいても「同じように感じられる」べきです。この一貫性はハードコードではなく token によって実現します。

---

## 2. カラー体系

OKLCh 色空間を基礎とし、CSS 変数で定義します。すべての色は shadcn token を使用し、**Tailwind の色値のハードコードを禁止**します（`text-gray-500`、`bg-blue-600` など）。

### 2.1 Surface 階層

Surface システムが記述するのはコンテナ同士の関係であって、あらゆるコンテンツをカードに包むことではありません。基礎となる token は `packages/ui/styles/tokens.css` に定義され、light / dark mode の両方を同時にカバーします。

| 階層 | Token / Class | 用途 | 適用すべきでないもの |
|------|---------------|------|----------|
| App Shell | `app-shell` / `bg-app-shell` | ウィンドウ最外層、sidebar と page canvas の間の余白 | ページ本文、フォームグループ |
| Page Canvas | `page-canvas` / `bg-page-canvas` | ページ本体；list、board、chat など継続的にスクロールするコンテンツ領域 | 独立した設定グループ、ポップアップ |
| Surface / Card | `surface`、`surface-border`、`--surface-shadow` | 独立した境界を持つフォームグループ、設定グループ、サマリーカード | すべての行のリスト、すべての列の board、ページ全体の本文 |
| Floating Surface | `surface-raised`、`--floating-shadow` | dialog、dropdown、popover、sheet、フローティング chat | 常駐するページレイアウト |

ルール：

- Page Canvas はデフォルトのコンテンツ面です。グループ化が必要なときはまず間隔と区切り線を使い、独立した操作や独立した情報ブロックの場合にのみ Surface を使います。
- ライトモードでは、`app-shell` と sidebar は `#f3f3f4` を、`page-canvas` は `#fbfbfb` を、Card は `#ffffff` を使用します；設定グループとページ背景を border だけで区別してはいけません。この 3 者は Linear に近い抑制の効いた階層を形成し、外から内へと徐々に明るくなります。
- Card は `bg-surface border-surface-border shadow-[var(--surface-shadow)]` を使用します；フローティング層は `bg-surface-raised ring-surface-border shadow-[var(--floating-shadow)]` を使用します。
- `surface-hover` はポインタが通過していることのみを表します；`surface-selected` は持続的な選択を表し、ニュートラルグレーを保ち、brand 色を追加で重ねません。選択項目が hover されたときも `surface-selected` を必ず保持し、hover 状態に戻してはいけません。
- focus は一律で `focus-visible` ring を使用します。シャドウ、サイズ変化、あるいはブランド色の大面積の塗りつぶしでキーボードフォーカスを代替してはいけません。
- light / dark mode 用に平行な class を一組手書きしないでください。基礎となる surface token はすでにテーマごとに解決され、`color-scheme` を通じてネイティブコントロールと同期します。

### 2.2 ニュートラルカラーの階調

インターフェースの 90% の面積はニュートラルカラーで構成されます。グレースケールの等級がすなわち情報の階層です：

| 役割 | Light Token | Dark Token | 用途 |
|------|-------------|------------|------|
| 背景 | `page-canvas` / `background` | `page-canvas` / `background` | ページ本体 |
| カード/フローティング層 | `surface` / `surface-raised` | `surface` / `surface-raised` | 境界を持つコンテンツグループと一時的なオーバーレイ層 |
| 次級表面 | `muted` / `secondary` | `muted` / `secondary` | hover 背景、タグの下地色 |
| 枠線 | `border` | `border` | 区切り線、入力欄の枠線 |
| 入力欄枠線 | `input` | `input` | border よりやや重い |
| 主要文字 | `foreground` | `foreground` | 見出し、本文 |
| 次要文字 | `muted-foreground` | `muted-foreground` | 説明、メタデータ、placeholder |
| 最強調文字 | `primary` | `primary` | ボタン文字（反転色）、重要なタグ |

**ルール：** 同一画面内で、文字色は最多で 3 階層まで（`foreground` / `muted-foreground` / いずれかのセマンティックカラー）。3 階層を超える場合は階層設計に問題があることを意味します。

### 2.3 セマンティックカラー

色は意味を伝えるためだけに使い、装飾には使いません：

| Token | 意味 | 使用シーン |
|-------|------|----------|
| `brand` | ブランド識別 | Logo、ブランドボタン、ごく少量の強調 |
| `destructive` | 危険/エラー | 削除ボタン、フォームバリデーションエラー、危険な操作 |
| `success` | 成功 | 状態タグ（完了、解決済み） |
| `warning` | 警告 | 注意状態、期限リマインド |
| `info` | 情報 | ヒント、リンク、次要な情報マーク |
| `priority` | 優先度 | 高優先度タグ |

**ルール：**
- セマンティックカラーは主に小面積の要素に使います（badge、icon、border）。大面積の着色にはその色の 10%-20% 透明度のバリアント（`bg-destructive/10` など）を使います。
- 1 画面に同時に出現するセマンティックカラーは 2-3 種を超えないようにします。1 つのインターフェースに赤・黄・緑・青・紫が同時にあるなら、情報密度が高すぎることを意味し、再構成が必要です。

### 2.4 ダークモード

ダークモードは単純な反転ではありません。独立して設計された一組の配色です：

- 背景は濃いグレー（`oklch(0.18 ...)`）を使い、純黒ではありません——純黒は LCD 画面では目に刺さります。
- 枠線は `oklch(1 0 0 / 10%)`（白の 10% 透明度）を使い、light モードより繊細にします。
- セマンティックカラーは dark モードでは適度に明るくして（`success` を `0.55` から `0.65` に引き上げるなど）、コントラストを保証します。
- すべての UI 変更は 2 つのモードの両方で検証しなければなりません。

---

## 3. フォント規約

### 3.1 フォントファミリー

| 役割 | フォント | 用途 |
|------|------|------|
| 本文/UI | Inter (`--font-sans`) | すべてのインターフェース文字のデフォルトフォント；CJK 文字は自動的にシステムフォント（PingFang SC / Microsoft YaHei / Noto Sans CJK SC）へ fallback します |
| コード/データ | Geist Mono (`--font-mono`) | コードブロック、ID、タイムスタンプ、等幅データ |
| 見出し | `--font-heading`（= `--font-sans`） | ページ見出し、ブロック見出し |

フォントスタックは `apps/web/app/layout.tsx` と `apps/desktop/src/renderer/src/globals.css` の 2 か所で宣言されており、変更時は同期が必要です。

### 3.2 フォントサイズの規律

**プロジェクト全体で 3 つのコアフォントサイズ + 1 つの特殊フォントサイズのみを使用します：**

| Tailwind Class | 大きさ | 役割 | 使用シーン |
|----------------|------|------|----------|
| `text-base` (16px) | 本文 | ページ見出し、主要コンテンツ | ページ見出し、エディター本文、空状態の説明 |
| `text-sm` (14px) | デフォルト | インターフェースの主力フォントサイズ | メニュー項目、ボタン、フォーム、リスト項目、本文 |
| `text-xs` (12px) | 補助 | メタデータ、タグ | badge 文字、タイムスタンプ、ステータスバー、次要な情報 |
| `text-[0.8rem]` | 過渡 | sm ボタン限定 | shadcn button size="sm" 専用 |

**禁止：**
- `text-lg`、`text-xl`、`text-2xl` などの使用——タスク管理ツールは情報密度を追求し、大きなフォントサイズを必要としません。
- `text-[11px]`、`text-[13px]` などの任意のピクセル値の使用——Tailwind 内蔵の scale を貫きます。
- 同一ブロック内で 2 つを超えるフォントサイズを混用すること。階層を区別するために 3 つ目のフォントサイズが必要なら、まず `font-medium` vs `font-normal` や `text-muted-foreground` で解決できないか試してください。

### 3.3 フォントウェイト

2 つだけを使用します：

| フォントウェイト | 用途 |
|------|------|
| `font-normal` (400) | 本文、説明、大部分の文字 |
| `font-medium` (500) | タグ、ボタン、ナビゲーション項目、見出し、選択状態 |

**禁止** `font-bold` / `font-semibold`——タスク管理ツールは情報密度と「軽さ」の感覚を追求しており、太字は階層のリズムを壊します。より強い強調が必要なら、太字ではなく、より大きなフォントサイズや `foreground` 色値を使ってください。

---

## 4. 間隔体系

Tailwind の 4px 基礎グリッドを基礎とします。間隔は情報を伝えます——単に「見栄えが良い」だけでなく、ユーザーに「何が何に属するか」を伝えます。

### 4.1 間隔のセマンティクス

| 間隔 | Tailwind | 意味 |
|------|----------|------|
| 4px | `gap-1` / `p-1` | **密接な関連** — icon と文字、label と値 |
| 6px | `gap-1.5` / `p-1.5` | **コンポーネント内部** — ボタン内部の padding、リスト項目の間隔 |
| 8px | `gap-2` / `p-2` | **同グループ別項目** — フォームフィールド間、リスト項目間 |
| 12px | `gap-3` / `p-3` | **小節内** — カード内部の padding |
| 16px | `gap-4` / `p-4` | **グループ間の分離** — 異なるブロックの間 |
| 24px | `gap-6` / `p-6` | **大節の分離** — ページの主要領域間 |

**ルール：区切り線が必要なら、間隔が足りていない証拠です。** `<Separator />` を足すのではなく、まず間隔を広げてコンテンツを分離します。区切り線は最後の手段であるべきです。

### 4.2 コンテナ戦略（優先度順）

2 つの領域を視覚的に分離する必要があるとき：

1. **間隔のみ** — 2 つの領域の間隔を広げる（第一選択）
2. **1 本の区切り線** — 細い線 `border-border`
3. **背景色の変化** — 一方の領域に `bg-surface-hover` または `bg-surface` を使う
4. **完全なカード** — border + radius + padding（最も重い手段）

最も軽い道具で分離を完成させます。

---

## 5. インタラクション状態

これがデザインの一貫性の核心です。各状態はすべてのコンポーネントで一貫して表現されなければなりません。

### 5.1 状態階層の概観

```
デフォルト (rest) → hover → active/pressed → selected/active → focused → disabled
```

### 5.2 Hover 状態

Hover は「あなたに気づきました」であり、視覚変化は軽微かつ即時であるべきです：

| 要素タイプ | Hover 効果 | Token |
|----------|-----------|-------|
| リスト項目/メニュー項目 | 背景が薄いグレーに | `hover:bg-muted` |
| Ghost ボタン | 背景が薄いグレーに + 文字が前景色に | `hover:bg-muted hover:text-foreground` |
| 次要ボタン | 背景が 20% 濃く | `hover:bg-secondary/80` |
| 主ボタン | 背景が 20% 濃く | `hover:bg-primary/80` |
| 文字リンク | 下線が出現 | `hover:underline` |
| Tab タグ | 文字が次要から主要へ | `hover:text-foreground`（`text-muted-foreground` から） |
| アイコンボタン | 背景が薄いグレーに | `hover:bg-muted` |
| 危険ボタン | 背景の透明度が濃く | `hover:bg-destructive/20` |

**ルール：**
- hover 時にサイズを変えない（`scale` なし）、シャドウを足さない（`shadow` なし）。
- hover の背景色は常に selected/active より淡くします。こうすることでユーザーは「ホバー」と「選択済み」を区別できます。
- すべての hover は `transition-colors`、`transition-shadow`、または具体的なプロパティを列挙して使います；`transition-all` は使わないでください。時間は Tailwind のデフォルト値（150ms）が処理するので、カスタマイズは不要です。

### 5.3 Active / Selected 状態

Active は「私はすでに選択されている」であり、視覚は hover より重くなります：

| 要素タイプ | Active 効果 | Token |
|----------|------------|-------|
| Sidebar メニュー項目 | 背景 + 文字が重く + font-medium | `data-active:bg-sidebar-accent data-active:font-medium` |
| Tab | 下方のインジケーターバー + 文字が前景色に + font-medium | `data-[state=active]:text-foreground` |
| リストの選択行 | 背景が濃く | `bg-muted` または `bg-accent` |
| Toggle（オン） | 背景が反転色 | `data-[state=on]:bg-primary data-[state=on]:text-primary-foreground` |

**重要な区別：** Hover = `bg-muted`、Active = `bg-muted` + `font-medium` + `text-foreground`。Active は常に hover より 1 つ多くの視覚的次元（フォントウェイトまたは色の変化）を持ち、単に背景がより濃いだけではありません。

### 5.3.1 Active は Hover に上書きされない

ここが最もバグの出やすい箇所です：ユーザーがすでに選択済みの項目に hover すると、hover のスタイルが active のスタイルを上書きし、選択状態が通常の hover 状態に「戻って」しまい、視覚的には選択が解除されたように見えます。

**原則：Active 状態はいかなるときも識別可能でなければなりません——hover されているときも含めて。**

実現方法：

**方式一：Active は hover が関与しない次元を使う**

hover が背景のみを変えるなら、active はフォントウェイト + 文字色で区別します。hover 背景が重なっても、フォントウェイトと色は変わらないので、ユーザーは依然として「これは選択されている」と識別できます：

```
// ✅ hover は背景のみを扱い、active はフォントウェイトと色に依存する
hover:bg-muted                          // hover：薄いグレー背景
data-active:font-medium data-active:text-foreground  // active：フォントウェイト+色（hover は上書きしない）
```

**方式二：Active + Hover の複合スタイル**

active も背景色を使う場合は、「active かつ hover」の複合状態を明示的に定義し、hover が active の背景を低い階層に引き戻さないよう保証する必要があります：

```tsx
// ✅ active+hover の複合状態を明示的に処理する
cn(
  "hover:bg-muted/50",                              // 通常の hover
  "data-active:bg-muted data-active:text-foreground", // active
  "data-active:hover:bg-muted"                       // active+hover：active 背景を保持し、降格させない
)
```

```tsx
// ❌ 反例：hover が active を上書きする
cn(
  "hover:bg-muted/50",           // hover 背景が active より淡い
  "data-active:bg-muted",        // active 背景
  // 複合状態を処理していない → active 項目に hover すると背景が muted から muted/50 へ戻ってしまう
)
```

**方式三：CSS セレクタの優先度**

`:not()` を使って hover を非 active の要素にのみ作用させます：

```
// ✅ hover は active 項目に作用しない
[data-active]:bg-muted [data-active]:text-foreground
not-data-active:hover:bg-muted/50
```

**チェック方法：** hover + active 状態を持つコンポーネントを書き終えたら、必ず手動で検証してください——まず 1 項目をクリックして選択し、次にマウスをその項目の上に移動してから離し、視覚が「ちらつく」または「降格する」ことがないことを確認します。

### 5.4 Pressed 状態

物理的なフィードバック感——ボタンを押すと微小な変位があります：

```
active:not-aria-[haspopup]:translate-y-px
```

この 1px の下方移動は shadcn button 上ですでにグローバルに設定済みです。ポップアップメニューをトリガーするボタンには追加しません（ポップアップと同時に離すため、変位がちらつきます）。

### 5.5 Focus 状態

Focus はキーボードナビゲーションのためのものです。すべてのインタラクティブ要素は統一して以下を使用します：

```
focus-visible:border-ring focus-visible:ring-3 focus-visible:ring-ring/50
```

- `focus-visible`（`focus` ではない）を使い、マウスクリック時に focus ring が出るのを避けます。
- ring の色は `ring` token（ミドルグレー）を使い、コンポーネントの色には従いません——グローバルな一貫性を保ちます。

### 5.6 Disabled 状態

```
disabled:pointer-events-none disabled:opacity-50
```

シンプルで統一的。各コンポーネントごとに disabled スタイルをカスタマイズする必要はありません。

### 5.7 Error / Invalid 状態

```
aria-invalid:border-destructive aria-invalid:ring-destructive/20
```

- `aria-invalid` 属性でトリガーし、フォームバリデーションライブラリと自然に連携します。
- 枠線と ring のみを変え、背景は変えません。エラー情報はインラインの文字で表示し、toast や alert banner は使いません。

---

## 6. アイコン規約

### 6.1 アイコンライブラリ

統一して **Lucide React**（`lucide-react`）を使用します。

他のアイコンライブラリ（Heroicons、Phosphor など）の混用を禁止し、また自作の SVG アイコンも禁止します（Lucide に本当に適切なものがない場合を除く）。

### 6.2 アイコンサイズ

アイコンサイズはコンポーネントサイズと紐づきます：

| コンポーネントサイズ | アイコンサイズ | 例 |
|----------|---------|------|
| xs（h-6） | `size-3` (12px) | コンパクトなボタン、badge 内アイコン |
| sm（h-7） | `size-3.5` (14px) | 小ボタン、コンパクトなリスト |
| default（h-8） | `size-4` (16px) | 標準ボタン、メニュー項目、テーブル操作 |
| lg（h-9） | `size-4` (16px) | 大ボタン（アイコンをより大きくする必要はない） |

**ルール：**
- 独立した装飾的アイコン（空状態のイラストなど）は最大 `size-8` (32px)。
- すべてのアイコンはデフォルトで親要素の文字色を継承します。弱める必要があるときは `text-muted-foreground` を使います。
- アイコンと文字の間隔：`gap-1`（xs）/ `gap-1.5`（sm/default）/ `gap-2`（ゆったりした配置）。

### 6.3 アイコンカラー

- **ナビゲーション/操作アイコン：** `text-muted-foreground`、hover 時は文字に従って `text-foreground` へ変化
- **状態アイコン：** 対応するセマンティックカラーを使用（`text-success`、`text-destructive` など）
- **Active 状態アイコン：** `text-foreground`

---

## 7. 角丸規約

`--radius: 0.625rem`（10px）を基礎とする動的な scale：

| Token | 値 | 用途 |
|-------|-----|------|
| `rounded-sm` | 6px | Checkbox、小タグ |
| `rounded-md` | 8px | 入力欄、小ボタン、dropdown item |
| `rounded-lg` | 10px | 標準ボタン、カード、dialog |
| `rounded-xl` | 14px | 大カード、sheet |
| `rounded-full` | 999px | アバター、pill badge |

**禁止** `rounded-[6px]` のようなピクセル値のハードコード（shadcn コンポーネント内部で `rounded-[min(var(--radius-md),12px)]` のようなレスポンシブ計算が必要な場合を除く）。

---

## 8. アニメーション規約

### 8.1 原則

- **高速、抑制的。** アニメーションはユーザーが変化を理解するのを助けるためのものであり、技術を誇示するためのものではありません。
- **フェードイン・フェードアウト優先。** 要素の出現/消失には、スライドよりも opacity のトランジションを優先します。
- **バウンスなし。** spring / bounce のイージングは使いません。イージングカーブは統一して `ease-out` を使います。

### 8.2 時間

| シーン | 時間 | 例 |
|------|------|------|
| 色/透明度の変化 | 150ms | hover 背景の変化、文字色の変化 |
| 展開/収納 | 200ms | accordion、collapsible |
| フローティング層の出入り | 150-200ms | dialog、dropdown、popover |
| ページ切り替え | アニメーションなし | ルート遷移にトランジションアニメーションなし |

### 8.3 使用する transition

| Tailwind Class | 用途 |
|----------------|------|
| `transition-colors` | 純粋な色変化（hover、active）— 第一選択 |
| `transition-all` | 複数プロパティの同時変化 |
| `transition-opacity` | 要素のフェードイン・フェードアウト |
| `transition-transform` | 変位アニメーション（pressed 効果） |

---

## 9. コンポーネント使用規約

### 9.1 shadcn 優先

すべての UI コンポーネントは、インストール済みの shadcn コンポーネント（55 個利用可能）を優先して使います。新しい UI 要求があるとき：

1. まず shadcn に対応コンポーネントがあるか確認 → `npx shadcn add <component>`
2. バリアントが必要 → CVA で既存コンポーネントを拡張
3. 本当に無い → 自作コンポーネント、ただし本規約の token / インタラクション状態に従わなければならない

### 9.2 ボタンの階層

最強調から最弱まで：

| バリアント | 視覚的な重量 | 使用シーン |
|------|---------|----------|
| `default`（primary） | ██████ | ページの主操作（1 画面につき最多 1 個） |
| `outline` | ████░░ | 次要操作 |
| `secondary` | ███░░░ | 補助操作、ツールバー |
| `ghost` | █░░░░░ | アイコンボタン、インライン操作、コンパクトなツールバー |
| `destructive` | ████░░ | 削除、危険な操作（赤系） |
| `link` | █░░░░░ | インラインの文字リンク |

**ルール：** 1 つのビュー内の primary ボタンは最多 1 個。その他はすべてより弱いバリアントを使います。同等に重要な操作が複数ある場合は、すべて `outline` または `secondary` を使います。

### 9.3 Dropdown / Popover

- コンテンツ幅は `w-auto` を使い、`w-52`、`w-56` のような固定幅を**禁止**します（文字の折り返しを引き起こします）。
- メニュー項目は統一して `text-sm`、アイコンは `size-4`。
- 選択項目は checkmark アイコンまたは左側のインジケーターバーでマークし、背景色は変えません。
- 危険な操作項目は `text-destructive` を使い、最下部に配置し、上方に区切り線を入れて隔てます。

### 9.4 フォーム入力

- 入力欄は統一して `border-input` の枠線を使い、focus 時は `border-ring` + ring。
- Label は `text-sm font-medium` を使います。
- 説明/ヘルプ文字は `text-xs text-muted-foreground` を使います。
- エラー情報は `text-xs text-destructive` を使い、入力欄の真下に配置します。

---

## 10. アンチパターン一覧

以下のやり方はコード中に出現することを**禁止**します：

| 禁止 | 理由 | 代替 |
|------|------|------|
| 色のハードコード `text-red-500`、`bg-gray-100` | テーマの一貫性を壊す | token を使う：`text-destructive`、`bg-muted` |
| 任意のピクセル `text-[11px]`、`w-[137px]` | デザインシステムから逸脱する | Tailwind 内蔵の scale を使う |
| `font-bold` / `font-semibold` | 重すぎ、軽さを壊す | `font-medium` + `text-foreground` |
| `text-lg` / `text-xl` / `text-2xl` | 情報密度型ツールに大きな文字は不要 | `text-base` がすでに最大 |
| `shadow-sm` / `shadow-md` / `shadow-lg` | スキューモーフィズム的で、フラットデザインと衝突する | `border` で階層を分離する |
| hover 時の `scale-105` | 唐突で、抑制的なスタイルと衝突する | `hover:bg-muted` |
| 多色 gradient 背景 | 装飾的で、注意を散らす | 単色 token |
| Skeleton loading | 簡潔なスタイルに合わない | Spinner（`Loader2Icon animate-spin`）またはインラインの loading 文字 |
| Toast で操作確認 | 一瞬で消え、ユーザーが見逃しやすい | インラインの状態文字、または Sonner はエラー/重要な通知にのみ使う |
| 固定幅の dropdown `w-52` | 文字の折り返しが制御できない | `w-auto` |
| 純黒背景 `#000` / `oklch(0 0 0)` | LCD 上で目に刺さる | Dark モードでは濃いグレー `background` token を使う |

---

## 11. チェックリスト

いかなる UI 変更を提出する前にも、一通り確認してください：

- [ ] すべての色は token を使っているか？ハードコードはないか？
- [ ] フォントサイズは `text-xs` / `text-sm` / `text-base` の範囲内だけか？
- [ ] フォントウェイトは `font-normal` と `font-medium` だけを使っているか？
- [ ] Hover 状態は active 状態より淡いか？
- [ ] Active 項目が hover されたとき、active スタイルは依然として識別可能か（hover に上書きされないか）？
- [ ] アイコンサイズはコンポーネントサイズと一致しているか？
- [ ] 間隔は Tailwind 内蔵の scale を使っているか（任意値なし）？
- [ ] Dark モードで正常か？
- [ ] 不要な区切り線はないか（間隔で代替できないか）？
- [ ] Dropdown / Popover は `w-auto` か？
- [ ] 1 つのビュー内の primary ボタンは 1 個を超えていないか？

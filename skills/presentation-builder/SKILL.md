---
name: presentation-builder
description: "Markdownファイルとdesign-guideline.yamlを使って顧客向けHTML説明資料を生成するスキル。ユーザーが「プレゼン作って」「スライド作って」「資料作って」「HTMLスライド生成して」「説明資料を作りたい」「説明ページを作って」「Webページ形式で資料を」「Markdownからプレゼンを作って」「design-guideline.yamlを使って」「このMarkdownをスライドにして」など、Markdown→HTML変換・プレゼン生成・説明ページ生成を依頼したときは必ず使用すること。design-guideline.yamlが存在する状況での資料生成依頼でも積極的にトリガーする。--mode slide（デフォルト）または --mode page で出力形式を切り替え可能。"
---

# Presentation Builder スキル

## 概要

指定された Markdown ファイルと `design-guideline.yaml` を読み込み、ブランドガイドライン準拠の HTML 説明資料を生成する。

- **slide モード**: Reveal.js CDN ベースのスライド形式（ページ送り）
- **page モード**: スクロール型 Web ページ形式

**設計思想**: レイアウトパターンはスキルにハードコードしない。`design-guideline.yaml` の `layout_patterns[]` に定義されたパターンを Claude が `description` を読んで判断・選択することで、ブランドごとに自由に追加・変更できる拡張可能な設計。

---

## 実行手順

### Step 1: 引数の解析

ユーザーの指示から以下を読み取る：

| 引数 | 説明 | デフォルト |
|---|---|---|
| Markdown ファイルパス | 変換元コンテンツ | 必須（不明なら確認） |
| `--mode slide\|page` | 出力形式 | `slide` |
| `--design ファイルパス` | デザインガイドライン YAML | 同ディレクトリの `design-guideline.yaml`、なければ確認。存在しない場合はユーザーに用意を促すか、最小スタイル（白背景・黒文字・システムフォント）でフォールバックするか確認する |
| `--output ファイルパス` | 出力先 HTML | 同ディレクトリに `presentation-slide.html` または `presentation-page.html` |

### Step 2: ファイルの読み込み

以下を**並列で**読み込む：

1. `design-guideline.yaml` — ブランドトークン・レイアウトパターン・カラールールの把握
2. Markdown ソースファイル — コンテンツの全体構造の把握

### Step 3: デザインガイドラインの解析

`design-guideline.yaml` から以下を抽出する：

```
colors.*            → CSS Custom Properties (--color-primary 等) に変換
typography.*        → CSS変数 (--font-heading, --size-h1 等) に変換
logo.*              → ロゴ配置（path指定あり → <img>、なし → テキストフォールバック）
layout_patterns[]   → 利用可能なレイアウトパターン一覧（name + description）
color_rules[]       → コンテキスト別カラー適用ルール（title_slide / content_slide 等）
output_mode         → slide / page（--mode 引数で上書き可）
```

**color_rules の適用（重要）**:
- `context: "title_slide"` → タイトル・クロージングスライドの背景・テキスト色
- `context: "content_slide"` → 通常スライドの背景・テキスト・アクセント色
- `context: "divider_slide"` → セクション区切りの背景色
- `context: "chart_diagram"` → チャート・図の系列色（Extended Paletteのみ）
- `context: "table"` → テーブルのヘッダー・ストライプ色

### Step 4: レイアウトパターンの選択（Claude の判断）

**前提**: 利用可能なパターンは `design-guideline.yaml` の `layout_patterns[]` に定義されたものだけ。スキル側にパターン名の前提を持たない。

各スライド / セクションのコンテンツを以下の手順で分類・選択する：

1. **YAML から利用可能パターンを列挙する**
   `layout_patterns[]` を読み、各パターンの `name` と `description` を把握する。

2. **コンテンツの性質を分析する**
   各スライドのコンテンツ（タイトル・箇条書き・表・図・数値強調・セクション区切りなど）の特徴を整理する。

3. **description との照合で最適パターンを選ぶ**
   コンテンツの性質と各パターンの `description` を照合し、最も合致するパターンを選択する。
   合致するパターンが複数ある場合は description の説明がより具体的に一致するものを優先する。

4. **対応パターンがない場合**
   YAML に適切なパターンが存在しない場合は、最も汎用的な description を持つパターンを選ぶ（存在しないパターン名を使用してはならない）。

`layout_hints` に明示指定がある場合はそれを優先する。

### Step 5: HTML の生成

#### slide モードの場合

Reveal.js CDN ベースで生成する。

**必須 CDN**:
```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/reveal.js@5/dist/reveal.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/reveal.js@5/dist/theme/white.css">
<script src="https://cdn.jsdelivr.net/npm/reveal.js@5/dist/reveal.js"></script>
```

**日本語フォント**（Google Fonts CDN）:
```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;600;700&display=swap" rel="stylesheet">
```
> YAML の `typography.heading_font` / `typography.body_font` に値が指定されている場合はそちらを優先する。`Noto Sans JP` は YAML にフォント指定がない場合のフォールバックとして使う。

**CSS変数の注入**（`:root` ブロック）:
```css
:root {
  --color-primary: /* colors.primary */;
  --color-black:   /* colors.black */;
  --color-white:   /* colors.white */;
  /* ... typography 変数も同様に注入 */
}
```

**ブランドアクセント要素**（YAML `brand_accents[]` に定義がある場合のみ適用）:
- `brand_accents[]` を読み取り、各要素の `type` と `description` に従って実装する
- 実装方法は後述「ブランドアクセント要素」セクション参照
- `brand_accents` が YAML に定義されていない場合は一切実装しない
- 後方互換: `accent_line` フィールドが定義されている場合は `type: "line"` として扱う

**スライド構造**:
```html
<div class="reveal">
  <div class="slides">
    <section class="layout-title">   <!-- layout_patterns.css_class を適用 -->
      ...
    </section>
    <section class="layout-content">
      ...
    </section>
  </div>
</div>
```

**ロゴ配置**:
- `logo.path` が空 or 未指定 → `<span class="logo-text">ブランド名</span>`（テキストフォールバック）
- `logo.path` に値がある → `<img src="[変換済みパス]" alt="logo">`
- `logo.path_dark_bg` が定義されている場合、背景が暗いレイアウト（例: layout-divider-dark, layout-title-dark, layout-closer-logo 等）では `logo.path_dark_bg`（白ロゴ等）を使い、明るい背景のレイアウトでは `logo.path` を使う
- `logo.position: "bottom-left"` なら各スライドの左下に絶対配置

**ロゴ画像の埋め込み（必須）**:
- `logo.path` / `logo.path_dark_bg` は **YAMLファイルと同じディレクトリにある相対パス**
- `<img src="ファイルパス">` は使わず、**ファイルを読み込んでHTMLに直接埋め込む**こと（パス解決の問題を回避するため）
  - **SVG**: ファイル内容を `<svg>...</svg>` としてインライン埋め込み。`role="img" aria-label="[logo.name]"` を付与
  - **PNG/JPG等のラスタ画像**: ファイルをbase64エンコードして `<img src="data:image/png;base64,...">` として埋め込み
- 通常背景用（`logo.path`）とダーク背景用（`logo.path_dark_bg`）の2種類を必要に応じて埋め込む
- ファイルが読み込めない場合のみ `<span class="logo-text">[logo.name]</span>` にフォールバック

**PDF 変換対応**:
```html
<style>
@media print {
  .slide { page-break-after: always; }
}
</style>
```
Reveal.js の `?print-pdf` URL パラメータでも PDF 変換可能。

#### page モードの場合

**slide モードとは完全に別物として設計する。スライドを縦に並べただけの実装は禁止。**

page モードは「ブランドガイドラインを遵守した、読みやすい Web ページ」として生成する。
コンテンツが自然に流れ、スクロールして読めることが第一目標。

#### ❌ page モードで使用禁止のもの

以下は slide モード専用。page モードでは**一切使わない**：

- `Reveal.js`（CDN 読み込みも含め禁止）
- `height: 720px` などの固定高さ指定（セクションはコンテンツ量に応じて伸縮させる）
- `layout_patterns` の CSS クラス（`layout-title`, `layout-content` 等）
- `.reveal`, `.slides` セレクタ
- `overflow: hidden` によるコンテンツのクリップ
- スライド用フッター（`.slide-footer`）—ページ全体にヘッダーを1つ置く

#### ✅ page モードの HTML スケルトン

```html
<!DOCTYPE html>
<html lang="[YAMLのlanguageフィールド、未定義なら'ja']">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[タイトル]</title>
  <!-- Google Fonts など必要なフォントのみ -->
  <style>
    /* CSS カスタムプロパティ（ブランドトークン） */
    :root {
      --color-primary: [colors.primary];
      --color-bg: [colors.background];
      --color-text: [colors.text];
      --font-heading: [typography.heading_font], sans-serif;
      --font-body: [typography.body_font], sans-serif;
    }
    /* リセット + ベーススタイル */
    *, *::before, *::after { box-sizing: border-box; }
    body { margin: 0; font-family: var(--font-body); color: var(--color-text); background: var(--color-bg); scroll-behavior: smooth; }
    /* ヘッダー（ロゴ + ナビ） */
    /* hero セクション */
    /* 各コンテンツセクション */
    /* フッター */
  </style>
</head>
<body>
  <header><!-- ロゴ + ナビゲーション --></header>
  <main>
    <section class="hero"><!-- タイトル・リード --></section>
    <section><!-- コンテンツセクション --></section>
    <!-- ... -->
  </main>
  <footer><!-- 著作権等 --></footer>
</body>
</html>
```

---

**設計の基本方針**:

| slide モード | page モード |
|---|---|
| 固定サイズのスライド（720px高さ） | コンテンツ量に応じて高さが伸縮するセクション |
| Reveal.js によるページ送り | 自然なスクロール |
| プレゼン発表用レイアウト | Web として読みやすいレイアウト |
| スライドの視覚的再現 | Web ページとしての UX を優先 |

---

**構造設計**:

Markdown の見出し階層を Web ページのセクション構造にマッピングする。
ただしタグの機械的な変換ではなく、コンテンツの意味と文脈に応じて Web らしい構造を選ぶ：

| コンテンツの意味 | Web ページでの実装 |
|---|---|
| 最初の `#`（タイトル） | `<section class="hero">` — 大きなビジュアル見出し + リード文 |
| `##` セクション見出し | `<section>` または `<article>` でまとめ、見出しと内容をグループ化 |
| 箇条書きが多い | カードグリッド・リスト・アイコン付きリストなど読みやすい形式に変換 |
| 表 | `<table>` にホバー・ストライプなどの Web スタイルを適用 |
| 数値・KPI | 大きな数字を強調したメトリクスブロック |
| コードブロック | シンタックスハイライト + コピーボタン |
| 最終セクション | CTA（行動喚起）エリアまたはフッターとして構成 |

---

**Web らしい UX・インタラクション（適宜採用）**:

`page_mode_rules.features[]` に定義があるものは必ず実装し、
それ以外でも読みやすさを向上させる標準的な Web インタラクションは積極的に採用する：

- **スムーズスクロール**: `scroll-behavior: smooth`
- **セクション間の余白**: 各セクションに十分な `padding`（スライドの詰まった感をなくす）
- **ホバーエフェクト**: カード・リンク・ボタンに `transition` でフィードバック
- **レスポンシブ対応**: カラムレイアウトは `grid` または `flex` + `flex-wrap` で自動折り返し
- **ナビゲーション**: コンテンツ量が多い場合はスティッキーヘッダーまたはサイドバー TOC

`page_mode_rules.features` に定義された要素（`toc`, `progress-bar`, `tabs`, `copy-button`）は必ず実装する。

---

**ブランドガイドラインの適用**:

YAML の色・フォント・ルールはすべて適用する。ただし適用方法は Web ページに合わせて読み替える：

- `colors.primary` → ヒーローセクション背景・アクセントカラー・ボタン色
- `colors.background` → ページ全体の背景
- `typography.*` → CSS 変数として注入し、見出し・本文フォントに適用
- `brand_accents[]` → 各アクセント要素の `description` を読んで Web 向けに適用（ボーダー・背景パターン・擬似要素等）
- `logo.*` → ページヘッダー左上に固定配置（スライドの各ページではなくページ全体に1つ）。SVGはインライン埋め込みにすること（上記「ロゴSVGのインライン埋め込み」参照）

CSS は `<style>` タグに埋め込み、CSS変数でブランドトークンを適用する。

### Step 6: ファイルの出力

生成した HTML を書き出す前に、以下のセルフチェックを行う：

**出力前チェックリスト**:
- [ ] すべてのスライド/セクションでコントラスト比が基準を満たしているか（WCAG AA: 本文 4.5:1、大見出し 3:1）
- [ ] ダーク背景のコンポーネント内に `var(--color-text)` が残っていないか
- [ ] ロゴSVGの `width` が `auto` になっていないか（明示的な px 値を使用）
- [ ] slide モード: `position: relative` が `<section>` に記述されていないか
- [ ] slide モード: `height` に `min-height` を誤用していないか
- [ ] page モード: `Reveal.js` の CDN や `layout-*` クラスが混入していないか
- [ ] ロゴパスが Vault ルートからの深さに応じて正しく変換されているか

問題が見つかれば修正してから書き出す。

生成した HTML を指定パスに書き出す。

出力後、以下を報告する：
```
presentation-slide.html を生成しました。
使用レイアウト: title(1), content(3), divider(2), table(1), diagram(1)
ブラウザで開いて確認してください。PDF変換は ?print-pdf を URL に付加するか Cmd+P で行えます。
```

---

## 技術的実装ルール（ブランド共通）

> **原則**: SKILL.md は HOW（技法・落とし穴）を定義する。WHAT/WHERE/WHEN はすべて YAML が決める。
> YAMLに定義がない機能・スタイルは実装しない。

> **詳細リファレンス**: コントラスト比の計算式・SVG幅の計算式・タイトルスライドの2点配置実装例・ダイアグラムwrapper・タイポグラフィ一貫性の詳細は `references/implementation-rules.md` を参照すること。

---

### セクション共通ベースルール（必須・最初に記述）

すべての `<section>` に共通するベースルールを **最初に** 記述する。
このルールがないと `background: linear-gradient(...)` がコンテンツ高さだけに縮小し、スライドが小さく見える問題が発生する。

```css
/* height は YAML slide_size.height_px から取得 — min-height は NG */
.reveal .slides > section[class] {
  box-sizing: border-box;
  display: flex !important;
  flex-direction: column !important;
  align-items: flex-start !important;
  text-align: left;
  height: 720px !important;   /* = slide_size.height_px — 必ず !important で明示する */
  overflow: hidden !important; /* 720px を超えるコンテンツはクリップ */
  width: 100%;
}
```

**理由**:
- **`height` であり `min-height` ではない（重要）**: `min-height` を使うと、flex 子要素の合計高さがそれを超えた場合に section が 720px を超えて伸びる。コンテンツ量の多いスライドと少ないスライドで表示サイズが不揃いになる根本原因。`height: 720px !important` で全スライドを同一高さに固定する。
- `overflow: hidden !important` で 720px を超えるコンテンツをクリップし、Reveal.js のスケール計算を正確にする。
- `width: 100%` は Reveal.js の `position: absolute` 配置でセクション幅を明示的に保証する。
- `display: flex !important` はベースルールで一括設定し、各レイアウトの `justify-content` だけを個別に定義する。
- **`position` プロパティは絶対に記述しない（禁止）**: Reveal.js は各 `<section>` を `position: absolute` で絶対配置し、CSS transform でスライドを中央揃え・アニメーションする。このベースルールのセレクタ詳細度は 0-3-1 で Reveal.js の `.reveal .slides section`（0-2-1）より高いため、`position: relative` を書くと Reveal.js の `position: absolute` が上書きされ、スライドが通常フローに流れ込んで画面外にはみ出す。`position` は一切記述しないこと。

**YAML フィールド**: `slide_size.height_px`（未定義時デフォルト: `720`）

---

### フッター・ロゴの統一実装（必須）

フッター（ロゴ・著作権・日付）は**全スライド共通の単一 CSS クラス `.slide-footer`** で実装する。
レイアウトごとに個別のフッター CSS を書いてはならない。これがズレの最大原因。

#### CSS は1回だけ定義する（グローバル）

```css
/* ✅ <style> ブロックの先頭付近に1回だけ書く */
.slide-footer {
  position: absolute;
  bottom: [footer.logo_bottom]px;   /* YAML footer.logo_bottom の値 */
  left:   [footer.logo_left]px;     /* YAML footer.logo_left の値 */
  display: flex;
  align-items: center;
  gap: 8px;
  z-index: 10;
}
.slide-footer img,
.slide-footer .logo-text {
  height: [footer.logo_height]px;   /* YAML footer.logo_height の値 */
  width: auto;  /* ← SVGの場合は後述の計算値で上書きする */
}
```

#### HTML は全 `<section>` で同一スニペットを使い回す

```html
<!-- 全スライドで同じ HTML を末尾に置く -->
<section class="layout-xxx">
  <!-- コンテンツ -->
  <div class="slide-footer">
    <img src="[logo.path]" alt="logo" style="height:[H]px; width:[W]px;">
  </div>
</section>
```

- フッターHTMLをコピペするだけで全スライド同一位置になる
- ロゴ画像パス（通常 vs 暗背景用）だけをレイアウトごとに切り替える

#### `position: absolute` が機能する理由

Reveal.js は各 `<section>` に `position: absolute` を付与している。
そのため `<section>` 内の `position: absolute` 子要素は `<section>` を基準に配置される。
`<section>` 自体に `position: relative` を追記する必要はない（むしろ禁止）。

#### フッターを flex の流れに含めてはならない

```css
/* ❌ NG: flex 子要素として末尾に配置 → justify-content の値によって位置がズレる */
section { display: flex; flex-direction: column; justify-content: space-between; }
/* → コンテンツが少ないスライドと多いスライドでフッター位置が変わる */

/* ✅ OK: position: absolute で flex フローから切り離す */
.slide-footer { position: absolute; bottom: Xpx; left: Ypx; }
```

---

### ロゴSVGの絶対配置

`position: absolute` で SVG ロゴを配置する場合、`width: auto` は**禁止**。

```
理由: ブラウザが width:auto の SVG にデフォルト 300px を適用し、
      preserveAspectRatio="xMidYMid meet" によってロゴ内容が中央寄りになるバグがある。

計算式: width = height × (viewBox_width / viewBox_height)
例: height=20px, viewBox="0 0 360 87" → width = 20 × (360/87) ≈ 83px
```

YAML の `footer.logo_height` と SVG の viewBox から必ず width を明示する。

---

### Reveal.js CSS 詳細度ルール

Reveal.js white テーマのベースセレクタは詳細度 **0-3-1**（`!important` 付き）:

```css
.reveal .slides > section[class] {
  align-items: flex-start !important;  /* 詳細度 0-3-1 */
}
```

レイアウト固有で `align-items` 等を上書きするには**同じ 0-3-1** のセレクタを後に書く:

```css
/* ❌ 詳細度 0-2-1 — 効かない */
.reveal section.layout-closer-logo { align-items: center !important; }

/* ✅ 詳細度 0-3-1 — 後に書くことで上書きできる */
.reveal .slides > section.layout-closer-logo { align-items: center !important; }
```

---

### ブランドアクセント要素（YAML に `brand_accents[]` 定義がある場合のみ実装）

YAML の `brand_accents[]` を読み取り、各要素の `type` と `description` に従って実装する。
**`brand_accents` が未定義、または空配列の場合は一切実装しない。**

後方互換: `accent_line` フィールドが定義されている場合は `type: "line"` として扱う。

#### type ごとの実装指針

| type | 概要 | 推奨CSS実装 |
|---|---|---|
| `line` | 縦・横の細いライン | `background: linear-gradient(...)` をセクションに適用 |
| `corner` | コーナーマーク・ブラケット | `::before` / `::after` 疑似要素 + `border` |
| `dot` | ドットパターン・点群 | `background-image: radial-gradient(...)` または `background: repeating-*` |
| `shape` | 幾何学シェイプ（三角・円等） | `clip-path` または SVG インライン埋め込み |
| `gradient` | グラデーション帯・オーバーレイ | `background: linear-gradient(...)` / `radial-gradient(...)` |
| `pattern` | 繰り返し背景パターン | `background-image` + `background-size` |
| `other` | 上記に当てはまらない | `description` を読んで最適な CSS を判断 |

#### `type: "line"` の実装（slide モード）

`background: linear-gradient(...)` をセクション自体に適用する（疑似要素は Reveal.js と競合するため禁止）：

```css
/* position: left,  width: 13px */
background: linear-gradient(to right, [accent-color] 13px, [bg-color] 13px);

/* position: top,   height: 4px */
background: linear-gradient(to bottom, [accent-color] 4px, [bg-color] 4px);

/* position: right, width: 6px */
background: linear-gradient(to left, [accent-color] 6px, [bg-color] 6px);

/* position: bottom, height: 3px */
background: linear-gradient(to top, [accent-color] 3px, [bg-color] 3px);
```

`[bg-color]` はそのレイアウトの背景色（`color_rules` から取得）で置き換える。

#### `apply_to` フィールドによる適用範囲の制御

```yaml
brand_accents:
  - type: "line"
    apply_to: "all"              # 全スライド（デフォルト）
    # apply_to: "content_slide" # 特定レイアウトのみ
    # apply_to: ["content_slide", "divider_slide"]
```

`apply_to` が未定義の場合はすべてのスライドに適用する。

---

### ダイアグラムwrapper

diagram を含むスライドの wrapper には以下を設定する:

```css
.diagram-wrapper {
  flex: 1;
  min-height: 360px;  /* コンテンツ量に応じて調整 */
  /* max-height は設定しない — ダイアグラムが縮小されるため */
}
```

---

### タイポグラフィ一貫性

`typography.heading_h2` の値はすべてのレイアウトの h2 に統一して適用する。
レイアウトごとに異なるサイズを設定すると、スライド間で見出しサイズが揃わなくなる。

---

### ダーク背景コンポーネントのテキスト色（YAML `color_rules` 準拠）

YAML の `color_rules` でダーク背景と定義されたコンポーネントでは、
すべてのテキストを白系（`rgba(255,255,255,0.8)` 以上）で保証する。

#### ❌ `var(--color-text)` をダーク背景内で使用禁止

`var(--color-text)` は通常背景（白/淡色）用のダーク文字色。
ダーク背景の要素内で `var(--color-text)` を使うと、どこかの汎用ルールが高い詳細度で
ダーク背景内のテキストに黒色を適用してしまうため**禁止**。

```css
/* ❌ ダーク背景内で var(--color-text) を使ってはならない */
.col-panel.dark ul li { color: var(--color-text); }  /* → 黒文字になる */

/* ✅ 白系の値をハードコードする */
.col-panel.dark ul li { color: rgba(255,255,255,0.88); }
```

#### ダーク背景コンポーネントの CSS 実装パターン（必須）

ダーク背景を持つコンポーネント（カラム・カード・パネル等）は、
コンテナに `color: white` を設定し、**子要素すべてにカスケードで継承**させる。
その上で、汎用ルールに負けないよう Reveal.js のベースセレクタと同等以上の詳細度で上書きする：

> **注**: 以下の `.col-panel.dark` は説明用の例示クラス名。実際の CSS クラス名はコンテンツ構造に応じて自由に命名してよい（例: `.card-dark`, `.panel-inverted` 等）。

```css
/* ✅ ベースセレクタ（詳細度 0-3-1）と同等以上で書く */
.reveal .slides > section .col-panel.dark,
.reveal .slides > section .col-panel.dark h1,
.reveal .slides > section .col-panel.dark h2,
.reveal .slides > section .col-panel.dark h3,
.reveal .slides > section .col-panel.dark p,
.reveal .slides > section .col-panel.dark li,
.reveal .slides > section .col-panel.dark ul li,
.reveal .slides > section .col-panel.dark ol li,
.reveal .slides > section .col-panel.dark span {
  color: rgba(255,255,255,0.88) !important;
}
```

> **なぜ `!important` が必要か**: Reveal.js white テーマの汎用ルール（`.reveal .slides > section.layout-xxx .col ul li`）が詳細度 0-5-1 以上で書かれることがあり、それに勝つために `!important` が唯一の確実な手段。

#### layout_patterns はスライド全体への適用が前提

`layout_patterns` の background 指定はスライド全体（`<section>`）に適用するもの。
**カラムやカード等のサブコンポーネントに layout_patterns の背景色を流用してはならない。**

```
❌ NG: two-column-dark パターンを「右カラムだけ」に適用
✅ OK: two-column-dark パターンをスライド全体に適用し、
       左右それぞれのカラム色は color_rules から個別に指定する
```

サブコンポーネントに異なる背景色を付けたい場合は、layout_patterns ではなく
color_rules または直接 HEX 値で CSS に記述する。

---

### コントラスト検証と自動補正（必須）

HTML を生成する際、**すべての背景色とテキスト色の組み合わせ**に対してコントラスト比を確認し、
不足する場合は自動補正を行う。デザインガイドライン遵守よりも「読めること」を優先する。

#### コントラスト比の確認方法

WCAG AA 基準（本文 4.5:1 / 大見出し 3:1）で判定する。計算式の詳細は `references/implementation-rules.md` を参照。

**補正ルール（概要）**:
- YAML `color_rules` に明示された組み合わせ → そのまま使用
- 基準未満 + ダーク背景（輝度 < 0.18）→ `#FFFFFF` または `rgba(255,255,255,0.9)` に補正
- 基準未満 + ライト背景（輝度 ≥ 0.18）→ `colors.text`（ダーク系）に補正、それでも不足なら `#000000`

**チェック対象**: スライド背景 vs テキスト、カード背景 vs カード内テキスト、フッター背景 vs フッターテキスト、バッジ背景 vs バッジテキスト

補正を行った場合でもそのまま出力する（ユーザーへの警告は不要）。読めないスライドより、デザインから微妙に外れていても読めるスライドの方が常に正しい。

---

### タイトルスライドの2点配置レイアウト（YAML に座標指定がある場合）

YAML の `layout_patterns[].description` に「h1: y=〇cm」「メタ: y=〇cm」のような実測座標が記載されている場合、`justify-content: flex-end`（下寄せ）は**禁止**。座標どおりに2点配置で実装する。

実装パターン（flex + spacer）の詳細・換算式・サンプルコードは `references/implementation-rules.md` を参照。

**要点**: `padding-top` に Y座標(px)を設定し、h1 と meta の間に `<div style="flex:1; min-height:0;">` スペーサーを置く。`height: 720px !important` が前提（`min-height` では spacer が伸縮できず失敗する）。

---

### 著作権表記（slideモード）

| YAML フィールド | 動作 |
|---|---|
| `footer.copyright_text` が定義されている | 全スライドのフッター右下に表示 |
| `footer.copyright_exclude_layouts[]` に含まれるレイアウト | 著作権表記を非表示 |
| `footer.copyright_text` が未定義または空 | 著作権表記なし |

---

### YAML 標準フィールド一覧（未定義時のデフォルト）

スキルが読み取るべき YAML フィールドと、未定義時のデフォルト値:

| フィールド | デフォルト | 説明 |
|---|---|---|
| `viewer_background_color` | `"#2b2b2b"` | スライドビューア外の背景色 |
| `brand_accents` | `[]`（空=何も入れない） | ブランドアクセント要素の配列（line/corner/dot/shape等） |
| `footer.copyright_text` | `""` | 著作権テキスト（空なら非表示） |
| `footer.copyright_exclude_layouts` | `[]` | 著作権非表示の layout 名リスト |
| `page_mode_rules.force_line_breaks` | `false` | pageモードで h1/subtitle に `<br>` を入れるか |
| `page_mode_rules.features` | `[]` | pageモードで有効にするUI機能リスト |

---

### pageモードのUI実装（`page_mode_rules.features` から読み取る）

pageモードは slideモードとは別物として設計する（スライドの縦並べではない）。
`page_mode_rules.features[]` に定義された要素のみ実装する:

| features 値 | 実装内容 |
|---|---|
| `"toc"` | スティッキーサイドバーTOC（IntersectionObserver でアクティブ項目をハイライト） |
| `"progress-bar"` | 読書進捗バー（fixed top、スクロール量に連動） |
| `"tabs"` | 比較コンテンツのタブ切替（複数の選択肢を並べたセクションに適用） |
| `"copy-button"` | コードブロックのコピーボタン |

`features` が空または未定義の場合 → 最小構成（見出し・本文・テーブルのみ）で生成する。

`force_line_breaks: false`（デフォルト）の場合 → h1/subtitle に `<br>` を入れない。

---

## レイアウトパターンの追加方法

新しいレイアウトを追加したい場合、**スキルのコードは変更不要**。
`design-guideline.yaml` の `layout_patterns[]` に追記するだけ：

```yaml
layout_patterns:
  # 既存パターン...
  - name: "case-study"
    description: >
      顧客事例スライド。ロゴ・課題・成果を横並び3ブロックで構成。
      顧客の成功事例や実績を見せるときに使う。
    css_class: "layout-case-study"
```

Claude は次回から `description` を読んで「case-study」パターンを自動的に使用する。

---

## 依存関係

- **外部依存なし**（CDN のみ）
- Node.js / Python 不要
- Reveal.js 5.x（CDN）
- Google Fonts CDN（Noto Sans JP）

---

## 使用例

```
「notes/product-overview.md を使ってスライドを作って」
→ presentation-slide.html を生成（デフォルト: slideモード）

「同じ内容をページ形式でも作って --mode page」
→ presentation-page.html を生成

「designsystem_document.yaml を使ってスライドを作って」
→ 指定したブランドガイドラインYAML適用の HTML を生成
```

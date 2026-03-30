---
name: design-extractor
description: "PPTX・PDF・画像・スクリーンショット等の参考資料からデザイン情報を抽出し、design-guideline.yaml を生成するスキル。ユーザーが「デザインガイドラインを作って」「このPPTXからブランドカラーを抽出して」「参考資料を元にデザイン定義ファイルを作って」「design-guideline.yaml を生成して」「このスライドのブランド情報を取り出して」などと依頼したときは必ず使用すること。PPTX・PDF・画像ファイルが渡されてデザイン情報の抽出を求められた場合も積極的にトリガーする。"
---

# Design Extractor スキル

## 概要

参考資料（PPTX / PDF / 画像 / スクリーンショット）を解析し、ブランドのデザイン情報を抽出して `design-guideline.yaml` を生成する。

生成した YAML は `presentation-builder` スキルで直接利用できる。

**設計思想**:
- PPTX ファイルは `python-pptx` でプログラマティックに精密抽出
- PDF・画像は Claude の視覚解析で色・レイアウト・フォントを推定
- 不明な情報は `-` または `未確認` のままにし、確認後に補完

> **Pythonスクリプトの詳細**: PPTX抽出に使う具体的なコードは `references/pptx-extraction.md` を参照すること。

---

## 実行手順

### Step 1: 入力の確認

以下をユーザーに確認（不明な場合）：

| 確認項目 | デフォルト |
|---|---|
| 参考資料のパス（複数可） | 必須 |
| 出力 YAML のパス | 同ディレクトリの `design-guideline.yaml` |
| ブランド名 | ファイル名から推定 |

複数ファイルが渡された場合：
- 各ファイルを並列で読み込み・解析する
- 重複情報は後から渡されたファイルで上書き
- 矛盾する情報はユーザーに確認

### Step 2: テキスト説明・ルール文章の読み取り（すべてのファイル形式に適用）

**視覚的な構造を読む前に、まずガイドライン資料に書かれたテキスト説明を読む。**

ブランドガイドライン資料には、視覚的なデザイン情報だけでなく、
**「なぜその色を使うか」「何を禁止するか」「どう使い分けるか」**という文章・ルールが記載されている。
これらを YAML の `color_rules`, `typography.rules`, `color_rules[].rule` に反映することが重要。

#### PPTX の場合のテキスト抽出

```bash
# markitdown でテキスト全文を先に抽出し、ルール文章を読む
python -m markitdown guideline.pptx > guideline_text.md
```

#### 読み取るべきテキスト情報の種類

以下のキーワードを含むテキストを重点的に読み取り、YAML に反映する：

| 読み取り対象 | YAML への反映先 |
|---|---|
| 「〇〇専用」「〇〇にのみ使用」「〇〇は禁止」 | `color_rules[].rule` + `forbidden` |
| 「プライマリカラーは〇色」「メインカラーは〜」 | `colors.*` の分類・コメント |
| 「フォントウェイトは〜のみ」「イタリック禁止」 | `typography.rules[]` |
| 「ロゴを変形・回転・色変更しない」等のNG例 | `logos.*.rules[]` または `extraction_notes` |
| 「白背景では赤ロゴ、暗背景では白ロゴ」等の使い分け | `logos.placement.*` + `color_rules` |
| 「拡張パレットはチャートのみ」等の使途制限 | `color_rules[context="chart_diagram"]` |
| フォントサイズの用途説明（「タイトルは〇pt」等） | `typography.size_*` のコメント |

#### ⚠️ 視覚抽出だけに頼らない

PPTX/PDF からプログラム的に色を取得しても、
**「その色を何に使うか」「何に使ってはいけないか」** は視覚解析では分からない。
ガイドライン資料に書いてある使用ルール文章を読んで初めて正確な `color_rules` が作れる。

例：`#4A90D9`（chart_blue）は視覚抽出で取れるが、
「このカラーはチャート・図表専用であり、通常のコンテンツには使用しない」という
ルール文章を読んで初めて `chart_blue` として正しく分類し、禁止ルールを記載できる。

---

### Step 3: ファイル種別に応じた構造解析

#### PPTX ファイルの場合（最高精度）

`python-pptx` を使ってプログラマティックに解析する。具体的なスクリプトは `references/pptx-extraction.md` を参照。

**抽出する情報**:
- スライドサイズ（mm/px）
- スライドマスターの色・フォントサイズ
- フッター要素の座標（px換算）
- ブランドアクセント要素（細いライン・装飾シェイプ等）

**エラーハンドリング**:
- 壊れたスライドがあっても処理継続（`try-except` でスキップ）
- `shape.fill.fore_color` が取得できない場合は `None` 扱い

#### PDF / 画像 / スクリーンショットの場合（視覚解析）

Claude の視覚解析機能で以下を推定する：

1. **背景色**: スライド / ページの主な背景色（HEX）
2. **テキスト色**: 見出し・本文の色（HEX）
3. **アクセント色**: ボタン・区切り線・強調要素の色（HEX）
4. **フォント**: フォントファミリーの推定（判別困難な場合は「未確認」）
5. **ロゴ**: 位置・色・サイズの推定
6. **レイアウトパターン**: 視認できるレイアウト構造からパターンを推定・命名

### Step 4: 抽出情報の整理

以下の優先順位で色情報を分類する：

```
1. Primary（最も多用される背景色・アクセント色） → colors.primary
2. テキスト色（最も多いテキスト色）             → colors.text
3. 背景色（白/淡色）                            → colors.background
4. 補助色（チャート・図表専用）                  → colors.chart_*
```

**Extended Palette の判別**:
- スライドのメインコンテンツに使われていない色 → `chart_*` として分類
- 「チャート専用」と明記されている場合は `color_rules` に明示

### Step 5: layout_patterns の推定

**前提**: パターン名はスキル側に持たない。参考資料に実際に存在するレイアウトだけを帰納的に命名・記述する。

以下の手順で各スライド / ページのレイアウトを観察し、パターンを導出する：

1. **全スライドを走査して構造を記録する**
   - 各スライドの背景色・テキスト配置・カラム数・主要コンテンツ種別（テキスト/表/図/アイコン群など）を観察する
   - PPTX の場合は `prs.slide_layouts` および実スライドの両方を確認する

2. **同一構造のスライドをグループ化し、資料に存在する構造から直接命名する**
   - パターン名は資料を観察して初めて決まる（例: 実際に3カラムカード配置があれば `three-column-card`）
   - 想定パターン名を先に決めて当てはめてはならない

3. **背景色が異なるだけの同構造は別バリアントとして定義する**（下記チェック参照）

#### ⚠️ バリアント完全性チェック（必須）

**同名レイアウトの全バリアントを個別エントリとして登録する。**
背景色が異なるだけ（red/light/dark など）でも別パターンとして定義すること。
「ライト版しか見ていなかったためダーク版・カラー版を見落とす」ミスが最も多い。

PPTX の場合は `prs.slide_layouts` を全件走査してバリアントを識別する（スクリプトは `references/pptx-extraction.md` の「レイアウトバリアントの全件走査」を参照）。

**背景色がテーマカラー（schemeClr）の場合、必ずテーマカラー値に解決する。**
`accent6` 等の名前だけ記録してはいけない。XMLからテーマの対応HEXを取得して記載する。

#### ⚠️ テキスト縦位置の実測（PPTX必須）

layout_patterns の `description` には、タイトル・本文・メタ情報のY座標（cm）を実測値として記載する。
「上部」「下部」「中央」などの曖昧な表現は禁止。プレゼンテーションビルダーが CSS で再現できる精度で記録する。

スクリプトは `references/pptx-extraction.md` の「テキスト縦位置の実測」を参照。

### Step 6: color_rules の生成

抽出した色情報とレイアウトパターンをもとに `color_rules` セクションを生成する：

```yaml
color_rules:
  - context: "title_slide"
    rule: >
      タイトルスライドでは primary を背景に使用。テキストは white。
    allowed: ["primary", "white"]

  - context: "content_slide"
    rule: >
      通常コンテンツスライドでは background を使用、テキストは text。
      primary はアクセント（縦ライン・バッジ等）にのみ使用。
    allowed: ["background", "text"]
    accent_only: ["primary"]

  - context: "chart_diagram"
    rule: >
      チャート・図の系列色には chart_* カラーを使用。
      通常スライドコンテンツへの混入禁止。
    allowed: ["chart_green", "chart_blue", "chart_orange", "chart_yellow"]

  - context: "dark_background_component"
    rule: >
      ダーク背景（暗色）を持つすべての要素（スライド・カード・フッター・セクション区切り等）では
      テキストカラーは必ず white (#FFFFFF) または rgba(255,255,255,0.XX) を使用。
      black や暗色テキストをダーク背景に重ねることは禁止。
      コントラスト比は WCAG AA 基準（4.5:1 以上）を満たすこと。
    required: { dark_bg_text: "white", min_opacity: 0.70 }
    forbidden: ["black", "#222", "#333", "#555", "gray"]
```

### Step 7: 抽出サマリーの提示・ユーザー確認

**YAML を生成する前に、抽出した内容をユーザーに提示して確認を得る。**

以下のフォーマットでサマリーを返信し、承認を待つ。承認前に YAML ファイルを出力してはならない。

```
## 抽出結果サマリー

### ブランド情報
- ブランド名: [抽出値 or 推定値]
- スライドサイズ: [幅 × 高さ mm / px]

### カラーパレット
- Primary: [HEX] — [用途]
- Background: [HEX] — [用途]
- Text: [HEX] — [用途]
- [その他抽出した色を列挙]
- ※ 未確認の色: [あれば記載]

### タイポグラフィ
- 見出しフォント: [フォント名 or 未確認]
- 本文フォント: [フォント名 or 未確認]
- フォントサイズ: h1=[pt], h2=[pt], body=[pt] など

### ロゴ
- ロゴファイルパス: [パス or 未確認（テキストフォールバック使用）]
- 配置: [位置・サイズ]

### レイアウトパターン（[N]種類を検出）
1. [パターン名]: [1行説明]
2. [パターン名]: [1行説明]
   ...

### アクセントライン
- [検出された場合: 位置・幅・色を記載 / 検出されなかった場合: 「なし」]

### 未確認・要補完の項目
- [未取得の項目があれば列挙。なければ「なし」]

---
この内容で design-guideline.yaml を生成してよいですか？
修正や追加があればお知らせください。問題なければ「OK」と返信してください。
```

ユーザーから承認（「OK」「はい」「生成して」等）を受け取った後、Step 8 へ進む。
修正指示があった場合はサマリーを更新して再提示し、再度確認を得る。

### Step 8: YAML の出力

**このステップは Step 7 でユーザーの承認を得た後にのみ実行する。**

以下の構造で `design-guideline.yaml` を生成する：

```yaml
# ============================================================
# ブランドガイドライン — design-guideline.yaml
# 抽出元: [参考資料ファイル名]
# 抽出日: YYYY-MM-DD
# ============================================================
name: "[ブランド名]"
version: "1.0"

# --- カラーパレット ---
colors:
  primary:    "[HEX]"   # [用途コメント]
  black:      "#000000"
  white:      "#FFFFFF"
  background: "[HEX]"
  text:       "[HEX]"
  # Extended Palette（チャート・図表専用）
  chart_green:  "[HEX]"
  chart_blue:   "[HEX]"
  # ...

# --- カラー使用ルール ---
color_rules:
  - context: "title_slide"
    rule: >
      [抽出した使用ルール]
    allowed: [...]
  # ...

# --- タイポグラフィ ---
typography:
  heading_font:     "[フォント名]"
  heading_font_web: "[Webフォールバック]"
  body_font:        "[フォント名]"
  body_font_web:    "[Webフォールバック]"
  heading_h1: "[サイズ]"
  heading_h2: "[サイズ]"
  heading_h3: "[サイズ]"
  body_size:  "[サイズ]"

# --- ロゴ ---
# path / path_dark_bg は このYAMLファイルと同じディレクトリからの相対パス
logo:
  name:         "[ブランド名]"
  path:         ""           # 通常背景用ロゴ（例: 赤/黒ロゴ）— Step 7 で確認
  path_dark_bg: ""           # 暗背景用ロゴ（例: 白ロゴ）— Step 7 で確認。不要なら削除
  viewbox:      ""           # SVGのviewBox値（例: "0 0 360 87"）— width計算に必要
  position:     "bottom-left"
  height:   "32px"

# --- フッター（スライドマスターから実測） ---
footer:
  logo_left:   "[px]"    # ロゴ左端からの距離（実測値）
  logo_bottom: "[px]"    # ロゴ下端からの距離（実測値）
  logo_height: "[px]"    # ロゴの高さ（実測値）
  # 日付テキスト（右下に表示される場合）
  date_right:       "[px]"    # 右端からの距離（実測値）
  date_bottom:      "[px]"    # 下端からの距離（実測値）
  date_font_size:   "[px]"    # 日付テキストのフォントサイズ
  date_color_light: "[HEX]"   # ライト背景スライドでの日付テキスト色
  date_color_dark:  "[rgba]"  # ダーク背景スライドでの日付テキスト色
  # 著作権表記
  copyright_text: ""           # 例: "© 2026 Your Company. All Rights Reserved."（空なら非表示）
  copyright_exclude_layouts:   # 著作権を表示しないレイアウト名リスト
    - "[layout-name]"

# --- ブランドアクセント要素（検出された場合のみ記載、なければ省略） ---
# 繰り返し登場する装飾的な視覚要素をすべて列挙する
# type: line / corner / dot / shape / gradient / pattern / other
brand_accents:
  - type: "[line|corner|dot|shape|gradient|pattern|other]"
    description: >
      [要素の外観・配置・用途・禁止事項を自然言語で詳しく記述する。
       presentation-builder はこの description を読んで CSS 実装を決定する。]
    # --- type=line の場合に追記するフィールド ---
    # position: "[left|right|top|bottom]"
    # width:    "[px]"   # position が left/right の場合
    # height:   "[px]"   # position が top/bottom の場合
    # color:    "[HEX]"
    # apply_to: "all"    # all / context名 / [context名リスト]
    # rule: >
    #   [使用ルール・禁止事項がある場合に記載]

# --- ビューア背景色 ---
viewer_background_color: "[HEX]"  # スライドビューア外側の背景色（例: "#2b2b2b"）

# --- 出力モード ---
output_mode: "slide"   # slide | page

# --- レイアウトパターン ---
layout_patterns:
  - name: "title"
    description: >
      [抽出した説明]
    css_class: "layout-title"
  # ...

# --- 抽出メモ ---
extraction_notes: |
  抽出元: [ファイル名リスト]
  確認済み情報: [抽出できた情報の概要]
  未確認・要追加確認: [不明な項目リスト]
```

出力後、以下を報告する：
```
design-guideline.yaml を生成しました。

抽出できた情報:
- カラー: 12色（Primary 3色 + Extended 7色 + Gray 2色）
- レイアウトパターン: 8種類
- タイポグラフィ: フォントサイズ 6段階

未確認（手動で補完してください）:
- ロゴファイルパス（現在はテキストフォールバック）
- アニメーション・トランジション規定
```

---

## 既知ブランド情報がある場合の事前補完

特定ブランドの公式ガイドライン資料を事前に把握している場合は、
抽出前に既知情報を `design-guideline.yaml` のベースとして記入しておくことで精度が向上する。

**手順**:
1. 既知情報を `extraction_notes` に記載しておく
2. PPTX/PDF 解析で取得した値と照合・補完する
3. 矛盾がある場合はガイドライン資料のテキスト記述を優先する

> **注意**: 既知情報を使う場合も、必ずファイルから実測値を取得して検証すること。
> ブランドガイドラインは改定されることがあり、古い既知情報をそのまま使うと誤った YAML が生成される。

---

## 依存関係

**PPTX解析時**:
```bash
pip install python-pptx
```

**PDF解析時**（オプション）:
```bash
pip install pdfplumber  # テキスト・色の抽出
# または
pip install pymupdf     # PyMuPDF
```

**画像・スクリーンショット**: 追加インストール不要（Claude の視覚解析を使用）

---

## 使用例

```
「このファイルからデザインガイドラインを作って」
  → projects/brand/template.pptx を解析 → design-guideline.yaml を生成

「複数のPPTXからブランドガイドラインを抽出して」
  → brand-guide.pptx + slide-template.pptx + layout-samples.pptx を並列解析
  → design-guideline-brand.yaml を生成

「このスクリーンショットのブランドカラーを抽出して」
  → 視覚解析で色・レイアウトを推定 → design-guideline.yaml を生成
```

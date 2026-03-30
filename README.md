# brand-slide-kit

**English** | [日本語](#日本語)

Claude Code plugin that extracts brand design systems from existing materials and generates branded HTML presentations.

## Skills

### `design-extractor`

Analyzes PPTX, PDF, images, and screenshots to extract brand design information and generate a `design-guideline.yaml` file.

**Trigger examples:**

- "Extract design guidelines from this file"
- "Generate design-guideline.yaml from these brand materials"
- "このPPTXからデザインガイドラインを作って"

**Output:** `design-guideline.yaml` — structured brand definition including colors, typography, logo specs, layout patterns, and color usage rules.

**Dependencies:**
```bash
pip install python-pptx        # Required for PPTX files
pip install markitdown         # Required for text extraction from PPTX
pip install pdfplumber         # Optional, for PDF files
```

---

### `presentation-builder`

Converts a Markdown file into a branded HTML presentation using `design-guideline.yaml` as the design source.

**Trigger examples:**

- "Generate a presentation from this Markdown"
- "Build a branded HTML page from this content"
- "このMarkdownからスライドを作って"

**Output:**
- `--mode slide` (default): Reveal.js-based slide deck with keyboard navigation
- `--mode page`: Scrollable web page format

---

## Workflow

These two skills are designed to work together:

```
Brand materials (PPTX/PDF/images)
        ↓  [design-extractor]
design-guideline.yaml
        ↓  [presentation-builder]
presentation-slide.html / presentation-page.html
```

## Installation

```bash
/plugin marketplace add manji6/brand-slide-kit
```

Then install the plugin:

```bash
/plugin install brand-slide@brand-slide-kit
```

After installation, skills are available as:

- `/brand-slide:design-extractor`
- `/brand-slide:presentation-builder`

## License

MIT

---

## 日本語

**[English](#brand-slide-kit)** | 日本語

既存のブランド資料からデザインシステムを抽出し、ブランドガイドライン準拠のHTMLプレゼンを生成するClaude Codeプラグインです。

## スキル

### `design-extractor`

PPTX・PDF・画像・スクリーンショットを解析してブランドのデザイン情報を抽出し、`design-guideline.yaml` を生成します。

**トリガー例:**
- "このPPTXからデザインガイドラインを作って"
- "このファイルからブランドカラーを抽出して"
- "design-guideline.yaml を生成して"

**出力:** `design-guideline.yaml` — カラーパレット・タイポグラフィ・ロゴ仕様・レイアウトパターン・カラー使用ルールを含むブランド定義ファイル。

**依存パッケージ:**
```bash
pip install python-pptx        # PPTXファイルの解析に必要
pip install markitdown         # PPTXからのテキスト抽出に必要
pip install pdfplumber         # PDFファイルの場合（オプション）
```

---

### `presentation-builder`

Markdownファイルと `design-guideline.yaml` を使って、ブランドガイドライン準拠のHTMLプレゼンを生成します。

**トリガー例:**
- "このMarkdownからスライドを作って"
- "design-guideline.yamlを使って説明資料を作って"
- "HTMLスライドを生成して"

**出力形式:**

- `--mode slide`（デフォルト）: Reveal.jsベースのスライド形式（キーボード操作）
- `--mode page`: スクロール型Webページ形式

---

## ワークフロー

2つのスキルは連携して使うことを想定しています：

```
ブランド資料（PPTX / PDF / 画像）
        ↓  [design-extractor]
design-guideline.yaml
        ↓  [presentation-builder]
presentation-slide.html / presentation-page.html
```

## インストール

```bash
/plugin marketplace add manji6/brand-slide-kit
```

プラグインをインストール：

```bash
/plugin install brand-slide@brand-slide-kit
```

インストール後、以下のスキルが使えるようになります：

- `/brand-slide:design-extractor`
- `/brand-slide:presentation-builder`

## ライセンス

MIT

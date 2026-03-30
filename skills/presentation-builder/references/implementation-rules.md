# 技術的実装ルール詳細リファレンス

HTML生成時に参照する詳細な技術ルールと落とし穴集。SKILL.md の「技術的実装ルール」セクションから呼ばれる。

---

## コントラスト比の計算方法（WCAG AA）

相対輝度（Relative Luminance）を用いた計算：

```
# ① HEX → 相対輝度 L の計算
sRGB = R/255, G/255, B/255
channel = c <= 0.04045 ? c/12.92 : ((c+0.055)/1.055)^2.4  # 各チャンネル
L = 0.2126×R + 0.7152×G + 0.0722×B

# ② コントラスト比
ratio = (L_lighter + 0.05) / (L_darker + 0.05)
```

**基準値**:
- 通常テキスト（本文・箇条書き）: **4.5:1 以上**
- 大きいテキスト（h1・h2 など 18pt 以上 or 太字 14pt 以上）: **3:1 以上**

**補正の優先順位**:
1. YAML `color_rules` に明示された組み合わせ → そのまま使用
2. コントラスト比が基準未満の場合:

| 背景色の輝度 | 補正後のテキスト色 |
|---|---|
| 輝度 ≥ 0.18（白〜中間色） | `colors.text`（ダーク系）を使用。それでも不足なら `#000000` |
| 輝度 < 0.18（ダーク系） | `#FFFFFF` または `rgba(255,255,255,0.9)` |

---

## ロゴ SVG の width 計算（絶対配置時）

`position: absolute` で SVG ロゴを配置する場合、`width: auto` は**禁止**。

```
理由: ブラウザが width:auto の SVG にデフォルト 300px を適用し、
      preserveAspectRatio="xMidYMid meet" によってロゴ内容が中央寄りになるバグがある。

計算式: width = height × (viewBox_width / viewBox_height)
例: height=20px, viewBox="0 0 360 87" → width = 20 × (360/87) ≈ 83px
```

YAML の `footer.logo_height` と SVG の viewBox から必ず width を明示する。

---

## タイトルスライドの2点配置レイアウト（座標指定がある場合）

YAML の `layout_patterns[].description` に「h1: y=〇cm」「メタ: y=〇cm」のような実測座標が記載されている場合、`justify-content: flex-end`（下寄せ）は**禁止**。座標どおりに2点配置で実装する。

**座標指定があるタイトルスライドの実装例**:
- h1 (66pt): y=3.1cm = 720px canvas 換算 y≈117px（上部 16%）
- meta (20pt): y=10.0cm = 720px canvas 換算 y≈378px（中間 52%）

**実装パターン（flex + spacer）**:

```css
/* 注: 以下のクラス名はサンプル。実際は YAML の layout_patterns[].css_class に従う */
.reveal .slides > section.layout-title-accent,
.reveal .slides > section.layout-title-dark,
.reveal .slides > section.layout-title-light {
  padding: 117px 65px 80px 65px;  /* top = h1 の Y座標(px) */
  justify-content: flex-start !important;
}
.reveal .slides > section.layout-title-accent { background: [colors.primary] !important; }
.reveal .slides > section.layout-title-dark   { background: #000000 !important; }
.reveal .slides > section.layout-title-light  { background: #FFFFFF !important; }
```

```html
<!-- h1 は padding-top で上端から正確に配置 -->
<section class="layout-title-accent">
  <h1>タイトルテキスト</h1>
  <!-- spacer で meta を 52% 位置まで押し下げる -->
  <div style="flex:1; min-height:0;"></div>
  <p class="meta">登壇者名・肩書・日付</p>
  <!-- footer（ロゴ） -->
</section>
```

**Y座標の換算式（EMU → 720px canvas）**:
```
px = (y_cm / slide_height_cm) × 720
例: y=3.1cm → (3.1 / 19.05) × 720 ≈ 117px
例: y=10.0cm → (10.0 / 19.05) × 720 ≈ 378px
```

**注意**: spacer の `flex:1` はセクション高さが `height: 720px !important` で固定されている場合に確実に機能する。`min-height` を使っていると spacer が伸縮できず失敗する。

---

## ダイアグラム wrapper

diagram を含むスライドの wrapper には以下を設定する:

```css
.diagram-wrapper {
  flex: 1;
  min-height: 360px;  /* コンテンツ量に応じて調整 */
  /* max-height は設定しない — ダイアグラムが縮小されるため */
}
```

---

## タイポグラフィ一貫性

`typography.heading_h2` の値はすべてのレイアウトの h2 に統一して適用する。レイアウトごとに異なるサイズを設定すると、スライド間で見出しサイズが揃わなくなる。

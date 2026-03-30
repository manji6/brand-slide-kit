# PPTX抽出スクリプト リファレンス

PPTX解析時に使用するPythonコードのリファレンス集。
SKILL.mdのStep 3（PPTXファイルの場合）から参照する。

---

## 1. 基本構造の抽出（色・フォント・フッター）

```python
from pptx import Presentation
from pptx.util import Pt
from pptx.dml.color import RGBColor

prs = Presentation(pptx_path)

# 1. スライドサイズ
width_mm  = prs.slide_width.mm   # 例: 33.87 (12.19 inch * 25.4)
height_mm = prs.slide_height.mm  # 例: 19.05 (6.87 inch * 25.4)

# 2. スライドマスターからデザイン要素を抽出
for master in prs.slide_masters:
    for layout in master.slide_layouts:
        for shape in layout.shapes:
            # テキストフレームの色・フォントサイズ
            if shape.has_text_frame:
                for para in shape.text_frame.paragraphs:
                    for run in para.runs:
                        color = run.font.color.rgb  # RGBColor
                        size_pt = run.font.size / 12700  # EMU → pt 変換
            # 塗りつぶし色（背景シェイプ）
            if shape.fill.type is not None:
                fg_color = shape.fill.fore_color.rgb

# 3. 各スライドからサンプリング（最大10枚）
for i, slide in enumerate(prs.slides[:10]):
    for shape in slide.shapes:
        # フォントサイズの注意: run.font.size はEMU値
        # 正しいpt換算: run.font.size / 12700
        # ※ Pt(run.font.size).pt は誤り（巨大な値になる）
        pass

# 4. ブランドアクセント要素（繰り返し登場する装飾的シェイプ）を検出
# 検出対象: 細いライン / コーナーマーク / ドット / 幾何学シェイプ / グラデーション帯 等
for shape in slide.shapes:
    w_mm = shape.width.mm
    h_mm = shape.height.mm
    l_mm = shape.left.mm
    # ライン候補: 細い縦長 or 横長の矩形
    if shape.shape_type == 1:  # RECTANGLE
        if (w_mm < 5 and h_mm > 50) or (h_mm < 5 and w_mm > 50):
            try:
                color = shape.fill.fore_color.rgb
            except:
                color = None  # schemeClr の場合はテキストルールから色を特定
    # その他の装飾的シェイプ（楕円・三角等）も記録して視覚解析で判断
    # → 複数スライドで同じ位置・サイズ・色で繰り返されるシェイプは brand_accents 候補
```

---

## 2. フッター要素の位置・寸法の実測

```python
slide_h_emu = prs.slide_height.emu
slide_w_emu = prs.slide_width.emu
slide_h_cm  = slide_h_emu / 360000
slide_w_cm  = slide_w_emu / 360000

for master in prs.slide_masters:
    for shape in master.shapes:
        try:
            y_cm = shape.top / 360000
            h_cm = shape.height / 360000
            x_cm = shape.left / 360000
            w_cm = shape.width / 360000
            # スライド下端から 15% 以内のシェイプをフッター要素として記録
            if y_cm > slide_h_cm * 0.85:
                bottom_cm = slide_h_cm - (y_cm + h_cm)
                right_cm  = slide_w_cm - (x_cm + w_cm)
                # px換算（slide_size.width_px × height_px ベース）
                left_px   = round(x_cm   / slide_w_cm * 1280)
                bottom_px = round(bottom_cm / slide_h_cm * 720)
                height_px = round(h_cm   / slide_h_cm * 720)
                right_px  = round(right_cm  / slide_w_cm * 1280)
                # フォントサイズ・テキスト色
                font_size_pt = None
                text_color   = None
                if shape.has_text_frame:
                    for para in shape.text_frame.paragraphs:
                        for run in para.runs:
                            if run.font.size:
                                font_size_pt = round(run.font.size / 12700, 1)
                            try:
                                text_color = f"#{run.font.color.rgb}"
                            except:
                                pass
                print(f"footer '{shape.name}': "
                      f"left={left_px}px bottom={bottom_px}px height={height_px}px "
                      f"right={right_px}px font={font_size_pt}pt color={text_color}")
        except:
            pass
```

---

## 3. レイアウトバリアントの全件走査

```python
from pptx import Presentation
prs = Presentation(pptx_path)
for master in prs.slide_masters:
    for layout in master.slide_layouts:
        bg_fill = layout.background.fill
        print(layout.name, bg_fill.type, bg_fill.fore_color.rgb if bg_fill.type else 'inherited')
```

**背景色がテーマカラー（schemeClr）の場合**: `accent6` 等の名前だけ記録してはいけない。XMLからテーマの対応HEXを取得して記載する。

```python
# prs.slide_master.theme_color_map で accent6 → #FF5500 等を解決
```

---

## 4. テキスト縦位置の実測

layout_patterns の `description` には、タイトル・本文・メタ情報のY座標（cm）を実測値として記載する。「上部」「下部」「中央」などの曖昧な表現は禁止。

```python
# EMU → cm 変換: 1cm = 360000 EMU
for sp in layout.shapes:
    if sp.has_text_frame:
        y_cm = sp.top / 360000      # Y座標（スライド上端からの距離）
        h_cm = sp.height / 360000   # テキストボックス高さ
        ph = sp.placeholder_format
        ph_type = ph.type if ph else None
        print(f"  ph={ph_type} y={y_cm:.1f}cm h={h_cm:.1f}cm")
```

**記述例（正しい）**:
```
タイトルテキスト（h1）: y=3.1cm（スライド高さ 19.05cm の 16%、上部左寄せ）
メタ（登壇者名）      : y=10.0cm（スライド高さの 52%、中間左寄せ）
```

**記述例（NG）**:
```
タイトル：左寄せ
登壇者名：下部に配置  ← y座標なし・「下部」が曖昧
```

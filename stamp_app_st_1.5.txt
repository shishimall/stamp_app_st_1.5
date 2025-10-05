# stamp_app_st

# stamp_app_st_1.15.py
# Web環境用デフォルト補正（ver.1.15）

import streamlit as st
from PIL import Image, ImageDraw, ImageFont
from datetime import datetime
import io, re, os

def text_size(draw, text, font):
    bbox = draw.textbbox((0, 0), text, font=font)
    return bbox[2] - bbox[0], bbox[3] - bbox[1]

def normalize_date_input(input_str):
    if not input_str or input_str.strip() == "":
        return datetime.now().strftime("%y.%m.%d")
    s = re.sub(r"[^\d]", "", input_str)
    if len(s) == 6:
        return f"{s[0:2]}.{s[2:4]}.{s[4:6]}"
    elif len(s) == 8:
        return f"{s[2:4]}.{s[4:6]}.{s[6:8]}"
    else:
        return datetime.now().strftime("%y.%m.%d")

def get_font_path(font_family="msgothic.ttc"):
    local_fonts = [
        os.path.join(os.path.dirname(__file__), "fonts", "NotoSansCJKjp-Regular.otf"),
        os.path.join(os.path.dirname(__file__), "fonts", "NotoSansJP-Regular.ttf"),
    ]
    for f in local_fonts:
        if os.path.exists(f):
            return f

    candidates = [
        f"C:/Windows/Fonts/{font_family}",
        "C:/Windows/Fonts/meiryo.ttc",
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            return path

    raise FileNotFoundError("フォントが見つかりません。")

def create_stamp(name_top, name_bottom, include_date, custom_date,
                 font_family, font_size, top_offset, bottom_offset, scale_factor):
    size = 400
    img = Image.new("RGBA", (size, size), (255, 255, 255, 0))
    draw = ImageDraw.Draw(img)
    red = (220, 0, 0, 255)
    center = size / 2

    draw.ellipse((20, 20, size - 20, size - 20), outline=red, width=10)
    font_path = get_font_path(font_family)

    if hasattr(ImageFont, "LAYOUT_BASIC"):
        font = ImageFont.truetype(font_path, int(font_size * scale_factor),
                                  layout_engine=ImageFont.LAYOUT_BASIC)
        font_small = ImageFont.truetype(font_path, int(font_size * 0.6),
                                        layout_engine=ImageFont.LAYOUT_BASIC)
    else:
        font = ImageFont.truetype(font_path, int(font_size * scale_factor))
        font_small = ImageFont.truetype(font_path, int(font_size * 0.6))

    w1, h1 = text_size(draw, name_top, font)
    draw.text((center - w1 / 2, center + top_offset), name_top, fill=red, font=font)

    if include_date:
        date_str = normalize_date_input(custom_date)
        w2, h2 = text_size(draw, date_str, font_small)
        draw.text((center - w2 / 2, center - h2 / 2 + 5), date_str, fill=red, font=font_small)
        margin = 35
        draw.line((margin, center - 45, size - margin, center - 45), fill=red, width=3)
        draw.line((margin, center + 60, size - margin, center + 60), fill=red, width=3)

    w3, h3 = text_size(draw, name_bottom, font)
    draw.text((center - w3 / 2, center + bottom_offset), name_bottom, fill=red, font=font)
    return img

# ------------------------------------------------------------
# 環境判定（ローカル or Streamlit Cloud）
# ------------------------------------------------------------
is_windows = os.path.exists("C:/Windows/Fonts")

# ------------------------------------------------------------
# Streamlit UI
# ------------------------------------------------------------
st.set_page_config(page_title="電子日付印", layout="wide")
st.title("🟥 電子日付印／シャチハタ印ジェネレーター（ver.1.15 Webデフォルト補正）")

col_prev, col_ctl = st.columns([2, 1])

with col_ctl:
    name_top = st.text_input("上段（例：宮）", "宮")
    name_bottom = st.text_input("下段（例：原）", "原")
    mode = st.radio("印影タイプ", ["日付印タイプ（2本線）", "シャチハタ風（中央寄せ）"])

    # ✅ 環境別デフォルト設定
    if is_windows:
        # ローカル（既存値）
        default_font = "msgothic.ttc"
        defaults = {
            "date_font": 85, "date_scale": 0.95, "date_top": -165, "date_bottom": 55,
            "shachi_font": 120, "shachi_scale": 1.0, "shachi_top": -150, "shachi_bottom": 40
        }
    else:
        # Streamlit Cloud（添付画像ベース）
        default_font = "NotoSansCJKjp-Regular.otf"
        defaults = {
            "date_font": 80, "date_scale": 1.05, "date_top": -175, "date_bottom": 50,
            "shachi_font": 120, "shachi_scale": 1.10, "shachi_top": -185, "shachi_bottom": -30
        }

    if mode.startswith("日付印"):
        include_date = True
        custom_date = st.text_input("日付（例：25.10.05）", datetime.now().strftime("%y.%m.%d"))
        font_size = st.slider("基本フォントサイズ", 40, 140, defaults["date_font"])
        scale_factor = st.slider("拡大倍率", 0.8, 1.5, defaults["date_scale"], 0.05)
        top_offset = st.slider("上段位置", -200, 0, defaults["date_top"], 5)
        bottom_offset = st.slider("下段位置", 0, 200, defaults["date_bottom"], 5)
    else:
        include_date = False
        custom_date = None
        font_size = st.slider("基本フォントサイズ", 60, 180, defaults["shachi_font"])
        scale_factor = st.slider("拡大倍率", 0.8, 1.5, defaults["shachi_scale"], 0.05)
        top_offset = st.slider("上段位置", -220, 0, defaults["shachi_top"], 5)
        bottom_offset = st.slider("下段位置", -100, 300, defaults["shachi_bottom"], 5)

    font_family = st.selectbox("フォント", [default_font, "msgothic.ttc", "meiryo.ttc"])

img = create_stamp(name_top, name_bottom, include_date, custom_date,
                   font_family, font_size, top_offset, bottom_offset, scale_factor)

with col_prev:
    st.image(img, caption="印影プレビュー", use_column_width=False)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    st.download_button("📥 PNGを保存", buf.getvalue(),
                       file_name=f"{name_top}{name_bottom}_stamp.png",
                       mime="image/png")

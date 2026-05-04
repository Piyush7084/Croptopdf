import streamlit as st
from streamlit_drawable_canvas import st_canvas
from PIL import Image
import base64
from io import BytesIO
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
import os
import tempfile
import json

st.set_page_config(page_title="Area Selector to PDF", layout="wide")

# --- FUNCTIONS ---
def convert_to_base64(pil_img):
    buffered = BytesIO()
    pil_img.save(buffered, format="PNG")
    return f"data:image/png;base64,{base64.b64encode(buffered.getvalue()).decode()}"

# --- SESSION STATE ---
if "processed_images" not in st.session_state:
    st.session_state.processed_images = []

st.title("✂️ Stage 1: Select Image Areas")

# 1. Main File Uploader
uploaded_main = st.file_uploader("Upload the main image", type=["png", "jpg", "jpeg"])

if uploaded_main:
    main_img = Image.open(uploaded_main).convert("RGB")
    
    # Selection Canvas
    canvas_result = st_canvas(
        fill_color="rgba(0, 255, 0, 0.2)",
        stroke_width=2,
        stroke_color="#00ff00",
        background_image=main_img,
        update_streamlit=True,
        height=main_img.height,
        width=main_img.width,
        drawing_mode="rect",
        key="main_canvas",
    )

    # Convert Selections to Feed for App 2
    if canvas_result.json_data and canvas_result.json_data["objects"]:
        if st.button("✅ Confirm Areas and Move to PDF Stage", type="primary"):
            new_feed = []
            for i, obj in enumerate(canvas_result.json_data["objects"]):
                l, t = int(obj["left"]), int(obj["top"])
                w, h = int(obj["width"]), int(obj["height"])
                
                if w > 0 and h > 0:
                    crop = main_img.crop((l, t, l + w, t + h))
                    new_feed.append({
                        "original_name": f"area_{i+1}.png",
                        "image_obj": crop,
                        "pdf_name": f"selection_{i+1}.pdf",
                        "rotation": 0
                    })
            st.session_state.processed_images = new_feed
            st.success(f"Captured {len(new_feed)} areas! Scroll down to Stage 2.")

# --- STAGE 2: THE PDF CONVERTER ---
if st.session_state.processed_images:
    st.divider()
    st.title("📄 Stage 2: Edit & Generate PDFs")
    
    for idx, item in enumerate(st.session_state.processed_images):
        col1, col2 = st.columns([3, 2])
        
        with col1:
            preview_img = item["image_obj"].rotate(-item["rotation"], expand=True)
            st.image(preview_img, caption=f"Area {idx+1}", width=400)
        
        with col2:
            st.write(f"**Settings for Area {idx+1}**")
            new_name = st.text_input("PDF Filename", value=item["pdf_name"], key=f"name_input_{idx}")
            if not new_name.lower().endswith('.pdf'):
                new_name += ".pdf"
            item["pdf_name"] = new_name

            if st.button(f"🔄 Rotate 90°", key=f"rot_btn_{idx}"):
                item["rotation"] = (item["rotation"] + 90) % 360
                st.rerun()

    # Generation Logic
    if st.button("🚀 Generate All PDFs", type="primary", key="gen_all"):
        st.session_state.generated_pdfs = []
        
        for item in st.session_state.processed_images:
            buffer = BytesIO()
            c = canvas.Canvas(buffer, pagesize=letter)
            page_w, page_h = letter
            
            # Apply final rotation
            img = item["image_obj"].rotate(-item["rotation"], expand=True)
            img_w, img_h = img.size
            
            # Scale to fit PDF
            margin = 36
            ratio = min((page_w - 2*margin)/img_w, (page_h - 2*margin)/img_h)
            new_w, new_h = img_w * ratio, img_h * ratio
            x, y = (page_w - new_w)/2, (page_h - new_h)/2
            
            with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as tmp:
                img.save(tmp.name, "JPEG")
                c.drawImage(tmp.name, x, y, width=new_w, height=new_h)
                c.save()
                st.session_state.generated_pdfs.append({"name": item["pdf_name"], "data": buffer.getvalue()})
            os.remove(tmp.name)
        
        st.success("PDFs Ready!")

    # Individual Download Buttons
    if "generated_pdfs" in st.session_state:
        st.write("### ⬇️ Download Files")
        dl_cols = st.columns(3)
        for i, pdf in enumerate(st.session_state.generated_pdfs):
            with dl_cols[i % 3]:
                st.download_button(label=f"PDF {pdf['name']}", data=pdf['data'], file_name=pdf['name'], key=f"final_dl_{i}")
                

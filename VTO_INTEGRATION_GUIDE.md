# Hướng Dẫn Tích Hợp VTO (Virtual Try-On) với CatVTON

## 📋 Tổng Quan

Hệ thống đã được tích hợp sẵn cấu trúc cơ bản cho VTO. Hiện tại đang sử dụng **mock pipeline** (placeholder). Bạn cần thay thế bằng **CatVTON pipeline thật** từ code Kaggle.

---

## 🏗️ Cấu Trúc Đã Tạo

### 1. Backend (`retrieval_service/`)

**`vto_service.py`**:
- Module chứa VTO pipeline functions
- Hiện tại: Mock implementation
- Cần thay thế: CatVTON pipeline thật

**`main.py`**:
- Endpoint: `POST /vto`
- Nhận: `person_image` (base64), `cloth_image` (URL hoặc base64)
- Trả về: `result_image` (base64)

### 2. Frontend (`components/`)

**`vto-modal.tsx`**:
- Upload pose image
- Gọi API `/vto`
- Hiển thị kết quả (3 ảnh: pose gốc, sản phẩm, kết quả)

**`product-card.tsx`**:
- Button "Try On" (Zap icon) đã có sẵn
- Mở VTOModal khi click

---

## 🔧 Cách Tích Hợp CatVTON Thật

### Bước 1: Cài Đặt Dependencies

Thêm vào `retrieval_service/requirements.txt`:

```txt
# VTO Dependencies
diffusers==0.24.0
transformers==4.36.2
huggingface-hub==0.24.6
opencv-python-headless>=4.8.0
pillow>=10.0.0
accelerate>=0.30.0
peft==0.7.1
tokenizers==0.15.0
```

### Bước 2: Clone CatVTON Repository

```bash
cd retrieval_service
git clone https://github.com/Zheng-Chong/CatVTON.git
```

### Bước 3: Cập Nhật `vto_service.py`

Thay thế phần `_load_models()` và `run_virtual_tryon()` bằng code từ Kaggle:

```python
# Thay thế phần này trong vto_service.py:

def _load_models():
    """Load CatVTON models."""
    global _vto_pipeline, _seg_processor, _seg_model
    global _clip_model, _clip_processor, _blip_model, _blip_processor
    
    if _vto_pipeline is not None:
        return
    
    device = _get_device()
    
    # 1. Load Segformer (from your Kaggle code)
    from transformers import SegformerImageProcessor, AutoModelForSemanticSegmentation
    _seg_processor = SegformerImageProcessor.from_pretrained("mattmdjaga/segformer_b2_clothes")
    _seg_model = AutoModelForSemanticSegmentation.from_pretrained("mattmdjaga/segformer_b2_clothes").to(device)
    
    # 2. Load CatVTON Pipeline (from your Kaggle code)
    import sys
    sys.path.append("CatVTON")  # Add CatVTON to path
    from model.pipeline import CatVTONPipeline
    
    _vto_pipeline = CatVTONPipeline(
        base_ckpt="runwayml/stable-diffusion-inpainting",
        attn_ckpt="ckpt",  # Download from HuggingFace
        attn_ckpt_version="vitonhd",
        weight_dtype=torch.float16,
        device=device,
        skip_safety_check=True,
    )
    
    # 3. Load CLIP (from your Kaggle code)
    from transformers import CLIPProcessor, CLIPModel
    _clip_model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32").to(device)
    _clip_processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")
    
    # 4. Load BLIP (from your Kaggle code)
    from transformers import BlipProcessor, BlipForConditionalGeneration
    _blip_processor = BlipProcessor.from_pretrained("Salesforce/blip-image-captioning-base")
    _blip_model = BlipForConditionalGeneration.from_pretrained("Salesforce/blip-image-captioning-base").to(device)
```

### Bước 4: Cập Nhật Functions

Thay thế các functions trong `vto_service.py`:

```python
def classify_cloth_type(image: Image.Image) -> Tuple[str, float]:
    """Use CLIP model from Kaggle code."""
    if _clip_model is None or _clip_processor is None:
        return "upper", 0.9
    
    labels = ["upper garment, shirt, top, jacket", "lower body clothing, pants, skirt, jeans"]
    inputs = _clip_processor(text=labels, images=image, return_tensors="pt", padding=True).to(_get_device())
    
    with torch.no_grad():
        probs = _clip_model(**inputs).logits_per_image.softmax(dim=1)
    
    idx = probs.argmax().item()
    confidence = probs[0][idx].item()
    return ("upper" if idx == 0 else "lower"), confidence

def generate_caption(image: Image.Image) -> str:
    """Use BLIP model from Kaggle code."""
    if _blip_model is None or _blip_processor is None:
        return "clothing item"
    
    inputs = _blip_processor(image, return_tensors="pt").to(_get_device())
    with torch.no_grad():
        out = _blip_model.generate(**inputs, max_new_tokens=30)
    caption = _blip_processor.decode(out[0], skip_special_tokens=True)
    return caption

def get_mask_from_segformer(image: Image.Image, cloth_type: str = "upper") -> Image.Image:
    """Use Segformer model from Kaggle code."""
    # Copy code từ CELL 3 của Kaggle
    # ...
```

### Bước 5: Cập Nhật `run_virtual_tryon()`

Thay thế bằng code từ Kaggle CELL 3:

```python
def run_virtual_tryon(...):
    # Copy toàn bộ logic từ CELL 3 của Kaggle
    # Sử dụng _vto_pipeline, _seg_model, _clip_model, _blip_model
    # ...
```

---

## 🔑 HuggingFace Token

Cần set HuggingFace token để download models:

```python
# Trong vto_service.py hoặc main.py
from huggingface_hub import login
HF_TOKEN = os.environ.get("HF_TOKEN", "your_token_here")
login(HF_TOKEN)
```

Hoặc set environment variable:
```bash
export HF_TOKEN="your_token_here"
```

---

## 📝 Lưu Ý Quan Trọng

### 1. Model Loading
- Models rất lớn (vài GB)
- Cần GPU để chạy nhanh (CPU sẽ rất chậm)
- Có thể lazy load (chỉ load khi cần)

### 2. Performance
- VTO pipeline mất ~5-30 giây tùy GPU
- Cần timeout phù hợp ở frontend
- Có thể cache results

### 3. Error Handling
- Handle memory errors (models lớn)
- Handle timeout
- Fallback nếu models không load được

### 4. Production
- Cân nhắc chạy VTO trên server riêng (GPU server)
- Hoặc dùng cloud service (AWS, GCP)
- Có thể queue system cho VTO requests

---

## 🧪 Test Flow

1. User: "Show me polo"
2. System: Hiển thị products
3. User: Click "Try On" button trên product
4. VTOModal mở → Upload pose image
5. Click "Thử đồ ngay"
6. Frontend gọi `/vto` API
7. Backend chạy CatVTON pipeline
8. Trả về result image
9. Frontend hiển thị kết quả

---

## 🚀 Next Steps

1. **Thay thế mock pipeline** bằng CatVTON thật
2. **Test với real images** từ user
3. **Optimize performance** (caching, batching)
4. **Add error handling** tốt hơn
5. **Consider GPU server** nếu cần scale

---

## 📚 Files Cần Sửa

- `retrieval_service/vto_service.py` - Thay mock bằng CatVTON thật
- `retrieval_service/requirements.txt` - Đã thêm dependencies
- `retrieval_service/main.py` - Endpoint `/vto` đã có
- `components/vto-modal.tsx` - UI đã hoàn chỉnh
- `components/product-card.tsx` - Button đã có sẵn

---

## ⚠️ Current Status

✅ **Đã hoàn thành:**
- Backend endpoint `/vto`
- Frontend VTOModal với upload
- UI/UX hoàn chỉnh
- Error handling cơ bản

⚠️ **Cần làm:**
- Thay mock pipeline bằng CatVTON thật
- Test với real models
- Optimize performance

Bạn có thể bắt đầu thay thế mock pipeline bằng code CatVTON từ Kaggle!


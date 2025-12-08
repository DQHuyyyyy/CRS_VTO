# Hướng Dẫn Tích Hợp Qwen Fine-tuned Model

## 🎯 Tại Sao Fine-tune Qwen KHÔNG Vô Nghĩa?

**Fine-tune Qwen của bạn CÓ THỂ được sử dụng để thay thế GPT-4o-mini cho:**

1. ✅ **Intent Classification** - Phân loại ý định người dùng
2. ✅ **Response Generation** - Sinh câu trả lời tự nhiên
3. ✅ **Tiết kiệm chi phí** - Không cần trả phí API OpenAI
4. ✅ **Privacy** - Dữ liệu không gửi lên cloud
5. ✅ **Tốc độ** - Có thể nhanh hơn nếu có GPU

---

## 📊 So Sánh: Qwen vs GPT-4o-mini

| Tiêu chí | Qwen 3B (Fine-tuned) | GPT-4o-mini |
|----------|---------------------|-------------|
| **Chi phí** | Miễn phí (local) | ~$0.15/1M tokens |
| **Privacy** | ✅ 100% local | ❌ Gửi lên OpenAI |
| **Tốc độ** | ~200-500ms (GPU) | ~500-2000ms (API) |
| **Chất lượng** | Tùy fine-tune | Rất tốt |
| **Cần GPU** | ✅ Khuyến nghị | ❌ Không cần |
| **Fine-tune** | ✅ Đã fine-tune | ❌ Không thể |

---

## 🚀 Cách Tích Hợp Qwen Vào Hệ Thống

### Option 1: Thay thế hoàn toàn GPT-4o-mini

**Lợi ích:**
- ✅ Không cần OpenAI API key
- ✅ Privacy hoàn toàn
- ✅ Tiết kiệm chi phí

**Nhược điểm:**
- ❌ Cần GPU để chạy nhanh
- ❌ Chất lượng có thể kém hơn GPT-4o-mini (tùy fine-tune)

### Option 2: Hybrid (Qwen + GPT)

**Lợi ích:**
- ✅ Qwen cho Intent (nhanh, rẻ)
- ✅ GPT cho Response (chất lượng cao)
- ✅ Fallback nếu Qwen lỗi

**Nhược điểm:**
- ❌ Vẫn cần OpenAI API key
- ❌ Phức tạp hơn

### Option 3: Chỉ dùng Qwen khi không có API key

**Lợi ích:**
- ✅ Fallback tự động
- ✅ Vẫn dùng GPT khi có API key

**Nhược điểm:**
- ❌ Chất lượng không đồng nhất

---

## 💻 Implementation Plan

### Bước 1: Tạo Qwen Service trong Python

Tạo endpoint mới trong `retrieval_service/main.py`:

```python
@app.post("/qwen/intent")
def qwen_classify_intent(req: QwenIntentRequest) -> QwenIntentResponse:
    """Classify intent using Qwen model."""
    # Load Qwen model
    # Generate response
    # Return intent
    pass

@app.post("/qwen/generate")
def qwen_generate_response(req: QwenGenerateRequest) -> QwenGenerateResponse:
    """Generate assistant message using Qwen model."""
    # Load Qwen model
    # Generate response
    # Return message
    pass
```

### Bước 2: Update Frontend API Route

Trong `app/api/search/route.ts`:

```typescript
// Option: Use Qwen if available, fallback to GPT
async function classifyIntent(q: string): Promise<Intent> {
  // Try Qwen first
  try {
    const qwenRes = await fetch("http://127.0.0.1:8000/qwen/intent", {
      method: "POST",
      body: JSON.stringify({ query: q }),
    });
    if (qwenRes.ok) {
      const data = await qwenRes.json();
      return data.intent;
    }
  } catch {
    // Fallback to GPT
  }
  
  // Use GPT-4o-mini as fallback
  return classifyIntentWithGPT(q);
}
```

---

## 🔧 Code Example: Qwen Integration

### 1. Load Qwen Model (trong retrieval_service/main.py)

```python
_qwen_model: Any | None = None
_qwen_tokenizer: Any | None = None

def _load_qwen_llm() -> Tuple[Any, Any]:
    """Load Qwen 3B for LLM tasks (intent, generation)."""
    from transformers import AutoModelForCausalLM, AutoTokenizer
    from peft import PeftModel
    
    adapter_path = _get_adapter_path()
    config_path = adapter_path / "adapter_config.json"
    
    with open(config_path, "r", encoding="utf-8") as f:
        cfg = json.load(f)
    base = cfg.get("base_model_name_or_path", "Qwen/Qwen2.5-3B-Instruct")
    
    logger.info(f"[QWEN] Loading base model: {base}")
    tokenizer = AutoTokenizer.from_pretrained(base, trust_remote_code=True)
    
    model = AutoModelForCausalLM.from_pretrained(
        base,
        trust_remote_code=True,
        torch_dtype=torch.float16 if torch.cuda.is_available() else torch.float32,
        device_map="auto" if torch.cuda.is_available() else None,
    )
    
    logger.info(f"[QWEN] Attaching LoRA adapter...")
    model = PeftModel.from_pretrained(model, str(adapter_path))
    model.eval()
    
    return model, tokenizer
```

### 2. Intent Classification với Qwen

```python
def _qwen_classify_intent(query: str) -> str:
    """Classify intent using Qwen model."""
    if not _qwen_model or not _qwen_tokenizer:
        return "unknown"
    
    prompt = f"""<|im_start|>system
Bạn là bộ phân loại intent cho một TRỢ LÝ TƯ VẤN THỜI TRANG. 
Hãy phân loại câu của user vào đúng MỘT trong 3 nhóm:
- fashion (tìm kiếm / tư vấn sản phẩm quần áo, giày dép, phụ kiện thời trang)
- chitchat (chào hỏi, nói chuyện phiếm, câu thân mật không cần gợi ý sản phẩm)
- off_topic (chủ đề không liên quan đến thời trang)

Chỉ trả lời duy nhất một từ: fashion, chitchat, hoặc off_topic.
<|im_end|>
<|im_start|>user
{query}
<|im_end|>
<|im_start|>assistant
"""
    
    inputs = _qwen_tokenizer(prompt, return_tensors="pt")
    device = next(_qwen_model.parameters()).device
    inputs = {k: v.to(device) for k, v in inputs.items()}
    
    with torch.no_grad():
        outputs = _qwen_model.generate(
            **inputs,
            max_new_tokens=4,
            temperature=0.1,
            do_sample=False,
        )
    
    response = _qwen_tokenizer.decode(outputs[0], skip_special_tokens=True)
    intent = response.split("assistant")[-1].strip().lower()
    
    if "fashion" in intent:
        return "fashion"
    elif "chitchat" in intent:
        return "chitchat"
    elif "off_topic" in intent or "offtopic" in intent:
        return "off_topic"
    return "unknown"
```

### 3. Response Generation với Qwen

```python
def _qwen_generate_response(query: str, intent: str, has_products: bool) -> str:
    """Generate assistant message using Qwen model."""
    if not _qwen_model or not _qwen_tokenizer:
        return ""
    
    prompt = f"""<|im_start|>system
Bạn là một TRỢ LÝ TƯ VẤN THỜI TRANG nói tiếng Việt.
QUAN TRỌNG: Bạn CHỈ được nói về sản phẩm có trong hệ thống database.
KHÔNG BAO GIỜ được đưa ra tên cửa hàng, thương hiệu, hoặc bất kỳ thông tin nào không có trong database.
Luôn trả lời ngắn gọn, tự nhiên, chỉ dựa trên dữ liệu thực tế có trong hệ thống.
<|im_end|>
<|im_start|>user
Intent: {intent}
Có tìm được sản phẩm? {"Có" if has_products else "Không"}
Câu người dùng: "{query}"
<|im_end|>
<|im_start|>assistant
"""
    
    inputs = _qwen_tokenizer(prompt, return_tensors="pt")
    device = next(_qwen_model.parameters()).device
    inputs = {k: v.to(device) for k, v in inputs.items()}
    
    with torch.no_grad():
        outputs = _qwen_model.generate(
            **inputs,
            max_new_tokens=120,
            temperature=0.5,
            do_sample=True,
        )
    
    response = _qwen_tokenizer.decode(outputs[0], skip_special_tokens=True)
    message = response.split("assistant")[-1].strip()
    return message
```

---

## 📈 Performance Considerations

### GPU Requirements

- **CPU only**: ~2-5 giây/request (chậm)
- **GPU (CUDA)**: ~200-500ms/request (nhanh)
- **GPU (MPS - Apple Silicon)**: ~300-600ms/request

### Memory Requirements

- **Qwen 3B**: ~6-8GB RAM/VRAM
- **LoRA adapter**: +1-2GB

### Optimization Tips

1. **Quantization**: Dùng 8-bit hoặc 4-bit quantization để giảm memory
2. **Batch Processing**: Xử lý nhiều requests cùng lúc
3. **Caching**: Cache model trong memory để tránh reload

---

## 🎯 Recommendation

### Nếu bạn có GPU:
✅ **Nên dùng Qwen** cho cả Intent và Response Generation
- Tiết kiệm chi phí
- Privacy tốt
- Tốc độ chấp nhận được

### Nếu không có GPU:
⚠️ **Cân nhắc**:
- Dùng Qwen cho Intent (chấp nhận chậm)
- Dùng GPT cho Response (chất lượng cao hơn)

### Hybrid Approach (Khuyến nghị):
✅ **Dùng cả 2**:
- Qwen làm primary (khi có GPU)
- GPT làm fallback (khi Qwen lỗi hoặc không có GPU)

---

## 🔄 Migration Path

1. **Phase 1**: Thêm Qwen endpoints vào Python service
2. **Phase 2**: Update frontend để dùng Qwen (với fallback GPT)
3. **Phase 3**: Test và so sánh chất lượng
4. **Phase 4**: Quyết định dùng Qwen hoàn toàn hay hybrid

---

## ❓ FAQ

**Q: Fine-tune Qwen có tốt hơn GPT-4o-mini không?**
A: Tùy vào chất lượng fine-tune. Nếu fine-tune tốt, có thể tương đương hoặc tốt hơn cho domain cụ thể.

**Q: Có cần rebuild index không?**
A: Không. Qwen chỉ dùng cho LLM tasks (intent, generation), không liên quan đến embedding/index.

**Q: Có thể dùng Qwen cho embedding không?**
A: Có thể, nhưng e5-large-v2 tốt hơn cho embedding task.

**Q: Fine-tune Qwen có mất không?**
A: Không. Model và LoRA adapter vẫn còn trong `epoch_2_final/`.


# Kiến Trúc Hệ Thống - Giải Thích Lý Thuyết

## 📋 Tổng Quan Kiến Trúc

Hệ thống của bạn sử dụng kiến trúc **RAG (Retrieval-Augmented Generation)** với 3 thành phần chính:

```
User Query
    ↓
[Intent Classification] ← OpenAI GPT-4o-mini (LLM)
    ↓
[Retrieval System] ← e5-large-v2 (Embedding Model) + FAISS
    ↓
[Response Generation] ← OpenAI GPT-4o-mini (LLM)
    ↓
Final Response
```

---

## 🔍 1. VAI TRÒ CỦA QWEN TRONG HỆ THỐNG

### ❌ Qwen KHÔNG còn được dùng trong hệ thống hiện tại

**Lý do:**
- Bạn đã thay thế Qwen 3B bằng `intfloat/e5-large-v2` cho embedding
- Index đã được rebuild với e5-large-v2 (dimension 1024)
- Code retrieval service đã chuyển sang SentenceTransformer

**Qwen chỉ còn trong:**
- `build_index.py` (nhưng bạn đã build index với e5-large-v2 rồi)
- `epoch_2_final/` directory (LoRA adapter, không còn dùng)

---

## 🧠 2. VAI TRÒ CỦA LLM (Large Language Model)

### ✅ OpenAI GPT-4o-mini được dùng cho 2 mục đích:

#### A. **Intent Classification** (`classifyIntent`)
- **Input**: User query (ví dụ: "Xin chào", "Tôi muốn mua áo polo")
- **Output**: `fashion` | `chitchat` | `off_topic`
- **Tại sao cần LLM?**
  - Hiểu ngữ cảnh và ý định người dùng
  - Phân biệt giữa câu hỏi thời trang vs chào hỏi vs chủ đề khác
  - Rule-based không đủ linh hoạt

#### B. **Assistant Message Generation** (`generateAssistantMessage`)
- **Input**: User query + Intent + HasProducts
- **Output**: Câu trả lời tự nhiên bằng tiếng Việt
- **Tại sao cần LLM?**
  - Tạo câu trả lời tự nhiên, lịch sự
  - Tùy chỉnh theo ngữ cảnh (có/không có sản phẩm)
  - Đảm bảo chỉ nói về sản phẩm trong database

---

## 🔎 3. VAI TRÒ CỦA RETRIEVAL SYSTEM

### ✅ e5-large-v2 (Embedding Model) + FAISS

#### A. **Embedding Model (e5-large-v2)**
- **Chức năng**: Chuyển đổi text → vector (embedding)
- **Dimension**: 1024
- **Tại sao cần embedding?**
  - Text không thể so sánh trực tiếp
  - Vector có thể đo độ tương tự (cosine similarity, L2 distance)
  - Tìm sản phẩm tương tự dựa trên ngữ nghĩa

**Ví dụ:**
```
Query: "Polo for men"
↓ Embedding
Vector: [0.12, -0.45, 0.78, ..., 0.23] (1024 dimensions)

Product: "Men's Polo Shirt"
↓ Embedding  
Vector: [0.11, -0.44, 0.79, ..., 0.24] (1024 dimensions)

→ Cosine similarity ≈ 0.95 → Rất tương tự!
```

#### B. **FAISS (Facebook AI Similarity Search)**
- **Chức năng**: Tìm kiếm nhanh trong không gian vector
- **Cách hoạt động**:
  1. Index tất cả embeddings của sản phẩm
  2. Query embedding → tìm top-K sản phẩm gần nhất
  3. Trả về danh sách sản phẩm có embedding tương tự

**Ví dụ:**
```
Query embedding → FAISS search (k=500)
↓
Top 500 sản phẩm có embedding gần nhất
↓
Hard filtering (category, sex, color, price)
↓
Final results
```

---

## 🎯 4. SO SÁNH: EMBEDDING MODEL vs LLM

| Tiêu chí | Embedding Model (e5-large-v2) | LLM (GPT-4o-mini) |
|----------|------------------------------|-------------------|
| **Mục đích** | Tạo vector biểu diễn text | Hiểu và sinh text |
| **Input** | Text | Text + Context |
| **Output** | Vector (1024 dimensions) | Text (câu trả lời) |
| **Tốc độ** | Rất nhanh (~10ms) | Chậm hơn (~500ms-2s) |
| **Chi phí** | Miễn phí (local) | Trả phí (API) |
| **Sử dụng** | Tìm kiếm tương tự | Phân loại, sinh câu trả lời |
| **Ví dụ** | "Polo" → vector | "Polo" → "Bạn muốn tìm áo polo cho nam hay nữ?" |

---

## 🔄 5. QUY TRÌNH HOẠT ĐỘNG CHI TIẾT

### Bước 1: User Query
```
User: "Tôi muốn mua áo polo cho nam, màu đen, giá dưới $50"
```

### Bước 2: Intent Classification (LLM)
```
OpenAI GPT-4o-mini
Input: "Tôi muốn mua áo polo cho nam..."
Output: "fashion"
```

### Bước 3: Retrieval (Embedding + FAISS)
```
a. Parse constraints:
   - category: "polo"
   - sex: "men"
   - color: "black"
   - price: max $50

b. Build query text: "men polo black"

c. Embedding (e5-large-v2):
   Query → Vector [0.12, -0.45, ...]

d. FAISS Search:
   - Tìm top-500 sản phẩm gần nhất
   - Distance threshold filtering

e. Hard Filtering:
   - Category match: "polo"
   - Sex match: "men"
   - Color match: "black"
   - Price <= $50

f. Return: Top-N products
```

### Bước 4: Response Generation (LLM)
```
OpenAI GPT-4o-mini
Input: 
  - Intent: "fashion"
  - HasProducts: true
  - Query: "Tôi muốn mua áo polo..."
  
Output: "Mình đã tìm được 15 sản phẩm áo polo cho nam màu đen với giá dưới $50..."
```

---

## 💡 6. TẠI SAO CẦN CẢ 2 LOẠI MODEL?

### Embedding Model (e5-large-v2):
- ✅ **Tìm kiếm nhanh**: Tìm trong 9490 sản phẩm trong vài ms
- ✅ **Hiểu ngữ nghĩa**: "Polo" tìm được "Polo Shirt", "Men's Polo"
- ✅ **Không tốn phí**: Chạy local, không cần API
- ❌ **Không sinh text**: Chỉ tìm, không tạo câu trả lời

### LLM (GPT-4o-mini):
- ✅ **Hiểu ý định**: Phân biệt fashion vs chitchat
- ✅ **Sinh câu trả lời**: Tạo text tự nhiên, lịch sự
- ✅ **Linh hoạt**: Tùy chỉnh theo ngữ cảnh
- ❌ **Chậm và tốn phí**: Cần API call, ~500ms-2s
- ❌ **Không tìm kiếm**: Không thể search trong 9490 sản phẩm

---

## 🎓 7. KIẾN TRÚC RAG (Retrieval-Augmented Generation)

Hệ thống của bạn là một **RAG System** điển hình:

```
┌─────────────────────────────────────────┐
│         RETRIEVAL PHASE                  │
│  (Embedding Model + FAISS)              │
│  - Tìm sản phẩm liên quan               │
│  - Hard filtering                       │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         GENERATION PHASE                │
│  (LLM - GPT-4o-mini)                    │
│  - Sinh câu trả lời dựa trên            │
│    sản phẩm đã tìm được                 │
└─────────────────────────────────────────┘
```

**Lợi ích:**
- ✅ LLM không cần "nhớ" 9490 sản phẩm
- ✅ LLM chỉ sinh text dựa trên kết quả retrieval
- ✅ Đảm bảo chỉ nói về sản phẩm có trong database
- ✅ Kết hợp sức mạnh của cả 2 loại model

---

## 📊 8. TÓM TẮT VAI TRÒ

| Component | Model | Vai trò | Vị trí |
|-----------|-------|--------|--------|
| **Embedding** | e5-large-v2 | Tạo vector, tìm kiếm | `retrieval_service/main.py` |
| **Search** | FAISS | Tìm kiếm nhanh trong vector space | `retrieval_service/main.py` |
| **Intent** | GPT-4o-mini | Phân loại ý định | `app/api/search/route.ts` |
| **Generation** | GPT-4o-mini | Sinh câu trả lời | `app/api/search/route.ts` |

---

## 🚀 9. CẢI THIỆN TRONG TƯƠNG LAI

### Có thể thay thế GPT-4o-mini bằng Qwen:
- ✅ **Rẻ hơn**: Chạy local, không cần API
- ✅ **Nhanh hơn**: Không cần network call
- ✅ **Privacy**: Dữ liệu không gửi lên cloud
- ❌ **Cần GPU**: Qwen 3B cần GPU để chạy nhanh
- ❌ **Chất lượng**: Có thể kém hơn GPT-4o-mini

### Có thể dùng Qwen cho:
1. **Intent Classification**: Fine-tune Qwen để phân loại intent
2. **Response Generation**: Dùng Qwen để sinh câu trả lời
3. **Hybrid**: GPT cho intent, Qwen cho generation

---

## ❓ 10. CÂU HỎI THƯỜNG GẶP

**Q: Tại sao không dùng Qwen cho embedding?**
A: e5-large-v2 được thiết kế chuyên cho embedding, tốt hơn Qwen cho task này.

**Q: Có thể dùng Qwen thay GPT-4o-mini không?**
A: Có, nhưng cần fine-tune và có GPU để chạy nhanh.

**Q: Retrieval có thể dùng LLM không?**
A: Không hiệu quả. LLM không thể search trong 9490 sản phẩm nhanh như FAISS.

**Q: Tại sao cần cả embedding và LLM?**
A: Embedding tìm kiếm, LLM sinh text. Mỗi cái làm việc mình giỏi nhất.


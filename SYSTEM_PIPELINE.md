# 🎯 Hệ Thống CRS (Conversational Recommendation System) - Pipeline & Công Nghệ

## 📋 Tổng Quan

Hệ thống CRS là một **hệ thống tư vấn thời trang thông minh** sử dụng kiến trúc **RAG (Retrieval-Augmented Generation)** kết hợp:
- **Embedding Model** để tìm kiếm sản phẩm
- **LLM (Large Language Model)** để hiểu ý định và sinh câu trả lời
- **Vector Database (FAISS)** để tìm kiếm nhanh

---

## 🏗️ Kiến Trúc Tổng Thể

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Next.js)                        │
│  - React + TypeScript                                           │
│  - Chat Interface (components/chat-interface.tsx)                │
│  - Product Cards (components/product-card.tsx)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTP POST /api/search
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Next.js API)                    │
│  - app/api/search/route.ts                                      │
│  - Intent Classification (Qwen/GPT)                             │
│  - Response Generation (Qwen/GPT)                                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTP POST /search
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│              RETRIEVAL SERVICE (FastAPI - Python)               │
│  - retrieval_service/main.py                                    │
│  - Embedding: e5-large-v2 (SentenceTransformer)                │
│  - Vector Search: FAISS                                         │
│  - LLM: Qwen 3B (Fine-tuned with LoRA)                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Pipeline Chi Tiết

### **BƯỚC 1: User Query Input**
```
User: "Tôi muốn mua áo polo cho nam, màu đen, giá dưới $50"
     ↓
Frontend (chat-interface.tsx)
- Parse user input
- Maintain conversation context
- Manage filters (color, price, category, etc.)
```

**Công nghệ:**
- **React** - UI framework
- **TypeScript** - Type safety
- **State Management** - React hooks (useState, useRef)

---

### **BƯỚC 2: Intent Classification**
```
Query → /api/search (Next.js API)
     ↓
Intent Classification:
  - Try Qwen 3B (timeout: 3s)
  - Fallback: GPT-4o-mini (OpenAI API)
     ↓
Output: "fashion" | "chitchat" | "off_topic "
```

**Công nghệ:**
- **Qwen 3B** (Fine-tuned with LoRA) - Primary
  - Model: `Qwen/Qwen2.5-3B-Instruct`
  - LoRA Adapter: `epoch_2_final/`
  - Framework: `transformers` + `peft`
- **GPT-4o-mini** (OpenAI) - Fallback
  - API: `https://api.openai.com/v1/chat/completions`
  - Model: `gpt-4o-mini`

**Lý do:**
- Qwen: Chạy local, không tốn phí, privacy tốt
- GPT: Fallback khi Qwen chậm/lỗi, chất lượng cao

---

### **BƯỚC 3: Query Processing & Slot Extraction**
```
Intent = "fashion"
     ↓
Parse constraints from query:
  - category: "polo"
  - sex: "men"
  - color: "black"
  - price: max $50
     ↓
Build query text: "men polo black"
```

**Công nghệ:**
- **Natural Language Processing** - Rule-based parsing
- **Slot Filling** - Extract structured data from text

---

### **BƯỚC 4: Embedding Generation**
```
Query Text: "men polo black"
     ↓
Embedding Model: e5-large-v2
  - Framework: SentenceTransformer
  - Dimension: 1024
  - Normalization: L2 normalized
     ↓
Query Vector: [0.12, -0.45, 0.78, ..., 0.23] (1024 dims)
```

**Công nghệ:**
- **SentenceTransformer** - Python library
- **Model**: `intfloat/e5-large-v2`
- **Embedding Dimension**: 1024
- **Normalization**: L2 (cosine similarity)

**Lý do:**
- e5-large-v2 được thiết kế chuyên cho embedding
- Hiểu ngữ nghĩa tốt (semantic search)
- Tốc độ nhanh (~10ms per query)

---

### **BƯỚC 5: Vector Similarity Search (FAISS)**
```
Query Vector → FAISS Index (products.faiss)
  - Index Type: IndexFlatL2
  - Total Products: 9490
  - Search: Top-K nearest neighbors (k=500)
     ↓
Top-K Candidates: [product_123, product_456, ...]
  - Distance scores: [0.12, 0.15, 0.18, ...]
```

**Công nghệ:**
- **FAISS (Facebook AI Similarity Search)**
  - Library: `faiss-cpu` (CPU) / `faiss-gpu` (GPU)
  - Index Type: `IndexFlatL2` (exact search)
  - Algorithm: L2 distance (Euclidean)

**Lý do:**
- Tìm kiếm cực nhanh trong không gian vector
- Scalable cho hàng triệu vectors
- Hỗ trợ cả CPU và GPU

---

### **BƯỚC 6: Hard Filtering**
```
Top-K Candidates → Hard Filters
  - Category: "polo" (substring match)
  - Sex: "men"
  - Color: "black"
  - Price: <= $50
  - Rating: (optional)
     ↓
Filtered Results: [product_123, product_789, ...]
```

**Công nghệ:**
- **Rule-based Filtering** - Python logic
- **Fuzzy Matching** - Substring/word boundary matching
- **Direct Database Search** - Fallback if ANN returns too few

**Lý do:**
- ANN chỉ tìm kiếm theo embedding similarity
- Hard filters đảm bảo constraints chính xác
- Fallback đảm bảo recall cho broad queries

---

### **BƯỚC 7: Response Generation**
```
Filtered Results + Query + Intent
     ↓
Response Generation:
  - Try Qwen 3B (timeout: 10s)
  - Fallback: GPT-4o-mini
     ↓
Assistant Message: "Mình đã tìm được 15 sản phẩm áo polo cho nam màu đen với giá dưới $50..."
```

**Công nghệ:**
- **Qwen 3B** (Fine-tuned) - Primary
  - Generation: Greedy decoding (CPU) / Sampling (GPU)
  - Max tokens: 80
- **GPT-4o-mini** - Fallback
  - Max tokens: 120
  - Temperature: 0.5

**Lý do:**
- Qwen: Local, không tốn phí
- GPT: Chất lượng cao, fallback khi Qwen chậm

---

### **BƯỚC 8: Response Rendering**
```
Assistant Message + Products
     ↓
Frontend (chat-interface.tsx)
  - Display message
  - Render product cards
  - Pagination
  - Client-side filtering
```

**Công nghệ:**
- **React Components** - Reusable UI
- **Client-side State** - Candidate universe management
- **Pagination** - Efficient rendering

---

## 🛠️ Công Nghệ Chi Tiết

### **1. Frontend Stack**

| Công nghệ | Version | Vai trò |
|-----------|---------|---------|
| **Next.js** | 14+ | React framework, SSR/SSG |
| **React** | 18+ | UI library |
| **TypeScript** | 5+ | Type safety |
| **Tailwind CSS** | 3+ | Styling |
| **shadcn/ui** | Latest | UI components |

**Files:**
- `components/chat-interface.tsx` - Main chat UI
- `components/product-card.tsx` - Product display
- `app/api/search/route.ts` - API gateway

---

### **2. Backend Stack (Python)**

| Công nghệ | Version | Vai trò |
|-----------|---------|---------|
| **FastAPI** | 0.115.5 | Web framework, API server |
| **Uvicorn** | 0.32.0 | ASGI server |
| **SentenceTransformer** | 2.2.0+ | Embedding model wrapper |
| **FAISS** | 1.9.0 | Vector similarity search |
| **PyTorch** | 2.0.0+ | Deep learning framework |
| **Transformers** | 4.40.0+ | Hugging Face models |
| **PEFT** | 0.12.0+ | LoRA adapter loading |
| **NumPy** | <2.0.0 | Numerical computing |

**Files:**
- `retrieval_service/main.py` - Main service
- `retrieval_service/requirements.txt` - Dependencies

---

### **3. AI/ML Models**

#### **A. Embedding Model: e5-large-v2**
```
Model: intfloat/e5-large-v2
Type: Sentence Transformer
Dimension: 1024
Framework: SentenceTransformer
Location: Auto-downloaded from Hugging Face
```

**Vai trò:**
- Chuyển đổi text → vector (embedding)
- Semantic search (tìm kiếm theo ngữ nghĩa)
- Tốc độ: ~10ms per query

---

#### **B. LLM: Qwen 3B (Fine-tuned)**
```
Base Model: Qwen/Qwen2.5-3B-Instruct
Fine-tuning: LoRA (Low-Rank Adaptation)
Adapter: epoch_2_final/
Parameters: ~3B (base) + ~10M (LoRA)
Framework: transformers + peft
```

**Vai trò:**
- Intent classification
- Response generation
- Tốc độ: ~200-500ms (GPU) / ~2-5s (CPU)

**LoRA Adapter:**
- Location: `epoch_2_final/`
- Files:
  - `adapter_model.safetensors` - Weights
  - `adapter_config.json` - Config
  - `tokenizer.json` - Tokenizer

---

#### **C. LLM: GPT-4o-mini (Fallback)**
```
Provider: OpenAI
Model: gpt-4o-mini
API: https://api.openai.com/v1/chat/completions
Cost: Pay-per-use
```

**Vai trò:**
- Fallback khi Qwen chậm/lỗi
- Intent classification (backup)
- Response generation (backup)

---

### **4. Vector Database: FAISS**

```
Index Type: IndexFlatL2 (exact search)
Total Vectors: 9490 (products)
Dimension: 1024
File: index_retrival/products.faiss
Metadata: index_retrival/meta.pkl
```

**Vai trò:**
- Tìm kiếm nhanh trong không gian vector
- Top-K nearest neighbor search
- L2 distance calculation

**Build Process:**
1. Load all products
2. Generate embeddings (e5-large-v2)
3. Build FAISS index
4. Save index + metadata

---

## 📊 Data Flow Diagram

```
┌─────────────┐
│   User      │
│   Query     │
└──────┬──────┘
       │
       ↓
┌─────────────────────────────────────┐
│  Frontend (React/Next.js)           │
│  - Parse input                      │
│  - Manage context                   │
└──────┬──────────────────────────────┘
       │ HTTP POST /api/search
       ↓
┌─────────────────────────────────────┐
│  API Gateway (Next.js API)          │
│  ┌──────────────────────────────┐  │
│  │ Intent Classification         │  │
│  │ - Qwen 3B (3s timeout)        │  │
│  │ - GPT-4o-mini (fallback)      │  │
│  └──────────────┬─────────────────┘  │
│                 ↓                     │
│         Intent: "fashion"             │
└──────┬──────────────────────────────┘
       │ HTTP POST /search
       ↓
┌─────────────────────────────────────┐
│  Retrieval Service (FastAPI)        │
│  ┌──────────────────────────────┐  │
│  │ 1. Embedding (e5-large-v2)   │  │
│  │    Query → Vector (1024 dim) │  │
│  └──────────────┬─────────────────┘  │
│                 ↓                     │
│  ┌──────────────────────────────┐  │
│  │ 2. FAISS Search               │  │
│  │    Top-K candidates (k=500)   │  │
│  └──────────────┬─────────────────┘  │
│                 ↓                     │
│  ┌──────────────────────────────┐  │
│  │ 3. Hard Filtering            │  │
│  │    Category, Sex, Color, etc. │  │
│  └──────────────┬─────────────────┘  │
│                 ↓                     │
│         Filtered Products             │
└──────┬──────────────────────────────┘
       │ JSON Response
       ↓
┌─────────────────────────────────────┐
│  API Gateway (Next.js API)          │
│  ┌──────────────────────────────┐  │
│  │ Response Generation           │  │
│  │ - Qwen 3B (10s timeout)       │  │
│  │ - GPT-4o-mini (fallback)      │  │
│  └──────────────┬─────────────────┘  │
│                 ↓                     │
│    Assistant Message + Products      │
└──────┬──────────────────────────────┘
       │ JSON Response
       ↓
┌─────────────────────────────────────┐
│  Frontend (React/Next.js)           │
│  - Display message                  │
│  - Render product cards             │
│  - Pagination                       │
└─────────────────────────────────────┘
```

---

## 🔧 Cấu Hình & Dependencies

### **Frontend (package.json)**
```json
{
  "dependencies": {
    "next": "^14.x",
    "react": "^18.x",
    "typescript": "^5.x",
    "tailwindcss": "^3.x"
  }
}
```

### **Backend (requirements.txt)**
```
fastapi==0.115.5
uvicorn[standard]==0.32.0
faiss-cpu==1.9.0
numpy<2.0.0
torch>=2.0.0
sentence-transformers>=2.2.0
transformers>=4.40.0
peft>=0.12.0
accelerate>=0.30.0
safetensors>=0.4.0
scikit-learn>=1.3.0
```

---

## 🎯 Tại Sao Chọn Các Công Nghệ Này?

### **1. e5-large-v2 (Embedding)**
- ✅ **Chuyên cho embedding**: Được train riêng cho semantic search
- ✅ **Tốc độ nhanh**: ~10ms per query
- ✅ **Chất lượng cao**: Top-tier embedding model
- ✅ **Miễn phí**: Open-source, chạy local

### **2. FAISS (Vector Search)**
- ✅ **Cực nhanh**: Tìm kiếm trong 9490 vectors trong vài ms
- ✅ **Scalable**: Hỗ trợ hàng triệu vectors
- ✅ **Flexible**: Nhiều index types (Flat, IVF, HNSW)
- ✅ **Production-ready**: Được Facebook sử dụng

### **3. Qwen 3B (LLM)**
- ✅ **Local**: Không cần API, privacy tốt
- ✅ **Fine-tuned**: Đã được train cho task cụ thể
- ✅ **Nhỏ gọn**: 3B parameters, có thể chạy trên GPU consumer
- ✅ **Miễn phí**: Không tốn phí API

### **4. GPT-4o-mini (Fallback)**
- ✅ **Chất lượng cao**: OpenAI model, rất tốt
- ✅ **Reliable**: Luôn available, không cần GPU
- ✅ **Fast**: API call nhanh (~500ms)
- ⚠️ **Tốn phí**: Pay-per-use

### **5. FastAPI (Backend)**
- ✅ **Nhanh**: Performance cao, async support
- ✅ **Type-safe**: Pydantic models
- ✅ **Auto-docs**: Swagger UI tự động
- ✅ **Modern**: Python 3.10+ features

### **6. Next.js (Frontend)**
- ✅ **SSR/SSG**: SEO friendly, fast initial load
- ✅ **API Routes**: Built-in API gateway
- ✅ **TypeScript**: Type safety
- ✅ **React**: Component-based, reusable

---

## 📈 Performance Metrics

| Component | Latency | Throughput |
|-----------|---------|------------|
| **Embedding (e5-large-v2)** | ~10ms | ~100 queries/s |
| **FAISS Search (k=500)** | ~5ms | ~1000 queries/s |
| **Hard Filtering** | ~1-5ms | ~5000 items/s |
| **Qwen Intent (GPU)** | ~200-500ms | ~2-5 queries/s |
| **Qwen Intent (CPU)** | ~2-5s | ~0.2 queries/s |
| **Qwen Generate (GPU)** | ~300-600ms | ~2-3 queries/s |
| **Qwen Generate (CPU)** | ~5-10s | ~0.1 queries/s |
| **GPT-4o-mini (API)** | ~500-2000ms | ~10 queries/s |

---

## 🚀 Deployment Architecture

```
┌─────────────────────────────────────────┐
│         Production Server               │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │  Next.js (Port 3000)             │  │
│  │  - Frontend                      │  │
│  │  - API Gateway                   │  │
│  └──────────────┬───────────────────┘  │
│                 │                       │
│  ┌──────────────▼───────────────────┐  │
│  │  FastAPI (Port 8000)             │  │
│  │  - Retrieval Service             │  │
│  │  - Embedding Model               │  │
│  │  - FAISS Index                   │  │
│  │  - Qwen LLM                      │  │
│  └──────────────────────────────────┘  │
│                                         │
│  External:                              │
│  - OpenAI API (GPT-4o-mini)             │
│  - Hugging Face (model downloads)       │
└─────────────────────────────────────────┘
```

---

## 📝 Tóm Tắt

### **Core Technologies:**
1. **Embedding**: e5-large-v2 (SentenceTransformer)
2. **Vector Search**: FAISS
3. **LLM (Primary)**: Qwen 3B (Fine-tuned with LoRA)
4. **LLM (Fallback)**: GPT-4o-mini (OpenAI)
5. **Backend**: FastAPI (Python)
6. **Frontend**: Next.js + React + TypeScript

### **Architecture Pattern:**
- **RAG (Retrieval-Augmented Generation)**
- **Hybrid LLM**: Qwen (local) + GPT (cloud fallback)
- **Two-stage Search**: ANN (FAISS) + Hard Filtering

### **Key Features:**
- ✅ Semantic search (ngữ nghĩa)
- ✅ Hard filtering (constraints chính xác)
- ✅ Intent classification
- ✅ Natural language response
- ✅ Fallback mechanisms
- ✅ Client-side optimization

---

**Tài liệu này mô tả đầy đủ pipeline và công nghệ của hệ thống CRS.**

# 📚 Tài Liệu Hệ Thống - CRS & VTO

## 📋 Mục Lục

1. [Tổng Quan Hệ Thống](#tổng-quan-hệ-thống)
2. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
3. [Hệ Thống CRS (Conversational Recommendation System)](#hệ-thống-crs)
4. [Hệ Thống VTO (Virtual Try-On)](#hệ-thống-vto)
5. [Chi Tiết Kỹ Thuật](#chi-tiết-kỹ-thuật)
6. [Luồng Xử Lý Chi Tiết](#luồng-xử-lý-chi-tiết)
7. [API Reference](#api-reference)

---

## 🎯 Tổng Quan Hệ Thống

Hệ thống là một **nền tảng tư vấn thời trang thông minh** kết hợp hai tính năng chính:

1. **CRS (Conversational Recommendation System)**: Hệ thống tư vấn sản phẩm thời trang thông qua hội thoại tự nhiên
2. **VTO (Virtual Try-On)**: Tính năng thử đồ ảo cho phép người dùng xem sản phẩm trên dáng người của mình

### Công Nghệ Sử Dụng

- **Frontend**: Next.js 14, React, TypeScript, Tailwind CSS
- **Backend**: FastAPI (Python), Uvicorn
- **AI/ML**: 
  - Embedding Model: `intfloat/e5-large-v2` (SentenceTransformer)
  - LLM: Qwen 3B (Fine-tuned với LoRA) + GPT-4o-mini (Fallback)
  - Vector Database: FAISS
  - VTO: CatVTON qua Custom HuggingFace Space (Yuhdeptraico102/VTO) sử dụng Gradio Client API
- **Storage**: LocalStorage (conversation history), FAISS Index (product vectors)

---

## 🏗️ Kiến Trúc Tổng Thể

```
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND (Next.js)                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  components/chat-interface.tsx                            │  │
│  │  - Chat UI, Conversation Management                       │  │
│  │  - State Management (messages, context, filters)        │  │
│  │  - API Calls to /api/search                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  components/product-card.tsx                              │  │
│  │  - Product Display                                         │  │
│  │  - VTO Modal Trigger                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  components/vto-modal.tsx                                 │  │
│  │  - Image Upload (drag & drop)                             │  │
│  │  - VTO API Call                                            │  │
│  │  - Result Display                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  lib/conversation-storage.ts                              │  │
│  │  - LocalStorage Management                                │  │
│  │  - Conversation CRUD Operations                           │  │
│  │  - Real-time Updates (Custom Events)                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                             │ HTTP POST /api/search
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              API GATEWAY (Next.js API Route)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  app/api/search/route.ts                                  │  │
│  │  - Intent Classification (Qwen/GPT)                       │  │
│  │  - Attribute Change Detection (Qwen/GPT)                  │  │
│  │  - Response Generation (Qwen/GPT)                         │  │
│  │  - Forward to Retrieval Service                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                             │ HTTP POST /search
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│          RETRIEVAL SERVICE (FastAPI - Python)                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  retrieval_service/main.py                                │  │
│  │  - Embedding: e5-large-v2                                 │  │
│  │  - Vector Search: FAISS                                    │  │
│  │  - Hard Filtering (category, sex, color, price, etc.)   │  │
│  │  - LLM: Qwen 3B (Intent, Response, Attribute Change)     │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  retrieval_service/vto_service.py                          │  │
│  │  - Gradio Client Integration                             │  │
│  │  - Image Processing (Base64 ↔ PIL)                       │  │
│  │  - VTO API Call to HuggingFace Space                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                             │ HTTP API
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│    CUSTOM HUGGINGFACE SPACE (Yuhdeptraico102/VTO)              │
│  Space: Yuhdeptraico102/VTO                                     │
│  - Virtual Try-On Processing                                    │
│  - Returns Result Image                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💬 Hệ Thống CRS (Conversational Recommendation System)

### 1. Tổng Quan

CRS là hệ thống tư vấn sản phẩm thời trang sử dụng **kiến trúc RAG (Retrieval-Augmented Generation)**:

- **Retrieval**: Tìm kiếm sản phẩm dựa trên embedding và FAISS
- **Augmented**: Bổ sung context từ conversation history
- **Generation**: Sinh câu trả lời tự nhiên bằng LLM

### 2. Các Thành Phần Chính

#### A. Frontend Components

##### `components/chat-interface.tsx`
**Vai trò**: Component chính quản lý toàn bộ giao diện chat và logic tư vấn

**Chức năng chính**:
- **Conversation Management**: 
  - Tạo mới, load, save, delete conversations
  - Auto-save với debounce (1 giây)
  - Auto-title generation từ message đầu tiên
  - Real-time updates qua Custom Events
  
- **State Management**:
  - `messages`: Lịch sử tin nhắn (user + assistant)
  - `context`: Context hiện tại (baseQuery + filters)
  - `candidateUniverse`: Tập sản phẩm candidate từ lần search gần nhất
  - `conversationHistory`: Lịch sử để reference resolution
  - `pendingAttributeQuestion`: Theo dõi attribute đang được hỏi

- **Constraint Parsing** (`parseConstraints`):
  - Extract: category, sex, color, material, theme, price, rating
  - Support: "men polo", "black shirt under $50", "cotton dress", etc.
  - Negative constraints: "not red", "exclude leather", etc.

- **Client-side Filtering** (`applyClientFilters`):
  - Filter candidate universe locally khi user refine
  - Support positive và negative filters
  - Flexible matching (substring, word boundary)

- **Reference Resolution** (`resolveReference`):
  - Resolve "this color", "that material", "these themes"
  - Dựa trên conversation history

- **Attribute Change Detection**:
  - Gọi LLM để detect user muốn thay đổi attribute
  - Hỏi user về preference mới nếu detect được

##### `components/product-card.tsx`
**Vai trò**: Hiển thị thông tin sản phẩm và trigger VTO modal

**Chức năng**:
- Display: image, name, category, price, rating
- Actions: "Try On" (VTO), "Buy" (external link)

##### `components/conversation-sidebar.tsx`
**Vai trò**: Sidebar hiển thị lịch sử conversations

**Chức năng**:
- List all conversations (sorted by updatedAt)
- Create new conversation
- Switch between conversations
- Delete conversation (with confirmation)
- Real-time updates (listen to `conversationUpdated` event)

##### `lib/conversation-storage.ts`
**Vai trò**: Utilities quản lý conversation trong LocalStorage

**Functions**:
- `saveConversation()`: Save/update conversation
- `getAllConversations()`: Get all conversations
- `getConversation(id)`: Get specific conversation
- `deleteConversation(id)`: Delete conversation
- `createNewConversation()`: Create new conversation
- `generateConversationTitle()`: Generate title from first message
- `formatRelativeTime()`: Format time (e.g., "2 min ago")

**Storage Format**:
```typescript
interface Conversation {
  id: string
  title: string
  createdAt: Date
  updatedAt: Date
  messages: Message[]
  context?: {
    baseQuery?: string | null
    filters: Filters
  }
  candidateUniverse?: Product[]
}
```

#### B. API Gateway (`app/api/search/route.ts`)

**Vai trò**: Trung gian giữa frontend và retrieval service

**Endpoints**:
- `POST /api/search`: Main search endpoint
- `POST /api/search` (with `action: "classify_attribute_change"`): Attribute change detection

**Luồng xử lý**:

1. **Intent Classification** (`classifyIntent`):
   - Try Qwen 3B first (timeout: 3s)
   - Fallback to GPT-4o-mini nếu Qwen fail
   - Output: `"fashion" | "chitchat" | "off_topic" | "unknown"`

2. **Attribute Change Detection** (`classifyAttributeChange`):
   - Try Qwen 3B first (timeout: 8s)
   - Fallback to GPT-4o-mini
   - Output: `{ is_attribute_change: boolean, attribute_type: "color"|"material"|"theme"|null, action: "replace"|"exclude"|null }`

3. **Response Generation** (`generateAssistantMessage`):
   - Try Qwen 3B first (timeout: 20s)
   - Fallback to GPT-4o-mini
   - Input: query, intent, hasProducts
   - Output: Natural language response in Vietnamese

4. **Forward to Retrieval Service**:
   - Nếu intent là `"fashion"` hoặc `"unknown"` → gọi `/search`
   - Nếu intent là `"chitchat"` hoặc `"off_topic"` → chỉ trả response, không search

#### C. Retrieval Service (`retrieval_service/main.py`)

**Vai trò**: Backend service xử lý search và LLM tasks

**Endpoints**:
- `POST /search`: Product search với slots/filters
- `POST /qwen/intent`: Intent classification
- `POST /qwen/generate`: Response generation
- `POST /qwen/attribute-change`: Attribute change detection
- `POST /vto`: Virtual Try-On

**Models**:

1. **Embedding Model** (`e5-large-v2`):
   - Framework: SentenceTransformer
   - Dimension: 1024
   - Normalization: L2 (cosine similarity)
   - Loaded at startup

2. **FAISS Index**:
   - Type: `IndexFlatL2` (exact search)
   - Total vectors: ~9490 products
   - Loaded from `index_retrival/products.faiss`

3. **Metadata**:
   - Loaded from `index_retrival/meta.pkl`
   - Contains: Category, Sex, Color, Material, Theme, Price, Rating, Image, Link, etc.

4. **Qwen LLM**:
   - Base: `Qwen/Qwen2.5-3B-Instruct`
   - LoRA Adapter: `epoch_2_final/`
   - Framework: `transformers` + `peft`
   - Loaded at startup (lazy load nếu fail)

**Search Logic** (`/search` endpoint):

1. **Pure Text Search** (no slots):
   - Encode query → embedding
   - FAISS search (k=5)
   - Return top results

2. **Slots Search** (with filters):
   - Build query text từ slots (sex + colors + category)
   - Encode query → embedding
   - FAISS search (k=200-500, tùy filters)
   - **Hard Filtering** (`pass_filters`):
     - **Sex**: Flexible matching (men/male/man, women/female/woman)
     - **Category**: Substring matching (polo in "polo shirt")
     - **Color**: Check trong color pool (split by "|")
     - **Price**: Range check (min/max)
     - **Negative Filters**: Exclude colors, materials, themes
   - **Direct Search Fallback**: Nếu chỉ có category → scan toàn bộ metadata
   - Sort by distance
   - Return top N results

**Filtering Logic** (`pass_filters` function):

```python
def pass_filters(idx: int) -> bool:
    m = _meta[idx]
    
    # Sex matching (flexible)
    if want_sex:
        product_sex = _normalize_text(m.get("Sex", ""))
        # Map: men → [men, male, man]
        if product_sex not in want_sex_variations:
            return False
    
    # Category matching (bidirectional)
    if want_cat:
        cat = _normalize_text(m.get("Category"))
        # Check: exact match OR substring match (both directions)
        if not ((cat == want_cat) or (want_cat in cat) or (cat in want_cat)):
            return False
    
    # Color matching
    if want_colors:
        cpool = " ".join(_normalize_text(x) for x in str(m.get("Color", "")).split("|"))
        for need in want_colors:
            if need not in cpool:
                return False
    
    # Price range
    if pmin is not None and price < pmin:
        return False
    if pmax is not None and price > pmax:
        return False
    
    # Negative filters (exclusion)
    if exclude_colors:
        # Check if product has excluded color
        if exclude_color_norm in product_color_pool:
            return False
    
    # ... similar for exclude_mats, exclude_themes
    
    return True
```

**Qwen LLM Functions**:

1. **`_qwen_classify_intent(query)`**:
   - Prompt: System message + user query
   - Output: "fashion" | "chitchat" | "off_topic" | "unknown"
   - Max tokens: 4

2. **`_qwen_generate_response(query, intent, has_products)`**:
   - Prompt: System message (Vietnamese, fashion advisor) + query + intent + hasProducts
   - Output: Natural language response
   - Max tokens: 80
   - Temperature: 0.5 (if GPU), 0 (if CPU)

3. **`_qwen_classify_attribute_change(query, conversation_history)`**:
   - Prompt: System message (attribute change classifier) + query + history context
   - Output: JSON `{ is_attribute_change, attribute_type, action }`
   - Max tokens: 50

### 3. Luồng Xử Lý CRS

#### Luồng Tìm Kiếm Cơ Bản

```
User: "Show me men polo"
     ↓
[Frontend] parseConstraints()
  - category: "polo"
  - sex: "men"
     ↓
[Frontend] POST /api/search
  { q: "Show me men polo", slots: { category: "polo", sex: "men" } }
     ↓
[API Gateway] classifyIntent()
  - Try Qwen (3s timeout)
  - Fallback GPT-4o-mini
  - Result: "fashion"
     ↓
[API Gateway] POST /search (Retrieval Service)
  { q: "Show me men polo", slots: { category: "polo", sex: "men" } }
     ↓
[Retrieval Service] Search Logic
  1. Build query text: "men polo"
  2. Encode → embedding (1024 dim)
  3. FAISS search (k=500)
  4. Hard filtering:
     - Category: "polo" in product category
     - Sex: "men" matches product sex
  5. Direct search fallback (if needed)
  6. Sort by distance
  7. Return top 12 products
     ↓
[Retrieval Service] Response
  { items: [Product1, Product2, ...] }
     ↓
[API Gateway] generateAssistantMessage()
  - Try Qwen (20s timeout)
  - Fallback GPT-4o-mini
  - Result: "Mình đã tìm được 12 sản phẩm polo cho nam..."
     ↓
[API Gateway] Response
  { intent: "fashion", items: [...], assistant_message: "..." }
     ↓
[Frontend] Display
  - Show assistant message
  - Show product cards (paginated)
```

#### Luồng Refine Search (Client-side Filtering)

```
User: "Show me men polo"
  → System returns 200 products
     ↓
User: "black color"
     ↓
[Frontend] parseConstraints()
  - color: "black"
     ↓
[Frontend] Update context
  - Merge filters: { category: "polo", sex: "men", color: "black" }
     ↓
[Frontend] applyClientFilters()
  - Filter candidateUniverse locally
  - No API call needed!
     ↓
[Frontend] Display filtered products
```

#### Luồng Attribute Change

```
User: "Show me men polo"
  → System returns products
     ↓
User: "I don't like this color"
     ↓
[Frontend] detectAttributeChangeIntent()
  - POST /api/search { action: "classify_attribute_change", query: "..." }
     ↓
[API Gateway] classifyAttributeChange()
  - Try Qwen (8s timeout)
  - Result: { is_attribute_change: true, attribute_type: "color", action: "exclude" }
     ↓
[Frontend] generatePreferenceQuestion()
  - "Bạn có hứng thú với màu nào?"
     ↓
[Frontend] Display question
  - Set pendingAttributeQuestion = "color"
     ↓
User: "pink"
     ↓
[Frontend] Detect answer to pending question
  - Extract color: "pink"
  - Replace color in context
  - Update filters: { excludeColor: ["black"], color: "pink" }
     ↓
[Frontend] applyClientFilters()
  - Filter candidateUniverse
     ↓
[Frontend] Display filtered products
```

---

## 👔 Hệ Thống VTO (Virtual Try-On)

### 1. Tổng Quan

VTO cho phép người dùng thử đồ ảo bằng cách:
1. Chọn sản phẩm từ kết quả tìm kiếm
2. Upload ảnh dáng người (pose image)
3. Hệ thống xử lý và trả về ảnh kết quả (person wearing the clothing)

**Model được sử dụng**: 
- **HuggingFace Space**: `Yuhdeptraico102/VTO` (Custom Space của bạn)
- **Core Model**: **CatVTON** (bên trong Space)
- **Integration**: Gradio Client API
- **Lưu ý**: 
  - Đây là Space riêng của bạn được deploy trên HuggingFace
  - Bên trong Space này, hệ thống core sử dụng **CatVTON** để xử lý Virtual Try-On
  - CatVTON là model chuyên dụng cho việc thử đồ ảo với chất lượng cao

### 2. Kiến Trúc VTO

```
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  components/product-card.tsx                              │  │
│  │  - "Try On" button (Zap icon)                             │  │
│  │  - Opens VTOModal                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  components/vto-modal.tsx                                 │  │
│  │  - Image upload (drag & drop or click)                    │  │
│  │  - Validation (type, size)                                │  │
│  │  - Preview pose image                                     │  │
│  │  - POST /vto (Backend API)                               │  │
│  │  - Display result (3 images: original, product, result)  │  │
│  │  - Download result                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                             │ HTTP POST /vto
                             │ { person_image: base64, cloth_image: URL, product_id }
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              RETRIEVAL SERVICE (FastAPI)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  POST /vto                                                │  │
│  │  - Load person_image (base64 → PIL)                      │  │
│  │  - Load cloth_image (URL → PIL)                          │  │
│  │  - Call vto_service.run_virtual_tryon()                   │  │
│  │  - Return result_image (PIL → base64)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  retrieval_service/vto_service.py                          │  │
│  │  - _init_gradio_client(): Initialize Gradio Client       │  │
│  │  - _save_image_to_temp(): Save PIL → temp file           │  │
│  │  - run_virtual_tryon(): Main VTO function                 │  │
│  │    • Save images to temp files                           │  │
│  │    • Call Gradio Client API                              │  │
│  │    • Load result image                                    │  │
│  │    • Clean up temp files                                  │  │
│  │  - image_to_base64(): PIL → base64                        │  │
│  │  - base64_to_image(): base64 → PIL                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                             │ Gradio Client API
                             │ client.predict(person_file, cloth_file, api_name="/vto_interface")
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│    CUSTOM HUGGINGFACE SPACE (Yuhdeptraico102/VTO)               │
│  Space: Yuhdeptraico102/VTO                                      │
│  Core Model: CatVTON                                            │
│  - Receives: person image, cloth image                          │
│  - Processes: Virtual Try-On using CatVTON model                │
│  - Returns: Result image path (temporary file)                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Chi Tiết Implementation

#### A. Frontend (`components/vto-modal.tsx`)

**State Management**:
- `poseImage`: Base64 string của ảnh pose
- `posePreview`: Base64 string để preview
- `resultImage`: Base64 string của ảnh kết quả
- `isProcessing`: Loading state
- `error`: Error message

**Functions**:

1. **`handleFileSelect(e)`**:
   - Validate file type (image/*)
   - Validate file size (max 10MB)
   - Read file as base64
   - Set `poseImage` and `posePreview`

2. **`handleTryOn()`**:
   - Validate `poseImage` exists
   - POST `/vto` với:
     ```json
     {
       "person_image": "data:image/jpeg;base64,...",
       "cloth_image": "https://...",
       "product_id": "..."
     }
     ```
   - Handle response:
     ```json
     {
       "status": "success",
       "result_image": "data:image/png;base64,...",
       "message": "..."
     }
     ```
   - Set `resultImage` và display

3. **`handleRemovePose()`**:
   - Clear `poseImage`, `posePreview`, `resultImage`
   - Reset file input

4. **`handleDownload()`**:
   - Create download link từ `resultImage`
   - Trigger download

#### B. Backend (`retrieval_service/vto_service.py`)

**Configuration**:
```python
VTO_SPACE = "Yuhdeptraico102/VTO"
HF_TOKEN = os.environ.get("HF_TOKEN", "...")
```

**Global Variables**:
- `_client`: Gradio Client instance (singleton)
- `_client_initialized`: Flag để tránh re-initialize

**Functions**:

1. **`_init_gradio_client()`**:
   - Initialize Gradio Client (lazy load, singleton)
   - Suppress Unicode output (Windows compatibility)
   - Set `HF_TOKEN` environment variable
   - Connect to HuggingFace Space
   - Return client instance

2. **`_save_image_to_temp(image, prefix)`**:
   - Save PIL Image to temporary file
   - Use `tempfile.mkstemp()` để tránh file lock (Windows)
   - Return file path

3. **`_load_image_from_path(image_path)`**:
   - Load image from file path
   - Convert to RGB

4. **`run_virtual_tryon(person_image, cloth_image, ...)`**:
   - Main VTO function
   - Save images to temp files
   - Call Gradio Client API:
     ```python
     result = client.predict(
         handle_file(person_temp),
         handle_file(cloth_temp),
         api_name="/vto_interface"
     )
     ```
   - Load result image from path
   - Clean up temp files
   - Return PIL Image

5. **`image_to_base64(image, format)`**:
   - Convert PIL Image to base64 string
   - Format: `"data:image/png;base64,..."`

6. **`base64_to_image(base64_str)`**:
   - Convert base64 string to PIL Image
   - Handle data URL prefix

#### C. Backend Endpoint (`retrieval_service/main.py`)

**Endpoint**: `POST /vto`

**Request Model**:
```python
class VTORequest(BaseModel):
    person_image: str  # Base64 encoded image
    cloth_image: str   # Base64 encoded image or product image URL
    product_id: Optional[str] = None
```

**Response Model**:
```python
class VTOResponse(BaseModel):
    result_image: str  # Base64 encoded result image
    status: str        # "success" | "error"
    message: Optional[str] = None
```

**Handler Logic**:
1. Load `person_image` (base64 → PIL)
2. Load `cloth_image` (URL → PIL hoặc base64 → PIL)
3. Call `vto_service.run_virtual_tryon()`
4. Convert result to base64
5. Return response

### 4. Luồng Xử Lý VTO

```
User clicks "Try On" button on product card
     ↓
[Frontend] Open VTOModal
     ↓
User uploads pose image (drag & drop or click)
     ↓
[Frontend] handleFileSelect()
  - Validate file (type, size)
  - Read as base64
  - Set posePreview
     ↓
User clicks "Thử đồ ngay"
     ↓
[Frontend] handleTryOn()
  - POST http://127.0.0.1:8000/vto
  {
    person_image: "data:image/jpeg;base64,...",
    cloth_image: "https://...",
    product_id: "..."
  }
     ↓
[Backend] POST /vto
  - Load person_image (base64 → PIL)
  - Load cloth_image (URL → PIL)
  - Call vto_service.run_virtual_tryon()
     ↓
[vto_service] run_virtual_tryon()
  1. Initialize Gradio Client (if not already)
  2. Save images to temp files
  3. Call client.predict(person_file, cloth_file, api_name="/vto_interface")
     ↓
[HuggingFace Space] Process VTO
  - Run CatVTON model (core model trong Yuhdeptraico102/VTO)
  - Return result image path
     ↓
[vto_service] Load result image
  - Load from path
  - Clean up temp files
  - Return PIL Image
     ↓
[Backend] Convert to base64
  - image_to_base64(result_image)
     ↓
[Backend] Return response
  {
    status: "success",
    result_image: "data:image/png;base64,...",
    message: "Virtual try-on completed successfully"
  }
     ↓
[Frontend] Display result
  - Show 3 images: original pose, product, result
  - Enable download button
```

### 5. Xử Lý Lỗi

**Frontend**:
- File validation errors (type, size)
- API errors (network, server)
- Display error message trong modal

**Backend**:
- Image loading errors (invalid base64, URL not found)
- Gradio Client errors (connection, API)
- VTO processing errors
- Return error response với message

**Windows Compatibility**:
- File locking issues → Use `tempfile.mkstemp()` và close handle immediately
- Unicode encoding errors → Suppress stdout/stderr during client init

---

## 🔧 Chi Tiết Kỹ Thuật

### 1. Embedding Model (e5-large-v2)

**Model**: `intfloat/e5-large-v2`
- **Framework**: SentenceTransformer
- **Dimension**: 1024
- **Normalization**: L2 (cosine similarity)
- **Usage**: Encode query text và product metadata

**Loading**:
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("intfloat/e5-large-v2")
```

**Encoding**:
```python
embeddings = model.encode(
    texts,
    convert_to_numpy=True,
    normalize_embeddings=True,
    show_progress_bar=False
)
```

### 2. FAISS Index

**Type**: `IndexFlatL2` (exact search)
- **Algorithm**: L2 distance (Euclidean)
- **Total Vectors**: ~9490 products
- **Dimension**: 1024 (matches embedding dimension)

**Loading**:
```python
import faiss
index = faiss.read_index("index_retrival/products.faiss")
```

**Search**:
```python
D, I = index.search(query_embedding, k=500)
# D: distances, I: indices
```

### 3. Qwen LLM

**Base Model**: `Qwen/Qwen2.5-3B-Instruct`
- **Parameters**: ~3B (base) + ~10M (LoRA)
- **Framework**: `transformers` + `peft`
- **LoRA Adapter**: `epoch_2_final/`

**Loading**:
```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-3B-Instruct", trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-3B-Instruct",
    trust_remote_code=True,
    dtype=torch.float16 if use_cuda else torch.float32,
    device_map="auto" if use_cuda else None,
    low_cpu_mem_usage=True,
)
model = PeftModel.from_pretrained(model, "epoch_2_final/")
model.eval()
```

**Generation**:
```python
inputs = tokenizer(prompt, return_tensors="pt")
inputs = {k: v.to(device) for k, v in inputs.items()}
with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=80,
        do_sample=use_sampling,
        temperature=0.5 if use_sampling else None,
    )
response = tokenizer.decode(outputs[0], skip_special_tokens=True)
```

### 4. CatVTON Model (Core VTO Model)

**Model**: CatVTON (Category-aware Virtual Try-On)
- **Location**: Deployed trong HuggingFace Space `Yuhdeptraico102/VTO`
- **Framework**: Diffusion-based model
- **Features**: 
  - Category-aware clothing try-on
  - High-quality image generation
  - Handles various clothing types (shirts, pants, dresses, etc.)

**Architecture**:
- CatVTON là một model Virtual Try-On chuyên dụng
- Sử dụng diffusion model để tạo ảnh kết quả
- Có khả năng nhận biết category của quần áo để xử lý phù hợp
- Được deploy trong Space riêng của bạn để dễ dàng truy cập qua API

**Note**: Model CatVTON được chạy trên HuggingFace Space, không phải local. Điều này giúp:
- Không cần cài đặt model nặng trên server local
- Tận dụng GPU của HuggingFace
- Dễ dàng scale và maintain

### 5. Gradio Client

**Library**: `gradio-client>=0.15.0`
- **Space**: `Yuhdeptraico102/VTO` (chứa CatVTON model)
- **API**: `client.predict(person_file, cloth_file, api_name="/vto_interface")`

**Initialization**:
```python
from gradio_client import Client
client = Client("Yuhdeptraico102/VTO")
```

**Usage**:
```python
from gradio_client import handle_file
result = client.predict(
    handle_file(person_temp_path),
    handle_file(cloth_temp_path),
    api_name="/vto_interface"
)
```

**How it works**:
1. Gradio Client kết nối đến Space `Yuhdeptraico102/VTO`
2. Space này chứa CatVTON model đã được deploy sẵn
3. Client gửi ảnh person và cloth lên Space
4. CatVTON model xử lý và trả về ảnh kết quả
5. Client nhận được path của ảnh kết quả

### 5. LocalStorage

**Key**: `"fashion_chat_conversations"`
**Format**: JSON array of Conversation objects
**Max Size**: 50 conversations (prevent overflow)

**Real-time Updates**:
- Dispatch `conversationUpdated` Custom Event sau mỗi modification
- Sidebar listens và reload conversations

---

## 📊 Luồng Xử Lý Chi Tiết

### 1. User Query → Product Results

```
┌─────────────┐
│ User Query  │
│ "men polo"  │
└──────┬──────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: parseConstraints()   │
│ - category: "polo"             │
│ - sex: "men"                    │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: POST /api/search     │
│ { q: "men polo",               │
│   slots: { category: "polo",   │
│            sex: "men" } }     │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ API Gateway: classifyIntent() │
│ - Try Qwen (3s)                │
│ - Fallback GPT                 │
│ - Result: "fashion"            │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ API Gateway: POST /search      │
│ (Retrieval Service)            │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Retrieval: Build Query Text    │
│ "men polo"                      │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Retrieval: Encode → Embedding  │
│ e5-large-v2                     │
│ [0.12, -0.45, ..., 0.23]       │
│ (1024 dims)                     │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Retrieval: FAISS Search         │
│ k=500                           │
│ Top candidates                  │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Retrieval: Hard Filtering      │
│ - Category: "polo" match        │
│ - Sex: "men" match              │
│ Filtered products               │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Retrieval: Sort by Distance    │
│ Top 12 products                 │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ API Gateway: generateResponse() │
│ - Try Qwen (20s)                │
│ - Fallback GPT                  │
│ - Result: "Mình đã tìm được..." │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: Display               │
│ - Assistant message             │
│ - Product cards (paginated)      │
└─────────────────────────────────┘
```

### 2. VTO Processing Flow

```
┌─────────────┐
│ User Action │
│ Click "Try  │
│ On" button  │
└──────┬──────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: Open VTOModal         │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ User: Upload Pose Image         │
│ (drag & drop or click)          │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: handleFileSelect()    │
│ - Validate (type, size)         │
│ - Read as base64                │
│ - Set posePreview               │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ User: Click "Thử đồ ngay"      │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: POST /vto             │
│ { person_image: base64,         │
│   cloth_image: URL,             │
│   product_id: "..." }            │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Backend: Load Images             │
│ - person_image: base64 → PIL    │
│ - cloth_image: URL → PIL        │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ vto_service: Save to Temp       │
│ - person_temp = tempfile        │
│ - cloth_temp = tempfile         │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ vto_service: Gradio Client      │
│ client.predict(                 │
│   handle_file(person_temp),    │
│   handle_file(cloth_temp),     │
│   api_name="/vto_interface"    │
│ )                               │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ HuggingFace Space: Process      │
│ - CatVTON model                 │
│   (trong Yuhdeptraico102/VTO)  │
│ - Return result image path      │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ vto_service: Load Result        │
│ - Load from path                │
│ - Clean up temp files           │
│ - Return PIL Image              │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Backend: Convert to Base64      │
│ image_to_base64(result_image)   │
└──────┬──────────────────────────┘
       │
       ↓
┌─────────────────────────────────┐
│ Frontend: Display Result        │
│ - Original pose                 │
│ - Product                       │
│ - Result (person wearing)       │
└─────────────────────────────────┘
```

---

## 📡 API Reference

### Frontend → Backend

#### `POST /api/search`

**Request**:
```typescript
{
  q: string
  k?: number
  n?: number
  slots?: {
    category?: string
    sex?: string
    color?: string[]
    material?: string[]
    theme?: string[]
    price?: { min?: number, max?: number }
    rating?: { min?: number, max?: number }
    exclude_color?: string[]
    exclude_material?: string[]
    exclude_theme?: string[]
  }
}
```

**Response**:
```typescript
{
  intent: "fashion" | "chitchat" | "off_topic" | "unknown"
  items: Array<{
    score: number
    meta: {
      Category: string
      Sex: string
      Color: string
      Material: string
      Theme: string
      PriceNum: number
      RatingNum: number
      Image: string
      Link: string
      Title: string
      Name: string
      // ... other fields
    }
  }>
  assistant_message?: string
}
```

#### `POST /api/search` (Attribute Change Detection)

**Request**:
```typescript
{
  action: "classify_attribute_change"
  query: string
  conversation_history?: Array<{
    query: string
    timestamp: string
  }>
}
```

**Response**:
```typescript
{
  is_attribute_change: boolean
  attribute_type: "color" | "material" | "theme" | null
  action: "replace" | "exclude" | null
}
```

#### `POST /vto`

**Request**:
```typescript
{
  person_image: string  // Base64 encoded image
  cloth_image: string   // Base64 encoded image or product image URL
  product_id?: string
}
```

**Response**:
```typescript
{
  status: "success" | "error"
  result_image: string  // Base64 encoded result image
  message?: string
}
```

### Backend Internal APIs

#### `POST /search` (Retrieval Service)

**Request**:
```python
{
  "q": str,
  "k": int | None,
  "n": int | None,
  "slots": {
    "category": str | None,
    "sex": str | None,
    "color": List[str] | None,
    "material": List[str] | None,
    "theme": List[str] | None,
    "price": {"min": float | None, "max": float | None} | None,
    "rating": {"min": float | None, "max": float | None} | None,
    "exclude_color": List[str] | None,
    "exclude_material": List[str] | None,
    "exclude_theme": List[str] | None,
  } | None
}
```

**Response**:
```python
{
  "items": [
    {
      "score": float,
      "meta": {
        "Category": str,
        "Sex": str,
        "Color": str,
        "Material": str,
        "Theme": str,
        "PriceNum": float,
        "RatingNum": float,
        "Image": str,
        "Link": str,
        "Title": str,
        "Name": str,
        # ... other fields
      }
    }
  ]
}
```

#### `POST /qwen/intent`

**Request**:
```python
{
  "query": str
}
```

**Response**:
```python
{
  "intent": "fashion" | "chitchat" | "off_topic" | "unknown"
}
```

#### `POST /qwen/generate`

**Request**:
```python
{
  "query": str,
  "intent": str,
  "has_products": bool
}
```

**Response**:
```python
{
  "message": str
}
```

#### `POST /qwen/attribute-change`

**Request**:
```python
{
  "query": str,
  "conversation_history": List[Dict[str, Any]] | None
}
```

**Response**:
```python
{
  "is_attribute_change": bool,
  "attribute_type": "color" | "material" | "theme" | None,
  "action": "replace" | "exclude" | None
}
```

---

## 🔐 Environment Variables

### Frontend (.env.local)
```bash
# Optional: OpenAI API key for GPT fallback
OPENAI_API_KEY=sk-...

# Optional: Retrieval service URL (default: http://127.0.0.1:8000)
RETRIEVAL_API_URL=http://127.0.0.1:8000
```

### Backend (retrieval_service/.env hoặc system env)
```bash
# HuggingFace token for VTO Space
HF_TOKEN=hf_...

# Optional: Retrieval service port (default: 8000)
PORT=8000
```

---

## 📝 Notes

### Performance Optimizations

1. **Lazy Loading**: 
   - Qwen LLM loaded at startup (có thể fail gracefully)
   - Gradio Client initialized on first VTO request

2. **Caching**:
   - Candidate universe cached trong frontend
   - Client-side filtering để tránh API calls

3. **Timeouts**:
   - Qwen Intent: 3s
   - Qwen Attribute Change: 8s
   - Qwen Generate: 20s
   - GPT fallback: No timeout (rely on fetch timeout)

4. **Pagination**:
   - Products displayed 9 per page
   - Client-side pagination

### Error Handling

1. **Frontend**:
   - Try-catch cho API calls
   - Fallback messages nếu LLM fail
   - Error display trong UI

2. **Backend**:
   - Graceful degradation nếu Qwen fail
   - GPT fallback cho LLM tasks
   - Error logging với traceback

### Windows Compatibility

1. **File Locking**:
   - Use `tempfile.mkstemp()` và close handle immediately
   - Wait 0.1s before deleting temp files

2. **Unicode Encoding**:
   - Suppress stdout/stderr during Gradio Client init
   - Use `contextlib.redirect_stdout` và `redirect_stderr`

---

## 🚀 Deployment

### Frontend
```bash
cd /path/to/project
pnpm install
pnpm build
pnpm start
```

### Backend
```bash
cd retrieval_service
pip install -r requirements.txt
python main.py
# hoặc
uvicorn main:app --host 0.0.0.0 --port 8000
```

### Environment Setup
1. Set `HF_TOKEN` environment variable
2. Set `OPENAI_API_KEY` (optional, for GPT fallback)
3. Ensure `index_retrival/` directory exists với `meta.pkl` và `products.faiss`
4. Ensure `epoch_2_final/` directory exists với LoRA adapter

---

## 📚 Tài Liệu Tham Khảo

- [Next.js Documentation](https://nextjs.org/docs)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [FAISS Documentation](https://github.com/facebookresearch/faiss)
- [SentenceTransformers Documentation](https://www.sbert.net/)
- [Gradio Client Documentation](https://www.gradio.app/docs/gradio-client)
- [Qwen Documentation](https://github.com/QwenLM/Qwen)

---

**Tài liệu này được tạo tự động dựa trên codebase hiện tại.**
**Cập nhật lần cuối: 2025-12-06**


# Hướng dẫn Setup và Test với Giao diện

## Bước 1: Cài đặt Dependencies

### Frontend (Next.js):
```bash
pnpm install
```

### Backend (Retrieval Service):
```bash
cd retrieval_service
pip install -r requirements.txt
cd ..
```

## Bước 2: Kiểm tra Files cần thiết

Đảm bảo các file sau tồn tại:

- ✅ `epoch_2_final/` - Thư mục chứa Qwen LoRA adapter
- ✅ `index_retrival/products.faiss` - FAISS index đã build với Qwen 3B
- ✅ `index_retrival/meta.pkl` - Metadata của products

## Bước 3: Start cả Frontend và Backend

Chạy lệnh sau để start cả hai service cùng lúc:

```bash
pnpm dev
```

Lệnh này sẽ:
- Start Next.js frontend tại `http://localhost:3000`
- Start Python retrieval service tại `http://127.0.0.1:8000`

**Lưu ý quan trọng:**
- Lần đầu tiên, retrieval service cần download Qwen model từ HuggingFace và load LoRA adapter
- Quá trình này có thể mất **3-5 phút** tùy vào tốc độ internet và máy tính
- Bạn sẽ thấy log trong terminal cho biết quá trình loading

## Bước 4: Test trên Giao diện

### 4.1. Mở trình duyệt

Truy cập: **http://localhost:3000**

### 4.2. Test Chat Interface

1. Nhập query vào ô chat, ví dụ:
   - "jeans"
   - "red dress"
   - "leather jacket for men"
   - "blue jeans under 100 dollars"

2. Nhấn Enter hoặc click nút Send

3. Xem kết quả:
   - Sản phẩm được hiển thị dưới dạng cards
   - Mỗi card hiển thị thông tin: title, price, rating, category, etc.
   - Assistant message (nếu có) sẽ hiển thị ở trên

### 4.3. Test Search Page

Truy cập: **http://localhost:3000/search**

- Nhập query vào ô search
- Chọn số lượng kết quả (k)
- Click "Search"
- Xem danh sách kết quả với scores

## Bước 5: Kiểm tra Service Status

### Kiểm tra Retrieval Service

Mở trình duyệt: **http://127.0.0.1:8000/docs**

Đây là Swagger UI của FastAPI, bạn có thể:
- Xem tất cả endpoints
- Test trực tiếp API
- Xem request/response schemas

### Kiểm tra Logs

Trong terminal chạy `pnpm dev`, bạn sẽ thấy:
- **web**: Logs từ Next.js (màu xanh lá)
- **py**: Logs từ Python service (màu cyan)

## Troubleshooting

### ❌ Lỗi OpenMP (OMP: Error #15)

**Triệu chứng:**
```
OMP: Error #15: Initializing libomp140.x86_64.dll, but found libiomp5md.dll already initialized.
```

**Giải pháp:**
- ✅ **Đã được fix tự động** trong code (biến môi trường `KMP_DUPLICATE_LIB_OK=TRUE` đã được set)
- Nếu vẫn gặp lỗi, thử set thủ công trong PowerShell:
  ```powershell
  $env:KMP_DUPLICATE_LIB_OK="TRUE"
  pnpm dev
  ```

### ❌ Frontend không kết nối được với Backend

**Triệu chứng:** 
- Chat interface hiển thị lỗi
- Không có kết quả search
- Error: `ECONNRESET` hoặc `fetch failed`

**Giải pháp:**
1. Kiểm tra retrieval service có đang chạy không: http://127.0.0.1:8000/docs
2. Kiểm tra biến môi trường `RETRIEVAL_API_URL` (nếu có)
3. Xem logs trong terminal để tìm lỗi
4. Nếu thấy lỗi OpenMP, service có thể đã crash - restart lại `pnpm dev`

### ❌ Service trả về 503 (Service Unavailable)

**Triệu chứng:**
- API trả về status 503
- Không có kết quả

**Giải pháp:**
- Model đang được load, đợi thêm vài phút
- Kiểm tra RAM/VRAM có đủ không (Qwen 3B cần ~6GB RAM)
- Xem logs trong terminal để biết tiến trình loading

### ❌ Lỗi "Missing epoch_2_final directory"

**Giải pháp:**
- Đảm bảo thư mục `epoch_2_final` có trong project root
- Kiểm tra đường dẫn: `E:\CURSOR\code\epoch_2_final`

### ❌ Lỗi "Missing products.faiss"

**Giải pháp:**
- Đảm bảo file `index_retrival/products.faiss` tồn tại
- Nếu chưa có, bạn cần build index trước:
  ```bash
  cd retrieval_service
  python build_index.py
  ```

### ❌ Python dependencies chưa được cài

**Giải pháp:**
```bash
cd retrieval_service
pip install -r requirements.txt
```

## Test Cases để thử

### Text Queries:
- ✅ "jeans"
- ✅ "red dress"
- ✅ "leather jacket"
- ✅ "blue sneakers"
- ✅ "winter coat"

### Slots Queries (qua chat interface):
- ✅ "I want jeans for men"
- ✅ "Show me red dresses"
- ✅ "blue jeans under 100 dollars"
- ✅ "leather jacket for women"

### Edge Cases:
- ✅ "hello" (chitchat - không tìm sản phẩm)
- ✅ "laptop" (off_topic - không tìm sản phẩm)
- ✅ "" (empty query - should fail)

## Performance Notes

- **First load**: 3-5 phút (download model + load adapter)
- **Subsequent loads**: 10-30 giây (chỉ load từ cache)
- **Query time**: < 1 giây (sau khi model đã load)

## Next Steps

Sau khi test thành công:
1. ✅ Service hoạt động đúng với Qwen 3B
2. ✅ Frontend kết nối được với backend
3. ✅ Search results hiển thị đúng
4. ✅ Chat interface hoạt động mượt mà

Bạn có thể tiếp tục phát triển các tính năng khác!


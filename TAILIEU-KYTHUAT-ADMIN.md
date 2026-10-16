# Tài liệu kỹ thuật Backend và Quản trị

## 1. Kiến trúc Backend

Backend là ứng dụng FastAPI chạy trên Python. Các route xác thực request bằng Pydantic, chọn chiến lược theo platform, gọi `LLMGateway`, kiểm tra policy và điều phối thao tác persistence. Cấu hình được tập trung trong `backend/config.py` bằng `pydantic_settings.BaseSettings`.

```text
HTTP Request
   → FastAPI / Pydantic validation
   → Platform Strategy
   → LLMGateway (timeout / retry / breaker / cache)
   → Strategy.parse_output
   → PolicyChecker
   → Supabase save (chỉ gemini/mock)
   → HTTP Response
```

## 2. Module và trách nhiệm

| Module | Trách nhiệm |
|---|---|
| `backend/main.py` | FastAPI lifecycle, CORS, request/response schema, dependency và điều phối route |
| `backend/config.py` | Đọc `.env`/environment, parse settings và bảo vệ secret |
| `backend/services/llm_gateway.py` | Gọi Gemini/mock, retry, cache, graceful degradation |
| `backend/services/circuit_breaker.py` | Trạng thái closed/open/half-open và giới hạn probe |
| `backend/strategies/` | Prompt và nhiệt độ theo kênh |
| `backend/verifiers/policy_checker.py` | Quét regex và trả warning/flagged terms |
| `backend/database/supabase_client.py` | Khởi tạo client và thao tác ghi bảng Supabase |
| `supabase/migrations/` | Schema migration có thể triển khai lặp lại theo môi trường |

Gateway không chứa logic database. Database module không xây prompt hoặc gọi LLM. Route kết nối các thành phần qua các interface hẹp.

## 3. Strategy Pattern

`BaseStrategy` là abstract base class:

- `build_prompt(product, desc) -> str`: tạo prompt riêng cho nền tảng.
- `parse_output(output) -> str`: chuẩn hóa đầu ra; implementation mặc định trim khoảng trắng đầu/cuối.

| Implementation | Prompt | Nhiệt độ |
|---|---|---:|
| `TikTokStrategy` | Hook 3 giây, visual cue, lời thoại và CTA | 0.9 |
| `FacebookStrategy` | AIDA, storytelling, CTA | 0.8 |
| `ShopeeStrategy` | Từ khóa SEO, 4–6 bullet, hashtag | 0.3 |

Registry trong `main.py` map giá trị request `platform` tới strategy. Thêm kênh mới cần class mới, đăng ký strategy, mở rộng Literal/schema và test tương ứng; không thêm nhánh prompt vào LLM gateway.

## 4. Resilience Gateway

### HTTP timeout

`LLMGateway` khởi tạo `httpx.AsyncClient` với `httpx.Timeout(10.0)`. Timeout này áp dụng cho request gateway tới upstream; không thay thế timeout tổng thể toàn ứng dụng hay giới hạn thời gian của thao tác database.

### Retry và exponential backoff với jitter

- Retry giới hạn bởi `retry_attempts` (mặc định 3).
- HTTP 429 và HTTP 5xx được xem là tạm thời; lỗi kết nối/request cũng được retry.
- Các lỗi HTTP khác không được retry tự động.
- Delay mỗi lần retry được lấy ngẫu nhiên trong `[0, min(max_delay, base_delay × 2^attempt)]`.
- Cấu hình nằm trong Pydantic settings:
  - `RETRY_ATTEMPTS`
  - `RETRY_BASE_DELAY_SECONDS`
  - `RETRY_MAX_DELAY_SECONDS`

### Circuit breaker

- Trạng thái: `CLOSED`, `OPEN`, `HALF_OPEN`.
- Sau 5 operation gateway thất bại liên tiếp, breaker mở.
- Trong OPEN, request được bỏ qua upstream và chuyển sang fallback.
- Sau 60 giây, một request được phép làm probe ở HALF_OPEN.
- Probe thành công đóng circuit; probe thất bại mở lại và bắt đầu lại thời gian phục hồi.
- Cấu hình: `CIRCUIT_FAILURE_THRESHOLD`, `CIRCUIT_RECOVERY_SECONDS`.

Lưu ý: một “failure” ở breaker là một operation gateway thất bại sau khi vòng retry hữu hạn của operation đó đã hoàn tất, không phải mỗi lần HTTP retry riêng lẻ.

### Cache và fallback

- Cache là dictionary trong process, khóa theo hash của prompt và temperature.
- TTL dữ liệu fresh mặc định 600 giây (10 phút).
- Entry hết TTL không được trả như cache bình thường, nhưng được giữ thêm một thời gian giới hạn (hiện tối thiểu 24 giờ) nhằm phục vụ stale fallback khi gateway lỗi/breaker mở.
- Nếu không có stale cache phù hợp, gateway trả nội dung mẫu và nguồn `sample`.
- Cache chỉ nằm trong bộ nhớ của một process; không chia sẻ giữa worker hoặc instance.
- Cấu hình fresh TTL bằng `CACHE_TTL_SECONDS`.

Response source:

| Source | Ý nghĩa |
|---|---|
| `gemini` | Nội dung mới từ Gemini |
| `mock` | Nội dung từ mock mode/mock endpoint |
| `cache` | Nội dung cũ dùng làm graceful degradation |
| `sample` | Bản mẫu dùng khi không có cache phù hợp |

## 5. Policy Checker

`PolicyChecker` dùng regex để phát hiện `100%`, `cam kết`, `số 1`, `trị dứt điểm`. Kết quả giữ nguyên text và bổ sung:

- `warnings`: thông báo rà soát.
- `flagged_terms`: danh sách nhãn cụm từ khớp.

Checker hiện không tự sửa nội dung, không xác minh bằng chứng quảng cáo, và không bao phủ toàn bộ quy định. Mọi thay đổi regex cần kiểm thử Unicode, chữ hoa/thường, dấu câu và false positives.

## 6. Tích hợp Supabase

### Migration

Migration chính: `supabase/migrations/001_initial_schema.sql`.

Bảng `public.generated_contents`:

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID | Primary key, mặc định `gen_random_uuid()` |
| `product_name` | TEXT | Tên sản phẩm |
| `platform` | TEXT | Kênh nội dung |
| `generated_text` | TEXT | Bản nháp sau parse và policy audit |
| `created_at` | TIMESTAMPTZ | Mặc định `now()` |

Migration tắt RLS theo yêu cầu hiện tại, nhưng thu hồi quyền `anon`/`authenticated` trên bảng và chỉ cấp quyền cho `service_role`. Vì RLS bị tắt, grants là biện pháp thiết yếu: phải xác nhận chúng được áp dụng chính xác trên project trước khi triển khai.

### Client và lưu dữ liệu

`backend/database/supabase_client.py`:

- `initialize_supabase_client(settings)` yêu cầu URL và service-role key; thiếu credential sẽ ném lỗi cấu hình rõ ràng.
- `save_generated_content(product_name, platform, generated_text, client=...)` insert vào `generated_contents` và trả response SDK.
- Lỗi Supabase/PostgREST hoặc HTTP được bọc thành `SupabasePersistenceError`.
- API gọi hàm sync này qua `asyncio.to_thread` để không chặn event loop.

Endpoint chỉ persist khi source là `gemini` hoặc `mock`. `cache`/`sample` không được ghi như nội dung vừa tạo. Nếu lưu thất bại, route trả HTTP 503 thay vì success response.

### Cấu hình secret

Pydantic settings đọc `.env` ở root dự án và thư mục cha của project:

| Biến | Phạm vi |
|---|---|
| `SUPABASE_URL` | Backend |
| `SUPABASE_SERVICE_ROLE_KEY` | Backend secret; quyền cao, không bao giờ đưa vào frontend |
| `GEMINI_API_KEY` | Backend secret |
| `GEMINI_MODEL` | Backend |
| `VITE_SUPABASE_URL` | Public frontend setting; chưa dùng trong luồng hiện tại |
| `VITE_SUPABASE_ANON_KEY` | Public/anon frontend setting; chưa dùng trong luồng hiện tại |

`.env.example` là mẫu không chứa giá trị production. Không commit `.env`. Khóa service role cần được lưu trong secret manager ở môi trường triển khai.

## 7. Khởi chạy và vận hành

### Backend local

```powershell
cd backend
.\venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

`GET /health` báo trạng thái breaker và database là `configured`/`unconfigured`. Trạng thái `configured` chỉ xác nhận client khởi tạo từ cấu hình, không phải health check toàn diện tới Postgres.

### Migration

Đưa migration vào Supabase CLI hoặc chạy trong SQL Editor theo quy trình đã phê duyệt. Trước khi production:

1. Kiểm tra URL/ref đúng môi trường.
2. Kiểm tra policy/grants và xác nhận anon/authenticated không thể ghi bảng.
3. Sao lưu/đánh giá dữ liệu hiện có trước khi thay đổi schema.
4. Chạy smoke test với bản ghi thử, sau đó xóa dữ liệu test theo quy trình.
5. Không gửi service-role key trong request từ frontend.

## 8. Xử lý lỗi và logging

- Lỗi upstream được chuyển thành `GeminiGatewayError`; log không chứa API key hoặc payload nhạy cảm.
- Lỗi cấu hình Supabase hiển thị thông báo hành động được nhưng không lộ giá trị credential.
- Lỗi persistence trả 503; cần log chi tiết ở server với correlation/request ID nếu bổ sung observability, đồng thời bảo đảm không ghi secrets.
- Không dùng bare `except` để tiếp tục giả vờ thành công.

## 9. Checklist triển khai

- [ ] Môi trường đã cài `backend/requirements.txt`.
- [ ] `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` và Gemini key được cấu hình qua secret manager.
- [ ] Migration đã chạy đúng project.
- [ ] Grants/RLS đã được rà soát.
- [ ] `GET /health` trả database `configured`.
- [ ] Test request có source `mock`/`gemini` tạo đúng một bản ghi.
- [ ] Test source `cache`/`sample` không tạo bản ghi.
- [ ] Test lỗi Supabase trả 503 và không lộ secrets.

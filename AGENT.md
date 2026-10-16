# AGENT.md — Quy tắc làm việc cho AI IDE

## ROLE

Bạn là kỹ sư phần mềm cấp cao làm việc trên SmartCopy AI, một ứng dụng tạo nội dung thương mại điện tử đa kênh. Ưu tiên tính đúng đắn, bảo mật, khả năng bảo trì, khả năng truy cập và kiểm chứng bằng mã nguồn hiện có. Trước khi sửa đổi, đọc luồng liên quan và tôn trọng thiết kế hiện tại; không tự giả định rằng tính năng được mô tả trong tài liệu đã được triển khai nếu chưa kiểm tra mã nguồn.

## Quy tắc chung

1. Thực hiện thay đổi hẹp, đầy đủ và có liên quan trực tiếp đến yêu cầu.
2. Không commit, in ra hoặc đưa secrets vào source, frontend bundle, lỗi HTTP, log hay ví dụ tài liệu.
3. Giữ tương thích API trừ khi yêu cầu chỉ rõ thay đổi; nếu đổi request/response, cập nhật frontend và tài liệu liên quan.
4. Dùng tên biến, cấu trúc module và quy ước của repository. Tránh tạo lớp hoặc abstraction không cần thiết.
5. Chạy kiểm tra nhỏ nhất nhưng có khả năng bắt lỗi của thay đổi: test backend liên quan, `npm run build`, `npm run lint`, hoặc kiểm tra migration.
6. Không tuyên bố đã xác minh kết nối thật tới Gemini/Supabase nếu chỉ chạy mock hoặc test bằng fake client.

## Quy tắc Backend

- API mới phải dùng **FastAPI**; schema request/response và cấu hình phải có kiểu rõ ràng.
- Đọc biến môi trường qua `pydantic_settings.BaseSettings` trong `backend/config.py`. Dùng `SecretStr` cho khóa Gemini và Supabase.
- Ưu tiên Supabase client/repository hiện có cho persistence. Không thêm SQLite làm database thay thế hoặc SQLAlchemy song song nếu không có yêu cầu kiến trúc cụ thể.
- Tuân thủ SRP: gateway gọi LLM; strategy tạo prompt; policy checker rà soát; module database/repository ghi dữ liệu; route điều phối các lớp.
- Thao tác mạng/DB lỗi phải được xử lý có chủ đích. Bắt ngoại lệ cụ thể khi có thể, ghi log an toàn ở ranh giới phù hợp và trả lỗi HTTP rõ ràng. Không dùng `except Exception: pass`, không trả nội dung thành công giả khi thao tác bắt buộc thất bại.
- Không trả exception nội bộ, request headers, URL có secrets, API key hoặc toàn bộ payload nhạy cảm cho người dùng.
- Các lời gọi I/O đồng bộ từ SDK không đồng bộ phải tránh chặn event loop (ví dụ `asyncio.to_thread`) hoặc chuyển sang client async phù hợp.
- Duy trì timeout của gateway, retry exponential backoff với jitter cho lỗi tạm thời và circuit breaker. Không retry vô hạn hoặc retry mọi lỗi 4xx.
- Không lưu kết quả cache/stale/sample như một lần sinh mới. Giữ nguồn dữ liệu rõ ràng trong response.
- Thay đổi schema phải đi cùng migration, đúng tên cột/thời gian, quyền truy cập và cập nhật tài liệu.

## Quy tắc Frontend

- Form phải là controlled components: state là nguồn dữ liệu duy nhất, có xử lý submit/validation rõ ràng.
- Dùng semantic HTML (`form`, `fieldset`, `legend`, `label`, `button`, `section`, headings). Mỗi `input`/`textarea` cần nhãn có liên kết đúng (`htmlFor`/`id`, hoặc label bao quanh hợp lệ).
- Trường lỗi cần `aria-invalid="true"` khi sai, thông báo lỗi có `role="alert"` và được liên kết bằng `aria-describedby` nếu phù hợp.
- Dùng button thật cho hành động; không dùng `div onClick`.
- Bảo đảm thao tác được bằng bàn phím, focus dễ nhìn, màu sắc có tương phản và trạng thái loading/error/success được thông báo phù hợp.
- Không gửi Gemini key, Supabase service role key hay bí mật server vào browser. Các biến `VITE_*` là dữ liệu công khai sau build.
- Tái sử dụng Tailwind và components hiện có; dùng lucide-react với biểu tượng trang trí được đánh dấu `aria-hidden`.

## Quy tắc Kiến trúc

### Strategy Pattern

- Giữ `BaseStrategy` làm hợp đồng cho `build_prompt(product, desc)` và `parse_output()`.
- Chiến lược TikTok, Facebook, Shopee giữ logic prompt riêng và cấu hình nhiệt độ tương ứng.
- Chọn strategy trong tầng điều phối/API; không đưa prompt nền tảng vào LLM gateway.

### Resilience Gateway

- Mọi yêu cầu Gemini phải đi qua backend `LLMGateway`; frontend không gọi provider trực tiếp.
- Giữ timeout HTTP 10 giây, retry hữu hạn có backoff+jitter, breaker mở sau 5 lỗi với 60 giây hồi phục và TTL cache 10 phút.
- Fallback phải xác định nguồn (`cache`, `sample`, `mock`, `gemini`) và không được che giấu lỗi persistence bắt buộc.

### Supabase và bảo mật

- Dùng `SUPABASE_SERVICE_ROLE_KEY` chỉ ở backend; frontend chỉ có thể dùng public/anon key nếu cần.
- Service-role key có quyền cao, do đó chỉ dùng server-side và giới hạn phạm vi cấp phát, lưu trữ.
- Migration hiện tắt RLS cho `generated_contents`; đồng thời quyền bảng phải giới hạn cho `service_role`, không cấp cho `anon`/`authenticated`.
- Không thay đổi chính sách RLS hoặc grants mà không đánh giá rủi ro và cập nhật migration/tài liệu.

### Policy Checker

- Checker hiện cảnh báo các cụm từ qua regex; không khẳng định rằng checker tự sửa nội dung hoặc bao phủ toàn bộ luật/quy định.
- Khi thay đổi danh sách, thêm test cho biến thể chữ hoa/thường, dấu tiếng Việt, dấu câu và false positives.

## Checklist trước khi hoàn tất

- [ ] Đã đọc code hiện có của các module bị ảnh hưởng.
- [ ] Không làm lộ secret hoặc thay đổi file `.env` ngoài yêu cầu rõ ràng.
- [ ] Các trạng thái stale/sample và lỗi persistence vẫn được phân biệt.
- [ ] Form mới vẫn có labels, error roles và trạng thái bàn phím.
- [ ] Đã chạy kiểm tra phù hợp và ghi rõ điều gì chưa thể xác minh.

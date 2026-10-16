# Tài liệu nghiệp vụ — SmartCopy AI

## 1. Tổng quan nghiệp vụ

SmartCopy AI hỗ trợ người bán tạo bản nháp nội dung sản phẩm theo định dạng quen thuộc của từng kênh. Người bán cung cấp tên, mô tả và nền tảng đích; hệ thống chọn chiến lược tương ứng, tạo nội dung, rà soát một số cụm từ quảng cáo có rủi ro và lưu kết quả mới vào Supabase.

Mục tiêu là giảm thời gian viết và tăng tính nhất quán giữa các kênh. SmartCopy AI không thay người bán xác minh chất lượng sản phẩm, quyền sử dụng hình ảnh, tính chính xác của tuyên bố hay việc tuân thủ toàn bộ chính sách nền tảng.

## 2. Vấn đề bài toán

### Nỗi đau của người bán

- Một sản phẩm cần nhiều phiên bản nội dung, mỗi kênh có cách trình bày khác nhau.
- Viết thủ công tốn thời gian, khó duy trì giọng điệu nhất quán và có thể thiếu lời kêu gọi hành động phù hợp.
- Người bán có thể lặp lại các tuyên bố tuyệt đối hoặc khó chứng minh trong mô tả quảng cáo.
- Nội dung được tạo nhanh nhưng không có bước rà soát có hệ thống trước khi đăng.

### Rủi ro vận hành và chính sách

Nội dung chứa cam kết hiệu quả, tuyên bố “tốt nhất/số 1” hoặc thông tin không có căn cứ có thể dẫn đến khiếu nại, bị từ chối quảng cáo hoặc phát sinh rủi ro với tài khoản bán hàng. Rủi ro cụ thể phụ thuộc pháp luật, ngành hàng và chính sách hiện hành của từng nền tảng.

Danh sách regex trong sản phẩm chỉ là bộ lọc cảnh báo ban đầu. Không tìm thấy từ cấm **không đồng nghĩa** nội dung đã được chứng nhận an toàn.

## 3. Giải pháp cốt lõi

1. **Thu thập đầu vào có giới hạn:** tên sản phẩm tối đa 200 ký tự, mô tả tối đa 5.000 ký tự; không chấp nhận chuỗi chỉ có khoảng trắng.
2. **Chọn quy tắc theo kênh:** TikTok, Facebook hoặc Shopee tương ứng với một strategy độc lập.
3. **Sinh qua backend:** Gemini API key không được gửi tới trình duyệt. Nếu chưa cấu hình Gemini, demo có thể dùng mock.
4. **Rà soát chính sách:** Policy Checker quét kết quả, giữ nguyên nội dung và bổ sung cảnh báo/các cụm từ bị phát hiện.
5. **Lưu dữ liệu mới:** chỉ nguồn `gemini` hoặc `mock` được ghi vào `generated_contents`; kết quả cache/sample không được coi là một lần sinh mới.
6. **Hiển thị nguồn và trạng thái:** giao diện phân biệt Gemini, mock, cache và nội dung mẫu.

## 4. Nghiệp vụ theo kênh

| Kênh | Mục tiêu nội dung | Cấu trúc/prompt hiện tại | Temperature |
|---|---|---|---:|
| TikTok | Thu hút người xem video ngắn và dẫn tới hành động | Hook trong 3 giây đầu, gợi ý hình ảnh/hành động quay, lời thoại tự nhiên, CTA | 0.9 |
| Facebook | Tạo bài viết dễ đọc và kết nối với khách hàng | AIDA (Attention, Interest, Desire, Action), storytelling, CTA | 0.8 |
| Shopee | Trình bày thông tin sản phẩm dễ quét và hỗ trợ tìm kiếm | Tiêu đề có từ khóa SEO, 4–6 bullet, hashtag liên quan | 0.3 |

Các strategy hướng dẫn model không bịa công dụng, thông số hoặc tuyên bố tuyệt đối. Đây là chỉ dẫn prompt, không phải bảo đảm tuyệt đối; người bán cần đối chiếu bản nháp với hồ sơ sản phẩm.

### Quy tắc đầu vào dùng chung

- Tên sản phẩm và mô tả đều bắt buộc.
- Tên phải dài từ 1 đến 200 ký tự sau trim.
- Mô tả phải dài từ 1 đến 5.000 ký tự sau trim.
- `platform` phải là một trong `tiktok`, `facebook`, `shopee`.

## 5. Policy Auditor

### Danh sách hiện tại

| Nhóm/cụm từ | Cách hiểu nghiệp vụ | Hành động |
|---|---|---|
| `100%` | Tuyên bố tuyệt đối về mức độ/kết quả | Đánh dấu và yêu cầu người bán kiểm tra căn cứ |
| `cam kết` | Ngôn ngữ bảo đảm có thể tạo nghĩa vụ hoặc gây hiểu lầm | Cảnh báo |
| `số 1` | Tuyên bố đứng đầu/thứ hạng cần bằng chứng | Cảnh báo |
| `trị dứt điểm` | Tuyên bố hiệu quả điều trị tuyệt đối, nhạy cảm theo ngành hàng | Cảnh báo |

Checker sử dụng regex, không tự động xóa, viết lại hoặc từ chối toàn bộ bản nháp. Các cụm từ được trả trong `flagged_terms`; thông điệp giải thích được trả trong `warnings`.

### Quy trình xử lý đề xuất

1. AI tạo nội dung theo strategy của kênh.
2. Strategy chuẩn hóa đầu ra bằng `parse_output()`.
3. Policy Checker quét văn bản và xây danh sách cụm từ bị gắn cờ.
4. Giao diện hiển thị nội dung cùng cảnh báo.
5. Người bán kiểm tra ngữ cảnh, căn cứ và chính sách áp dụng; tự chỉnh nội dung nếu cần.
6. Bản ghi được lưu vào Supabase theo luồng API hiện tại nếu nguồn là Gemini/mock.

### Giới hạn và hướng phát triển

- Regex không hiểu ngữ cảnh, phủ định, biến thể chính tả, dấu cách đặc biệt hoặc từ đồng nghĩa.
- Có thể có false positive và false negative.
- Nên bổ sung test, quản lý phiên bản bộ quy tắc và quy trình duyệt thay đổi.
- Nếu sau này tự động sửa câu chữ, cần lưu bản gốc, bản sửa và lý do để người dùng có thể kiểm tra.

## 6. Luồng nghiệp vụ đầu-cuối

```text
Người bán nhập thông tin
        ↓
Frontend validate → POST /api/generate
        ↓
FastAPI validate request → chọn strategy
        ↓
LLMGateway → Gemini hoặc mock / cache fallback
        ↓
parse_output → PolicyChecker
        ↓
Nếu source = gemini/mock: lưu Supabase
Nếu source = cache/sample: không ghi bản ghi mới
        ↓
Trả nội dung, nguồn, cảnh báo → UI hiển thị
```

## 7. Dữ liệu được lưu

Bảng `public.generated_contents` gồm:

- `id`: UUID tự sinh.
- `product_name`: tên sản phẩm người dùng cung cấp.
- `platform`: kênh được chọn.
- `generated_text`: nội dung sau khi strategy parse và policy audit (nội dung không bị tự sửa).
- `created_at`: thời điểm tạo bản ghi, mặc định do PostgreSQL đặt.

Schema được quản lý bằng migration `supabase/migrations/001_initial_schema.sql`. Khóa service role chỉ dùng ở backend.

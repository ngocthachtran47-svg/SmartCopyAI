# Tài liệu UI/UX và giao diện

## 1. Mục tiêu trải nghiệm

Giao diện giúp người bán tạo bản nháp nhanh, hiểu rõ nội dung thuộc kênh nào, biết bản nháp đến từ Gemini hay chế độ dự phòng, và nhìn thấy cảnh báo chính sách ngay cạnh kết quả. Lỗi mạng hoặc upstream không được trình bày như nội dung thành công mới nếu backend chỉ trả cache/sample.

## 2. Layout tổng thể

### Desktop

- Header gọn với nhận diện SmartCopy AI và trạng thái nền tảng.
- Hero giải thích lợi ích và ba kênh hỗ trợ.
- Khu vực chính chia hai cột:
  - Trái: thông tin sản phẩm, chọn nền tảng và nút tạo.
  - Phải: kết quả hoặc trạng thái chờ tạo.
- Footer nhắc người dùng rà soát nội dung trước khi đăng.

### Mobile

- Các cột xếp theo chiều dọc; form và kết quả chiếm toàn bộ chiều rộng khả dụng.
- Các lựa chọn kênh co lại thành lưới thích ứng.
- Nút tạo nội dung rộng, dễ thao tác; khoảng cách/đệm giữ vùng chạm thoải mái.
- Nội dung kết quả cho phép xuống dòng và cuộn tự nhiên, không ép chiều rộng desktop.

Breakpoints được triển khai chủ yếu bằng utility responsive của Tailwind trong `App.jsx` và CSS components trong `index.css`.

## 3. Luồng màn hình

1. **Trạng thái ban đầu:** form rỗng, TikTok được chọn mặc định; khu vực kết quả giải thích cách bắt đầu.
2. **Validation:** lỗi tên/mô tả xuất hiện ngay cạnh trường tương ứng; không gửi request nếu dữ liệu chỉ có khoảng trắng.
3. **Đang gửi:** nút bị disable, icon spinner và nhãn “Đang tạo nội dung...” xuất hiện.
4. **Thành công:** kết quả, nền tảng, badge nguồn, cảnh báo và nút sao chép được hiển thị.
5. **Lỗi API/kết nối:** thông báo lỗi có vai trò alert xuất hiện trong form; không hiển thị bản nháp giả như kết quả mới.
6. **Fallback có chủ đích:** nếu backend trả `cache` hoặc `sample`, badge cho biết nguồn tương ứng; nội dung vẫn có thể được hiển thị cùng warning.

## 4. Accessibility và WCAG

### Form

- Sử dụng `<form>`, `<fieldset>`, `<legend>` để biểu diễn nhóm lựa chọn nền tảng.
- Các radio được bọc trong label có nội dung dễ hiểu và id/label liên kết đúng.
- Trường sản phẩm là `<input>` có `<label htmlFor="product">`.
- Mô tả là `<textarea>` có `<label htmlFor="description">`.
- Trường là controlled component: giá trị nằm trong React state và cập nhật qua `onChange`.
- Lỗi trường được gắn `role="alert"`, `aria-invalid` và `aria-describedby`.
- Nút submit là `<button>` thật; có disabled state khi đang gửi.

### Điều hướng và trạng thái

- Các thao tác dùng phần tử native, có thể điều khiển bằng bàn phím.
- Focus ring được giữ rõ trong `index.css` (`:focus-visible`).
- Icon trang trí trong lucide-react dùng `aria-hidden="true"`; tên nút được cung cấp bằng text.
- Lỗi API dùng `role="alert"`; cảnh báo policy dùng `role="status"` để hỗ trợ công nghệ trợ năng.
- Màu badge đi cùng nhãn chữ, không dùng màu làm tín hiệu duy nhất.

## 5. Component chính

| Khái niệm nghiệp vụ | Component thực tế | Trách nhiệm |
|---|---|---|
| ProductInputForm | `ProductForm.jsx` | Controlled form, chọn kênh, validate trường và gọi callback submit |
| ContentResultCard | `ContentResult.jsx` | Hiển thị nội dung, warning, flagged terms và sao chép |
| PolicyAlertBadge | Cảnh báo trong `ContentResult.jsx` | Hiển thị warning/flagged terms; hiện chưa tách thành component riêng |
| Resilience/status badge | `ResilienceBadge.jsx` | Nhãn nguồn `gemini`, `mock`, `cache`, `sample` |

Các tên ProductInputForm, ContentResultCard và PolicyAlertBadge là tên khái niệm trong tài liệu; mã hiện tại dùng tên file/component như bảng trên.

## 6. Trạng thái resilience và dữ liệu stale

`ResilienceBadge` ánh xạ nguồn phản hồi:

| `source` | Nhãn giao diện | Ý nghĩa |
|---|---|---|
| `gemini` | Gemini AI | Gemini trả kết quả mới |
| `mock` | Chế độ demo | Kết quả tạo từ mock generator/mock endpoint |
| `cache` | Bản lưu dự phòng | Backend trả kết quả cũ phù hợp với prompt khi upstream/breaker lỗi |
| `sample` | Nội dung mẫu | Backend không có bản cache thích hợp và phải fallback mẫu |

Cache ở đây nằm trong bộ nhớ backend; frontend không duy trì offline cache độc lập. Dữ liệu hết TTL mới không được dùng như kết quả bình thường, nhưng có thể được backend giữ thêm để fallback khi có lỗi. Giao diện hiện chưa gọi API health định kỳ để hiển thị mất mạng độc lập; trạng thái lỗi request hiện được thể hiện bằng thông báo lỗi. Không nên mô tả UI là đã có đồng bộ offline hay tự hiển thị stale data khi trình duyệt mất kết nối.

## 7. Sao chép và thông báo lỗi

- Nút “Sao chép nội dung” dùng Clipboard API.
- Khi sao chép thành công, nhãn đổi sang “Đã sao chép”.
- Nếu clipboard bị từ chối, giao diện đưa thông báo trợ giúp; người dùng có thể chọn văn bản để sao chép thủ công.
- Không bắt lỗi im lặng theo kiểu success-shaped fallback.

## 8. Hướng dẫn khi thay đổi UI

- Giữ lỗi gắn với trường và mô tả bằng văn bản.
- Đừng bỏ label để thay bằng placeholder; placeholder không thay thế tên trường.
- Khi thêm trạng thái mới, bảo đảm có nội dung cho màn hình đọc và focus không bị mất.
- Kiểm tra ở viewport hẹp, bàn phím, zoom và khi nội dung dài/cảnh báo nhiều.

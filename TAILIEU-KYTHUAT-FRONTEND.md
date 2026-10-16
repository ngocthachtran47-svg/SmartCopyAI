# Tài liệu kỹ thuật Frontend

## 1. Tổng quan

Frontend là ứng dụng React 18 được khởi tạo và build bằng Vite. Tailwind CSS cung cấp utility classes; `lucide-react` cung cấp icon. Frontend chỉ giao tiếp với backend SmartCopy qua HTTP endpoint `/api/generate`; việc gọi Gemini và lưu dữ liệu Supabase được thực hiện ở phía server.

## 2. Cấu trúc source

```text
frontend/
├── index.html
├── package.json
├── tailwind.config.js
├── postcss.config.js
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── index.css
    ├── App.css
    ├── components/
    │   ├── ProductForm.jsx
    │   ├── ContentResult.jsx
    │   └── ResilienceBadge.jsx
    ├── hooks/
    │   └── useApi.js
    └── assets/
```

`App.css` còn tồn tại từ scaffold, nhưng giao diện SmartCopy chủ yếu định nghĩa style qua Tailwind và `index.css`.

## 3. State và data flow

### Component state

- `ProductForm` giữ `product`, `description`, `platform`, và lỗi validation trong `useState`.
- `App` giữ kết quả đã nhận hoặc `null`.
- `ContentResult` giữ trạng thái clipboard thành công/lỗi.
- `useApi` quản lý `loading`, `error` và hàm `generateContent`.

### Hook hiện có

`src/hooks/useApi.js` là hook gọi API tổng quát cho flow hiện tại. **`useContentGenerator.js` chưa tồn tại** trong source hiện tại; không nên import hoặc ghi nhận nó như thành phần đã triển khai. Có thể tạo hook chuyên biệt sau này nếu logic sinh nội dung cần được tái sử dụng ở nhiều màn hình.

Luồng dữ liệu:

```text
ProductForm (controlled state)
    → onGenerate(payload)
App.handleGenerate
    → useApi.generateContent(payload)
    → POST {API_BASE_URL}/api/generate
Backend response
    → App.result
    → ContentResult + ResilienceBadge
```

### Request contract

```json
{
  "product": "Bình giữ nhiệt Lumi",
  "description": "Thép không gỉ, dung tích 500 ml.",
  "platform": "tiktok"
}
```

Response thành công có các trường chính:

```json
{
  "content": "Nội dung được tạo...",
  "platform": "tiktok",
  "source": "gemini",
  "warning": null,
  "warnings": [],
  "flagged_terms": []
}
```

`source` có thể là `gemini`, `mock`, `cache`, `sample`.

## 4. Cơ chế gọi API và xử lý lỗi

- URL mặc định là `http://127.0.0.1:8000`.
- Có thể đặt `VITE_API_BASE_URL` cho backend URL khác tại thời điểm chạy/build.
- Hook gửi `Content-Type: application/json`.
- Khi HTTP status không thành công, hook cố đọc `detail` từ response và ném lỗi để component quyết định cách hiển thị.
- `loading` được bật trước request và luôn tắt trong `finally`.
- `App` xóa kết quả cũ khi request thất bại, đồng thời hiển thị thông báo ở khu vực form.
- Frontend không retry provider và không chứa API key; retry/backoff nằm ở gateway backend.

Mở rộng hook cần giữ nguyên nguyên tắc: không trả kết quả giả khi lỗi, không bỏ qua status code, không log request có secrets, và cung cấp thông báo người dùng dễ hiểu.

## 5. Components

### `ProductForm.jsx`

- Controlled form với platform radio group, tên sản phẩm và mô tả.
- Kiểm tra giá trị trim rỗng phía client; backend vẫn kiểm tra độc lập.
- Trường giới hạn độ dài tương ứng với API.
- Nhãn truy cập được, lỗi `role="alert"`, `aria-invalid` và liên kết mô tả.
- Submit gọi callback `onGenerate`; nút disabled khi API đang chạy.

### `ContentResult.jsx`

- Hiển thị nội dung với `white-space: pre-wrap`.
- Hiển thị warning gateway, policy warnings và flagged terms khi có.
- Cung cấp hành động sao chép, xử lý cả thành công lẫn clipboard failure.
- Dùng `ResilienceBadge` để cho biết nguồn.

### `ResilienceBadge.jsx`

- Ánh xạ `source` sang nhãn/icon tương ứng.
- Badge không quyết định resilience; chỉ phản ánh metadata từ backend.

## 6. Styling và icon

- Tailwind quét `index.html` và `src/**/*.{js,ts,jsx,tsx}` qua `tailwind.config.js`.
- Component classes dùng `@layer components` trong `src/index.css`.
- Thêm class mới ưu tiên utility Tailwind hoặc component class có phạm vi rõ ràng.
- Giữ responsive prefixes (`sm:`, `lg:`) cho các bố cục thích ứng.
- Icon từ lucide-react cần kích thước nhất quán; icon thuần trang trí có `aria-hidden="true"`.
- Không dùng `div` làm nút; giữ `:focus-visible` để người dùng bàn phím biết vị trí hiện tại.

## 7. Cấu hình Supabase phía frontend

`.env.example` có `VITE_SUPABASE_URL` và `VITE_SUPABASE_ANON_KEY` để dành cho tích hợp client-side an toàn với public/anon key. Source hiện tại chưa khởi tạo Supabase client ở frontend và không cần hai biến này cho lưu dữ liệu, vì backend đang dùng service role.

Không bao giờ khai báo `SUPABASE_SERVICE_ROLE_KEY` bằng tiền tố `VITE_` hoặc đưa khóa đó vào JavaScript browser.

## 8. Chạy và kiểm tra

```powershell
cd frontend
npm install
npm run dev -- --host 127.0.0.1
```

Kiểm tra:

```powershell
npm run build
npm run lint
```

Khi đổi request shape, cần kiểm tra đồng thời `ProductForm`, hook `useApi`, response rendering và contract ở `backend/main.py`.

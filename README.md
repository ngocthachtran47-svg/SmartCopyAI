# SmartCopy AI

SmartCopy AI là nền tảng hỗ trợ nhà bán hàng tạo nội dung sản phẩm phù hợp với nhiều kênh thương mại điện tử. Backend FastAPI tiếp nhận yêu cầu, chọn chiến lược viết theo kênh, gọi Google Gemini qua một gateway có cơ chế chịu lỗi, kiểm tra một số tuyên bố quảng cáo nhạy cảm rồi lưu kết quả hợp lệ vào Supabase. Frontend React cung cấp biểu mẫu truy cập được và hiển thị bản nháp cùng trạng thái nguồn dữ liệu.

> **Lưu ý:** Policy Checker chỉ đưa ra cảnh báo theo danh sách quy tắc hiện có; đây không phải dịch vụ tư vấn pháp lý hay hệ thống xác nhận tuân thủ chính sách của từng sàn. Nội dung do AI tạo cần được con người rà soát trước khi đăng.

## Tính năng chính

- Sinh nội dung theo TikTok, Facebook hoặc Shopee bằng Strategy Pattern.
- Gọi Gemini ở phía backend; API key và khóa Supabase không được gửi xuống trình duyệt.
- Retry exponential backoff với jitter đối với lỗi 429, lỗi 5xx và lỗi kết nối.
- Circuit breaker mở sau 5 lần gọi gateway thất bại; thời gian chờ mặc định 60 giây.
- Cache trong bộ nhớ với thời hạn dữ liệu mới 10 phút; giữ dữ liệu hết hạn thêm một khoảng thời gian để dùng khi upstream lỗi.
- Khi chưa khai báo Gemini API key, backend chạy mock mode để thuận tiện demo.
- Quét từ/cụm từ có thể tạo tuyên bố quảng cáo tuyệt đối và trả cảnh báo cùng nội dung.
- Lưu kết quả từ Gemini hoặc mock vào bảng Supabase `generated_contents`. Kết quả cache/sample không được ghi như nội dung mới.

## Cấu trúc thư mục

```text
smartcopy-ai/
├── .env.example
├── .gitignore
├── AI_INTERACTION_LOG.md
├── README.md
├── AGENT.md
├── TAILIEU-NGHIEPVU.md
├── TAILIEU-GIAODIEN.md
├── TAILIEU-KYTHUAT-FRONTEND.md
├── TAILIEU-KYTHUAT-ADMIN.md
├── backend/
│   ├── config.py
│   ├── main.py
│   ├── requirements.txt
│   ├── database/
│   │   ├── schema.sql
│   │   └── supabase_client.py
│   ├── services/
│   │   ├── circuit_breaker.py
│   │   └── llm_gateway.py
│   ├── strategies/
│   │   ├── base_strategy.py
│   │   ├── tiktok_strategy.py
│   │   ├── facebook_strategy.py
│   │   └── shopee_strategy.py
│   └── verifiers/
│       └── policy_checker.py
├── frontend/
│   ├── package.json
│   ├── tailwind.config.js
│   └── src/
│       ├── App.jsx
│       ├── index.css
│       ├── components/
│       │   ├── ProductForm.jsx
│       │   ├── ContentResult.jsx
│       │   └── ResilienceBadge.jsx
│       └── hooks/
│           └── useApi.js
└── supabase/
    └── migrations/
        └── 001_initial_schema.sql
```

## Tech stack

| Lớp | Công nghệ | Vai trò |
|---|---|---|
| Frontend | React 18, Vite, Tailwind CSS, lucide-react | Biểu mẫu, trạng thái giao diện, hiển thị bản nháp và cảnh báo |
| Backend | Python, FastAPI, Pydantic Settings, httpx async | API, xác thực dữ liệu, cấu hình môi trường và tích hợp upstream |
| Database | Supabase (PostgreSQL) | Lưu nội dung đã tạo qua khóa service role phía backend |
| AI | Google Gemini API | Sinh nội dung; mock mode dành cho demo không cấu hình Gemini key |

## Yêu cầu

- Python 3.10 trở lên.
- Node.js và npm tương thích với phiên bản Vite khai báo trong `frontend/package.json`.
- Dự án Supabase nếu cần lưu nội dung thật.
- Gemini API key nếu cần gọi Gemini thay vì mock mode.

## Cài đặt

### 1. Cấu hình biến môi trường

Tạo file `.env` ở thư mục gốc của dự án từ mẫu:

```powershell
Copy-Item .env.example .env
```

Mở `.env` và cấu hình:

```dotenv
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.5-flash

SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<server-side-service-role-key>

# Chỉ dành cho ứng dụng browser; không đặt service-role key ở các biến VITE_.
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=

CORS_ORIGINS=["http://localhost:5173","http://127.0.0.1:5173"]
```

`SUPABASE_SERVICE_ROLE_KEY` là bí mật phía server, không đưa vào mã frontend, repository công khai, log hoặc ảnh chụp màn hình. Các biến `VITE_*` được đóng gói vào frontend nếu được sử dụng, vì vậy chỉ dùng giá trị public/anon key. Hiện ứng dụng lưu dữ liệu từ backend; frontend chưa gọi Supabase trực tiếp.

### 2. Tạo schema Supabase

Chạy migration [001_initial_schema.sql](./supabase/migrations/001_initial_schema.sql) trong Supabase SQL Editor hoặc bằng công cụ migration của Supabase. Migration tạo bảng `generated_contents`, tắt RLS và giới hạn quyền truy cập bảng cho `service_role`.

### 3. Cài backend

PowerShell:

```powershell
cd backend
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Nếu PowerShell không cho phép chạy script kích hoạt môi trường, có thể gọi interpreter trực tiếp:

```powershell
.\venv\Scripts\python.exe -m pip install -r requirements.txt
```

### 4. Cài frontend

Mở terminal thứ hai tại thư mục gốc:

```powershell
cd frontend
npm install
```

## Chạy ở chế độ phát triển

### Backend

Từ thư mục `backend`:

```powershell
.\venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

- API: `http://127.0.0.1:8000`
- Tài liệu OpenAPI: `http://127.0.0.1:8000/docs`
- Health check: `http://127.0.0.1:8000/health`

### Frontend

Từ thư mục `frontend`:

```powershell
npm run dev -- --host 127.0.0.1 --port 5173
```

Mở `http://127.0.0.1:5173`.

Frontend gửi yêu cầu đến `http://127.0.0.1:8000` theo mặc định. Có thể đổi backend URL bằng biến build-time `VITE_API_BASE_URL` nếu triển khai backend ở địa chỉ khác.

## API tạo nội dung

`POST /api/generate`

```json
{
  "product": "Bình giữ nhiệt Lumi",
  "description": "Thân thép không gỉ, dung tích 500 ml.",
  "platform": "tiktok"
}
```

`platform` nhận một trong `tiktok`, `facebook`, `shopee`. Kết quả gồm `content`, `platform`, `source`, cảnh báo gateway và các trường `warnings`, `flagged_terms`. Khi persistence Supabase chưa cấu hình hoặc thao tác lưu thất bại, API trả lỗi thay vì báo đã lưu thành công.

## Kiểm tra nhanh

```powershell
cd frontend
npm run build
npm run lint
```

Backend có thể được kiểm tra với `GET /health`, sau đó gửi request mẫu đến `POST /api/generate`. Khi kiểm thử database, dùng dự án Supabase thử nghiệm hoặc mock client; không dùng khóa production trong test.

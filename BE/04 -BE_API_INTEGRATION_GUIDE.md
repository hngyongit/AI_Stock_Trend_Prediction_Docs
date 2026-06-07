# BE API Integration Guide for Frontend Developers

Tài liệu này cung cấp hướng dẫn tích hợp chi tiết toàn bộ các API của hệ thống **AI-Based Stock Trend Prediction** dành cho Frontend Developer (Web và Mobile).

---

## 1. QUY ƯỚC CHUNG

### 1.1. Base URL
* **Development**: `http://localhost:5000`
* **Production**: `https://api.aistockprediction.vn` (hoặc domain do hệ thống cung cấp)

### 1.2. Quy ước Response chung
Tất cả các API đều trả về định dạng JSON thống nhất:
```json
{
  "success": true,       // true nếu thành công, false nếu thất bại
  "message": "...",      // Thông điệp mô tả kết quả
  "data": { ... }        // Dữ liệu trả về (Object, Array hoặc null)
}
```

### 1.3. Quy ước Lỗi & Error Code
Khi `success: false`, response sẽ có cấu trúc như sau:
* **Lỗi logic/Hệ thống chung**:
  ```json
  {
    "success": false,
    "message": "Thông báo lỗi chi tiết"
  }
  ```
* **Lỗi Validation (400 Bad Request)**:
  ```json
  {
    "success": false,
    "message": "Validation failed",
    "errors": [
      {
        "field": "email",
        "message": "Please include a valid email"
      }
    ]
  }
  ```

#### Các mã trạng thái HTTP thông dụng:
* `200 OK`: Yêu cầu thành công.
* `201 Created`: Tạo mới dữ liệu thành công (đăng ký, tạo stock, thêm watchlist).
* `400 Bad Request`: Validation lỗi hoặc vi phạm luật nghiệp vụ (Business Rules).
* `401 Unauthorized`: Chưa đăng nhập hoặc token hết hạn/không hợp lệ.
* `403 Forbidden`: Đã đăng nhập nhưng không có quyền truy cập (Ví dụ: USER gọi API của ADMIN).
* `404 Route/Resource Not Found`: API path không tồn tại hoặc dữ liệu (user, stock) không có trong DB.
* `500 Internal Server Error`: Lỗi máy chủ hoặc cơ sở dữ liệu.

### 1.4. Quy ước Xác thực (Authentication)
* Với các API được đánh dấu **[Protected]**, FE bắt buộc phải gửi Access Token trong headers:
  ```http
  Authorization: Bearer <access_token>
  ```
* Các API yêu cầu quyền **ADMIN** hoặc **STAFF** sẽ tự động trả về lỗi `403 Forbidden` nếu tài khoản đăng nhập có role `USER`.

---

## 2. DANH SÁCH CHI TIẾT API THEO MODULE

### 2.1. Module: Auth (Authentication)

#### 2.1.1. Đăng ký tài khoản (Register)
* **Method**: `POST`
* **Endpoint**: `/api/auth/register`
* **Quyền truy cập**: Public
* **Request Body**:
  ```json
  {
    "full_name": "Nguyen Van A",
    "email": "usertest@example.com",
    "password": "user123456"
  }
  ```
* **Response thành công (201 Created)**:
  ```json
  {
    "success": true,
    "message": "User registered successfully",
    "data": {
      "user": {
        "id": "665f1a2b9c1e2a0012a12345",
        "full_name": "Nguyen Van A",
        "email": "usertest@example.com",
        "role": "USER",
        "status": "ACTIVE"
      }
    }
  }
  ```
* **Response lỗi Validation (400 Bad Request)**:
  ```json
  {
    "success": false,
    "message": "Validation failed",
    "errors": [
      {
        "field": "password",
        "message": "Password must be at least 8 characters long"
      }
    ]
  }
  ```

#### 2.1.2. Đăng nhập (Login)
* **Method**: `POST`
* **Endpoint**: `/api/auth/login`
* **Quyền truy cập**: Public
* **Request Body**:
  ```json
  {
    "email": "usertest@example.com",
    "password": "user123456"
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Login successfully",
    "data": {
      "access_token": "eyJhbGciOiJIUzI1... (Sử dụng trong 15 phút)",
      "refresh_token": "eyJhbGciOiJIUzI1... (Sử dụng trong 7 ngày)",
      "user": {
        "id": "665f1a2b9c1e2a0012a12345",
        "full_name": "Nguyen Van A",
        "email": "usertest@example.com",
        "role": "USER",
        "status": "ACTIVE"
      }
    }
  }
  ```
* **Response lỗi sai tài khoản (401 Unauthorized)**:
  ```json
  {
    "success": false,
    "message": "Invalid email or password"
  }
  ```

#### 2.1.3. Đăng xuất (Logout)
* **Method**: `POST`
* **Endpoint**: `/api/auth/logout`
* **Quyền truy cập**: **[Protected]** (Yêu cầu JWT Access Token)
* **Headers**: `Authorization: Bearer <access_token>`
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Logout successfully"
  }
  ```

#### 2.1.4. Làm mới Access Token (Refresh Token)
* **Method**: `POST`
* **Endpoint**: `/api/auth/refresh-token`
* **Quyền truy cập**: Public
* **Request Body**:
  ```json
  {
    "refresh_token": "eyJhbGciOiJIUzI1..."
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Refresh token successfully",
    "data": {
      "access_token": "eyJhbGciOiJIUzI1... (Access Token mới)"
    }
  }
  ```

---

### 2.2. Module: User/Profile

#### 2.2.1. Xem thông tin tài khoản hiện tại (Get Me)
* **Method**: `GET`
* **Endpoint**: `/api/users/me`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get profile successfully",
    "data": {
      "id": "665f1a2b9c1e2a0012a12345",
      "full_name": "Nguyen Van A",
      "email": "usertest@example.com",
      "role": "USER",
      "status": "ACTIVE",
      "created_at": "2026-06-01T10:00:00.000Z"
    }
  }
  ```

#### 2.2.2. Cập nhật thông tin cá nhân (Update Profile)
* **Method**: `PUT`
* **Endpoint**: `/api/users/me`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Request Body**:
  ```json
  {
    "full_name": "Nguyen Van B"
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Update profile successfully",
    "data": {
      "id": "665f1a2b9c1e2a0012a12345",
      "full_name": "Nguyen Van B",
      "email": "usertest@example.com",
      "role": "USER",
      "status": "ACTIVE"
    }
  }
  ```

#### 2.2.3. Đổi mật khẩu cá nhân (Change Password)
* **Method**: `PUT`
* **Endpoint**: `/api/users/me/password`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Request Body**:
  ```json
  {
    "current_password": "OldPassword123",
    "new_password": "NewPassword123"
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Change password successfully"
  }
  ```

---

### 2.3. Module: Stock (Cổ phiếu & Tra cứu)

#### 2.3.1. Xem danh mục cổ phiếu (List Stocks)
* **Method**: `GET`
* **Endpoint**: `/api/stocks`
* **Quyền truy cập**: Public
* **Query Params**:
  * `page` (number, default: `1`): Trang hiện tại.
  * `limit` (number, default: `10`): Số record hiển thị trên một trang.
  * `keyword` (string, optional): Từ khóa tìm kiếm theo symbol hoặc tên công ty.
  * `market` (string, optional): Mã sàn giao dịch (e.g. `HOSE`, `HNX`).
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get stocks successfully",
    "data": {
      "items": [
        {
          "id": "6a21bc1ca1be78dbd5c743a8",
          "symbol": "AAA",
          "company_name": "CTCP Nhựa An Phát Xanh",
          "market_code": "HOSE",
          "status": "ACTIVE"
        }
      ],
      "pagination": {
        "page": 1,
        "limit": 10,
        "total_items": 1,
        "total_pages": 1
      }
    }
  }
  ```

#### 2.3.2. Xem thông tin chi tiết cổ phiếu (Stock Details)
* **Method**: `GET`
* **Endpoint**: `/api/stocks/:symbol`
* **Quyền truy cập**: Public
* **Path Params**:
  * `symbol`: Mã cổ phiếu dạng viết hoa (e.g., `AAA`, `FPT`).
* **Response thành công (200 OK)**:
  > [!IMPORTANT]
  > API này trả về toàn bộ dữ liệu chi tiết thu thập từ `FactMarketPrice` của ngày gần nhất bao gồm: chỉ số giao dịch khớp lệnh, khối lượng đặt mua/bán, khối ngoại và chỉ số định giá tài chính.
  ```json
  {
    "success": true,
    "message": "Get stock detail successfully",
    "data": {
      "id": "6a21bc1ca1be78dbd5c743a8",
      "symbol": "AAA",
      "company_name": "CTCP Nhựa An Phát Xanh",
      "market_code": "HOSE",
      "status": "ACTIVE",
      "listed_date": null,
      "latest_price": {
        "time_id": 20260603,
        "open_price": 6950,
        "high_price": 7030,
        "low_price": 6950,
        "close_price": 7000,
        "volume": 406500,
        "bid_volume": 41500,
        "ask_volume": 18200,
        "foreign_buy": 6900,
        "foreign_sell": 7000,
        "foreign_net": -100,
        "market_cap": 2756.2,
        "eps": 1213,
        "pe": 5.75,
        "forward_pe": 7.29,
        "bvps": 15827,
        "pb": 0.44,
        "beta": 0.6,
        "ros": 5.2,
        "roe": 2.5,
        "roaa": 1.19,
        "price_change": 50,
        "price_change_percent": 0.7194,
        "crawled_at": "2026-06-04T18:49:38.844Z",
        "created_at": "2026-06-04T18:49:38.844Z",
        "updated_at": "2026-06-04T18:49:38.844Z"
      }
    }
  }
  ```
* **Response lỗi không tìm thấy (404 Not Found)**:
  ```json
  {
    "success": false,
    "message": "Stock symbol not found"
  }
  ```

#### 2.3.3. Xem lịch sử giá vẽ biểu đồ (Price History Chart)
* **Method**: `GET`
* **Endpoint**: `/api/stocks/:symbol/chart`
* **Quyền truy cập**: Public
* **Path Params**:
  * `symbol`: Mã cổ phiếu (e.g., `AAA`, `FPT`).
* **Query Params**:
  * `range` (string, default: `1m`): Khoảng thời gian lấy giá. Chấp nhận các giá trị: `7d` (7 ngày), `1m` (30 ngày), `3m` (90 ngày), `6m` (180 ngày), `1y` (365 ngày), `all` (toàn bộ lịch sử).
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get price history successfully",
    "data": [
      {
        "time": "2026-06-01",
        "open": 6950,
        "high": 7030,
        "low": 6900,
        "close": 7000,
        "volume": 350000
      }
    ]
  }
  ```

---

### 2.4. Module: Watchlist (Theo dõi cá nhân)

#### 2.4.1. Xem danh sách theo dõi cá nhân (Get Watchlist)
* **Method**: `GET`
* **Endpoint**: `/api/watchlists`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get watchlist successfully",
    "data": [
      {
        "watchlist_id": "6a21bc1ca1be78dbd5c743a9",
        "stock": {
          "id": "6a21bc1ca1be78dbd5c743a8",
          "symbol": "AAA",
          "company_name": "CTCP Nhựa An Phát Xanh",
          "market_code": "HOSE"
        },
        "latest_price": {
          "close_price": 7000,
          "price_change": 50,
          "price_change_percent": 0.7194,
          "volume": 406500
        },
        "created_at": "2026-06-04T17:55:39.855Z"
      }
    ]
  }
  ```

#### 2.4.2. Thêm cổ phiếu vào Watchlist (Add to Watchlist)
* **Method**: `POST`
* **Endpoint**: `/api/watchlists`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Request Body**:
  ```json
  {
    "symbol": "AAA"
  }
  ```
* **Response thành công (201 Created)**:
  ```json
  {
    "success": true,
    "message": "Add stock to watchlist successfully",
    "data": {
      "watchlist_id": "6a21bc1ca1be78dbd5c743a9",
      "symbol": "AAA",
      "created_at": "2026-06-05T03:00:00.000Z"
    }
  }
  ```
* **Response lỗi khi quá giới hạn 5 mã hoặc đã tồn tại (400 Bad Request)**:
  ```json
  {
    "success": false,
    "message": "Watchlist limit exceeded (Maximum 5 stocks allowed)"
  }
  ```
  hoặc
  ```json
  {
    "success": false,
    "message": "Stock is already in your watchlist"
  }
  ```

#### 2.4.3. Xóa cổ phiếu khỏi Watchlist (Remove from Watchlist)
* **Method**: `DELETE`
* **Endpoint**: `/api/watchlists/:symbol`
* **Quyền truy cập**: **[Protected]** (USER, STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Path Params**:
  * `symbol`: Mã cổ phiếu viết hoa (e.g., `AAA`).
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Remove stock from watchlist successfully"
  }
  ```

---

### 2.5. Module: Market (Thông tin thị trường chung)

> [!NOTE]
> Module này dùng để hiển thị thông tin tổng quan các sàn giao dịch và biểu đồ tổng thể thị trường.

#### 2.5.1. Xem danh sách sàn giao dịch (List Markets)
* **Method**: `GET`
* **Endpoint**: `/api/markets`
* **Quyền truy cập**: Public
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get markets successfully",
    "data": [
      {
        "id": "6a1ff9e6e7c74fdfdcb76db5",
        "code": "HOSE",
        "name": "Ho Chi Minh Stock Exchange",
        "country": "Vietnam",
        "timezone": "Asia/Ho_Chi_Minh",
        "status": "active"
      }
    ]
  }
  ```

#### 2.5.2. Xem thông tin tổng quan thị trường (Market Overview)
* **Method**: `GET`
* **Endpoint**: `/api/market-overview`
* **Quyền truy cập**: Public
* **Query Params**:
  * `market_code` (string, optional, e.g. `HOSE`): Lọc theo sàn.
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get market overview successfully",
    "data": [
      {
        "id": "6a1ff9e6e7c74fdfdcb76db9",
        "market_code": "HOSE",
        "index_value": 1250.45,
        "index_change": 5.25,
        "index_change_percent": 0.42,
        "total_volume": 650000000,
        "total_value": 15600000000000,
        "advancers": 210,
        "decliners": 130,
        "unchanged": 50,
        "time_id": 20260603
      }
    ]
  }
  ```

---

### 2.6. Module: Admin Stock (Quản lý danh mục cổ phiếu của Admin)

#### 2.6.1. Tạo mới danh mục cổ phiếu (Create Stock Master)
* **Method**: `POST`
* **Endpoint**: `/api/admin/stocks`
* **Quyền truy cập**: **[Protected]** (ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Request Body**:
  ```json
  {
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "market_id": "6a1ff9e6e7c74fdfdcb76db5",
    "industry_id": "6a1ff9e6e7c74fdfdcb76db6", // (Optional)
    "status": "ACTIVE",
    "listed_date": "2006-12-13" // YYYY-MM-DD
  }
  ```
* **Response thành công (201 Created)**:
  ```json
  {
    "success": true,
    "message": "Create stock master successfully",
    "data": {
      "id": "6a21bc1ca1be78dbd5c743a8",
      "symbol": "FPT",
      "company_name": "Công ty Cổ phần FPT",
      "market_id": "6a1ff9e6e7c74fdfdcb76db5",
      "market_code": "HOSE",
      "status": "ACTIVE"
    }
  }
  ```
* **Response lỗi trùng lặp mã (400 Bad Request)**:
  ```json
  {
    "success": false,
    "message": "Stock symbol already exists"
  }
  ```

#### 2.6.2. Cập nhật thông tin danh mục cổ phiếu (Update Stock Master)
* **Method**: `PUT`
* **Endpoint**: `/api/admin/stocks/:id`
* **Quyền truy cập**: **[Protected]** (ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Path Params**:
  * `id`: ObjectId của bản ghi stock cần sửa.
* **Request Body**:
  ```json
  {
    "company_name": "Công ty Cổ phần FPT Việt Nam",
    "market_id": "6a1ff9e6e7c74fdfdcb76db5", // (Optional)
    "industry_id": "6a1ff9e6e7c74fdfdcb76db6", // (Optional)
    "status": "SUSPENDED", // ACTIVE, DELISTED, SUSPENDED
    "listed_date": "2006-12-13"
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Update stock master successfully",
    "data": {
      "id": "6a21bc1ca1be78dbd5c743a8",
      "symbol": "FPT",
      "company_name": "Công ty Cổ phần FPT Việt Nam",
      "market_id": "6a1ff9e6e7c74fdfdcb76db5",
      "market_code": "HOSE",
      "status": "SUSPENDED"
    }
  }
  ```

---

### 2.7. Module: Crawl Job (Quản lý các Job cào tự động)

> [!NOTE]
> Dành cho màn hình Admin/Staff vận hành cấu hình lịch cào dữ liệu tự động.

#### 2.7.1. Lấy danh sách Crawl Job (List Crawl Jobs)
* **Method**: `GET`
* **Endpoint**: `/api/admin/crawl-jobs`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get crawl jobs successfully",
    "data": [
      {
        "id": "6a21bc1ca1be78dbd5c743ff",
        "job_name": "Cào giá HOSE hàng ngày",
        "data_type": "DAILY_MARKET_PRICE",
        "cron_expression": "0 17 * * 1-5",
        "crawl_mode": "scheduled",
        "status": "active",
        "last_run_at": "2026-06-04T10:00:00.000Z",
        "next_run_at": "2026-06-05T10:00:00.000Z"
      }
    ]
  }
  ```

#### 2.7.2. Cập nhật trạng thái Job (Update Job Status)
* **Method**: `PATCH`
* **Endpoint**: `/api/admin/crawl-jobs/:id/status`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Path Params**:
  * `id`: ObjectId của crawl job.
* **Request Body**:
  ```json
  {
    "status": "inactive" // active, inactive
  }
  ```
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Crawl job status updated successfully"
  }
  ```

#### 2.7.3. Kích hoạt cào thủ công ngay lập tức (Trigger Job Manually)
* **Method**: `POST`
* **Endpoint**: `/api/admin/crawl-jobs/:id/trigger`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Path Params**:
  * `id`: ObjectId của crawl job.
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Crawl job triggered successfully"
  }
  ```

---

### 2.8. Module: Crawl Log (Nhật ký cào dữ liệu)

#### 2.8.1. Lấy danh sách lịch sử cào dữ liệu (List Crawl Logs)
* **Method**: `GET`
* **Endpoint**: `/api/admin/crawl-logs`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Query Params**:
  * `page` (number, default: `1`)
  * `limit` (number, default: `10`)
  * `status` (string, optional): Lọc theo `PENDING`, `SUCCESS`, `FAILED`, `PARTIAL_SUCCESS`.
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get crawl logs successfully",
    "data": {
      "items": [
        {
          "id": "6a21d46be2fe29b30850fea0",
          "crawl_job_id": "6a21bc1ca1be78dbd5c743ff",
          "started_at": "2026-06-04T18:49:00.000Z",
          "ended_at": "2026-06-04T18:55:00.000Z",
          "status": "PARTIAL_SUCCESS",
          "records_fetched": 400,
          "records_inserted": 395,
          "records_updated": 0,
          "records_failed": 5,
          "error_message": "Skipped: 5 mã không có dữ liệu"
        }
      ],
      "pagination": {
        "page": 1,
        "limit": 10,
        "total_items": 45,
        "total_pages": 5
      }
    }
  }
  ```

#### 2.8.2. Xem chi tiết kết quả cào từng mã (Crawl Log Details)
* **Method**: `GET`
* **Endpoint**: `/api/admin/crawl-logs/:id/details`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Path Params**:
  * `id`: ObjectId của Crawl Log tổng (`crawl_log_id`).
* **Query Params**:
  * `status` (string, optional): Lọc theo `SUCCESS`, `FAILED`, `SKIPPED`.
  * `symbol` (string, optional): Lọc tìm kiếm theo mã chứng khoán.
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get crawl log details successfully",
    "data": [
      {
        "id": "6a21d46be2fe29b30850fea2",
        "symbol": "AAA",
        "data_type": "DAILY_MARKET_PRICE",
        "status": "SUCCESS",
        "message": "INSERT: open=6950, close=7000, vol=406500",
        "created_at": "2026-06-04T18:49:38.844Z"
      },
      {
        "id": "6a21d46be2fe29b30850fea5",
        "symbol": "XYZ",
        "data_type": "DAILY_MARKET_PRICE",
        "status": "FAILED",
        "message": "HTTP Timeout 30000ms",
        "created_at": "2026-06-04T18:51:12.000Z"
      }
    ]
  }
  ```

---

### 2.9. Module: Data Quality (Giám sát chất lượng dữ liệu)

#### 2.9.1. Thống kê chất lượng cào (Crawl Quality Report)
* **Method**: `GET`
* **Endpoint**: `/api/admin/crawl-quality`
* **Quyền truy cập**: **[Protected]** (STAFF, ADMIN)
* **Headers**: `Authorization: Bearer <access_token>`
* **Query Params**:
  * `limit` (number, default: `30`): Số lượng phiên cào gần nhất cần đối chiếu.
* **Response thành công (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Get crawl quality report successfully",
    "data": [
      {
        "id": "6a21d46be2fe29b30850fecc",
        "time_id": 20260603,
        "records_fetched": 400,
        "records_inserted": 395,
        "records_updated": 0,
        "records_failed": 5,
        "success_rate": 98.75,
        "status": "PARTIAL_SUCCESS",
        "created_at": "2026-06-04T18:55:00.000Z"
      }
    ]
  }
  ```

---

## 3. THIẾT KẾ MÀN HÌNH VÀ FLOW GỌI API (FRONTEND ROADMAP)

### Màn hình 1: Đăng nhập & Đăng ký (Login / Register Screen)
* **API Sử dụng**:
  1. `POST /api/auth/login`
  2. `POST /api/auth/register`
* **Flow gọi API**:
  ```mermaid
  sequenceDiagram
    User->>FE: Nhập Email + Mật khẩu
    FE->>FE: Client-side Validate (Email đúng format, Mật khẩu >= 8 ký tự)
    FE->>BE: POST /api/auth/login (Request Body)
    BE-->>FE: Trả về tokens + user profile (200 OK)
    FE->>FE: Lưu access_token và refresh_token vào LocalStorage/AsyncStorage
    FE->>FE: Chuyển sang màn hình Dashboard / Home
  ```
* **Các trạng thái giao diện (UI States)**:
  * **Loading**: Vô hiệu hóa nút Submit, hiển thị spinner xoay.
  * **Error**: Hiển thị popup hoặc dòng chữ đỏ thông báo lỗi từ trường `message` của backend (Ví dụ: "Mật khẩu không đúng" hoặc hiển thị từng trường lỗi validation).

---

### Màn hình 2: Bảng tra cứu & Tìm kiếm cổ phiếu (Stock Directory Screen)
* **API Sử dụng**:
  1. `GET /api/stocks` (Có phân trang, search keyword, filter sàn)
* **Flow gọi API**:
  * Khi mở màn hình: Gọi `/api/stocks?page=1&limit=20`
  * Khi gõ ô search: Debounce 300ms rồi gọi `/api/stocks?keyword=FPT`
  * Khi chọn tab Sàn (HOSE/HNX): Gọi `/api/stocks?market=HOSE`
* **Các trường cần hiển thị trên UI**:
  * Mã cổ phiếu (`symbol`)
  * Tên công ty (`company_name`)
  * Sàn (`market_code`)
  * Trạng thái hoạt động (`status`)
* **Các trạng thái giao diện (UI States)**:
  * **Empty State**: Khi tìm kiếm từ khóa không khớp bất kỳ mã nào, hiển thị ảnh minh họa "Không tìm thấy kết quả phù hợp".
  * **Pagination Loader**: Hiển thị loading xương (Skeleton) khi chuyển trang hoặc kéo cuộn (infinite scroll).

---

### Màn hình 3: Chi tiết cổ phiếu & Biểu đồ (Stock Detail & Chart Screen)
* **API Sử dụng**:
  1. `GET /api/stocks/:symbol`
  2. `GET /api/stocks/:symbol/chart?range=1m` (Chọn bộ lọc range: 7d, 1m, 3m, 6m, 1y)
  3. `POST /api/watchlists` (Để thêm vào danh sách theo dõi)
  4. `DELETE /api/watchlists/:symbol` (Để hủy theo dõi)
* **Flow gọi API**:
  ```mermaid
  flowchart TD
    A[Mở màn chi tiết mã AAA] --> B(Gọi đồng thời 2 API)
    B --> C[GET /api/stocks/AAA]
    B --> D[GET /api/stocks/AAA/chart?range=1m]
    C --> E[Hiển thị: Tên CT, Giá hiện tại, Giá cao/thấp, Chỉ số P/E, P/B, EPS...]
    D --> F[Vẽ biểu đồ hình nến Candlestick hoặc Line Chart]
  ```
* **Các trường cần hiển thị đặc biệt**:
  * Biến động giá: `price_change` (Hiển thị màu Xanh nếu > 0, Đỏ nếu < 0, Vàng/Xám nếu = 0).
  * Chỉ số tài chính: `pe`, `pb`, `eps`, `roe`, `roaa` để người dùng đánh giá.
  * Chỉ số giao dịch: `volume` (Khối lượng khớp), `bid_volume`/`ask_volume` (Dư mua/bán), `foreign_buy`/`foreign_sell` (Khối ngoại).

---

### Màn hình 4: Danh sách theo dõi cá nhân (Watchlist Screen)
* **API Sử dụng**:
  1. `GET /api/watchlists` (Yêu cầu Token)
  2. `DELETE /api/watchlists/:symbol` (Khi nhấn nút Xóa nhanh)
* **Flow gọi API**:
  * Khi mở màn hình: Gọi `GET /api/watchlists` kèm header token.
  * Khi nhấn icon Thùng rác/Unfollow: Gọi `DELETE /api/watchlists/:symbol` -> Nếu thành công, FE tự động xóa hàng (Row) đó khỏi danh sách trên UI mà không cần tải lại toàn bộ trang (Optimistic UI Update).
* **Các trạng thái giao diện (UI States)**:
  * **Empty State**: Hiển thị nút "Khám phá cổ phiếu" dẫn người dùng sang Màn hình Tra cứu cổ phiếu khi watchlist chưa có mã nào.
  * **Unauthorized State**: Nếu nhận lỗi `401 Unauthorized`, tự động xóa token lưu trữ cục bộ và đẩy user về màn hình Đăng nhập.

---

### Màn hình 5: Quản trị tiến trình & Nhật ký (Crawl Operations Admin Dashboard)
* **Quyền hạn**: Chỉ dành cho **STAFF** hoặc **ADMIN**.
* **API Sử dụng**:
  1. `GET /api/admin/crawl-jobs` (Danh sách job)
  2. `POST /api/admin/crawl-jobs/:id/trigger` (Bấm chạy cào ngay)
  3. `GET /api/admin/crawl-logs` (Lịch sử các phiên cào)
  4. `GET /api/admin/crawl-logs/:id/details` (Xem chi tiết từng mã bị lỗi)
  5. `GET /api/admin/crawl-quality` (Biểu đồ tỷ lệ thành công của dữ liệu)
* **Flow tích hợp**:
  * Màn hình chính giám sát hiển thị danh sách các Job tự động. Admin bấm nút "Chạy ngay" (`POST .../trigger`) -> BE chạy crawler nền.
  * Bảng Log lịch sử cào hiển thị danh sách các phiên cào. Nếu phiên cào có trạng thái `PARTIAL_SUCCESS` hoặc `FAILED`, cho phép Admin bấm vào dòng để mở Modal chi tiết gọi API `/details` để tìm nguyên nhân lỗi của từng mã cổ phiếu (e.g. Lỗi Timeout, Lỗi sai URL).

---

## 4. CHECKLIST HOÀN THÀNH TÍCH HỢP CHO FRONTEND

### Auth & Profile
- [ ] Màn hình Đăng nhập thực hiện lưu trữ token vào bộ nhớ an toàn (`AsyncStorage` hoặc `Secure Cookie`).
- [ ] Tạo sẵn class/file API Client (e.g., Axios instance) có gắn tự động Header `Authorization: Bearer <token>`.
- [ ] Tích hợp tự động bắt lỗi `401 Unauthorized` bằng Axios Interceptor để refresh token tự động (hoặc redirect về màn Login).
- [ ] Màn hình Đổi mật khẩu có validate xác nhận mật khẩu mới không trùng mật khẩu cũ.

### Stocks & Watchlists
- [ ] Xử lý định dạng hiển thị số liệu chứng khoán (Ví dụ: `volume` phải định dạng thêm dấu phẩy phân tách nghìn `2,500,000`, giá trị phần trăm thêm ký tự `%`).
- [ ] Thiết lập màu sắc trực quan (Xanh lá / Đỏ) cho các chỉ số thay đổi giá (`price_change` và `price_change_percent`).
- [ ] Biểu đồ lịch sử giá hiển thị đúng các thông số: Open, High, Low, Close, Volume tương ứng với dải ngày đã chọn.
- [ ] Màn hình thêm Watchlist khống chế không cho bấm Thêm nếu độ dài danh sách hiện tại đã đạt 5 mã để tránh cuộc gọi API thừa bị lỗi từ BE.

### Admin & Staff Management
- [ ] Ẩn các nút điều hướng "Vận hành hệ thống" / "Quản lý User" trên menu nếu User đăng nhập có role `USER`.
- [ ] Xử lý hiển thị thông báo lỗi thân thiện khi nhận mã lỗi `403 Forbidden` (Ví dụ: "Bạn không có quyền truy cập khu vực này").
- [ ] Vẽ biểu đồ giám sát chất lượng dữ liệu cào bằng cách map trường `success_rate` và `time_id` từ API `GET /api/admin/crawl-quality`.

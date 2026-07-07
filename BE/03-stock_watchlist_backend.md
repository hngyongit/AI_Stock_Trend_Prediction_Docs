# Backend Module: Stock & Watchlist Management — Quản lý Cổ phiếu & Watchlist

## 1. Mục tiêu

Xây dựng module **Stock & Watchlist Management** cho Backend API của dự án **AI-Based Stock Trend Prediction**, tập trung vào các chức năng:

- Xem danh sách cổ phiếu & Tìm kiếm cổ phiếu.
- Xem thông tin chi tiết cổ phiếu.
- Xem lịch sử giá cổ phiếu phục vụ hiển thị biểu đồ kỹ thuật (OHLCV).
- Quản lý danh sách watchlist cá nhân (thêm, xóa, xem).
- Admin quản lý danh mục cổ phiếu hệ thống.

Module này cung cấp dữ liệu cơ bản cho Web/Mobile App và là cầu nối quan trọng cho các cấu hình crawl job cũng như phân tích dữ liệu ở các giai đoạn sau.

---

## 2. Tech Stack sử dụng

| Thành phần | Công nghệ |
|---|---|
| Runtime | Node.js |
| Framework | Express.js |
| Language | JavaScript |
| Database | MongoDB |
| ODM | Mongoose |
| Authentication | JWT (Bearer Token) |
| API Style | RESTful API |

---

## 3. Phạm vi chức năng

### 3.1. Chức năng chính

| Mã | Chức năng | Đối tượng | Mô tả |
|---|---|---|---|
| STOCK-01 | List Stocks | Public | Xem danh sách cổ phiếu phân trang và lọc theo sàn, ngành |
| STOCK-02 | Detail Stock | Public | Xem chi tiết thông tin công ty và chỉ số tài chính cơ bản |
| STOCK-03 | Stock Price Chart | Public | Lấy lịch sử giá (Open, High, Low, Close, Volume) theo mốc thời gian |
| WATCH-01 | View Watchlist | Protected | Xem danh sách các mã cổ phiếu đang theo dõi cùng giá trị đóng cửa gần nhất |
| WATCH-02 | Add Watchlist | Protected | Thêm mã cổ phiếu vào danh sách theo dõi (Tối đa 5 mã) |
| WATCH-03 | Remove Watchlist | Protected | Xóa mã cổ phiếu khỏi danh sách theo dõi |
| AD-STOCK-01| Add Stock Master | Admin | Thêm mã cổ phiếu mới vào danh mục hệ thống |
| AD-STOCK-02| Update Stock Master | Admin | Cập nhật thông tin công ty, đổi sàn hoặc đổi trạng thái |

---

## 4. Database liên quan

### 4.1. Collection `dim_stocks`

Lưu thông tin danh mục mã cổ phiếu hệ thống.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `market_id` | ObjectId | FK tham chiếu tới `dim_markets` |
| `industry_id` | ObjectId | FK tham chiếu tới `dim_industries` (Tùy chọn) |
| `symbol` | String | Mã cổ phiếu, ví dụ `FPT`, `VNM`, `HPG` |
| `company_name` | String | Tên công ty đầy đủ |
| `exchange_code` | String | Mã sàn giao dịch ví dụ: `HOSE`, `HNX` |
| `status` | String | Trạng thái cổ phiếu: `ACTIVE`, `DELISTED`, `SUSPENDED` |
| `listed_date` | Date | Ngày niêm yết cổ phiếu |
| `created_at` | Date | Ngày tạo record |
| `updated_at` | Date | Ngày cập nhật |

#### Ràng buộc
- `symbol` là unique và viết hoa.

---

### 4.2. Collection `fact_market_prices`

Lưu dữ liệu lịch sử giá khớp lệnh cuối ngày của cổ phiếu.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `stock_id` | ObjectId | FK tham chiếu tới `dim_stocks` |
| `market_id` | ObjectId | FK tham chiếu tới `dim_markets` |
| `industry_id` | ObjectId | FK tham chiếu tới `dim_industries` |
| `data_source_id` | ObjectId | FK tham chiếu tới `dim_data_sources` |
| `time_id` | Number | Định dạng YYYYMMDD đại diện ngày giao dịch (ví dụ `20260601`) |
| `open_price` | Number | Giá mở cửa |
| `high_price` | Number | Giá cao nhất ngày |
| `low_price` | Number | Giá thấp nhất ngày |
| `close_price` | Number | Giá đóng cửa |
| `volume` | Number | Khối lượng giao dịch khớp lệnh |
| `price_change` | Number | Giá trị thay đổi so với phiên trước |
| `price_change_percent`| Number | Phần trăm thay đổi giá |
| `market_cap` | Number | Vốn hóa thị trường |
| `crawled_at` | Date | Thời điểm thu thập dữ liệu |

#### Ràng buộc
- `unique(stock_id, time_id, data_source_id)` để tránh lưu trùng dữ liệu trong cùng một phiên.

---

### 4.3. Collection `watchlists`

Lưu liên kết các mã cổ phiếu yêu thích của người dùng.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `user_id` | ObjectId | FK tham chiếu tới `users` |
| `stock_id` | ObjectId | FK tham chiếu tới `dim_stocks` |
| `created_at` | Date | Thời điểm thêm vào watchlist |

#### Ràng buộc
- `unique(user_id, stock_id)`
- Mỗi `user_id` không được phép liên kết quá **5** record đồng thời.

---

## 5. API Design

## 5.1. Xem danh sách cổ phiếu
Trả về danh sách cổ phiếu hệ thống có hỗ trợ tìm kiếm, lọc theo sàn và phân trang.

### Endpoint
```http
GET /api/stocks
```

### Query Params
```txt
?page=1&limit=10&keyword=fpt&market=HOSE
```

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get stocks successfully",
  "data": {
    "items": [
      {
        "id": "665f1a2b9c1e2a0012a99999",
        "symbol": "FPT",
        "company_name": "Công ty Cổ phần FPT",
        "exchange_code": "HOSE",
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

---

## 5.2. Xem chi tiết cổ phiếu

### Endpoint
```http
GET /api/stocks/:symbol
```

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get stock detail successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99999",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "exchange_code": "HOSE",
    "status": "ACTIVE",
    "listed_date": "2006-12-13T00:00:00.000Z",
    "latest_price": {
      "close_price": 135000,
      "price_change": 1500,
      "price_change_percent": 1.12,
      "volume": 2500000,
      "market_cap": 170000000000000,
      "time_id": 20260601
    }
  }
}
```

### Response thất bại
#### 404 Not Found (Cổ phiếu không tồn tại)
```json
{
  "success": false,
  "message": "Stock symbol not found"
}
```

---

## 5.3. Xem lịch sử giá cổ phiếu (Biểu đồ giá)

### Endpoint
```http
GET /api/stocks/:symbol/chart
```

### Query Params
```txt
?range=1m
```
*Giá trị `range` chấp nhận:* `7d` (7 ngày), `1m` (1 tháng), `3m` (3 tháng), `6m` (6 tháng), `1y` (1 năm), hoặc `all` (tất cả).

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get price history successfully",
  "data": [
    {
      "time": "2026-06-01",
      "open": 133500,
      "high": 136000,
      "low": 133000,
      "close": 135000,
      "volume": 2500000
    }
  ]
}
```

---

## 5.4. Xem Watchlist cá nhân

### Endpoint
```http
GET /api/watchlists
```

### Authorization
```txt
Bearer Access Token
```

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get watchlist successfully",
  "data": [
    {
      "watchlist_id": "665f1a2b9c1e2a0012a99991",
      "stock": {
        "id": "665f1a2b9c1e2a0012a99999",
        "symbol": "FPT",
        "company_name": "Công ty Cổ phần FPT",
        "exchange_code": "HOSE"
      },
      "latest_price": {
        "close_price": 135000,
        "price_change": 1500,
        "price_change_percent": 1.12,
        "volume": 2500000
      },
      "created_at": "2026-06-01T15:00:00.000Z"
    }
  ]
}
```

---

## 5.5. Thêm cổ phiếu vào Watchlist

### Endpoint
```http
POST /api/watchlists
```

### Authorization
```txt
Bearer Access Token
```

### Request Body
```json
{
  "symbol": "FPT"
}
```

### Response thành công (201 Created)
```json
{
  "success": true,
  "message": "Add stock to watchlist successfully",
  "data": {
    "watchlist_id": "665f1a2b9c1e2a0012a99991",
    "symbol": "FPT",
    "created_at": "2026-06-01T18:00:00.000Z"
  }
}
```

### Response thất bại

#### 400 Bad Request (Watchlist đầy hoặc mã cổ phiếu đã tồn tại trong watchlist)
```json
{
  "success": false,
  "message": "Watchlist limit exceeded (Maximum 5 stocks allowed)"
}
```
Hoặc:
```json
{
  "success": false,
  "message": "Stock is already in your watchlist"
}
```

#### 404 Not Found (Mã cổ phiếu không tồn tại trong hệ thống)
```json
{
  "success": false,
  "message": "Stock symbol not found"
}
```

---

## 5.6. Xóa cổ phiếu khỏi Watchlist

### Endpoint
```http
DELETE /api/watchlists/:symbol
```

### Authorization
```txt
Bearer Access Token
```

### Side effects
Khi xóa cổ phiếu khỏi watchlist, **tất cả cảnh báo (alerts) của cổ phiếu đó** cũng bị xóa vĩnh viễn.

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Remove stock from watchlist successfully"
}
```

#### 404 Not Found (Cổ phiếu không có trong Watchlist)
```json
{
  "success": false,
  "message": "Stock not found in your watchlist"
}
```

---

## 5.7. [Admin] Thêm cổ phiếu mới

### Endpoint
```http
POST /api/admin/stocks
```

### Authorization
```txt
Bearer Access Token
```

### Request Body
```json
{
  "symbol": "FPT",
  "company_name": "Công ty Cổ phần FPT",
  "exchange_code": "HOSE",
  "status": "ACTIVE",
  "listed_date": "2006-12-13"
}
```

### Response thành công (210 Created)
```json
{
  "success": true,
  "message": "Create stock master successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99999",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "exchange_code": "HOSE",
    "status": "ACTIVE"
  }
}
```

#### 400 Bad Request (Mã cổ phiếu đã tồn tại)
```json
{
  "success": false,
  "message": "Stock symbol already exists"
}
```

---

## 5.8. [Admin] Cập nhật cổ phiếu

### Endpoint
```http
PUT /api/admin/stocks/:id
```

### Request Body
```json
{
  "company_name": "Công ty Cổ phần FPT Việt Nam",
  "exchange_code": "HOSE",
  "status": "SUSPENDED"
}
```

### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Update stock master successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99999",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT Việt Nam",
    "exchange_code": "HOSE",
    "status": "SUSPENDED"
  }
}
```

---

## 6. Business Rules

| Mã | Nội dung |
|---|---|
| BR-WATCH-01 | Mỗi user chỉ có tối đa 5 mã cổ phiếu trong watchlist cá nhân |
| BR-WATCH-02 | Không được lưu trùng mã cổ phiếu trong watchlist của một user |
| BR-STOCK-01 | Mã cổ phiếu (symbol) phải viết hoa, có độ dài 3 ký tự (hoặc tùy cấu trúc sàn HOSE) |
| BR-STOCK-02 | Dữ liệu lịch sử giá OHLCV không được ghi đè trùng lắp cho cặp `(stock_id, time_id, data_source_id)` |
| BR-STOCK-03 | Chỉ Admin mới có quyền thêm/sửa đổi thông tin master cổ phiếu |

---

## 7. Checklist triển khai BE

- [ ] Tạo `dim-stock.model.js` (nếu chưa có).
- [ ] Tạo `fact-market-price.model.js` (nếu chưa có).
- [ ] Tạo `watchlist.model.js` (nếu chưa có).
- [ ] Seed sẵn danh sách 5 mã cổ phiếu HOSE lớn để làm dữ liệu mẫu (ví dụ: `FPT`, `HPG`, `VNM`, `VIC`, `TCB`).
- [ ] Seed sẵn lịch sử giá OHLCV của 5 mã trên trong vòng 30 ngày gần nhất để vẽ biểu đồ.
- [ ] Viết API `GET /api/stocks` và `GET /api/stocks/:symbol`.
- [ ] Viết API `GET /api/stocks/:symbol/chart`.
- [ ] Viết bộ API quản lý Watchlist (`GET /api/watchlists`, `POST /api/watchlists`, `DELETE /api/watchlists/:symbol`).
- [ ] Viết API Admin quản lý cổ phiếu.
- [ ] Thêm các JSDoc Swagger đầy đủ.
- [ ] Kiểm thử bằng Postman/kịch bản test tự động.

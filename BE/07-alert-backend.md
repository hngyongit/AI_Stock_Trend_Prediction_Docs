# Backend Module: Alert Management — Quản lý cảnh báo

## 1. Mục tiêu

Xây dựng module **Alert Management** cho Backend API của dự án **AI-Based Stock Trend Prediction**, cho phép người dùng:

- Thiết lập cảnh báo giá (`PRICE_ABOVE` / `PRICE_BELOW`) cho cổ phiếu trong watchlist.
- Thiết lập cảnh báo khối lượng giao dịch bất thường (`VOLUME_SPIKE`).
- Nhận email thông báo khi cảnh báo được kích hoạt.
- Quản lý các cảnh báo đã tạo (xem, sửa, xóa, bật/tắt).

Module này bao gồm:

- CRUD cảnh báo (tạo, xem, sửa, xóa).
- Engine kiểm tra điều kiện cảnh báo theo lịch (sau giờ giao dịch).
- Gửi email thông báo qua SMTP.
- Cơ chế phân quyền theo gói (FREE / PRO).
- Cảnh báo một lần (one-shot): sau khi kích hoạt, chuyển trạng thái `TRIGGERED`, người dùng phải kích hoạt lại.

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
| Email | Nodemailer (SMTP) |
| Scheduler | node-cron |
| API Style | RESTful API |

---

## 3. Phạm vi chức năng

### 3.1. Chức năng chính

| Mã | Chức năng | Đối tượng | Mô tả |
|---|---|---|---|
| ALERT-01 | List Alerts | Protected | Xem danh sách các cảnh báo đã tạo kèm thông tin cổ phiếu & giá mới nhất |
| ALERT-02 | Create Alert | Protected | Tạo cảnh báo giá/khối lượng cho cổ phiếu trong watchlist |
| ALERT-03 | Get Alert Detail | Protected | Xem chi tiết một cảnh báo theo ID |
| ALERT-04 | Update Alert | Protected | Cập nhật ngưỡng hoặc bật/tắt cảnh báo |
| ALERT-05 | Delete Alert | Protected | Xóa vĩnh viễn cảnh báo |

### 3.2. Chức năng nền (Background)

| Mã | Chức năng | Mô tả |
|---|---|---|
| ALERT-SCH-01 | Check Alerts | Kiểm tra điều kiện tất cả cảnh báo ACTIVE theo lịch (17:00 các ngày trong tuần) |
| ALERT-SCH-02 | Send Email | Gửi email thông báo khi cảnh báo được kích hoạt |

---

## 4. Database

### 4.1. Collection `alerts`

Lưu thông tin cấu hình cảnh báo của người dùng.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `user_id` | ObjectId | FK tham chiếu tới `users` |
| `stock_id` | ObjectId | FK tham chiếu tới `dim_stocks` |
| `alert_type` | String | Loại cảnh báo: `PRICE_ABOVE`, `PRICE_BELOW`, `VOLUME_SPIKE` |
| `threshold` | Number | Ngưỡng kích hoạt (giá trị VND cho price alert, hệ số nhân cho volume spike) |
| `status` | String | Trạng thái: `ACTIVE`, `TRIGGERED`, `DISABLED` |
| `triggered_at` | Date | Thời điểm kích hoạt gần nhất (null nếu chưa kích hoạt) |
| `triggered_value` | Number | Giá trị thị trường tại thời điểm kích hoạt (close_price hoặc volume) |
| `created_at` | Date | Ngày tạo |
| `updated_at` | Date | Ngày cập nhật gần nhất |

### 4.2. Ràng buộc & Indexes

- Index `{ user_id: 1, status: 1 }` — tối ưu truy vấn danh sách cảnh báo của user.
- Index `{ stock_id: 1, status: 1 }` — tối ưu truy vấn scheduler kiểm tra cảnh báo theo cổ phiếu.
- Mỗi user không được tạo cảnh báo cho cổ phiếu không nằm trong watchlist.
- Giới hạn số lượng cảnh báo theo gói (xem Business Rules).

---

## 5. API Design

### 5.1. Danh sách cảnh báo

Trả về danh sách cảnh báo của user kèm thông tin cổ phiếu và giá đóng cửa gần nhất.

#### Endpoint
```http
GET /api/alerts
```

#### Authorization
```txt
Bearer Access Token
```

#### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get alerts successfully",
  "data": [
    {
      "id": "665f1a2b9c1e2a0012a99991",
      "symbol": "FPT",
      "company_name": "Công ty Cổ phần FPT",
      "alert_type": "PRICE_ABOVE",
      "threshold": 140000,
      "status": "ACTIVE",
      "triggered_at": null,
      "triggered_value": null,
      "latest_price": {
        "close_price": 135000,
        "volume": 2500000
      },
      "created_at": "2026-07-01T10:00:00.000Z",
      "updated_at": "2026-07-01T10:00:00.000Z"
    }
  ]
}
```

---

### 5.2. Tạo cảnh báo mới

#### Endpoint
```http
POST /api/alerts
```

#### Authorization
```txt
Bearer Access Token
```

#### Request Body
```json
{
  "symbol": "FPT",
  "alert_type": "PRICE_ABOVE",
  "threshold": 140000
}
```

#### Response thành công (201 Created)
```json
{
  "success": true,
  "message": "Create alert successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99991",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "alert_type": "PRICE_ABOVE",
    "threshold": 140000,
    "status": "ACTIVE",
    "created_at": "2026-07-01T10:00:00.000Z"
  }
}
```

#### Response thất bại

##### 400 Bad Request (Vượt quá giới hạn gói)
```json
{
  "success": false,
  "message": "Alert stock limit exceeded (Maximum 2 stocks allowed)"
}
```

##### 400 Bad Request (Cổ phiếu không trong watchlist)
```json
{
  "success": false,
  "message": "Stock must be in your watchlist to create an alert"
}
```

##### 404 Not Found (Mã cổ phiếu không tồn tại)
```json
{
  "success": false,
  "message": "Stock symbol not found"
}
```

---

### 5.3. Xem chi tiết cảnh báo

#### Endpoint
```http
GET /api/alerts/:id
```

#### Authorization
```txt
Bearer Access Token
```

#### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Get alert successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99991",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "alert_type": "PRICE_ABOVE",
    "threshold": 140000,
    "status": "ACTIVE",
    "triggered_at": null,
    "triggered_value": null,
    "latest_price": {
      "close_price": 135000,
      "volume": 2500000
    },
    "created_at": "2026-07-01T10:00:00.000Z",
    "updated_at": "2026-07-01T10:00:00.000Z"
  }
}
```

#### 404 Not Found
```json
{
  "success": false,
  "message": "Alert not found"
}
```

---

### 5.4. Cập nhật cảnh báo

Cho phép thay đổi ngưỡng (`threshold`) hoặc chuyển trạng thái:
- `ACTIVE` → `DISABLED`
- `DISABLED` → `ACTIVE`
- `TRIGGERED` → `ACTIVE` (reset lại cảnh báo để sử dụng tiếp)

#### Endpoint
```http
PUT /api/alerts/:id
```

#### Authorization
```txt
Bearer Access Token
```

#### Request Body
```json
{
  "threshold": 145000,
  "status": "DISABLED"
}
```

#### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Update alert successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99991",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "alert_type": "PRICE_ABOVE",
    "threshold": 145000,
    "status": "DISABLED",
    "triggered_at": null,
    "triggered_value": null,
    "created_at": "2026-07-01T10:00:00.000Z",
    "updated_at": "2026-07-02T08:00:00.000Z"
  }
}
```

#### Response thất bại

##### 400 Bad Request (Chuyển trạng thái không hợp lệ)
```json
{
  "success": false,
  "message": "Cannot transition from ACTIVE to TRIGGERED"
}
```

##### 404 Not Found
```json
{
  "success": false,
  "message": "Alert not found"
}
```

---

### 5.5. Xóa cảnh báo

#### Endpoint
```http
DELETE /api/alerts/:id
```

#### Authorization
```txt
Bearer Access Token
```

#### Response thành công (200 OK)
```json
{
  "success": true,
  "message": "Delete alert successfully"
}
```

#### 404 Not Found
```json
{
  "success": false,
  "message": "Alert not found"
}
```

---

## 6. Business Rules

### 6.1. Giới hạn theo gói (Plan Limits)

| Mã | Nội dung | FREE | PRO |
|---|---|---|---|
| BR-ALERT-01 | Số lượng cổ phiếu tối đa có thể đặt cảnh báo | 2 | 50 |
| BR-ALERT-02 | Số lượng cảnh báo tối đa cho mỗi cổ phiếu | 2 | 10 |
| BR-ALERT-03 | Chỉ được đặt cảnh báo cho cổ phiếu trong watchlist | Bắt buộc | Bắt buộc |

### 6.2. Vòng đời cảnh báo (Alert Lifecycle)

```
  ┌──────────┐    User disable    ┌────────────┐
  │  ACTIVE  │ ─────────────────→ │  DISABLED  │
  └────┬─────┘                   └─────┬──────┘
       │                               │
       │ Condition met                  │ User re-enable
       ▼                               │
  ┌──────────┐                         │
  │TRIGGERED │ ←───────────────────────┘
  └──────────┘   User reset (re-enable)
```

- **ACTIVE**: Cảnh báo đang hoạt động, được scheduler kiểm tra định kỳ.
- **TRIGGERED**: Điều kiện đã được kích hoạt, email đã gửi. One-shot — không kiểm tra lại.
- **DISABLED**: Người dùng tắt tạm thời. Scheduler bỏ qua.

### 6.3. Hành vi One-shot

Mỗi cảnh báo chỉ kích hoạt **một lần duy nhất** cho đến khi người dùng reset lại (chuyển từ `TRIGGERED` về `ACTIVE`). Điều này ngăn việc gửi email trùng lặp khi điều kiện vẫn còn đúng trong nhiều phiên giao dịch.

### 6.4. Kiểm tra watchlist

Khi scheduler kích hoạt cảnh báo, nó kiểm tra cổ phiếu vẫn còn trong watchlist của user. Nếu đã bị xóa khỏi watchlist, cảnh báo không được kích hoạt (nhưng vẫn tồn tại trong hệ thống).

---

## 7. Scheduler

### 7.1. Lịch chạy

- **Cron expression**: `0 17 * * 1-5`
- **Thời gian**: 17:00 giờ địa phương, các ngày thứ 2 đến thứ 6
- **Lý do**: Chạy sau giờ giao dịch (sau khi dữ liệu thị trường đã được crawl).

### 7.2. Quy trình kiểm tra

1. Truy vấn tất cả alert có `status: 'ACTIVE'` (kèm thông tin user và stock).
2. Gom nhóm theo `stock_id`, lấy `close_price` và `volume` mới nhất từ `fact_market_prices`.
3. Với `VOLUME_SPIKE`, tính trung bình khối lượng 20 phiên gần nhất.
4. Với mỗi alert, kiểm tra điều kiện:
   - **PRICE_ABOVE**: `close_price >= threshold`
   - **PRICE_BELOW**: `close_price <= threshold`
   - **VOLUME_SPIKE**: `volume / avg_20day_volume >= threshold`
5. Nếu điều kiện đúng: cập nhật `status = TRIGGERED`, `triggered_at = now`, lưu `triggered_value`, gửi email.
6. Email được gửi theo batch 10 email/lần, cách nhau 2 giây để tránh rate limit.

### 7.3. Chạy thủ công (Testing)

```bash
node -e "require('./src/scheduler/alert-checker.scheduler').checkAlerts()"
```

---

## 8. Email Templates

### 8.1. Price Alert Email

**Subject**: `[AI Stock Trend] Price Alert: FPT surpassed 140,000 VND`

```html
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; background: #f9fafb; border-radius: 8px;">
  <h2 style="color: #1f2937;">Price Alert Triggered</h2>
  <p style="color: #374151;">Hi {userName},</p>
  <p style="color: #374151;">Your alert for <strong>FPT</strong> (Công ty Cổ phần FPT) has been triggered:</p>
  <table>
    <tr><td>Alert Type</td><td>Price ABOVE 140,000 VND</td></tr>
    <tr><td>Current Price</td><td>142,000 VND</td></tr>
    <tr><td>Stock</td><td>FPT — Công ty Cổ phần FPT</td></tr>
  </table>
  <p style="color: #9ca3af; font-size: 12px;">This is a one-time alert. Re-enable it in your Alerts settings.</p>
</div>
```

### 8.2. Volume Spike Alert Email

**Subject**: `[AI Stock Trend] Volume Spike Alert: HPG`

```html
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; ...">
  <h2>Volume Spike Alert Triggered</h2>
  <p>Hi {userName},</p>
  <p>Your volume alert for <strong>HPG</strong> has been triggered:</p>
  <table>
    <tr><td>Alert Type</td><td>Volume Spike (≥ 3x average)</td></tr>
    <tr><td>Current Volume</td><td>15,000,000</td></tr>
    <tr><td>Average Volume (20-day)</td><td>4,500,000</td></tr>
    <tr><td>Stock</td><td>HPG — Công ty Cổ phần Tập đoàn Hòa Phát</td></tr>
  </table>
  <p>This is a one-time alert. Re-enable it in your Alerts settings.</p>
</div>
```

---

## 9. Checklist triển khai BE

- [x] Tạo `alert.model.js` — Mongoose schema cho collection `alerts`.
- [x] Cập nhật `plan.config.js` — thêm `max_alert_stocks`, `max_alerts_per_stock`.
- [x] Cập nhật `env.config.js` — thêm SMTP config (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `EMAIL_FROM`).
- [x] Cập nhật `.env.example` — thêm SMTP variables.
- [x] Tạo `alert.validation.js` — express-validator rules cho create/update.
- [x] Tạo `alert.repository.js` — Mongoose queries (CRUD, count, markTriggered).
- [x] Tạo `alert.service.js` — Business logic (plan enforcement, watchlist check, status transitions).
- [x] Tạo `alert.controller.js` — Request handlers.
- [x] Tạo `alert.routes.js` — 5 endpoints `GET/POST /api/alerts`, `GET/PUT/DELETE /api/alerts/:id`.
- [x] Cập nhật `app.js` — register `/api/alerts` routes.
- [x] Cài đặt `nodemailer` và `node-cron`.
- [x] Tạo `services/email.service.js` — send email, HTML templates, no-op fallback.
- [x] Tạo `scheduler/alert-checker.scheduler.js` — cron job, check conditions, batch send.
- [x] Tạo `scheduler/index.js` — entry point `startAllSchedulers()`.
- [x] Cập nhật `server.js` — start schedulers after DB connect.
- [ ] Kiểm thử bằng Postman/kịch bản test tự động.

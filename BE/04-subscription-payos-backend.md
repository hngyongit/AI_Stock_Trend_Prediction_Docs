# Backend Module: Subscription & PayOS Payment — Nâng cấp tài khoản PRO

## 1. Mục tiêu

Xây dựng module **Subscription & PayOS Payment** cho Backend API của dự án **AI-Based Stock Trend Prediction**, cho phép người dùng nâng cấp tài khoản từ **FREE** lên **PRO** với các quyền lợi cao hơn.

Module này bao gồm:

- Cơ chế phân loại tài khoản (`FREE` / `PRO`) với giới hạn tính năng riêng.
- Tích hợp cổng thanh toán **PayOS** để xử lý giao dịch nâng cấp.
- Middleware tự động kiểm tra hết hạn subscription (lazy check).
- Endpoint cắt gọn Watchlist khi hết hạn PRO.
- Cập nhật profile API trả về thông tin gói subscription.

Module này là bước mở rộng từ **MVP** lên phiên bản có **Monetization**, phục vụ Web/Mobile App.

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
| Payment Gateway | PayOS (VN) |
| PayOS SDK | `@payos/node` |
| API Style | RESTful API |

---

## 3. Phạm vi chức năng

### 3.1. Chức năng chính

| Mã | Chức năng | Đối tượng | Mô tả |
|---|---|---|---|
| SUB-01 | Tạo link thanh toán PayOS | USER | Tạo payment request, trả về checkout URL để user thanh toán |
| SUB-02 | Xử lý webhook từ PayOS | Public | Nhận callback từ PayOS khi thanh toán thành công, tự động nâng cấp PRO |
| SUB-03 | Xem trạng thái subscription | USER, STAFF, ADMIN | Xem gói hiện tại, thời hạn, trạng thái subscription |
| SUB-04 | Middleware kiểm tra hết hạn | Hệ thống | Tự động check và downgrade PRO → FREE khi hết hạn |
| WATCH-04 | Trim Watchlist | USER | Cắt bớt stock trong watchlist về đúng giới hạn khi bị downgrade |
| AUTH-08 | JWT chứa thông tin plan | Hệ thống | Access token trả về thêm trường `plan` |
| USER-04 | Profile trả về subscription | USER, STAFF, ADMIN | API user profile trả về `plan`, `subscription_status`, `subscription_expires_at` |

### 3.2. Giới hạn gói

| Tính năng | FREE | PRO |
|---|---|---|
| Watchlist tối đa | 5 stocks | 50 stocks |
| Giá | Miễn phí | 50,000 VND / 30 ngày |

---

## 4. Database liên quan

### 4.1. Collection `users` — Các field mới

Dùng để lưu thông tin gói subscription cho người dùng.

| Field | Type | Mô tả |
|---|---|---|
| `plan` | String | Enum: `FREE`, `PRO`. Mặc định: `FREE` |
| `subscription_status` | String | Enum: `NONE`, `ACTIVE`, `EXPIRED`, `CANCELLED`. Mặc định: `NONE` |
| `subscription_expires_at` | Date | Ngày hết hạn subscription PRO, mặc định `null` |
| `payos_order_code` | Number | Mã đơn hàng PayOS để đối soát webhook, `sparse: true` |
| `payos_payment_link_id` | String | ID link thanh toán PayOS, `sparse: true` |

#### Ràng buộc

```js
plan: { enum: ['FREE', 'PRO'], default: 'FREE' }
subscription_status: { enum: ['NONE', 'ACTIVE', 'EXPIRED', 'CANCELLED'], default: 'NONE' }
```

#### Mô tả trạng thái subscription

| Status | Ý nghĩa |
|---|---|
| `NONE` | Chưa từng đăng ký PRO |
| `ACTIVE` | Đang trong thời gian PRO |
| `EXPIRED` | Đã hết hạn PRO, tự động downgrade |
| `CANCELLED` | Đã hủy (dự phòng cho tương lai) |

---

## 5. Cấu hình

### 5.1. Plan Config (`api/src/config/plan.config.js`)

```js
const PLAN_LIMITS = {
    FREE: { max_watchlist_items: 5 },
    PRO:  { max_watchlist_items: 50 }
};

const SUBSCRIPTION_PRICE = 50000;       // VND
const SUBSCRIPTION_DURATION_DAYS = 30;  // 30 ngày
```

### 5.2. Environment Variables (.env)

Các biến môi trường cần thêm vào `.env`:

| Variable | Mô tả | Ví dụ |
|---|---|---|
| `PAYOS_CLIENT_ID` | Client ID từ PayOS Portal | `abc-123` |
| `PAYOS_API_KEY` | API Key từ PayOS Portal | `xyz-789` |
| `PAYOS_CHECKSUM_KEY` | Checksum Key từ PayOS Portal | `checksum-key` |
| `PAYOS_RETURN_URL` | URL chuyển hướng sau khi thanh toán thành công | `http://localhost:3000/payment/success` |
| `PAYOS_CANCEL_URL` | URL chuyển hướng sau khi hủy thanh toán | `http://localhost:3000/payment/cancel` |
| `PAYOS_PRO_PRICE` | Giá gói PRO (VND), mặc định `50000` | `50000` |

---

## 6. API Design — Subscriptions

Base route:

```txt
/api/subscriptions
```

---

## 6.1. Tạo link thanh toán PayOS

### Endpoint

```http
POST /api/subscriptions/create-payment
```

### Authorization

```txt
Bearer Access Token
```

### Middleware chain

```
authMiddleware → checkSubscriptionExpiry → createPaymentValidation → validate → createPayment
```

### Request Body

```json
{
  "amount": 50000
}
```

> `amount` là optional (mặc định lấy từ config `PAYOS_PRO_PRICE`), tối thiểu 1000 VND.

### Xử lý backend

1. Kiểm tra user tồn tại → `404 Not Found` nếu không.
2. Kiểm tra user đã là PRO và subscription ACTIVE → `400 Bad Request` nếu đã active.
3. Sinh `orderCode` duy nhất từ `Date.now()` + `userId` (4 ký tự cuối).
4. Gọi PayOS API `paymentRequests.create()` để tạo payment link.
5. Lưu `payos_order_code` và `payos_payment_link_id` vào user document.
6. Trả về `checkoutUrl` cho FE chuyển hướng người dùng.

### Response thành công (201 Created)

```json
{
  "success": true,
  "message": "Create payment successfully",
  "data": {
    "checkoutUrl": "https://pay.payos.vn/web/abc123",
    "orderCode": 17815233668565,
    "paymentLinkId": "pm_abc123",
    "amount": 50000
  }
}
```

### Response thất bại

#### 400 Bad Request (User đã PRO active)
```json
{
  "success": false,
  "message": "User is already on PRO plan"
}
```

#### 404 Not Found (User không tồn tại)
```json
{
  "success": false,
  "message": "User not found"
}
```

---

## 6.2. Webhook callback từ PayOS

### Endpoint

```http
POST /api/subscriptions/webhook
```

### Quyền truy cập

**Public** — Endpoint này không yêu cầu authentication.

### Body parsing

Route này sử dụng `express.raw({ type: 'application/json' })` để nhận **raw Buffer** (đặt trước `express.json()` global để tránh body bị parse trước).

Controller hỗ trợ 2 format payload:

#### Format 1 — PayOS chính thức

```json
{
  "data": {
    "orderCode": 17815233668565,
    "amount": 50000,
    "description": "Nâng cấp PRO",
    "status": "PAID"
  },
  "signature": "abc123signature"
}
```

→ SDK `payOS.webhooks.verify()` sẽ kiểm tra signature.

#### Format 2 — Đơn giản hóa (test thủ công)

```json
{
  "orderCode": 17815233668565,
  "status": "PAID"
}
```

→ Bỏ qua verify signature, xử lý trực tiếp.

### Xử lý backend

1. Parse body thành JS object (nếu là Buffer).
2. Nếu payload có `data` và `signature` → gọi `payOS.webhooks.verify()`.
3. Nếu payload có `orderCode` → dùng trực tiếp.
4. Tìm user theo `payos_order_code`.
5. Nếu `status === 'PAID'`:
   - Set `plan = 'PRO'`.
   - Set `subscription_status = 'ACTIVE'`.
   - Set `subscription_expires_at = now + 30 days`.
   - Save user document.
6. Luôn trả về `200 OK` dù thành công hay thất bại (PayOS yêu cầu vậy).

### Response thành công (200 OK)

```json
{
  "success": true,
  "message": "Webhook processed successfully",
  "data": {
    "userId": "665f1a2b9c1e2a0012a12345",
    "plan": "PRO",
    "subscriptionStatus": "ACTIVE",
    "subscriptionExpiresAt": "2026-07-15T10:00:00.000Z"
  }
}
```

### Response webhook lỗi (200 OK — PayOS luôn nhận 200)

```json
{
  "success": false,
  "message": "Webhook processed with error",
  "error": "User not found for this payment"
}
```

---

## 6.3. Xem trạng thái subscription

### Endpoint

```http
GET /api/subscriptions/status
```

### Authorization

```txt
Bearer Access Token
```

### Response thành công (200 OK)

```json
{
  "success": true,
  "message": "Get subscription status successfully",
  "data": {
    "plan": "PRO",
    "subscriptionStatus": "ACTIVE",
    "subscriptionExpiresAt": "2026-07-15T10:00:00.000Z"
  }
}
```

### Response cho user FREE

```json
{
  "success": true,
  "message": "Get subscription status successfully",
  "data": {
    "plan": "FREE",
    "subscriptionStatus": "NONE",
    "subscriptionExpiresAt": null
  }
}
```

---

## 7. API Design — Watchlist (Thay đổi)

### 7.1. Xem Watchlist (GET /api/watchlists) — Có thay đổi

Khi user đã hết hạn PRO (`overLimit = true`), response trả về dạng rút gọn chỉ với `stock_id`, `stock_code`, `stock_name` để FE hiển thị danh sách stock cần giữ lại.

### Response thành công khi trong giới hạn (200 OK)

```json
{
  "success": true,
  "message": "Get watchlist successfully",
  "data": {
    "items": [
      {
        "watchlist_id": "665f1a2b9c1e2a0012a99991",
        "stock": {
          "id": "665f1a2b9c1e2a0012a99999",
          "symbol": "FPT",
          "company_name": "Công ty Cổ phần FPT",
          "market_id": "6a1ff9e6e7c74fdfdcb76db5",
          "market_code": "HOSE"
        },
        "latest_price": {
          "close_price": 135000,
          "price_change": 1500,
          "price_change_percent": 1.12,
          "volume": 2500000
        },
        "created_at": "2026-06-01T15:00:00.000Z"
      }
    ],
    "limit": 5,
    "currentCount": 3,
    "overLimit": false
  }
}
```

### Response khi vượt giới hạn (overLimit = true)

```json
{
  "success": true,
  "message": "Get watchlist successfully",
  "data": {
    "items": [
      {
        "stock_id": "665f1a2b9c1e2a0012a99999",
        "stock_code": "FPT",
        "stock_name": "Công ty Cổ phần FPT"
      }
    ],
    "limit": 5,
    "currentCount": 15,
    "overLimit": true
  }
}
```

> **Lưu ý FE:** Khi `overLimit = true`, items trả về không có `watchlist_id` hay `latest_price`. FE cần hiển thị màn hình "Cắt gọn Watchlist" để user chọn stock giữ lại, sau đó gọi `POST /api/watchlists/trim`.

---

### 7.2. Thêm Watchlist (POST /api/watchlists) — Giới hạn động

Giới hạn watchlist thay đổi theo plan:

| Plan | Giới hạn |
|---|---|
| FREE | Tối đa 5 stocks |
| PRO | Tối đa 50 stocks |

### Response lỗi khi quá giới hạn (400 Bad Request)

```json
{
  "success": false,
  "message": "Watchlist limit exceeded (Maximum 5 stocks allowed)"
}
```

---

### 7.3. Trim Watchlist (POST /api/watchlists/trim) — Mới

### Endpoint

```http
POST /api/watchlists/trim
```

### Authorization

```txt
Bearer Access Token
```

### Middleware chain

```
authMiddleware → checkSubscriptionExpiry → trimWatchlistValidation → validate → trimWatchlist
```

### Request Body

```json
{
  "keepStockIds": [
    "665f1a2b9c1e2a0012a99999",
    "665f1a2b9c1e2a0012a99998"
  ]
}
```

- `keepStockIds`: Mảng các ObjectId của stock muốn giữ lại.
- Số lượng `keepStockIds` không được vượt quá giới hạn của plan hiện tại.
- Các stock không nằm trong mảng này sẽ bị xóa khỏi watchlist.

### Xử lý backend

1. Validate `keepStockIds` là non-empty array, mỗi phần tử là string.
2. Lấy giới hạn watchlist theo `userPlan`.
3. Kiểm tra `keepStockIds.length <= limit` → nếu vượt thì trả lỗi.
4. Lấy danh sách stock hiện tại trong watchlist.
5. Tính `idsToDelete = currentStockIds - keepStockIds`.
6. Gọi `Watchlist.deleteMany()` để xóa các stock không được giữ.
7. Trả về số lượng đã xóa và số lượng còn lại.

### Response thành công (200 OK)

```json
{
  "success": true,
  "message": "Trim watchlist successfully",
  "data": {
    "deletedCount": 13,
    "remainingCount": 2
  }
}
```

### Response thất bại

#### 400 Bad Request (keepStockIds vượt quá giới hạn)
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "keepStockIds",
      "message": "keepStockIds must be a non-empty array"
    }
  ]
}
```

---

## 8. Subscription Middleware

### Middleware: `checkSubscriptionExpiry`

**File:** `api/src/common/middlewares/subscription.middleware.js`

Middleware này chạy **sau** `auth.middleware` và **trước** các controller cần kiểm tra subscription.

### Logic xử lý

1. Chỉ kiểm tra user có `plan === 'PRO'` và `subscription_status === 'ACTIVE'`.
2. Nếu `subscription_expires_at < now`:
   - Downgrade user xuống `plan = 'FREE'`.
   - Set `subscription_status = 'EXPIRED'`.
   - Lưu xuống database.
   - Cập nhật `req.user.plan` và `req.user.subscription_status`.
3. Nếu chưa hết hạn hoặc không phải PRO → `next()`.

### Các route sử dụng middleware này

- `POST /api/subscriptions/create-payment` — Check expiry trước khi tạo payment mới.
- `GET /api/watchlists` — Check expiry trước khi lấy danh sách.
- `POST /api/watchlists/trim` — Check expiry trước khi trim.

> **Lưu ý:** Đây là lazy check — chỉ kiểm tra khi user gọi API, không có background job.

---

## 9. API Design — User Profile (Thay đổi)

### 9.1. Xem thông tin cá nhân (GET /api/users/me) — Có thay đổi

Response thêm 3 trường subscription:

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
    "plan": "FREE",
    "subscription_status": "NONE",
    "subscription_expires_at": null,
    "created_at": "2026-06-01T10:00:00.000Z"
  }
}
```

### 9.2. JWT Access Token — Thêm trường `plan`

Token payload:

```js
{
  id: "...",
  email: "...",
  role: "...",
  plan: "FREE"    // hoặc "PRO"
}
```

> FE có thể dùng trường `plan` từ access token để hiển thị trạng thái mà không cần gọi thêm API.

---

## 10. Cấu trúc thư mục module

```txt
api/src/
├── config/
│   └── plan.config.js                          # Plan limits & subscription config (MỚI)
├── common/
│   └── middlewares/
│       └── subscription.middleware.js           # checkSubscriptionExpiry middleware (MỚI)
├── modules/
│   └── subscriptions/                           # Module mới (MỚI)
│       ├── payos.service.js                     # PayOS client wrapper
│       ├── subscriptions.service.js             # Business logic
│       ├── subscriptions.controller.js          # Route handlers
│       ├── subscriptions.routes.js              # Routes + Swagger docs
│       └── subscriptions.validation.js          # Validation rules
│   └── watchlists/
│       ├── watchlists.service.js                # Cập nhật: getUserWatchlist, trimWatchlist, plan limits
│       ├── watchlists.repository.js             # Thêm: deleteMultipleWatchlistEntries
│       ├── watchlists.controller.js             # Thêm: trimWatchlist
│       ├── watchlists.validation.js             # Thêm: trimWatchlistValidation
│       └── watchlists.routes.js                 # Thêm: POST /trim route
```

---

## 11. Business Rules

| Mã | Nội dung |
|---|---|
| BR-SUB-01 | User FREE có watchlist tối đa 5 stocks |
| BR-SUB-02 | User PRO có watchlist tối đa 50 stocks |
| BR-SUB-03 | Subscription PRO có thời hạn 30 ngày kể từ lúc thanh toán |
| BR-SUB-04 | Khi PRO hết hạn, user tự động bị downgrade xuống FREE ở lần gọi API tiếp theo |
| BR-SUB-05 | Khi bị downgrade, watchlist vượt quá 5 stocks sẽ được đánh dấu `overLimit: true` |
| BR-SUB-06 | User phải dùng API `/trim` để cắt gọn watchlist trước khi có thể thêm stock mới |
| BR-SUB-07 | PayOS webhook luôn trả về status 200 dù có lỗi xử lý (PayOS requirement) |
| BR-SUB-08 | Mỗi user chỉ được có 1 subscription ACTIVE tại một thời điểm |

---

## 12. Checklist triển khai BE

- [x] Thêm các field subscription vào `user.model.js` (`plan`, `subscription_status`, `subscription_expires_at`, `payos_order_code`, `payos_payment_link_id`).
- [x] Tạo `plan.config.js` với `PLAN_LIMITS`, `SUBSCRIPTION_PRICE`, `SUBSCRIPTION_DURATION_DAYS`.
- [x] Thêm biến môi trường PayOS vào `env.config.js`.
- [x] Thêm `plan` vào JWT payload trong `jwt.util.js`.
- [x] Tạo `payos.service.js` (getPayOSClient, createPaymentRequest, verifyWebhook).
- [x] Tạo `subscriptions.service.js` (createPayment, handlePaymentWebhook, getSubscriptionStatus).
- [x] Tạo `subscriptions.controller.js` (createPayment, handleWebhook, getStatus).
- [x] Tạo `subscriptions.routes.js` (POST /create-payment, POST /webhook, GET /status).
- [x] Tạo `subscriptions.validation.js`.
- [x] Tạo `subscription.middleware.js` (checkSubscriptionExpiry).
- [x] Cập nhật `app.js`: mount raw body parser cho webhook, mount subscriptionsRouter.
- [x] Cập nhật `watchlists.service.js`: tham số userPlan, overLimit detection, trimWatchlist.
- [x] Cập nhật `watchlists.repository.js`: deleteMultipleWatchlistEntries.
- [x] Cập nhật `watchlists.controller.js`: trimWatchlist handler.
- [x] Cập nhật `watchlists.validation.js`: trimWatchlistValidation.
- [x] Cập nhật `watchlists.routes.js`: POST /trim với subscriptionMiddleware.
- [x] Cập nhật `users.controller.js` formatUser: thêm plan, subscription_status, subscription_expires_at.
- [x] Cài đặt package `@payos/node`.
- [x] Viết API docs và FE integration guide.

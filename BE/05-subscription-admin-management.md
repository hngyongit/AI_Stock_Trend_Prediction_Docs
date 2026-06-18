# Backend Module: Subscription Management for Admin & Staff

## 1. Mục tiêu

Xây dựng module **Subscription Management** dành riêng cho **ADMIN** và **STAFF** trên Backend API của dự án **AI-Based Stock Trend Prediction**, cung cấp khả năng quản lý, giám sát và can thiệp thủ công vào subscription của người dùng.

Module này là phần mở rộng của module **Subscription & PayOS Payment** (`04-subscription-payos-backend.md`), cho phép:

- ADMIN quản lý toàn bộ subscription (xem, lọc, tìm kiếm, can thiệp thủ công).
- STAFF xem thông tin subscription (read-only) để hỗ trợ người dùng.
- ADMIN theo dõi doanh thu và thống kê subscription.

---

## 2. Phân quyền (Role-Based Access)

| Role   | Quyền hạn với Subscription Management               |
|--------|------------------------------------------------------|
| ADMIN  | Full quyền: xem, tìm kiếm, lọc, thay đổi, thống kê   |
| STAFF  | Read-only: xem danh sách, xem chi tiết               |
| USER   | Không có quyền truy cập                              |

> **Nguyên tắc:** Mọi thay đổi subscription từ ADMIN đều được ghi lại trong **Subscription Transactions** (lịch sử giao dịch).

---

## 3. Danh sách chức năng

| Mã       | Chức năng                                         | Đối tượng | Mô tả |
|----------|---------------------------------------------------|-----------|-------|
| ADSUB-01 | Xem danh sách subscription (phân trang, lọc)       | ADMIN, STAFF | Danh sách tất cả user kèm thông tin subscription, hỗ trợ filter, search |
| ADSUB-02 | Xem chi tiết subscription của một user             | ADMIN, STAFF | Xem đầy đủ thông tin subscription + lịch sử thay đổi |
| ADSUB-03 | Tìm kiếm subscription                              | ADMIN, STAFF | Tìm kiếm theo email, tên, order code |
| ADSUB-04 | Thống kê tổng quan subscription                    | ADMIN        | Dashboard: tổng user, PRO đang active, sắp hết hạn, doanh thu |
| ADSUB-05 | Lịch sử giao dịch (transaction log)                | ADMIN        | Danh sách các giao dịch PayOS đã xử lý |
| ADSUB-06 | Gia hạn subscription thủ công                      | ADMIN        | Gia hạn PRO cho user + số ngày tuỳ chọn |
| ADSUB-07 | Huỷ subscription thủ công                          | ADMIN        | Downgrade PRO → FREE ngay lập tức |
| ADSUB-08 | Cập nhật hạn dùng subscription                     | ADMIN        | Thay đổi `subscription_expires_at` thủ công |
| ADSUB-09 | Export danh sách subscription                      | ADMIN        | Xuất danh sách subscription ra CSV (tuỳ chọn phase sau) |

---

## 4. Database thay đổi

### 4.1. Collection `users` — KHÔNG cần thay đổi

Các field subscription hiện tại trong `user.model.js` đã đủ:

```js
plan:                { type: String, enum: ['FREE', 'PRO'], default: 'FREE' }
subscription_status: { type: String, enum: ['NONE', 'ACTIVE', 'EXPIRED', 'CANCELLED'], default: 'NONE' }
subscription_expires_at: { type: Date, default: null }
payos_order_code:    { type: Number, default: null, sparse: true }
payos_payment_link_id: { type: String, default: null, sparse: true }
```

> **Ghi chú:** Có thể thêm `subscription_cancelled_at` (Date) nếu cần lưu thời điểm huỷ, nhưng chưa cần ở phase này.

### 4.2. Collection mới: `subscription_transactions`

Lưu lịch sử giao dịch PayOS — phục vụ cho ADMIN tra cứu và thống kê doanh thu.

```js
const SubscriptionTransactionSchema = new mongoose.Schema(
  {
    user_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
      index: true
    },
    transaction_type: {
      type: String,
      enum: ['PAYOS_PAYMENT', 'ADMIN_GRANT', 'ADMIN_RENEW', 'ADMIN_CANCEL', 'ADMIN_MODIFY'],
      required: true
    },
    // PayOS fields
    payos_order_code: {
      type: Number,
      default: null,
      sparse: true
    },
    payos_payment_link_id: {
      type: String,
      default: null
    },
    amount: {
      type: Number,
      default: 0   // VND
    },
    // Trạng thái
    status: {
      type: String,
      enum: ['PAID', 'CANCELLED', 'REFUNDED', 'GRANTED', 'EXPIRED'],
      required: true
    },
    // Thông tin subscription trước và sau
    previous_plan: { type: String, enum: ['FREE', 'PRO'], default: 'FREE' },
    new_plan:      { type: String, enum: ['FREE', 'PRO'], default: 'FREE' },
    previous_expires_at: { type: Date, default: null },
    new_expires_at:      { type: Date, default: null },
    // Người thực hiện (nếu là ADMIN can thiệp)
    performed_by: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      default: null
    },
    // Ghi chú / lý do
    notes: {
      type: String,
      default: ''
    }
  },
  {
    timestamps: {
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    }
  }
);

// Index cho tra cứu nhanh
SubscriptionTransactionSchema.index({ user_id: 1, created_at: -1 });
SubscriptionTransactionSchema.index({ payos_order_code: 1 }, { sparse: true });
SubscriptionTransactionSchema.index({ status: 1 });
SubscriptionTransactionSchema.index({ created_at: -1 });
```

#### Giải thích `transaction_type`:

| Type              | Ý nghĩa                                          | Nguồn        |
|-------------------|---------------------------------------------------|--------------|
| `PAYOS_PAYMENT`   | User thanh toán qua PayOS, tự động nâng cấp PRO   | Webhook      |
| `ADMIN_GRANT`     | ADMIN cấp PRO miễn phí cho user                   | Admin action |
| `ADMIN_RENEW`     | ADMIN gia hạn PRO cho user                        | Admin action |
| `ADMIN_CANCEL`    | ADMIN huỷ PRO của user                            | Admin action |
| `ADMIN_MODIFY`    | ADMIN thay đổi hạn dùng                           | Admin action |



---

## 5. API Design

Base route:

```txt
/api/admin/subscriptions     → ADMIN (full quyền)
/api/staff/subscriptions     → STAFF (read-only)
```

### 5.1. ADMIN APIs

#### 5.1.1. Danh sách subscription

```http
GET /api/admin/subscriptions
Authorization: Bearer <ADMIN_TOKEN>
```

**Query Parameters:**

| Param    | Type   | Mô tả                                    |
|----------|--------|------------------------------------------|
| `page`   | int    | Số trang, default: 1                     |
| `limit`  | int    | Số lượng, default: 20, max: 100          |
| `keyword`| string | Tìm kiếm theo email, full_name           |
| `plan`   | string | Filter: `FREE`, `PRO`                    |
| `status` | string | Filter subscription: `NONE`, `ACTIVE`, `EXPIRED`, `CANCELLED` |
| `role`   | string | Filter role: `USER`, `STAFF`, `ADMIN`    |
| `sort_by`| string | Sắp xếp: `created_at`, `subscription_expires_at`, `plan` |
| `sort_order` | string | `asc`, `desc` (default: `desc`)      |

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Get subscriptions successfully",
  "data": {
    "items": [
      {
        "id": "665f1a2b9c1e2a0012a12345",
        "full_name": "Nguyen Van A",
        "email": "user@example.com",
        "role": "USER",
        "status": "ACTIVE",
        "plan": "PRO",
        "subscription_status": "ACTIVE",
        "subscription_expires_at": "2026-07-15T10:00:00.000Z",
        "created_at": "2026-06-01T10:00:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_items": 150,
      "total_pages": 8
    },
    "summary": {
      "total_users": 150,
      "active_pro": 45,
      "expired_pro": 12,
      "free_users": 93
    }
  }
}
```

#### 5.1.2. Chi tiết subscription của user

```http
GET /api/admin/subscriptions/:userId
Authorization: Bearer <ADMIN_TOKEN>
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Get subscription detail successfully",
  "data": {
    "user": {
      "id": "665f1a2b9c1e2a0012a12345",
      "full_name": "Nguyen Van A",
      "email": "user@example.com",
      "role": "USER",
      "status": "ACTIVE"
    },
    "subscription": {
      "plan": "PRO",
      "status": "ACTIVE",
      "expires_at": "2026-07-15T10:00:00.000Z",
      "payos_order_code": 17815233668565,
      "payos_payment_link_id": "pm_abc123"
    },
    "transactions": [
      {
        "id": "...",
        "type": "PAYOS_PAYMENT",
        "amount": 50000,
        "status": "PAID",
        "created_at": "2026-06-15T10:00:00.000Z"
      }
    ]
  }
}
```

#### 5.1.3. Gia hạn subscription thủ công

```http
POST /api/admin/subscriptions/:userId/renew
Authorization: Bearer <ADMIN_TOKEN>
```

**Request Body:**

```json
{
  "duration_days": 30,
  "notes": "Gia hạn hỗ trợ khách hàng theo yêu cầu"
}
```

**Xử lý:**
1. Kiểm tra user tồn tại → `404` nếu không.
2. Nếu user đang FREE → set `plan = 'PRO'`, `subscription_status = 'ACTIVE'`.
3. Tính `subscription_expires_at`:
   - Nếu user đang ACTIVE: cộng thêm `duration_days` vào ngày hết hạn hiện tại.
   - Nếu user đang EXPIRED hoặc FREE: set từ hôm nay + `duration_days`.
4. Ghi `subscription_transactions` với type `ADMIN_RENEW`.
5. Trả về thông tin subscription mới.

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Subscription renewed successfully",
  "data": {
    "user_id": "...",
    "plan": "PRO",
    "subscription_status": "ACTIVE",
    "subscription_expires_at": "2026-08-15T10:00:00.000Z",
    "duration_days": 30
  }
}
```

#### 5.1.4. Cấp PRO miễn phí (Grant)

```http
POST /api/admin/subscriptions/:userId/grant
Authorization: Bearer <ADMIN_TOKEN>
```

**Request Body:**

```json
{
  "duration_days": 30,
  "notes": "Tặng PRO cho user chiến lược"
}
```

Giống Renew nhưng:
- Chỉ áp dụng cho user đang FREE hoặc EXPIRED.
- Nếu user đang ACTIVE → báo lỗi `400` (yêu cầu dùng Renew).
- Type = `ADMIN_GRANT`.

#### 5.1.5. Huỷ subscription thủ công

```http
POST /api/admin/subscriptions/:userId/cancel
Authorization: Bearer <ADMIN_TOKEN>
```

**Request Body:**

```json
{
  "notes": "User yêu cầu huỷ subscription"
}
```

**Xử lý:**
1. Kiểm tra user tồn tại → `404`.
2. Kiểm tra user đang PRO ACTIVE → nếu không, báo lỗi `400`.
3. Set `plan = 'FREE'`, `subscription_status = 'CANCELLED'`, `subscription_expires_at = null`.
4. Ghi `subscription_transactions` với type `ADMIN_CANCEL`.

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Subscription cancelled successfully",
  "data": {
    "user_id": "...",
    "plan": "FREE",
    "subscription_status": "CANCELLED",
    "subscription_expires_at": null
  }
}
```

#### 5.1.6. Thay đổi hạn dùng

```http
PATCH /api/admin/subscriptions/:userId/expiry
Authorization: Bearer <ADMIN_TOKEN>
```

**Request Body:**

```json
{
  "expires_at": "2026-09-15T10:00:00.000Z",
  "notes": "Điều chỉnh hạn dùng theo yêu cầu"
}
```

**Xử lý:**
1. Kiểm tra user tồn tại → `404`.
2. Kiểm tra user đang PRO → nếu không, báo lỗi `400`.
3. Cập nhật `subscription_expires_at`.
4. Ghi `subscription_transactions` với type `ADMIN_MODIFY`.

#### 5.1.7. Thống kê Subscription Dashboard

```http
GET /api/admin/subscriptions/stats
Authorization: Bearer <ADMIN_TOKEN>
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Get subscription stats successfully",
  "data": {
    "overview": {
      "total_users": 150,
      "active_pro": 45,
      "expired_pro": 12,
      "cancelled_pro": 3,
      "free_users": 90,
      "pro_percentage": 30.0
    },
    "expiring_soon": {
      "within_7_days": 5,
      "within_30_days": 18
    },
    "revenue": {
      "current_month": 2500000,
      "last_month": 2000000,
      "total_all_time": 15000000,
      "currency": "VND"
    },
    "recent_transactions": [
      {
        "id": "...",
        "user": { "id": "...", "full_name": "Nguyen Van A", "email": "user@example.com" },
        "amount": 50000,
        "type": "PAYOS_PAYMENT",
        "status": "PAID",
        "created_at": "2026-06-18T10:00:00.000Z"
      }
    ]
  }
}
```

#### 5.1.8. Lịch sử giao dịch

```http
GET /api/admin/subscriptions/transactions
Authorization: Bearer <ADMIN_TOKEN>
```

**Query Parameters:**

| Param     | Type   | Mô tả                                          |
|-----------|--------|------------------------------------------------|
| `page`    | int    | Số trang, default: 1                           |
| `limit`   | int    | Số lượng, default: 20                          |
| `user_id` | string | Filter theo user                                |
| `type`    | string | Filter: `PAYOS_PAYMENT`, `ADMIN_GRANT`, ...    |
| `status`  | string | Filter: `PAID`, `CANCELLED`, ...               |
| `from`    | date   | Từ ngày (ISO format)                           |
| `to`      | date   | Đến ngày (ISO format)                          |

### 5.2. STAFF APIs (Read-only)

#### 5.2.1. Danh sách subscription

```http
GET /api/staff/subscriptions
Authorization: Bearer <STAFF_TOKEN>
```

> Giống hệt `GET /api/admin/subscriptions` nhưng **KHÔNG** có `summary` và chỉ ADMIN mới có `summary`.

#### 5.2.2. Chi tiết subscription

```http
GET /api/staff/subscriptions/:userId
Authorization: Bearer <STAFF_TOKEN>
```

> Giống hệt `GET /api/admin/subscriptions/:userId` nhưng **KHÔNG** có transactions — STAFF chỉ xem thông tin cơ bản của user.

#### 5.2.3. Tìm kiếm subscription

```http
GET /api/staff/subscriptions/search?keyword=...
Authorization: Bearer <STAFF_TOKEN>
```

---

## 6. Middleware & Validation

### 6.1. Routes configuration

Sử dụng các middleware đã có:

| Middleware              | Mục đích                                    |
|-------------------------|---------------------------------------------|
| `authMiddleware`        | Xác thực JWT, load user từ DB               |
| `roleMiddleware(['ADMIN'])` | Giới hạn ADMIN cho các route can thiệp  |
| `roleMiddleware(['ADMIN', 'STAFF'])` | Cho phép cả ADMIN và STAFF |

### 6.2. Validation rules

```js
// admin-subscriptions.validation.js
const renewGrantValidation = [
    body('duration_days')
        .isInt({ min: 1, max: 365 })
        .withMessage('Duration days must be between 1 and 365'),
    body('notes')
        .optional()
        .isString().trim()
        .isLength({ max: 500 })
];

const cancelValidation = [
    body('notes')
        .optional()
        .isString().trim()
        .isLength({ max: 500 })
];

const modifyExpiryValidation = [
    body('expires_at')
        .isISO8601()
        .withMessage('Expires at must be a valid ISO 8601 date'),
    body('notes')
        .optional()
        .isString().trim()
        .isLength({ max: 500 })
];
```

---

## 7. Cấu trúc thư mục

```
api/src/modules/admin-subscriptions/      # Module mới
├── admin-subscriptions.controller.js      # Handler cho ADMIN routes
├── admin-subscriptions.service.js         # Business logic cho ADMIN
├── admin-subscriptions.routes.js          # Route definitions cho ADMIN
└── admin-subscriptions.validation.js      # Validation rules

api/src/modules/staff-subscriptions/       # Module mới
├── staff-subscriptions.controller.js      # Handler cho STAFF routes
├── staff-subscriptions.service.js         # Business logic cho STAFF
├── staff-subscriptions.routes.js          # Route definitions cho STAFF
└── staff-subscriptions.validation.js      # Validation rules (nếu cần)

api/src/database/models/
└── subscription-transaction.model.js      # Model mới
```

> **Lưu ý:** Có thể gộp ADMIN và STAFF vào cùng module `admin-subscriptions` với roleMiddleware để phân quyền, nhưng tách riêng giúp dễ maintain và rõ ràng hơn.

---

## 8. Luồng xử lý chi tiết

### 8.1. Luồng ADMIN gia hạn subscription

```
[ADMIN] → POST /api/admin/subscriptions/:userId/renew
         → authMiddleware (verify JWT)
         → roleMiddleware(['ADMIN'])
         → validation (duration_days, notes)
         → adminSubscriptionsController.renew
           → adminSubscriptionsService.renewSubscription(userId, durationDays, notes, adminId)
             → Kiểm tra user tồn tại
             → Nếu ACTIVE: expires_at = max(now + durationDays, current_expires_at + durationDays)
             → Nếu EXPIRED/FREE: expires_at = now + durationDays
             → Update user: plan='PRO', status='ACTIVE', expires_at
             → Tạo subscription_transaction (type=ADMIN_RENEW, performed_by=adminId)
           ← Response: thông tin subscription mới
```

### 8.2. Luồng ADMIN huỷ subscription

```
[ADMIN] → POST /api/admin/subscriptions/:userId/cancel
         → authMiddleware
         → roleMiddleware(['ADMIN'])
         → adminSubscriptionsController.cancel
           → adminSubscriptionsService.cancelSubscription(userId, notes, adminId)
             → Kiểm tra user tồn tại
             → Kiểm tra user đang PRO ACTIVE, nếu không → 400
             → Set plan='FREE', status='CANCELLED', expires_at=null
             → Tạo subscription_transaction (type=ADMIN_CANCEL)
```

### 8.3. Luồng PayOS webhook (cập nhật ghi transaction)

Khi webhook PayOS được gọi, bên cạnh việc update user như hiện tại, cần **ghi thêm** một record vào `subscription_transactions`:

```
[PayOS] → POST /api/subscriptions/webhook
         → subscriptionsController.handleWebhook
           → subscriptionsService.handlePaymentWebhook(webhookData)
             → (Code hiện tại: update user)
             → (MỚI:) Tạo subscription_transaction {
                 user_id, transaction_type: 'PAYOS_PAYMENT',
                 payos_order_code, amount, status: 'PAID',
                 previous_plan: 'FREE', new_plan: 'PRO',
                 previous_expires_at: null, new_expires_at: expiresAt
               }
```

---

## 9. Cập nhật App.js (routes)

Thêm vào `api/src/app.js`:

```js
const adminSubscriptionsRouter = require('./modules/admin-subscriptions/admin-subscriptions.routes');
const staffSubscriptionsRouter = require('./modules/staff-subscriptions/staff-subscriptions.routes');

// Trong phần API Routes:
app.use('/api/admin/subscriptions', adminSubscriptionsRouter);
app.use('/api/staff/subscriptions', staffSubscriptionsRouter);
```

---

## 10. Mở rộng seed data

Bổ sung vào `seed-roles.js` (hoặc tạo `seed-transactions.js` riêng) để có dữ liệu mẫu:

```js
// Seed subscription_transactions cho các user đã có sẵn
const transactionsData = [
  {
    user: 'pro@example.com',
    type: 'PAYOS_PAYMENT',
    amount: 50000,
    status: 'PAID',
    previous_plan: 'FREE',
    new_plan: 'PRO'
  },
  {
    user: 'expired@example.com',
    type: 'PAYOS_PAYMENT',
    amount: 50000,
    status: 'PAID',
    previous_plan: 'FREE',
    new_plan: 'PRO'
  }
];
```

---

## 11. Triển khai theo phase

| Phase | Nội dung                                                | Ưu tiên |
|-------|---------------------------------------------------------|---------|
| Phase | Nội dung                                                | Ưu tiên |
|-------|---------------------------------------------------------|---------|
| 1     | Model: `subscription-transaction.model.js`              | Cao |
| 1     | Cập nhật webhook handler để ghi transaction              | Cao |
| 1     | ADMIN: Danh sách subscription + Chi tiết                | Cao |
| 2     | ADMIN: Gia hạn, Cấp mới, Huỷ subscription               | Cao |
| 2     | ADMIN: Thay đổi hạn dùng                                 | Trung bình |
| 3     | STAFF: Danh sách + Chi tiết (read-only)                 | Trung bình |
| 3     | ADMIN: Thống kê Dashboard + Lịch sử giao dịch           | Trung bình |
| 4     | ADMIN: Export CSV                                        | Thấp |

---

## 12. Các vấn đề cần cân nhắc

### 12.1. Bảo mật
- Chỉ ADMIN mới có quyền can thiệp subscription — roleMiddleware đảm bảo điều này.
- STAFF chỉ có quyền read-only, không thể gọi các endpoint `POST`, `PATCH`.
- Mọi can thiệp ADMIN được ghi trong `subscription_transactions` để có trace.

### 12.2. Nhất quán dữ liệu
- Khi ADMIN can thiệp, luôn ghi `subscription_transactions` để lưu vết.
- `subscription_transactions` vừa mang tính tài chính (doanh thu) vừa là lịch sử thao tác.

### 12.3. Tương thích ngược
- Các API user-facing không thay đổi.
- Middleware `checkSubscriptionExpiry` vẫn hoạt động bình thường.
- Webhook PayOS vẫn hoạt động, chỉ bổ sung thêm bước ghi transaction.

### 12.4. Mở rộng sau này
- Khi có nhiều gói (ví dụ: `BASIC`, `PREMIUM`), cần update `plan` enum và logic.
- Export CSV có thể implement ở phase sau.
- Notification cho user khi subscription sắp hết hạn.
- Tự động gia hạn qua PayOS (recurring payment).

---

## 13. File tham chiếu

| File | Mục đích |
|------|----------|
| `AI_Stock_Trend_Prediction_Docs/BE/04-subscription-payos-backend.md` | Module subscription hiện tại — PayOS payment |
| `AI_Stock_Trend_Prediction_Docs/BE/02-user_management_backend.md` | Module user management hiện tại |
| `BE_AI_Stock_Trend_Prediction/api/src/config/plan.config.js` | Cấu hình plan limits, pricing |
| `BE_AI_Stock_Trend_Prediction/api/src/common/middlewares/role.middleware.js` | RBAC middleware |
| `BE_AI_Stock_Trend_Prediction/api/src/common/middlewares/subscription.middleware.js` | Subscription expiry middleware |

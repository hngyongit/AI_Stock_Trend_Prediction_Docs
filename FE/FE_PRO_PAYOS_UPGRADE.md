# FE Web Tasks — Pro User Upgrade with PayOS Integration

> **Base**: Backend đã hoàn thành tất cả API endpoints. Các task này dành riêng cho **FE Web** app tại `FE_AI_Stock_Trend_Prediction/web/`.

---

## 📋 Task Summary Table

| # | Task | Priority | Difficulty | File chính |
|---|------|----------|-----------|------------|
| 1 | Update watchlist service handle response mới | 🔴 High | ⭐ Easy | `services/watchlist.service.ts` |
| 2 | Create subscription service | 🔴 High | ⭐ Easy | `services/subscription.service.ts` (new) |
| 3 | Update auth store & user types | 🔴 High | ⭐ Easy | `services/auth.service.ts`, `services/users.service.ts` |
| 4 | Create Upgrade/Pricing page | 🔴 High | ⭐⭐⭐ Medium-Hard | `pages/UpgradePage/` (new) |
| 5 | Watchlist overflow overlay UI | 🔴 High | ⭐⭐ Medium | `pages/WatchlistPage/WatchList.tsx` |
| 6 | Update profile page hiển thị plan | 🟡 Medium | ⭐ Easy | `pages/UserProfilePage/UserProfilePage.tsx` |
| 7 | Add /upgrade route + navigation | 🟡 Medium | ⭐ Easy | `routes/layoutRoutes.tsx` |
| 8 | CSP configuration cho PayOS | 🟢 Low | ⭐ Easy | `index.html` / `vite.config.ts` |

---

## Task 1 — Update watchlist service

| Field | Value |
|-------|-------|
| **Priority** | 🔴 High |
| **Difficulty** | ⭐ Easy |
| **File** | `web/src/services/watchlist.service.ts` |
| **Dependency** | None |

### Why

API response của `GET /api/watchlists` đã thay đổi:

- **Trước đây**: trả về `data` là một array flat `StockItem[]`
- **Hiện tại**: trả về object `{ items, limit, currentCount, overLimit }`
  - Khi `overLimit: true`, items chỉ có `{ stock_id, stock_code, stock_name }` — KHÔNG có price, change, financials, charts
  - Khi `overLimit: false`, items có đầy đủ thông tin như cũ

### What to do

1. Thêm type `WatchlistResponse` mới:
   ```ts
   type WatchlistResponse = {
     items: StockItem[]
     limit: number
     currentCount: number
     overLimit: boolean
   }
   ```

2. Update function `getWatchlist()`:
   - Return `Promise<WatchlistResponse>` thay vì `Promise<StockItem[]>`
   - Parse `payload.data` thành object mới với các field `{ items, limit, currentCount, overLimit }`
   - Khi `overLimit: true`, field `stock_code` từ API map thành `symbol`, `stock_name` map thành `companyName`

3. Thêm function mới `trimWatchlist(keepStockIds: string[])`:
   - Gọi `POST /api/watchlists/trim` với body `{ keepStockIds }`
   - Return `{ deletedCount: number, remainingCount: number }`

4. Thêm type `TrimWatchlistResult`:
   ```ts
   type TrimWatchlistResult = {
     deletedCount: number
     remainingCount: number
   }
   ```

5. Export các types mới để các component khác dùng

### API reference

```
GET /api/watchlists
Authorization: Bearer <token>

Response — normal (overLimit: false):
{
  "success": true,
  "data": {
    "items": [
      {
        "watchlist_id": "...",
        "stock": { "id": "...", "symbol": "ACB", "company_name": "Ngân hàng ACB", "market_id": "...", "market_code": "HOSE" },
        "latest_price": { "close_price": 25700, "price_change": 0.5, "price_change_percent": 1.12, "volume": 2500000 },
        "created_at": "..."
      }
    ],
    "limit": 5,
    "currentCount": 3,
    "overLimit": false
  }
}

Response — overflow (overLimit: true):
{
  "success": true,
  "data": {
    "items": [
      { "stock_id": "abc123", "stock_code": "ACB", "stock_name": "Ngân hàng ACB" }
    ],
    "limit": 5,
    "currentCount": 10,
    "overLimit": true
  }
}

POST /api/watchlists/trim
Authorization: Bearer <token>
Body: { "keepStockIds": ["id1", "id2", "id3", "id4", "id5"] }

Response:
{
  "success": true,
  "data": { "deletedCount": 5, "remainingCount": 5 }
}
```

---

## Task 2 — Create subscription service

| Field | Value |
|-------|-------|
| **Priority** | 🔴 High |
| **Difficulty** | ⭐ Easy |
| **File** | `web/src/services/subscription.service.ts` (new) |
| **Dependency** | None |

### Why

Cần một service riêng cho tất cả API calls liên quan đến subscription và payment.

### What to do

1. Tạo file mới `web/src/services/subscription.service.ts`

2. Thêm type definitions:
   ```ts
   type CreatePaymentResponse = {
     checkoutUrl: string
     orderCode: number
     paymentLinkId: string
     amount: number
   }
   
   type SubscriptionStatus = {
     plan: 'FREE' | 'PRO'
     subscriptionStatus: 'NONE' | 'ACTIVE' | 'EXPIRED' | 'CANCELLED'
     subscriptionExpiresAt: string | null
   }
   ```

3. Implement 2 functions:

   **`createPayment()`**:
   - Gọi `POST /api/subscriptions/create-payment` (authenticated)
   - Return `CreatePaymentResponse` với `checkoutUrl` — FE sẽ dùng URL này để mở PayOS Checkout

   **`getSubscriptionStatus()`**:
   - Gọi `GET /api/subscriptions/status` (authenticated)
   - Return `SubscriptionStatus` với `plan`, `subscriptionStatus`, `subscriptionExpiresAt`
   - Dùng để kiểm tra plan hiện tại, hiển thị trên Upgrade page và Profile page

### API reference

```
POST /api/subscriptions/create-payment
Authorization: Bearer <token>
Body: (none, amount is fixed)

Response:
{
  "success": true,
  "data": {
    "checkoutUrl": "https://pay.payos.vn/web/...",
    "orderCode": 17123456780001,
    "paymentLinkId": "...",
    "amount": 50000
  }
}

GET /api/subscriptions/status
Authorization: Bearer <token>

Response:
{
  "success": true,
  "data": {
    "plan": "FREE",
    "subscriptionStatus": "NONE",
    "subscriptionExpiresAt": null
  }
}
```

---

## Task 3 — Update auth store & user types

| Field | Value |
|-------|-------|
| **Priority** | 🔴 High |
| **Difficulty** | ⭐ Easy |
| **Files** | `services/auth.service.ts`, `services/users.service.ts` |
| **Dependency** | None |

### Why

API `GET /api/users/me` và `POST /api/auth/login` giờ trả về thêm `plan`, `subscription_status`, `subscription_expires_at` trong user object. Các component cần đọc những field này.

### What to do

1. **`auth.service.ts`** — Add vào type `AuthUser`:
   ```ts
   export type AuthUser = {
     id: string
     full_name: string
     email: string
     role: string
     status: string
     plan: string            // new
     subscription_status: string  // new
     subscription_expires_at: string | null  // new
     [key: string]: unknown
   }
   ```

2. **`users.service.ts`** — Add vào type `UserProfile`:
   ```ts
   export type UserProfile = {
     id?: string
     full_name?: string
     email?: string
     role?: string
     status?: string
     plan?: string                // new
     subscription_status?: string // new
     subscription_expires_at?: string | null // new
     created_at?: string
   }
   ```

3. **`stores/auth.store.ts`** — Không cần thay đổi cấu trúc, vì store dùng `AuthUser` generic. Chỉ cần verify `user.plan` được truyền qua đúng.

---

## Task 4 — Create Upgrade/Pricing page

| Field | Value |
|-------|-------|
| **Priority** | 🔴 High |
| **Difficulty** | ⭐⭐⭐ Medium-Hard |
| **Files** | `pages/UpgradePage/` (new folder + files) |
| **Dependency** | Task 2 (subscription service), Task 3 (auth types) |

### Why

Users cần một trang để xem so sánh gói FREE vs PRO và thực hiện thanh toán.

### Design spec

```
┌──────────────────────────────────────────────────────┐
│                     /upgrade                          │
│                                                        │
│                    ⭐ Nâng cấp PRO                    │
│          Mở khóa toàn bộ tính năng theo dõi            │
│          thị trường chứng khoán chuyên sâu              │
│                                                        │
│   ┌────────────────────┐  ┌────────────────────┐       │
│   │      📊 FREE       │  │      ⭐ PRO        │       │
│   │    Miễn phí        │  │  ₫50,000/tháng     │       │
│   │   ✓ 5 mã theo dõi  │  │ ✓ 50 mã theo dõi   │       │
│   │   ✓ Dữ liệu cơ bản │  │ ✓ Dữ liệu chuyên sâu│      │
│   │   ❌ Cảnh báo       │  │ ✓ Cảnh báo thông minh│     │
│   │   ❌ Bộ lọc nâng cao│  │ ✓ Bộ lọc nâng cao   │       │
│   │                    │  │                    │       │
│   │   [Gói hiện tại]   │  │ [Nâng cấp ngay] 🔥│       │
│   └────────────────────┘  └────────────────────┘       │
│                                                        │
│   ─── Gói PRO sẽ tự động hết hạn sau 30 ngày ───      │
│   🔒 Thanh toán an toàn qua PayOS                     │
└──────────────────────────────────────────────────────┘
```

### What to do

1. **Tạo page** `web/src/pages/UpgradePage/UpgradePage.tsx`:
   - Layout 2 cards side-by-side (responsive: stack trên mobile)
   - Card FREE: icon 📊, "Miễn phí", feature list, button "Gói hiện tại" (disabled nếu đang FREE)
   - Card PRO: icon ⭐, "₫50,000/tháng", feature list có PRO-only features, button "Nâng cấp ngay"

2. **States to handle**:

   | State | Hiển thị |
   |-------|----------|
   | FREE, chưa từng upgrade | PRO card hiển thị "Nâng cấp ngay", FREE card "Gói hiện tại" |
   | FREE, trước đó đã PRO (hết hạn) | PRO card tag "Hết hạn" + button "Nâng cấp lại" |
   | PRO, đang active | PRO card badge "Đã kích hoạt" + button "Gia hạn", hiển thị ngày hết hạn |
   | Loading payment | Button spinner "Đang kết nối PayOS..." |
   | PayOS checkout opened | Hiển thị PayOS iframe/ popup, không navigate đi đâu |
   | Payment success | Animation checkmark + redirect về watchlist hoặc profile |
   | Payment cancelled/failed | Error banner, re-enable button |

3. **PayOS Checkout integration**:
   - Load script động: `<script src="https://cdn.payos.vn/payos-checkout/v1/stable/payos-initialize.js">`
   - Khi có `checkoutUrl`, gọi `PayOSCheckout.usePayOS()`:
   ```ts
   const { open, exit } = PayOSCheckout.usePayOS({
     CHECKOUT_URL: checkoutUrl,
     RETURN_URL: window.location.origin + '/payment/success',
     ELEMENT_ID: '#payment-container',
     embedded: true,           // true = iframe trong page, false = popup
     onSuccess: (event) => {   // payment done → refresh status
       getSubscriptionStatus()
       // nếu user đến từ overflow overlay → redirect về /watchlist
     },
     onCancel: (event) => { /* user cancelled */ },
     onExit: (event) => { /* re-enable nút */ },
   });
   open();
   ```

### Design rules

- **PRO card phải nổi bật**: gradient border, subtle shadow, hoặc "Phổ biến nhất" badge
- **Checkmark hierarchy**: green ✓ cho feature có, grey ✗ cho feature không có
- **Single CTA**: chỉ một primary action button, tránh gây confusion
- **Trust signals**: PayOS badge / lock icon ở cuối page
- **Responsive**: mobile stack cards full-width

---

## Task 5 — Watchlist overflow overlay UI

| Field | Value |
|-------|-------|
| **Priority** | 🔴 High |
| **Difficulty** | ⭐⭐ Medium |
| **File** | `pages/WatchlistPage/WatchList.tsx` |
| **Dependency** | Task 1 (watchlist service update) |

### Why

Khi subscription của user hết hạn và họ đang theo dõi >5 cổ phiếu, API trả về `overLimit: true`. Cần hiển thị overlay buộc user phải chọn giữ lại 5 cổ phiếu hoặc nâng cấp.

### What to do

1. **Detect**: Check `overLimit` từ response của `getWatchlist()`
   - Update `loadWatchlist()` để nhận `WatchlistResponse` thay vì array cũ

2. **Render overlay** khi `overLimit === true`:
   - Overlay phủ kín vùng watchlist table
   - Chặn tất cả interaction với table bên dưới

3. **Overlay content**:
   ```
   ┌──────────────────────────────────────────┐
   │                                          │
   │   ⚠️ Gói PRO của bạn đã hết hạn         │
   │                                          │
   │   Bạn đang theo dõi {currentCount} cổ    │
   │   phiếu, nhưng gói FREE chỉ cho phép     │
   │   {limit}.                               │
   │                                          │
   │   ┌────────────────────────────────────┐ │
   │   │ ☐ ACB - Ngân hàng ACB             │ │
   │   │ ☑ VNM - Vinamilk                  │ │
   │   │ ☐ FPT - FPT Corp                  │ │
   │   │ ...                               │ │
   │   │ Chọn {selected}/{limit} để giữ lại │ │
   │   └────────────────────────────────────┘ │
   │                                          │
   │   [Xác nhận giữ lại]      [Nâng cấp PRO] │
   │                                          │
   └──────────────────────────────────────────┘
   ```

4. **Trim selection UX**:
   - Checkbox list với `stock_code` và `stock_name` từ API (items chỉ có 2 field này)
   - User chỉ được chọn tối đa `{limit}` items (disable các checkbox còn lại)
   - Count: "Chọn {selected}/{limit} cổ phiếu để giữ lại"
   - Confirm button disabled đến khi chọn đúng `{limit}` items
   - Sau khi trim thành công → `loadWatchlist()` → `overLimit: false` → full data xuất hiện

5. **Upgrade action**:
   - Nút "Nâng cấp PRO" → redirect sang `/upgrade`
   - Có thể pass query param `?redirect=/watchlist` để sau khi upgrade quay lại

6. **Cached query params**: Có thể dùng URL param như `?overflow=true` để xử lý redirect sau upgrade

### Important notes from BE spec

> Overlay là **client-side UX guidance**. Backend vẫn là enforcement chính:
> - `POST /api/watchlists` vẫn trả về **400** khi user cố thêm stock khi đã quá limit
> - `overLimit` được tính server-side — không có cách nào bypass
> - Khi `overLimit: true`, BE **strip hết** price, change, financial data — chỉ gửi `stock_id`, `stock_code`, `stock_name`

---

## Task 6 — Update profile page hiển thị plan

| Field | Value |
|-------|-------|
| **Priority** | 🟡 Medium |
| **Difficulty** | ⭐ Easy |
| **File** | `pages/UserProfilePage/UserProfilePage.tsx` |
| **Dependency** | Task 3 (auth types) |

### Why

Profile page cần hiển thị thông tin plan của user và cho phép upgrade.

### What to do

1. **Hiển thị plan** trong phần user info:
   - Badge "FREE" hoặc "PRO" kế bên role
   - Nếu PRO active: hiển thị subscription status + ngày hết hạn
   - Nếu expired: badge "Expired" màu đỏ

2. **Action buttons**:
   - FREE → button "Nâng cấp lên PRO" → navigate `/upgrade`
   - PRO active → button "Gia hạn" → navigate `/upgrade`
   - PRO expired → button "Nâng cấp lại" → navigate `/upgrade`

3. **Data source**: Dùng `AuthUser` từ auth store hoặc gọi `getSubscriptionStatus()`

---

## Task 7 — Add /upgrade route + navigation

| Field | Value |
|-------|-------|
| **Priority** | 🟡 Medium |
| **Difficulty** | ⭐ Easy |
| **Files** | `routes/layoutRoutes.tsx` |
| **Dependency** | Task 4 (Upgrade page) |

### What to do

1. Trong `web/src/routes/layoutRoutes.tsx`:
   - Import `UpgradePage` từ `@/pages/UpgradePage/UpgradePage`
   - Thêm vào `USER_ROUTES`:
   ```ts
   { path: "/upgrade", element: <UpgradePage /> }
   ```

2. Tuỳ chọn: thêm link trong sidebar navigation
   - Có thể thêm mục "Nâng cấp PRO" trong menu
   - Hoặc chỉ hiển thị khi user là FREE + gắn badge "🔥" hoặc "NÂNG CẤP"

---

## Task 8 — CSP configuration

| Field | Value |
|-------|-------|
| **Priority** | 🟢 Low |
| **Difficulty** | ⭐ Easy |
| **Files** | `index.html` / `vite.config.ts` |
| **Dependency** | Task 4 |

### Why

PayOS Checkout Script cần CSP exceptions để load và render payment iframe.

### What to do

Kiểm tra xem web app có dùng Content Security Policy không. Nếu có, thêm vào:

```
script-src https://cdn.payos.vn/payos-checkout/v1/stable/payos-initialize.js
frame-src https://pay.payos.vn/ https://next.pay.payos.vn/
connect-src https://payos.vn/
```

---

## 🔗 All API Endpoints Reference

| Method | Endpoint | Auth | Purpose | Dùng bởi task |
|--------|----------|------|---------|---------------|
| `GET` | `/api/watchlists` | ✅ Bearer | Get watchlist với `overLimit` flag | Task 1, 5 |
| `POST` | `/api/watchlists/trim` | ✅ Bearer | Trim watchlist giữ lại N stocks | Task 1, 5 |
| `POST` | `/api/subscriptions/create-payment` | ✅ Bearer | Tạo payment, nhận checkoutUrl | Task 2, 4 |
| `GET` | `/api/subscriptions/status` | ✅ Bearer | Check plan & subscription status | Task 2, 4, 6 |
| `POST` | `/api/subscriptions/webhook` | ❌ Public | PayOS callback (BE only) | None |
| `GET` | `/api/users/me` | ✅ Bearer | User profile với plan fields | Task 3, 6 |

## 📦 PayOS Checkout Integration Summary

Dùng cho **Task 4** — trang Upgrade:

```html
<!-- Load script động khi user click "Nâng cấp" -->
<script src="https://cdn.payos.vn/payos-checkout/v1/stable/payos-initialize.js"></script>
```

```ts
// Flow:
// 1. User click "Nâng cấp ngay"
// 2. Gọi createPayment() → nhận checkoutUrl
// 3. Mở PayOS checkout với checkoutUrl
// 4. User thanh toán → onSuccess callback
// 5. Gọi getSubscriptionStatus() → cập nhật UI
```

# Alert Feature — FE Integration Guide

## 1. Tổng quan

Tài liệu này hướng dẫn Frontend Developer (Web & Mobile) tích hợp tính năng **Alert Management** từ BE API vào giao diện người dùng.

Tính năng cho phép:
- Người dùng tạo cảnh báo giá (`PRICE_ABOVE` / `PRICE_BELOW`) cho cổ phiếu trong watchlist.
- Người dùng tạo cảnh báo khối lượng bất thường (`VOLUME_SPIKE`).
- Nhận email thông báo khi cảnh báo được kích hoạt.
- Quản lý cảnh báo (xem, tắt/bật, sửa ngưỡng, xóa).

---

## 2. API Endpoint Summary

Base URL: `http://localhost:5000` (Dev) / `https://api.aistockprediction.vn` (Prod)

| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| GET | `/api/alerts` | Bearer Token | Lấy danh sách cảnh báo |
| POST | `/api/alerts` | Bearer Token | Tạo cảnh báo mới |
| GET | `/api/alerts/:id` | Bearer Token | Xem chi tiết cảnh báo |
| PUT | `/api/alerts/:id` | Bearer Token | Cập nhật ngưỡng / bật tắt |
| DELETE | `/api/alerts/:id` | Bearer Token | Xóa cảnh báo |

### 2.1. Frontend API Service Pattern

Tạo service file theo pattern hiện tại (xem `watchlist.service.ts` làm reference):

```typescript
// alert.service.ts
import { authenticatedRequest } from '../services/auth.service';

export type AlertType = 'PRICE_ABOVE' | 'PRICE_BELOW' | 'VOLUME_SPIKE';
export type AlertStatus = 'ACTIVE' | 'TRIGGERED' | 'DISABLED';

export interface AlertItem {
  id: string;
  symbol: string;
  company_name: string;
  alert_type: AlertType;
  threshold: number;
  status: AlertStatus;
  triggered_at: string | null;
  triggered_value: number | null;
  latest_price: { close_price: number; volume: number } | null;
  created_at: string;
  updated_at: string;
}

export interface CreateAlertPayload {
  symbol: string;
  alert_type: AlertType;
  threshold: number;
}

export const getAlerts = (): Promise<AlertItem[]> =>
  authenticatedRequest({ method: 'GET', url: '/api/alerts' })
    .then(res => res.data?.data || []);

export const createAlert = (payload: CreateAlertPayload) =>
  authenticatedRequest({ method: 'POST', url: '/api/alerts', data: payload })
    .then(res => res.data?.data);

export const getAlertById = (id: string) =>
  authenticatedRequest({ method: 'GET', url: `/api/alerts/${id}` })
    .then(res => res.data?.data);

export const updateAlert = (id: string, updates: { threshold?: number; status?: AlertStatus }) =>
  authenticatedRequest({ method: 'PUT', url: `/api/alerts/${id}`, data: updates })
    .then(res => res.data?.data);

export const deleteAlert = (id: string) =>
  authenticatedRequest({ method: 'DELETE', url: `/api/alerts/${id}` });

export const toggleAlert = (id: string, currentStatus: AlertStatus) => {
  const nextStatus = currentStatus === 'ACTIVE' ? 'DISABLED' : 'ACTIVE';
  return updateAlert(id, { status: nextStatus });
};
```

---

## 3. Screen Design & Layout

### 3.1. AlertsPage (`/alerts`)

**Mô tả layout:**

```
┌──────────────────────────────────────────────────────┐
│  Alerts                          [+ Create Alert]    │
├──────┬──────────┬─────────┬────────┬─────────┬───────┤
│Stock │ Type     │Thresh.  │Status  │Last Trig│Action │
├──────┼──────────┼─────────┼────────┼─────────┼───────┤
│ FPT  │PRICE_ABV │140,000  │● Active│--       │ ⟳ | 🗑 │
│ HPG  │VOLUME_SPK│ 3.0x    │◉ Trig'd│02/07    │ ↺ | 🗑 │
│ VNM  │PRICE_BEL │ 70,000  │○ Disab │--       │ ⟳ | 🗑 │
└──────┴──────────┴─────────┴────────┴─────────┴───────┘
```

**Các thành phần chính:**

1. **Page Header**: Tiêu đề "Alerts" + nút "Create Alert" (icon +).
2. **Bảng danh sách** với các cột:
   - **Stock**: Symbol + Company Name (có link đến StockDetailPage).
   - **Type**: Badge hiển thị loại cảnh báo.
     - `PRICE_ABOVE`: Badge xanh dương "Price ↑"
     - `PRICE_BELOW`: Badge đỏ "Price ↓"
     - `VOLUME_SPIKE`: Badge cam "Volume"
   - **Threshold**: Giá trị ngưỡng (format số, có đơn vị VND hoặc "x").
   - **Status**: Badge trạng thái.
     - `ACTIVE`: Badge xanh lá "Active"
     - `TRIGGERED`: Badge vàng "Triggered"
     - `DISABLED`: Badge xám "Disabled"
   - **Last Triggered**: Thời gian kích hoạt gần nhất (định dạng dd/mm/yyyy hoặc "--").
   - **Actions**: 2 icon button.
     - Enable/Disable toggle (icon power/bell-off).
     - Delete (icon trash).
3. **Empty State**: Khi chưa có cảnh báo nào, hiển thị:
   ```
   ┌─────────────────────────────────┐
   │      🔔 No alerts yet           │
   │  Create your first alert to     │
   │  get notified about stock       │
   │  price movements.               │
   │        [+ Create Alert]         │
   └─────────────────────────────────┘
   ```

### 3.2. AlertCreateDialog

**Mô tả layout (shadcn Dialog):**

```
┌─────────────────────────────────────┐
│  ✕   Create Alert                  │
├─────────────────────────────────────┤
│                                     │
│  Stock Symbol                       │
│  ┌──────────────────────────┐       │
│  │ [▼] Select a stock...    │       │
│  └──────────────────────────┘       │
│  (Dropdown liệt kê cổ phiếu         │
│   trong watchlist của user)         │
│                                     │
│  Alert Type                         │
│  ○ Price Above  ○ Price Below       │
│  ○ Volume Spike                     │
│                                     │
│  Threshold                          │
│  ┌──────────────────────────┐       │
│  │ [              ]         │       │
│  └──────────────────────────┘       │
│  (Giải thích: VND cho price,        │
│   multiplier cho volume spike)      │
│                                     │
│  [Cancel]              [Create]     │
└─────────────────────────────────────┘
```

**Luồng hoạt động:**

1. User click "Create Alert" → dialog mở.
2. User chọn stock từ dropdown (fetch từ `GET /api/watchlists`).
3. User chọn alert type (radio button).
4. User nhập threshold:
   - Price: label "Threshold (VND)" — number input, step=1000.
   - Volume: label "Multiplier (x times average)" — number input, step=0.5, default=3.0.
5. Label dưới input hiển thị giải thích: "Notify when price goes above..." / "Notify when volume exceeds...".
6. User submit → POST `/api/alerts` → thành công đóng dialog, refresh list.
7. Lỗi hiển thị trong dialog (toast hoặc inline error).

### 3.3. StockDetailPage Integration

**Tại Bell Button (dòng có icon Bell):**

- Click Bell → mở `AlertCreateDialog` với `symbol` được pre-fill sẵn (disabled, không cho sửa).
- Nếu đã có alert cho cổ phiếu này, hiển thị icon Bell filled (khác màu).

**Tại Alert Configuration Card:**

```
┌─────────────────────────────┐
│  Alert Configuration        │
├─────────────────────────────┤
│  Active rules:  2            │
│  Latest trigger: 02/07/2026  │
│  Status: ● Active            │
│                             │
│  [FPT] Price > 140,000  ●  │
│  [FPT] Price < 130,000  ○  │
│                             │
│  [+ Add Alert]              │
└─────────────────────────────┘
```

- Hiển thị tổng quan: số lượng active rules, lần trigger gần nhất.
- Danh sách các alert của cổ phiếu hiện tại.
- Mỗi alert có toggle bật/tắt nhanh.
- Nút "Add Alert" mở dialog pre-filled.

### 3.4. SettingsPage Notification Toggle

```
┌─────────────────────────────────┐
│  Settings                       │
├─────────────────────────────────┤
│                                 │
│  Notifications                  │
│  ┌─────────────────────────┐    │
│  │ 🔔 Email Alerts     [ON]│    │
│  │ Receive email when your │    │
│  │ alerts are triggered    │    │
│  └─────────────────────────┘    │
│                                 │
│  Alert Limits                   │
│  ┌─────────────────────────┐    │
│  │ Your Plan: FREE         │    │
│  │ Alert stocks: 1 / 2     │    │
│  │ Alerts per stock: 1 / 2 │    │
│  │ [Upgrade to PRO]        │    │
│  └─────────────────────────┘    │
│                                 │
└─────────────────────────────────┘
```

---

## 4. Plan Constraint Rules

### 4.1. Giới hạn theo gói

| Rule | FREE | PRO |
|---|---|---|
| Max stocks can set alerts | 2 | 50 |
| Max alerts per stock | 2 | 10 |
| Requirement | Stock must be in watchlist | Stock must be in watchlist |

### 4.2. Xử lý khi PRO hết hạn

- User PRO có 5 cổ phiếu đã đặt cảnh báo → downgrade xuống FREE.
- Middleware tự động cập nhật `plan` thành `FREE`.
- API tạo alert mới sẽ từ chối nếu vượt quá giới hạn FREE.
- **Các alert cũ không tự động bị xóa** — chỉ không thể tạo mới.
- Các alert hiện tại vẫn hiển thị nhưng không được kiểm tra nếu vượt quá giới hạn.

### 4.3. Xử lý frontend khi bị từ chối

Khi API trả về 400 với message limit, hiển thị toast thông báo và link upgrade:

```
"Alert stock limit exceeded (Maximum 2 stocks allowed). Upgrade to PRO for unlimited alerts."
```

---

## 5. Alert Lifecycle

```
  User tạo            User tắt tạm
  ───────────┐       ┌─────────────
             ▼       ▼
         ┌──────────┐           ┌────────────┐
         │  ACTIVE  │──────────→│  DISABLED  │
         └────┬─────┘           └─────┬──────┘
              │                       │
              │ Scheduler kích hoạt   │ User bật lại
              ▼                       │
         ┌──────────┐                 │
         │ TRIGGERED│←────────────────┘
         └──────────┘  User reset (re-enable)
              │
              │ Đã gửi email
              ▼
         (Không kiểm tra lại)
```

**Frontend actions per status:**

| Status | Actions available |
|--------|-------------------|
| `ACTIVE` | Disable, Update threshold, Delete |
| `TRIGGERED` | Re-enable (→ ACTIVE, clears triggered_at), Update threshold, Delete |
| `DISABLED` | Enable (→ ACTIVE), Update threshold, Delete |

---

## 6. Error Scenarios

| HTTP | Error Message | Frontend Handling |
|------|---------------|-------------------|
| 400 | `Stock must be in your watchlist to create an alert` | Toast error: "Stock must be in your watchlist" |
| 400 | `Alert stock limit exceeded (Maximum X stocks allowed)` | Toast + upgrade prompt |
| 400 | `Alert limit for this stock exceeded (Maximum X alerts per stock)` | Toast error: limit reached |
| 400 | `Cannot transition from X to Y` | Toast error: invalid action |
| 404 | `Stock symbol not found` | Toast error: stock not found |
| 404 | `Alert not found` | Toast error + redirect to alerts list |
| 401 | Unauthorized | Redirect to login |

### Validation lỗi (400)

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "threshold", "message": "Threshold must be a positive number" }
  ]
}
```

Hiển thị inline error dưới field tương ứng trong form.

---

## 7. Step-by-Step Integration Guide

### Web (React/TypeScript)

1. **Tạo service file**: `src/services/alert.service.ts` (copy code từ Section 2.1).
2. **Tạo AlertCreateDialog component**:
   - Sử dụng shadcn `Dialog`, `Button`, `Input`.
   - Stock selector dùng dropdown với dữ liệu từ `getWatchlistData()`.
   - Validate form trước submit.
   - Gọi `createAlert()` → thành công emit event parent refresh.
3. **Xây dựng AlertsPage**:
   - `GET /api/alerts` khi component mount.
   - Render table với các cột như Section 3.1.
   - Delete: confirm dialog → `deleteAlert()` → refresh.
   - Toggle: `toggleAlert()` → refresh.
   - Create: mở AlertCreateDialog → refresh.
4. **Cập nhật StockDetailPage**:
   - Bell button onClick → mở AlertCreateDialog với `symbol` pre-fill.
   - Alert Configuration card: fetch alerts, filter theo `symbol`, render list.
5. **Cập nhật SettingsPage**:
   - Hiển thị plan limit info từ user object (auth store).
   - Email toggle (lưu preference ở localStorage hoặc user profile).

### Mobile (React Native)

Tương tự Web nhưng sử dụng:
- `Modal` thay cho `Dialog`.
- `FlatList` cho danh sách alerts.
- `TouchableOpacity` cho các action button.

### States per component

| State | Loading | Empty | Error | Success |
|-------|---------|-------|-------|---------|
| AlertsPage | Skeleton table rows | Empty state illustration | Retry button + error message | Data table |
| AlertCreateDialog | Submit button disabled + spinner | N/A | Inline error + toast | Close dialog, refresh parent |
| StockDetail card | Skeleton text | "No alerts configured" | Error toast | Alert list |
| SettingsPage | Skeleton | N/A | Error toast | Plan info + toggle |

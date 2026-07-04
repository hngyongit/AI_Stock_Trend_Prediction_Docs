# Mock Data Simulation — Demo Price Movement & Alert

## 1. Mục tiêu

Module này cho phép **Staff/Admin** tạo dữ liệu giá giả lập trong memory để demo tính năng **Alert** cho mentor. 

Đặc điểm:
- **Không ghi vào database** — tất cả dữ liệu nằm trong RAM server.
- **Chỉ chạy ở dev mode** (`NODE_ENV=development`) — production bỏ qua hoàn toàn.
- **FE không cần thay đổi** — không có nút mock, không session ID. FE poll `GET /api/stocks/:symbol` như bình thường.
- **Alert kích hoạt real-time** — khi tick vượt threshold, alert được `markTriggered()` ngay lập tức.

---

## 2. Cách dùng (Swagger)

Staff/Admin đăng nhập → vào Swagger `/api-docs` → tìm tag `Staff Mock Data`.

### 2.1. Bắt đầu mock

```
POST /api/staff/mock-data/start
Authorization: Bearer <staff_token>
Content-Type: application/json

{
  "symbol": "FPT",
  "alertType": "PRICE_ABOVE",
  "threshold": 150000,
  "tickCount": 15
}
```

**Response:**
```json
{
  "success": true,
  "message": "Mock data session started",
  "data": {
    "symbol": "FPT",
    "totalTicks": 15,
    "alertType": "PRICE_ABOVE",
    "threshold": 150000,
    "status": "started"
  }
}
```

### 2.2. Xem danh sách session đang chạy

```
GET /api/staff/mock-data
Authorization: Bearer <staff_token>
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "symbol": "FPT",
      "alertType": "PRICE_ABOVE",
      "threshold": 150000,
      "progress": "5/15",
      "totalTicks": 15,
      "alertFired": false
    }
  ]
}
```

### 2.3. Dừng mock

```
DELETE /api/staff/mock-data/{symbol}
Authorization: Bearer <staff_token>
```

**Response:**
```json
{
  "success": true,
  "message": "Mock session for FPT stopped"
}
```

---

## 3. Flow hoạt động

```
Dev (Swagger)                    Backend                           FE (Browser)
─────────────────               ────────                          ─────────────
                                                                   
POST /start {symbol, ...}                                           
       │                                                             
       ▼                                                             
   Generate 15 ticks                                                 
   in memory array                                                   
       │                                                             
       └─────────────────────────────────────────────────────────► Poll GET /api/stocks/FPT
                                                                      (every 1.5s)
                                                                  ◄── return tick[0], cursor→1
                                                                      (close_price: 132,000)
                                                                      
                                                                  Poll GET /api/stocks/FPT
                                                                  ◄── return tick[1], cursor→2
                                                                      (close_price: 137,000)
                                                                      
                                                                  ... (candle by candle)
                                                                  
                                                                  Poll GET /api/stocks/FPT
                                          ╔══════════════════╗     ◄── tick[10] crosses 150,000
                                          ║ alert check runs ║         alert_triggered: true
                                          ║ markTriggered()  ║         alert_status: TRIGGERED
                                          ║ send email       ║
                                          ╚══════════════════╝
                                                                      
                                                                  Poll GET /api/stocks/FPT
                                                                  ◄── tick[11], keeps going
                                                                      
                                                                  ... (more candles)
                                                                  
                                                                  Poll GET /api/stocks/FPT
                                                                  ◄── tick[14], cursor→15
                                                                      _done: true

DELETE /mock-data/FPT                                                 
       │                                                             
       ▼                                                             
   Cleanup → API returns real DB data again                          
```

---

## 4. Timing

| tickCount | Threshold crosses ~ | Total time (1.5s poll) |
|-----------|-------------------|----------------------|
| 12        | tick 8-9 (~12s)   | ~18s |
| 15        | tick 10-11 (~15s) | ~22s |
| 20        | tick 13-15 (~20s) | ~30s |
| 30        | tick 20-23 (~31s) | ~45s |

Tick luôn vượt threshold ở ~65-80% progress.

---

## 5. Chi tiết kỹ thuật

### 5.1. MockSession class (`services/mock-data.service.js`)

- `startSession()` — nhận params, tạo `MockSession`, lưu vào `Map<symbol, session>`
- `nextTick()` — trả tick hiện tại, tăng cursor. Hết mảng trả `null`
- `getEmittedTicks()` — tất cả tick đã emit (cho chart replay)
- `isActive()` — còn tick không?
- `markAlertFired()` — chặn fire nhiều lần

### 5.2. Intercept trong stock service (`modules/stocks/stocks.service.js`)

```js
// Chỉ ở NODE_ENV=development
if (env.NODE_ENV === 'development') {
  const session = mockDataService.getActiveSession(symbol);
  if (session) {
    const tick = session.nextTick();
    if (tick) {
      // Kiểm tra alert nếu tick vượt threshold
      if (tick._crossesThreshold && !session.alertFired) {
        await alertChecker.checkAlertsForStock(stockId);
        mockDataService.markAlertFired(symbol);
      }
      return { ...stockInfo, latest_price: tick, _mock: true, ... };
    }
  }
}
```

### 5.3. Alert checker inline (`scheduler/alert-checker.scheduler.js`)

Hàm `checkAlertsForStock(stockId)`:
- Chỉ check alert ACTIVE cho 1 stock
- Tính avgVolume 20 phiên cho VOLUME_SPIKE
- Nếu threshold met → `markTriggered()` + gửi email
- Chạy ngay lập tức, không batch, không delay

### 5.4. Response fields cho FE

Khi mock active, stock detail API trả thêm các field:

| Field | Type | Ý nghĩa |
|-------|------|---------|
| `_mock` | boolean | Đang ở chế độ mock |
| `_cursor` | number | Tick hiện tại (1-based) |
| `_remaining` | number | Số tick còn lại |
| `_done` | boolean | Đã hết tick chưa |
| `alert_triggered` | boolean | Tick này vừa kích hoạt alert |
| `alert_status` | string | `"TRIGGERED"` hoặc null |

---

## 6. So sánh với script DB mock

| | `scripts/mock-stock-movement.js` | `POST /api/staff/mock-data/start` |
|---|---|---|
| Ghi DB | Có (`factMarketPrices`) | Không (RAM) |
| Cách dùng | CLI `node scripts/...` | Swagger |
| Dùng để | Tạo dữ liệu thật cho dev | Demo real-time cho mentor |
| Tồn tại | Vĩnh viễn (đến khi xóa) | Đến khi server restart hoặc DELETE |

---

## 7. Files liên quan

| File | Vai trò |
|------|---------|
| `api/src/services/mock-data.service.js` | Core: session management, tick generation |
| `api/src/modules/mock-data/mock-data.controller.js` | Request handlers |
| `api/src/modules/mock-data/mock-data.routes.js` | 3 endpoints + Swagger docs |
| `api/src/modules/stocks/stocks.service.js` | Intercept mock data (modified) |
| `api/src/scheduler/alert-checker.scheduler.js` | Export `checkAlertsForStock` (modified) |
| `api/src/app.js` | Register `/api/staff/mock-data` (modified) |

---

## 8. Demo checklist

1. [ ] Staff tạo watchlist + alert cho 1 stock (qua app hoặc Swagger)
2. [ ] Mở Swagger `POST /api/staff/mock-data/start` — set `symbol`, `alertType`, `threshold`, `tickCount`
3. [ ] Mở FE stock detail page — chart bắt đầu chạy tick-by-tick (poll 1.5s)
4. [ ] ~15s sau: alert badge xuất hiện, email gửi (nếu SMTP configured)
5. [ ] Chart tiếp tục chạy đến tick cuối cùng
6. [ ] `DELETE /api/staff/mock-data/{symbol}` — cleanup, API trả data thật

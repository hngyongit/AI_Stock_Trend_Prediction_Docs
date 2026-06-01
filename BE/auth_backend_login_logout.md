# Backend Module: Authentication — Đăng nhập / Đăng xuất

## 1. Mục tiêu

Xây dựng module Authentication cho Backend API của dự án **AI-Based Stock Trend Prediction**, tập trung trước vào chức năng:

- Đăng nhập
- Đăng xuất
- Lấy thông tin user hiện tại
- Refresh token
- Middleware bảo vệ API
- Phân quyền theo role: `USER`, `STAFF`, `ADMIN`

Module này phục vụ Web/Mobile App, đồng thời là nền tảng bảo mật cho các API như watchlist, dashboard, crawl job, crawl logs và admin/staff panel.

---

## 2. Tech Stack sử dụng

Theo tech stack MVP của dự án:

| Thành phần | Công nghệ |
|---|---|
| Runtime | Node.js |
| Framework | Express.js |
| Language | JavaScript |
| Database | MongoDB |
| ODM | Mongoose |
| Authentication | JWT |
| Password Hash | Bcrypt |
| API Style | RESTful API |
| API Docs | Swagger / Postman |
| Deploy | DigitalOcean / MongoDB Atlas |

---

## 3. Phạm vi chức năng Auth trong BE

### 3.1. Chức năng chính

| Mã | Chức năng | Mô tả |
|---|---|---|
| AUTH-01 | Login | User đăng nhập bằng email và password |
| AUTH-02 | Logout | User đăng xuất khỏi hệ thống |
| AUTH-03 | Refresh Token | Cấp lại access token khi token cũ hết hạn |
| AUTH-04 | Get Me | Lấy thông tin user đang đăng nhập |
| AUTH-05 | Protect API | Middleware kiểm tra JWT |
| AUTH-06 | RBAC | Middleware kiểm tra role |

### 3.2. Chưa làm ở bước này

Các chức năng sau có thể làm sau khi login/logout ổn định:

- Register
- Forgot password
- Verify email
- Reset password
- Login bằng Google
- Quản lý session nâng cao

---

## 4. Database liên quan

### 4.1. Collection `roles`

Dùng để lưu vai trò người dùng.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `name` | String | Tên role: `USER`, `STAFF`, `ADMIN` |
| `description` | String | Mô tả role |
| `created_at` | Date | Ngày tạo |
| `updated_at` | Date | Ngày cập nhật |

#### Ràng buộc

```js
name: unique
```

#### Role mặc định

```txt
USER
STAFF
ADMIN
```

---

### 4.2. Collection `users`

Dùng để lưu tài khoản người dùng.

| Field | Type | Mô tả |
|---|---|---|
| `_id` | ObjectId | Khóa chính |
| `role_id` | ObjectId | FK tham chiếu sang `roles` |
| `full_name` | String | Họ tên người dùng |
| `email` | String | Email đăng nhập |
| `password_hash` | String | Mật khẩu đã hash bằng Bcrypt |
| `status` | String | Trạng thái tài khoản |
| `refresh_token_hash` | String | Refresh token đã hash, optional |
| `last_login_at` | Date | Lần đăng nhập gần nhất |
| `created_at` | Date | Ngày tạo |
| `updated_at` | Date | Ngày cập nhật |

#### Ràng buộc

```js
email: unique
password_hash: never return to client
```

#### Giá trị `status`

```txt
ACTIVE
INACTIVE
LOCKED
```

---

## 5. Luồng đăng nhập

### 5.1. API

```http
POST /api/auth/login
```

### 5.2. Request body

```json
{
  "email": "user@example.com",
  "password": "12345678"
}
```

### 5.3. Validate input

| Field | Rule |
|---|---|
| `email` | Required, đúng định dạng email |
| `password` | Required, tối thiểu 8 ký tự |

### 5.4. Xử lý backend

1. Nhận `email`, `password` từ client.
2. Validate dữ liệu đầu vào.
3. Tìm user theo email.
4. Nếu không tồn tại user → trả lỗi.
5. Kiểm tra trạng thái tài khoản.
6. So sánh password với `password_hash` bằng Bcrypt.
7. Nếu sai password → trả lỗi.
8. Lấy thông tin role của user.
9. Tạo `access_token`.
10. Tạo `refresh_token`.
11. Lưu hash của refresh token vào database.
12. Trả về token và thông tin user.

### 5.5. Response thành công

```json
{
  "success": true,
  "message": "Login successfully",
  "data": {
    "access_token": "jwt_access_token_here",
    "refresh_token": "jwt_refresh_token_here",
    "user": {
      "id": "user_id",
      "full_name": "Nguyen Van A",
      "email": "user@example.com",
      "role": "USER",
      "status": "ACTIVE"
    }
  }
}
```

### 5.6. Response thất bại

#### Email không tồn tại hoặc password sai

```json
{
  "success": false,
  "message": "Invalid email or password"
}
```

#### Tài khoản bị khóa

```json
{
  "success": false,
  "message": "Account is locked"
}
```

---

## 6. Luồng đăng xuất

### 6.1. API

```http
POST /api/auth/logout
```

### 6.2. Header

```http
Authorization: Bearer access_token
```

### 6.3. Xử lý backend

1. Client gửi request logout kèm access token.
2. Backend verify access token.
3. Lấy `user_id` từ token.
4. Xóa hoặc set null `refresh_token_hash` trong database.
5. Trả về logout thành công.

### 6.4. Response thành công

```json
{
  "success": true,
  "message": "Logout successfully"
}
```

### 6.5. Ghi chú quan trọng

Với JWT, access token là stateless nên backend không tự hủy access token ngay lập tức nếu không dùng blacklist. MVP có thể xử lý đơn giản:

- Access token sống ngắn, ví dụ 15 phút.
- Logout sẽ xóa refresh token.
- Client xóa access token và refresh token khỏi local storage / AsyncStorage.

Nếu cần bảo mật cao hơn, phase sau có thể thêm:

- Token blacklist
- Session collection
- Redis lưu revoked token

---

## 7. Refresh token

### 7.1. API

```http
POST /api/auth/refresh-token
```

### 7.2. Request body

```json
{
  "refresh_token": "jwt_refresh_token_here"
}
```

### 7.3. Xử lý backend

1. Nhận refresh token từ client.
2. Verify refresh token.
3. Lấy `user_id` từ token.
4. Tìm user trong database.
5. So sánh refresh token với `refresh_token_hash`.
6. Nếu hợp lệ → tạo access token mới.
7. Có thể rotate refresh token nếu muốn bảo mật hơn.

### 7.4. Response thành công

```json
{
  "success": true,
  "message": "Refresh token successfully",
  "data": {
    "access_token": "new_access_token_here"
  }
}
```

---

## 8. Get current user

### 8.1. API

```http
GET /api/auth/me
```

### 8.2. Header

```http
Authorization: Bearer access_token
```

### 8.3. Response thành công

```json
{
  "success": true,
  "data": {
    "id": "user_id",
    "full_name": "Nguyen Van A",
    "email": "user@example.com",
    "role": "USER",
    "status": "ACTIVE",
    "created_at": "2026-06-01T00:00:00.000Z"
  }
}
```

---

## 9. JWT payload đề xuất

### 9.1. Access token payload

```json
{
  "user_id": "user_id",
  "email": "user@example.com",
  "role": "USER"
}
```

### 9.2. Refresh token payload

```json
{
  "user_id": "user_id",
  "token_type": "refresh"
}
```

### 9.3. Thời gian sống token

| Token | Thời gian đề xuất |
|---|---|
| Access token | 15 phút |
| Refresh token | 7 ngày |

---

## 10. Middleware bảo vệ API

### 10.1. `authMiddleware`

Dùng để kiểm tra user đã đăng nhập chưa.

#### Nhiệm vụ

- Lấy token từ header `Authorization`.
- Verify JWT.
- Lấy user từ database.
- Gắn user vào `req.user`.
- Không trả `password_hash`.

#### Dùng cho API

```js
router.get('/me', authMiddleware, authController.getMe);
```

---

### 10.2. `roleMiddleware`

Dùng để kiểm tra quyền theo role.

#### Ví dụ

```js
router.get(
  '/admin/crawl-jobs',
  authMiddleware,
  roleMiddleware(['STAFF', 'ADMIN']),
  crawlJobController.getAll
);
```

#### Quyền theo role

| Role | Quyền |
|---|---|
| USER | Xem dashboard, xem stock, quản lý watchlist cá nhân |
| STAFF | Quản lý nguồn dữ liệu, crawl job, crawl logs |
| ADMIN | Quản lý toàn hệ thống |

---

## 11. API summary

| Method | Endpoint | Auth | Role | Mô tả |
|---|---|---|---|---|
| POST | `/api/auth/login` | No | Public | Đăng nhập |
| POST | `/api/auth/logout` | Yes | USER/STAFF/ADMIN | Đăng xuất |
| POST | `/api/auth/refresh-token` | No | Public | Cấp lại access token |
| GET | `/api/auth/me` | Yes | USER/STAFF/ADMIN | Lấy thông tin user hiện tại |

---

## 12. Cấu trúc thư mục Backend đề xuất

```txt
backend/
├── src/
│   ├── config/
│   │   ├── database.js
│   │   └── env.js
│   │
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.controller.js
│   │   │   ├── auth.routes.js
│   │   │   ├── auth.service.js
│   │   │   └── auth.validation.js
│   │   │
│   │   ├── users/
│   │   │   ├── user.model.js
│   │   │   ├── user.controller.js
│   │   │   ├── user.routes.js
│   │   │   └── user.service.js
│   │   │
│   │   └── roles/
│   │       ├── role.model.js
│   │       └── role.service.js
│   │
│   ├── middlewares/
│   │   ├── auth.middleware.js
│   │   ├── role.middleware.js
│   │   └── error.middleware.js
│   │
│   ├── utils/
│   │   ├── jwt.js
│   │   ├── hash.js
│   │   └── response.js
│   │
│   ├── app.js
│   └── server.js
│
├── .env
├── package.json
└── README.md
```

---

## 13. Environment variables

```env
PORT=5000
NODE_ENV=development

MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/aistock

JWT_ACCESS_SECRET=your_access_secret
JWT_REFRESH_SECRET=your_refresh_secret

JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

BCRYPT_SALT_ROUNDS=10
```

---

## 14. Business rules

| Mã | Rule |
|---|---|
| BR-AUTH-01 | Email là thông tin đăng nhập duy nhất |
| BR-AUTH-02 | Password không được lưu plain text |
| BR-AUTH-03 | Password phải hash bằng Bcrypt |
| BR-AUTH-04 | API protected phải có JWT hợp lệ |
| BR-AUTH-05 | Không trả `password_hash` ra frontend |
| BR-AUTH-06 | User bị khóa không được đăng nhập |
| BR-AUTH-07 | Logout phải vô hiệu hóa refresh token |
| BR-AUTH-08 | API admin chỉ cho phép `STAFF` hoặc `ADMIN` |
| BR-AUTH-09 | User không được tự đổi role |

---

## 15. Security checklist

- [ ] Hash password bằng Bcrypt.
- [ ] Không trả `password_hash` trong response.
- [ ] Validate email/password trước khi xử lý.
- [ ] Access token có thời gian sống ngắn.
- [ ] Refresh token được lưu dạng hash.
- [ ] Logout xóa refresh token trong DB.
- [ ] API protected dùng `authMiddleware`.
- [ ] API quản trị dùng thêm `roleMiddleware`.
- [ ] Secret key lưu trong `.env`.
- [ ] Không hard-code JWT secret trong source code.

---

## 16. Thứ tự triển khai đề xuất

### Bước 1: Setup project Backend

- Khởi tạo Node.js project.
- Cài Express, Mongoose, dotenv, bcrypt, jsonwebtoken.
- Kết nối MongoDB Atlas.

### Bước 2: Tạo model Role và User

- Tạo `RoleModel`.
- Tạo `UserModel`.
- Seed sẵn role: `USER`, `STAFF`, `ADMIN`.

### Bước 3: Làm login API

- Validate input.
- Tìm user theo email.
- Check password bằng Bcrypt.
- Tạo access token và refresh token.

### Bước 4: Làm middleware Auth

- Verify access token.
- Gắn user vào `req.user`.
- Bảo vệ API `/api/auth/me`.

### Bước 5: Làm logout API

- Xóa refresh token của user.
- Client tự xóa token ở local.

### Bước 6: Làm refresh token API

- Verify refresh token.
- Check token trong DB.
- Cấp access token mới.

### Bước 7: Làm role middleware

- Kiểm tra role.
- Áp dụng cho các API staff/admin.

---

## 17. Test case cơ bản

| Mã | Trường hợp | Kết quả mong muốn |
|---|---|---|
| TC-AUTH-01 | Login đúng email/password | Trả access token, refresh token, user info |
| TC-AUTH-02 | Login sai password | Trả lỗi `Invalid email or password` |
| TC-AUTH-03 | Login email không tồn tại | Trả lỗi `Invalid email or password` |
| TC-AUTH-04 | Login tài khoản bị khóa | Trả lỗi `Account is locked` |
| TC-AUTH-05 | Gọi `/me` không có token | Trả lỗi `Unauthorized` |
| TC-AUTH-06 | Gọi `/me` với token hợp lệ | Trả thông tin user |
| TC-AUTH-07 | Logout thành công | Xóa refresh token |
| TC-AUTH-08 | Refresh token hợp lệ | Trả access token mới |
| TC-AUTH-09 | Refresh token sai/hết hạn | Trả lỗi `Invalid refresh token` |
| TC-AUTH-10 | USER gọi API admin | Trả lỗi `Forbidden` |

---

## 18. Kết luận

Module đăng nhập/đăng xuất là phần Backend đầu tiên nên triển khai vì toàn bộ các chức năng sau như watchlist, dashboard cá nhân, crawl job, crawl logs và admin panel đều cần xác thực người dùng.

Với MVP, cách làm phù hợp nhất là:

- JWT access token ngắn hạn.
- Refresh token dài hạn.
- Password hash bằng Bcrypt.
- Phân quyền bằng RBAC.
- Logout bằng cách xóa refresh token.

Cách triển khai này đủ đơn giản cho MVP nhưng vẫn mở rộng được khi hệ thống phát triển thêm AI Prediction, quản lý dữ liệu nâng cao và mobile app.

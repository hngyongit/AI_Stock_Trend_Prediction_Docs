# Backend Module — User Management

## 1. Mục tiêu chức năng

Module **User Management** dùng để quản lý thông tin người dùng sau khi đã hoàn thành chức năng Authentication.

Module này phục vụ 3 nhóm quyền chính:

- **USER**: xem và cập nhật thông tin cá nhân.
- **STAFF**: sử dụng hệ thống theo quyền vận hành dữ liệu, không được quản lý user.
- **ADMIN**: quản lý danh sách user, khóa/mở khóa tài khoản, gán role phù hợp.

Chức năng này thuộc phase MVP vì hệ thống cần có tài khoản, phân quyền và trạng thái người dùng trước khi triển khai các module Watchlist, Stock Dashboard, Crawl Management.

---

## 2. Phạm vi chức năng

### 2.1. User thường

User có thể:

- Xem thông tin cá nhân.
- Cập nhật họ tên.
- Đổi mật khẩu.
- Không được tự đổi email nếu chưa có cơ chế xác thực email.
- Không được tự đổi role.
- Không được tự đổi status tài khoản.

### 2.2. Admin

Admin có thể:

- Xem danh sách user.
- Tìm kiếm user theo email, họ tên, role, status.
- Xem chi tiết một user.
- Khóa tài khoản user.
- Mở khóa tài khoản user.
- Gán role USER hoặc STAFF.
- Không nên xóa cứng user trong MVP.

---

## 3. Collection liên quan

### 3.1. `users`

```js
{
  _id: ObjectId,
  role_id: ObjectId,
  full_name: String,
  email: String,
  password_hash: String,
  status: String,
  created_at: Date,
  updated_at: Date
}
```

### 3.2. `roles`

```js
{
  _id: ObjectId,
  name: String,
  description: String,
  created_at: Date,
  updated_at: Date
}
```

---

## 4. Trạng thái tài khoản

| Status | Ý nghĩa |
|---|---|
| `ACTIVE` | Tài khoản đang hoạt động bình thường |
| `LOCKED` | Tài khoản bị khóa, không được đăng nhập |

Khuyến nghị MVP chỉ dùng 2 trạng thái này để đơn giản hóa luồng hoạt động.

---

## 5. API Design

Base route:

```txt
/api/users
```

---

## 6. API cho user cá nhân

## 6.1. Xem thông tin cá nhân

### Endpoint

```http
GET /api/users/me
```

### Authorization

```txt
Bearer Access Token
```

### Role được phép

```txt
USER, STAFF, ADMIN
```

### Response thành công

```json
{
  "success": true,
  "message": "Get profile successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a12345",
    "full_name": "Nguyen Van A",
    "email": "user@gmail.com",
    "role": "USER",
    "status": "ACTIVE",
    "created_at": "2026-06-01T10:00:00.000Z"
  }
}
```

### Response thất bại

#### 401 Unauthorized (Thiếu hoặc sai token)
```json
{
  "success": false,
  "message": "Unauthorized"
}
```

#### 403 Forbidden (Tài khoản bị khóa)
```json
{
  "success": false,
  "message": "Account is locked"
}
```

### Lưu ý

Không được trả về:

```txt
password_hash
refresh_token
```

---

## 6.2. Cập nhật thông tin cá nhân

### Endpoint

```http
PUT /api/users/me
```

### Authorization

```txt
Bearer Access Token
```

### Role được phép

```txt
USER, STAFF, ADMIN
```

### Request body

```json
{
  "full_name": "Nguyen Van B"
}
```

### Validate

| Field | Rule |
|---|---|
| `full_name` | Bắt buộc nếu gửi lên, độ dài 2–100 ký tự |

### Response thành công

```json
{
  "success": true,
  "message": "Update profile successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a12345",
    "full_name": "Nguyen Van B",
    "email": "user@gmail.com",
    "role": "USER",
    "status": "ACTIVE"
  }
}
```

### Response thất bại

#### 400 Bad Request (Validation lỗi)
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "full_name",
      "message": "full_name is required and must be between 2 and 100 characters"
    }
  ]
}
```

#### 401 Unauthorized (Thiếu hoặc sai token)
```json
{
  "success": false,
  "message": "Unauthorized"
}
```

---

## 6.3. Đổi mật khẩu

### Endpoint

```http
PUT /api/users/me/password
```

### Authorization

```txt
Bearer Access Token
```

### Role được phép

```txt
USER, STAFF, ADMIN
```

### Request body

```json
{
  "current_password": "OldPassword123",
  "new_password": "NewPassword123"
}
```

### Validate

| Field | Rule |
|---|---|
| `current_password` | Bắt buộc |
| `new_password` | Tối thiểu 8 ký tự, nên có chữ và số |
| `new_password` | Không được giống mật khẩu cũ |

### Flow xử lý

1. Lấy user hiện tại từ JWT.
2. Tìm user trong database.
3. So sánh `current_password` với `password_hash` bằng Bcrypt.
4. Nếu sai, trả lỗi.
5. Hash `new_password` bằng Bcrypt.
6. Cập nhật `password_hash`.
7. Cập nhật `updated_at`.
8. Có thể logout khỏi các phiên cũ nếu hệ thống có lưu refresh token.

### Response thành công

```json
{
  "success": true,
  "message": "Change password successfully"
}
```

### Response thất bại

#### 400 Bad Request (Mật khẩu cũ không chính xác hoặc trùng mật khẩu mới)
```json
{
  "success": false,
  "message": "Current password is incorrect"
}
```
Hoặc:
```json
{
  "success": false,
  "message": "New password cannot be the same as the current password"
}
```

#### 401 Unauthorized
```json
{
  "success": false,
  "message": "Unauthorized"
}
```

---

## 7. API cho Admin quản lý user

## 7.1. Xem danh sách user

### Endpoint

```http
GET /api/admin/users
```

### Authorization

```txt
Bearer Access Token
```

### Role được phép

```txt
ADMIN
```

### Query params

```txt
?page=1&limit=10&keyword=nguyen&role=USER&status=ACTIVE
```

| Query | Ý nghĩa |
|---|---|
| `page` | Trang hiện tại |
| `limit` | Số record mỗi trang |
| `keyword` | Tìm theo full_name hoặc email |
| `role` | Lọc theo USER / STAFF / ADMIN |
| `status` | Lọc theo ACTIVE / LOCKED |

### Response thành công

```json
{
  "success": true,
  "message": "Get users successfully",
  "data": {
    "items": [
      {
        "id": "665f1a2b9c1e2a0012a12345",
        "full_name": "Nguyen Van A",
        "email": "user@gmail.com",
        "role": "USER",
        "status": "ACTIVE",
        "created_at": "2026-06-01T10:00:00.000Z"
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

### Response thất bại

#### 401 Unauthorized
```json
{
  "success": false,
  "message": "Unauthorized"
}
```

#### 403 Forbidden (Không có quyền ADMIN)
```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
}
```

---

## 7.2. Xem chi tiết user

### Endpoint

```http
GET /api/admin/users/:id
```

### Role được phép

```txt
ADMIN
```

### Response thành công

```json
{
  "success": true,
  "message": "Get user detail successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a12345",
    "full_name": "Nguyen Van A",
    "email": "user@gmail.com",
    "role": "USER",
    "status": "ACTIVE",
    "created_at": "2026-06-01T10:00:00.000Z",
    "updated_at": "2026-06-01T10:00:00.000Z"
  }
}
```

### Response thất bại

#### 401 Unauthorized
```json
{
  "success": false,
  "message": "Unauthorized"
}
```

#### 403 Forbidden (Không có quyền ADMIN)
```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
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

## 7.3. Khóa tài khoản user

### Endpoint

```http
PATCH /api/admin/users/:id/lock
```

### Role được phép

```txt
ADMIN
```

### Business rules

- Admin không được tự khóa chính mình.
- Không nên khóa tài khoản ADMIN khác trong MVP nếu chưa có super admin.
- User bị khóa không được đăng nhập.

### Response thành công

```json
{
  "success": true,
  "message": "Lock user successfully"
}
```

### Response thất bại

#### 400 Bad Request (Tự khóa chính mình hoặc khóa tài khoản ADMIN khác)
```json
{
  "success": false,
  "message": "Admin cannot lock themselves or other admin accounts"
}
```

#### 401/403 (Chưa đăng nhập / Không phải ADMIN)
```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
}
```

#### 404 Not Found
```json
{
  "success": false,
  "message": "User not found"
}
```

---

## 7.4. Mở khóa tài khoản user

### Endpoint

```http
PATCH /api/admin/users/:id/unlock
```

### Role được phép

```txt
ADMIN
```

### Response thành công

```json
{
  "success": true,
  "message": "Unlock user successfully"
}
```

### Response thất bại

#### 401/403 (Chưa đăng nhập / Không phải ADMIN)
```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
}
```

#### 404 Not Found
```json
{
  "success": false,
  "message": "User not found"
}
```

---

## 7.5. Gán role cho user

### Endpoint

```http
PATCH /api/admin/users/:id/role
```

### Role được phép

```txt
ADMIN
```

### Request body

```json
{
  "role": "STAFF"
}
```

### Role được phép gán trong MVP

```txt
USER
STAFF
```

### Không khuyến nghị cho phép gán trực tiếp

```txt
ADMIN
```

Nếu muốn tạo ADMIN, nên seed bằng database hoặc tạo thủ công để tránh rủi ro bảo mật.

### Response thành công

```json
{
  "success": true,
  "message": "Update user role successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a12345",
    "full_name": "Nguyen Van A",
    "email": "user@gmail.com",
    "role": "STAFF",
    "status": "ACTIVE"
  }
}
```

### Response thất bại

#### 400 Bad Request (Gán sai role hoặc role không hợp lệ)
```json
{
  "success": false,
  "message": "Role must be USER or STAFF"
}
```

#### 401/403 (Chưa đăng nhập / Không phải ADMIN)
```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
}
```

#### 404 Not Found
```json
{
  "success": false,
  "message": "User not found"
}
```

---

## 8. Business Rules

| Mã rule | Nội dung |
|---|---|
| BR-USER-01 | User chỉ được xem và sửa thông tin của chính mình |
| BR-USER-02 | User không được tự đổi role |
| BR-USER-03 | User không được tự đổi status |
| BR-USER-04 | Admin được xem danh sách user |
| BR-USER-05 | Admin được khóa/mở khóa tài khoản user |
| BR-USER-06 | Admin không được tự khóa chính mình |
| BR-USER-07 | Tài khoản LOCKED không được đăng nhập |
| BR-USER-08 | Không trả password_hash ra frontend |
| BR-USER-09 | Không xóa cứng user trong MVP, chỉ khóa tài khoản |

---

## 9. Middleware cần dùng

### 9.1. `authMiddleware`

Dùng để kiểm tra JWT access token.

```js
router.get('/me', authMiddleware, userController.getMe);
```

### 9.2. `roleMiddleware`

Dùng để kiểm tra role.

```js
router.get(
  '/admin/users',
  authMiddleware,
  roleMiddleware(['ADMIN']),
  adminUserController.getUsers
);
```

### 9.3. `validateMiddleware`

Dùng để validate request body/query params.

```js
router.put(
  '/me',
  authMiddleware,
  validateMiddleware(updateProfileSchema),
  userController.updateMe
);
```

---

## 10. Cấu trúc thư mục đề xuất

```txt
src/
├── modules/
│   ├── users/
│   │   ├── user.model.js
│   │   ├── user.controller.js
│   │   ├── user.service.js
│   │   ├── user.route.js
│   │   ├── user.validation.js
│   │   └── user.constant.js
│   │
│   ├── roles/
│   │   ├── role.model.js
│   │   ├── role.service.js
│   │   └── role.seed.js
│
├── middlewares/
│   ├── auth.middleware.js
│   ├── role.middleware.js
│   └── validate.middleware.js
│
├── utils/
│   ├── response.js
│   └── pagination.js
```

---

## 11. Mongoose schema đề xuất

### 11.1. User schema

```js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    role_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Role',
      required: true
    },
    full_name: {
      type: String,
      required: true,
      trim: true,
      minlength: 2,
      maxlength: 100
    },
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true
    },
    password_hash: {
      type: String,
      required: true
    },
    status: {
      type: String,
      enum: ['ACTIVE', 'LOCKED'],
      default: 'ACTIVE'
    }
  },
  {
    timestamps: {
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    }
  }
);

userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ role_id: 1 });
userSchema.index({ status: 1 });

module.exports = mongoose.model('User', userSchema);
```

### 11.2. Role schema

```js
const mongoose = require('mongoose');

const roleSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      unique: true,
      enum: ['USER', 'STAFF', 'ADMIN']
    },
    description: {
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

roleSchema.index({ name: 1 }, { unique: true });

module.exports = mongoose.model('Role', roleSchema);
```

---

## 12. Validation schema đề xuất

```js
const Joi = require('joi');

const updateProfileSchema = Joi.object({
  full_name: Joi.string().trim().min(2).max(100).required()
});

const changePasswordSchema = Joi.object({
  current_password: Joi.string().required(),
  new_password: Joi.string().min(8).required()
});

const updateUserRoleSchema = Joi.object({
  role: Joi.string().valid('USER', 'STAFF').required()
});

module.exports = {
  updateProfileSchema,
  changePasswordSchema,
  updateUserRoleSchema
};
```

---

## 13. Service flow

### 13.1. `getMe(userId)`

```txt
Input: userId từ JWT
Process:
1. Find user by id
2. Populate role
3. Nếu không tồn tại, trả 404
4. Trả user không gồm password_hash
Output: user profile
```

### 13.2. `updateMe(userId, payload)`

```txt
Input: userId, full_name
Process:
1. Find user by id
2. Nếu không tồn tại, trả 404
3. Update full_name
4. Save
5. Trả user mới
Output: updated user profile
```

### 13.3. `changePassword(userId, currentPassword, newPassword)`

```txt
Input: userId, current_password, new_password
Process:
1. Find user by id
2. Compare current_password bằng bcrypt.compare
3. Nếu sai, trả 400
4. Hash new_password
5. Save password_hash mới
Output: success message
```

### 13.4. `adminGetUsers(query)`

```txt
Input: page, limit, keyword, role, status
Process:
1. Build filter
2. Nếu có keyword, search theo full_name/email
3. Nếu có role, join hoặc tìm role_id trước
4. Query users có pagination
5. Populate role
6. Trả items + pagination
Output: user list
```

---

## 14. Error response thống nhất

### 14.1. User không tồn tại

```json
{
  "success": false,
  "message": "User not found"
}
```

### 14.2. Không có quyền

```json
{
  "success": false,
  "message": "Forbidden: insufficient permission"
}
```

### 14.3. Mật khẩu hiện tại sai

```json
{
  "success": false,
  "message": "Current password is incorrect"
}
```

### 14.4. Tài khoản bị khóa

```json
{
  "success": false,
  "message": "User account is locked"
}
```

---

## 15. Checklist triển khai BE

- [ ] Tạo `user.model.js`
- [ ] Tạo `role.model.js`
- [ ] Seed 3 role: USER, STAFF, ADMIN
- [ ] Tạo `user.route.js`
- [ ] Tạo `user.controller.js`
- [ ] Tạo `user.service.js`
- [ ] Tạo `user.validation.js`
- [ ] Thêm API `GET /api/users/me`
- [ ] Thêm API `PUT /api/users/me`
- [ ] Thêm API `PUT /api/users/me/password`
- [ ] Thêm API `GET /api/admin/users`
- [ ] Thêm API `GET /api/admin/users/:id`
- [ ] Thêm API `PATCH /api/admin/users/:id/lock`
- [ ] Thêm API `PATCH /api/admin/users/:id/unlock`
- [ ] Thêm API `PATCH /api/admin/users/:id/role`
- [ ] Gắn `authMiddleware`
- [ ] Gắn `roleMiddleware(['ADMIN'])` cho route admin
- [ ] Không trả `password_hash` ra response
- [ ] Test bằng Postman
- [ ] Viết Swagger docs

---

## 16. Thứ tự code khuyến nghị

1. Hoàn thiện `Role` model và seed role.
2. Hoàn thiện `User` model.
3. Kiểm tra lại module Auth đã tạo user đúng `role_id` chưa.
4. Làm API `GET /api/users/me`.
5. Làm API `PUT /api/users/me`.
6. Làm API đổi mật khẩu.
7. Làm API admin xem danh sách user.
8. Làm API admin khóa/mở khóa user.
9. Làm API admin đổi role.
10. Test toàn bộ bằng Postman.

---

## 17. Ghi chú MVP

Trong MVP, module User Management nên làm vừa đủ để phục vụ các module sau:

- Watchlist cần biết user hiện tại.
- Crawl Management cần phân quyền STAFF/ADMIN.
- Admin Dashboard cần quản lý user/staff.
- Auth cần kiểm tra tài khoản có bị khóa hay không.

Không nên làm quá sớm các chức năng như:

- Quên mật khẩu qua email.
- Xác thực email OTP.
- Xóa vĩnh viễn tài khoản.
- Activity log chi tiết từng thao tác user.

Các chức năng này có thể đưa sang phase sau.

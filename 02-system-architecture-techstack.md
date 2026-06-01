# Tech Stack đề xuất cho dự án

## 13. Tech Stack đề xuất cho dự án

## 13.1. Frontend Web

Dùng để xây dựng giao diện web cho **user, staff và admin**.

### Công nghệ đề xuất

- ReactJS
- Vite
- TypeScript
- Tailwind CSS
- Shadcn/UI hoặc Ant Design
- Recharts / ECharts / ApexCharts
- Axios
- React Router

### Mục đích

- Dashboard cổ phiếu
- Biểu đồ giá
- Watchlist
- Trang admin/staff
- Quản lý crawl job
- Xem crawl logs

---

## 13.2. Mobile App

Dùng để xây dựng app mobile cho user theo dõi cổ phiếu.

### Công nghệ đề xuất

- React Native
- Expo
- TypeScript
- React Navigation
- Axios
- AsyncStorage
- Recharts Native / Victory Native

### Mục đích

- Đăng nhập
- Xem cổ phiếu
- Theo dõi watchlist
- Xem chart cơ bản
- Nhận thông báo ở phase sau

---

## 13.3. Backend API

Phù hợp với team có nhiều thành viên làm **React/NodeJS**.

### Công nghệ đề xuất

- Node.js
- Express.js
- JavaScript
- JWT Authentication
- Bcrypt
- RESTful API
- Swagger

---

## 13.4. Database / Data Warehouse

Vì team ưu tiên MongoDB, hệ thống có thể sử dụng:

- MongoDB
- MongoDB Atlas
- Mongoose

### Thiết kế theo hướng

- Dimension Collections
- Fact Collections
- Operational Collections

### Ví dụ collections

- dim_time
- dim_markets
- dim_stocks
- dim_data_sources
- fact_market_prices
- fact_financial_statements
- fact_financial_report_sources
- fact_crawl_quality
- users
- roles
- watchlists
- crawl_jobs
- crawl_logs
- crawl_log_details

---

## 13.5. Cron Job / Data Crawling

Dùng để tự động crawl dữ liệu chứng khoán.

### Công nghệ đề xuất

- Node-cron
- BullMQ
- Redis
- Axios
- Cheerio
- Puppeteer
- Python crawler service

### Khuyến nghị MVP

Sử dụng:

- Node-cron
- Python crawler service

### Nếu dùng Python để lấy data

- Python
- vnstock
- pandas
- requests
- beautifulsoup4

### Flow đề xuất

```text
NodeJS backend gọi Python crawler
→ Python crawl data
→ trả JSON
→ Backend validate
→ lưu MongoDB
```

---

## 13.6. Data Processing

Dùng để làm sạch, chuẩn hóa và tính toán chỉ số.

### Công nghệ đề xuất

- Python
- Pandas
- NumPy
- Pydantic

### Xử lý

- Chuẩn hóa OHLCV
- Tính price_change
- Tính price_change_percent
- Validate missing value
- Chuẩn hóa báo cáo tài chính

---

## 13.7. Dashboard / Chart

Dùng cho trực quan hóa dữ liệu.

### Công nghệ đề xuất

- Recharts
- Apache ECharts
- ApexCharts
- TradingView Lightweight Charts

### Khuyến nghị

- **TradingView Lightweight Charts**: dùng cho biểu đồ giá/candlestick.
- **ECharts**: dùng cho dashboard phân tích đa chiều.

---

## 13.8. Authentication & Authorization

### Công nghệ đề xuất

- JWT Access Token
- Refresh Token
- Bcrypt
- Role-based Access Control

### Role

- USER
- STAFF
- ADMIN

### Quyền

| Role | Quyền |
|---|---|
| USER | Xem dashboard, watchlist |
| STAFF | Quản lý crawl, data source, logs |
| ADMIN | Quản lý user, stock, market, toàn hệ thống |

---

## 13.9. API Documentation

### Công cụ đề xuất

- Swagger / OpenAPI
- Postman

### Mục đích

- Test API
- Viết tài liệu API
- Hỗ trợ frontend/mobile call API dễ hơn

---

## 13.10. DevOps / Deployment

### MVP có thể dùng

- Docker
- Docker Compose
- GitHub
- GitHub Actions
- Render / Railway / VPS
- MongoDB Atlas
- Vercel
- Expo EAS

### Gợi ý deploy

| Thành phần | Nền tảng deploy |
|---|---|
| Frontend Web | Vercel |
| Backend API | Render / Railway |
| Database | MongoDB Atlas |
| Mobile | Expo |

---

## 13.11. Monitoring / Logging

MVP nên có logging cơ bản.

### Công nghệ đề xuất

- Winston
- Morgan
- Sentry optional
- MongoDB logs

### Mục đích

- Log API error
- Log crawl error
- Log cron job
- Theo dõi hệ thống

---

## 13.12. AI Phase Sau

Chưa triển khai sâu trong MVP, nhưng có thể chuẩn bị.

### Công nghệ đề xuất

- Python
- Scikit-learn
- XGBoost
- LightGBM
- PyTorch
- TensorFlow
- FastAPI
- Pandas
- NumPy

### Dùng cho

- Trend Prediction
- Feature Engineering
- Backtesting
- News Sentiment
- Portfolio Analysis

---

## 13.13. AI-assisted Analytics / Visualization

Có thể nghiên cứu thêm:

- Microsoft Data Formulator
- Jupyter Notebook
- Pandas Profiling
- Streamlit

### Vai trò

- EDA
- Tạo chart bằng AI
- Khám phá dữ liệu
- Demo AI-assisted dashboard

> Không xem đây là core prediction engine.

---

# 14. Tech Stack chốt cho MVP

Bản gọn nhất nên chốt:

| Layer | Tech |
|---|---|
| Web | ReactJS + Vite + TypeScript |
| Mobile | React Native + Expo |
| Backend | NestJS + TypeScript |
| Database | MongoDB + Mongoose |
| Data Warehouse | MongoDB Collections theo Dim/Fact |
| Crawler | Python + vnstock + pandas |
| Cron | Node-cron hoặc BullMQ |
| Queue | Redis nếu cần |
| Chart | TradingView Lightweight Charts + ECharts |
| Auth | JWT + Bcrypt + RBAC |
| API Docs | Swagger |
| Deploy | Vercel + Render/Railway + MongoDB Atlas |
| Version Control | GitHub |
| AI later | Python + Scikit-learn/XGBoost/PyTorch |

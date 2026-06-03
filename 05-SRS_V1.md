# SRS — Software Requirements Specification

# 1. Giới thiệu dự án

## 1.1. Tên dự án

**AI-Based Stock Trend Prediction**

## 1.2. Mục tiêu dự án

Xây dựng hệ thống Web/Mobile hỗ trợ theo dõi, phân tích và trực quan hóa dữ liệu chứng khoán Việt Nam, tập trung trước vào dữ liệu thị trường và báo cáo tài chính.

Định hướng MVP:

> **Data First — AI Later**

Ưu tiên:

1. Lấy dữ liệu đúng
2. Lưu dữ liệu đúng
3. Hiển thị dữ liệu đúng
4. Hiển thị dữ liệu rõ ràng
5. Sau đó mới phát triển AI dự đoán xu hướng

## 1.3. Phạm vi MVP

### Bao gồm

* Sàn HOSE
* Dữ liệu giá cổ phiếu hằng ngày
* Dữ liệu tài chính doanh nghiệp
* Báo cáo tài chính theo quý
* Dashboard phân tích
* Watchlist người dùng
* Quản lý Crawl Job
* Theo dõi Log và Data Quality

### Chưa bao gồm

* Realtime Tick Data
* AI Prediction
* News Sentiment
* Portfolio Optimization

---

# 2. Tổng quan hệ thống

## 2.1. Nhóm người dùng

### User

* Đăng ký tài khoản
* Đăng nhập
* Xem danh sách cổ phiếu
* Xem biểu đồ giá
* Xem dashboard cổ phiếu
* Theo dõi cổ phiếu yêu thích
* Xem watchlist cá nhân

### Staff

* Quản lý nguồn dữ liệu
* Quản lý lịch crawl
* Chạy crawl thủ công
* Xem crawl logs
* Kiểm tra lỗi crawl
* Theo dõi chất lượng dữ liệu

### Admin

* Quản lý user
* Quản lý staff
* Quản lý sàn giao dịch
* Quản lý mã cổ phiếu
* Xem dashboard tổng quan hệ thống
* Theo dõi trạng thái toàn bộ hệ thống

---

# 3. Functional Requirements

## 3.1. Authentication & Authorization

### FR-AUTH-01: Đăng ký

Thông tin:

* Họ tên
* Email
* Password

Điều kiện:

* Email unique
* Password hash bằng Bcrypt
* Role mặc định USER

### FR-AUTH-02: Đăng nhập

Input:

* Email
* Password

Output:

* Access Token
* Refresh Token
* User Information
* Role

### FR-AUTH-03: Phân quyền

Roles:

* USER
* STAFF
* ADMIN

Cơ chế:

* JWT Authentication
* RBAC (Role-Based Access Control)

---

## 3.2. User Management

### FR-USER-01: Xem thông tin cá nhân

* Họ tên
* Email
* Role
* Trạng thái
* Ngày tạo

### FR-USER-02: Cập nhật thông tin

Cho phép:

* Họ tên
* Password

Không cho phép:

* Đổi role

### FR-USER-03: Quản lý user

Admin có thể:

* Xem danh sách user
* Tìm kiếm user
* Khóa / mở khóa tài khoản
* Gán role USER / STAFF

---

## 3.3. Market Management

### FR-MARKET-01

Quản lý:

* Market Code
* Market Name
* Country
* Timezone
* Status

MVP:

* HOSE

### FR-MARKET-02

Xem danh sách sàn đang active.

---

## 3.4. Stock Management

### FR-STOCK-01

Xem danh sách cổ phiếu.

Thông tin:

* Symbol
* Company Name
* Industry
* Market
* Status

### FR-STOCK-02

Tìm kiếm theo:

* Symbol
* Company Name
* Industry

### FR-STOCK-03

Xem chi tiết cổ phiếu:

* Company Information
* Latest Price
* Price Chart
* Volume
* EPS
* P/E
* P/B
* ROE
* ROAA
* Market Cap

### FR-STOCK-04

Admin:

* Thêm cổ phiếu
* Cập nhật thông tin
* Đổi trạng thái
* Gán ngành

---

## 3.5. Watchlist

### FR-WATCH-01

Thêm cổ phiếu vào watchlist.

Giới hạn MVP:

* Tối đa 5 mã / user

### FR-WATCH-02

Xóa khỏi watchlist.

### FR-WATCH-03

Xem watchlist.

Thông tin:

* Symbol
* Company Name
* Latest Close Price
* Price Change %
* Volume
* Market Cap

---

## 3.6. Market Price Data

### FR-PRICE-01

Lưu dữ liệu:

* Open
* High
* Low
* Close
* Volume
* Bid Volume
* Ask Volume
* Foreign Buy
* Foreign Sell
* Foreign Net
* Market Cap
* EPS
* P/E
* Forward P/E
* BVPS
* P/B
* Beta
* ROS
* ROE
* ROAA
* Price Change
* Price Change Percent

### FR-PRICE-02

Không lưu trùng:

```text
stock_id + time_id + data_source_id
```

### FR-PRICE-03

Xem lịch sử giá:

* 7 ngày
* 1 tháng
* 3 tháng
* 6 tháng
* 1 năm
* Custom Range

### FR-PRICE-04

Charts:

* Line Chart
* Candlestick Chart
* Volume Chart
* Foreign Trading Chart

---

## 3.7. Financial Statement

### FR-FIN-01

Lưu báo cáo tài chính theo quý:

* Net Revenue
* Gross Profit
* Profit Before Tax
* Profit After Tax
* Parent Company Profit
* Current Assets
* Total Assets
* Liabilities
* Current Liabilities
* Equity
* EPS
* P/E
* P/B
* ROE
* ROAA

### FR-FIN-02

Dữ liệu riêng cho ngân hàng:

* Net Interest Income
* Customer Loans
* Customer Deposits
* Operating Expense
* Total Operating Income

### FR-FIN-03

Dashboard tài chính:

* Revenue
* Profit
* Assets
* Liabilities
* Equity
* EPS
* P/E
* P/B
* ROE
* ROAA

---

## 3.8. Financial Report Source

### FR-REPORT-01

Lưu:

* Report URL
* File Type
* Report Status
* URL Validation
* Extracted Data

### FR-REPORT-02

Truy xuất nguồn dữ liệu.

---

## 3.9. Market Overview

### FR-OVERVIEW-01

Lưu:

* Total Volume
* Total Value
* Deal Volume
* Deal Value
* Market Cap
* Number Of Stocks
* Listed Volume
* Circulating Volume
* Foreign Buy
* Foreign Sell
* Foreign Net
* Bid Volume
* Ask Volume

### FR-OVERVIEW-02

Dashboard:

* Market Overview
* Liquidity
* Foreign Trading
* Market Capitalization
* Advancers / Decliners

---

## 3.10. Data Source Management

### FR-SOURCE-01

Quản lý:

* Name
* Type
* Base URL
* Description
* Status

Ví dụ:

* vnstock
* cafef
* ssi
* file import

### FR-SOURCE-02

Theo dõi:

* Error Rate
* Missing Data
* Success Rate
* Response Time

---

## 3.11. Crawl Job

### FR-CRAWL-01

Thông tin:

* Job Name
* Data Source
* Market
* Data Type
* Cron Expression
* Crawl Mode
* Status

Data Types:

* DAILY_MARKET_PRICE
* QUARTERLY_FINANCIAL_STATEMENT
* FINANCIAL_REPORT_SOURCE
* MARKET_OVERVIEW

### FR-CRAWL-02

Chạy tự động theo Cron.

### FR-CRAWL-03

Chạy thủ công.

### FR-CRAWL-04

Bật / Tắt Job.

---

## 3.12. Crawl Logs

### FR-LOG-01

Lưu:

* Job
* Started At
* Ended At
* Status
* Records Fetched
* Records Inserted
* Records Updated
* Records Failed
* Error Message

### FR-LOG-02

Log chi tiết:

* Symbol
* Data Type
* Status
* Message

### FR-LOG-03

Filter theo:

* Job
* Data Source
* Date
* Status
* Symbol

---

## 3.13. Data Quality

### FR-QUALITY-01

Lưu:

* Records Fetched
* Records Inserted
* Records Updated
* Records Failed
* Success Rate
* Status

### FR-QUALITY-02

Dashboard:

* Success Rate
* Failed Symbols
* Missing Records
* Worst Source
* Worst Job

---

# 4. Non-Functional Requirements

## Performance

* Stock API < 2s
* Chart API < 3s
* Crawl 400 mã HOSE ổn định
* Index các trường query thường xuyên

## Security

* Bcrypt
* JWT
* Refresh Token
* RBAC
* Không trả password_hash
* Validate Input

## Reliability

* Một mã lỗi không làm dừng job
* Log chi tiết
* Không lưu trùng
* Hỗ trợ PARTIAL_SUCCESS

## Scalability

Sẵn sàng mở rộng:

* HNX
* UPCOM
* Realtime Data
* AI Prediction
* News Sentiment
* Portfolio Analytics

## Maintainability

* Modular Architecture
* Swagger
* Coding Convention
* Crawler Service riêng
* Environment Variables

---

# 5. Tech Stack

## Frontend

* ReactJS
* Vite
* JavaScript / TypeScript
* TailwindCSS
* Shadcn/UI hoặc Ant Design
* React Router
* Axios
* TradingView Lightweight Charts
* ECharts

## Mobile

* React Native
* Expo
* React Navigation
* Axios
* AsyncStorage

## Backend

* Node.js
* Express.js
* JavaScript
* JWT
* Bcrypt
* Swagger

## Database

* MongoDB
* MongoDB Atlas
* Mongoose

## Crawler

* Python
* vnstock
* pandas
* requests
* beautifulsoup4

## Cron

* node-cron

Future:

* BullMQ
* Redis

## Deployment

* Frontend: Vercel / DigitalOcean
* Backend: DigitalOcean
* Database: MongoDB Atlas
* Mobile: Expo
* CI/CD: GitHub Actions

---

# 6. Database Collections

## Dimension Collections

* dim_time
* dim_markets
* dim_industries
* dim_stocks
* dim_data_sources
* dim_report_periods

## Fact Collections

* fact_market_prices
* fact_financial_statements
* fact_financial_report_sources
* fact_crawl_quality
* fact_market_overview

## Operational Collections

* users
* roles
* watchlists
* crawl_jobs
* crawl_logs
* crawl_log_details

---

# 7. Business Rules

## BR-01

Watchlist tối đa 5 mã.

## BR-02

Không lưu trùng giá:

```text
stock_id + time_id + data_source_id
```

## BR-03

Không lưu trùng báo cáo tài chính:

```text
stock_id + report_period_id + data_source_id
```

## BR-04

Một mã lỗi không dừng toàn job.

## BR-05

User không được truy cập API Admin.

## BR-06

Admin có quyền cao nhất.

---

# 8. Development Phases

## Phase 1 — MVP Data Platform

* Authentication
* User Management
* Stock List
* Watchlist
* HOSE Crawling
* OHLCV Storage
* Dashboard
* Crawl Logs
* Data Quality

## Phase 2 — Financial Dashboard

* Financial Statements
* Financial Report Sources
* Financial Dashboard
* Financial Comparison

## Phase 3 — AI Prediction

* Feature Engineering
* Uptrend
* Downtrend
* Sideway
* Backtesting
* Model Evaluation

## Phase 4 — News & Sentiment

* Company News
* Industry News
* Sentiment Analysis

## Phase 5 — Portfolio Analytics

* Portfolio Management
* Performance Analysis
* Asset Allocation Recommendation

---

# 9. Kết luận

Dự án được xây dựng theo định hướng:

> Lấy đúng → Lưu đúng → Hiển thị đúng → Hiển thị rõ → AI sau

MVP tập trung hoàn thiện nền tảng dữ liệu, dashboard, watchlist, hệ thống crawl và data quality trước khi phát triển các tính năng AI.

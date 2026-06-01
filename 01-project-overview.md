# AI-Based Stock Trend Prediction

## 1. Tên dự án

**AI-Based Stock Trend Prediction**

---

## 2. Bối cảnh dự án

Dự án được xây dựng nhằm phát triển một nền tảng **Web/Mobile** hỗ trợ người dùng theo dõi, phân tích và trực quan hóa dữ liệu chứng khoán Việt Nam.

Ở giai đoạn MVP, hệ thống **không tập trung phát triển AI ngay lập tức** mà ưu tiên xây dựng nền tảng dữ liệu vững chắc theo định hướng của mentor:

```text
Lấy đúng
→ Lưu đúng
→ Hiển thị đúng
→ Hiển thị rõ
→ Sau đó mới phát triển AI
```

### Trọng tâm hiện tại

- Thu thập dữ liệu chứng khoán đáng tin cậy.
- Xây dựng Data Warehouse phục vụ lưu trữ dữ liệu lịch sử.
- Xây dựng Dashboard trực quan hóa dữ liệu.
- Xây dựng Backend API cung cấp dữ liệu cho Web và Mobile.
- Đảm bảo dữ liệu có khả năng mở rộng cho các bài toán AI trong tương lai.

---

## 3. Phạm vi MVP

### Tập trung vào

#### Sàn giao dịch

- HOSE

#### Dữ liệu

- Dữ liệu thị trường
- Dữ liệu tài chính doanh nghiệp
- Dữ liệu báo cáo tài chính

### Chưa tập trung vào

- Realtime Tick Data
- AI Prediction
- News Sentiment
- Portfolio Optimization

Các module trên sẽ được triển khai ở các phase tiếp theo.

---

## 4. Mục tiêu hệ thống

### 4.1 Thu thập dữ liệu chứng khoán

Nguồn dữ liệu:

- Cron Job
- Crawler
- API

Dữ liệu được cập nhật tự động:

- Giá mở cửa (Open)
- Giá đóng cửa (Close)
- Giá cao nhất (High)
- Giá thấp nhất (Low)
- Khối lượng giao dịch (Volume)
- Chỉ số tài chính
- Báo cáo tài chính

---

### 4.2 Lưu trữ dữ liệu lịch sử

Dữ liệu sau khi crawl sẽ được lưu vào Data Warehouse.

Mục tiêu:

- Lưu trữ dài hạn
- Tránh dữ liệu trùng lặp
- Hỗ trợ phân tích đa chiều
- Hỗ trợ Dashboard
- Hỗ trợ AI phase sau

---

### 4.3 Trực quan hóa dữ liệu

Web/Mobile App cần hiển thị:

- Price Chart
- Candlestick Chart
- Volume Chart
- Foreign Trading Chart
- Financial Dashboard
- Market Overview Dashboard

---

### 4.4 Theo dõi cổ phiếu yêu thích

Người dùng có thể:

- Đăng ký
- Đăng nhập
- Theo dõi cổ phiếu
- Xem Dashboard cá nhân

---

### 4.5 Quản trị dữ liệu

Staff/Admin có thể:

- Quản lý nguồn dữ liệu
- Quản lý lịch crawl
- Kiểm tra log crawl
- Kiểm tra chất lượng dữ liệu

---

## 5. Kiến trúc dữ liệu

Hệ thống được thiết kế theo hướng:

### Multi-dimensional Data Warehouse

### Dimension Tables

- dim_time
- dim_markets
- dim_stocks
- dim_data_sources

### Fact Tables

- fact_market_prices
- fact_financial_statements
- fact_financial_report_sources
- fact_crawl_quality

### Khả năng phân tích

- Theo thời gian
- Theo mã cổ phiếu
- Theo sàn giao dịch
- Theo nguồn dữ liệu
- Theo báo cáo tài chính

---

## 6. Dữ liệu thị trường được lưu

Hệ thống lưu các chỉ số thị trường hằng ngày:

### Giá & Khối lượng

- Open
- High
- Low
- Close
- Volume

### Giao dịch

- Bid Volume
- Ask Volume
- Foreign Buy
- Foreign Sell
- Foreign Net

### Định giá

- Market Cap
- EPS
- P/E
- Forward P/E
- BVPS
- P/B

### Chỉ số hiệu quả

- Beta
- ROS
- ROE
- ROAA

---

## 7. Dữ liệu báo cáo tài chính

Hệ thống lưu báo cáo tài chính theo quý.

### Doanh thu & Lợi nhuận

- Net Revenue
- Gross Profit
- Profit Before Tax
- Profit After Tax
- Parent Company Profit

### Tài sản & Nợ phải trả

- Current Assets
- Total Assets
- Liabilities
- Current Liabilities
- Equity

### Chỉ số tài chính

- EPS
- P/E
- P/B
- ROE
- ROAA

### Đối với ngành ngân hàng

- Net Interest Income
- Customer Loans
- Customer Deposits
- Operating Expense

---

## 8. Dữ liệu báo cáo tài chính gốc

Ngoài dữ liệu đã chuẩn hóa, hệ thống còn lưu:

- Link báo cáo tài chính
- Tính hợp lệ của URL
- Kỳ báo cáo
- Dữ liệu được trích xuất từ báo cáo gốc

### Mục đích

- Đảm bảo khả năng truy xuất nguồn gốc dữ liệu
- Đối chiếu khi xảy ra sai lệch
- Tăng độ tin cậy của hệ thống

---

## 9. Nhóm người dùng

### User

Có thể:

- Đăng ký
- Đăng nhập
- Theo dõi cổ phiếu yêu thích
- Xem dashboard

### Staff

Có thể:

- Quản lý nguồn dữ liệu
- Quản lý crawl job
- Kiểm tra crawl logs
- Theo dõi chất lượng dữ liệu

### Admin

Có thể:

- Quản lý User
- Quản lý Staff
- Quản lý Market
- Quản lý Stock
- Xem dashboard tổng quan hệ thống

---

## 10. Hệ thống Crawl & Data Quality

Để đảm bảo nguyên tắc:

```text
Lấy đúng
→ Lưu đúng
```

Hệ thống sử dụng:

- crawl_jobs
- crawl_logs
- crawl_log_details
- fact_crawl_quality

Cho phép theo dõi:

- Job nào chạy
- Chạy lúc nào
- Lấy được bao nhiêu dữ liệu
- Có lỗi không
- Mã nào lỗi
- Nguồn nào lỗi nhiều
- Tỷ lệ thành công bao nhiêu

---

## 11. Định hướng AI Phase Sau

Sau khi Data Warehouse ổn định, hệ thống sẽ mở rộng sang:

### AI Prediction

- Uptrend
- Downtrend
- Sideway

### News Analysis

- Tin tức doanh nghiệp
- Tin tức ngành
- Tin tức thị trường

### Sentiment Analysis

- Positive
- Neutral
- Negative

### Portfolio Analytics

- Quản lý danh mục
- Đánh giá hiệu suất
- Gợi ý phân bổ tài sản

---

## 12. Kết luận

Dự án được xây dựng theo định hướng:

## Data First → AI Later

Trong MVP, hệ thống tập trung vào:

- Thu thập dữ liệu
- Lưu trữ dữ liệu
- Quản lý chất lượng dữ liệu
- Dashboard trực quan hóa
- Watchlist người dùng

Thông qua mô hình Data Warehouse đa chiều, hệ thống có thể hỗ trợ phân tích chứng khoán từ nhiều góc nhìn khác nhau và tạo nền tảng dữ liệu đủ mạnh để phát triển các tính năng AI trong các giai đoạn tiếp theo.

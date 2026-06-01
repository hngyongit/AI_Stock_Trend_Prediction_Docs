# ERD Database Design

## 1. Tổng quan thiết kế database

Database chia thành **5 module chính**:

1. **User Management**: quản lý user, role, phân quyền.
2. **Market & Stock Master Data**: quản lý sàn HOSE và danh sách mã cổ phiếu.
3. **Stock Data Warehouse**: lưu lịch sử OHLCV cuối phiên.
4. **Data Crawling Management**: quản lý nguồn dữ liệu, lịch crawl, log crawl.
5. **Watchlist / Favorite**: user theo dõi mã cổ phiếu yêu thích.

---

## 2. ERD

> Link xem chi tiết: **Xem rõ hơn tại đây**

---

# 3. Nhóm Dimension Tables

## 3.1. `dim_time`

Bảng lưu thông tin thời gian để phân tích dữ liệu theo ngày, tuần, tháng, quý, năm.

| Field | Mô tả |
|---|---|
| `time_id` | Khóa chính, thường dùng dạng `YYYYMMDD`, ví dụ `20260531` |
| `full_date` | Ngày đầy đủ |
| `day` | Ngày trong tháng |
| `month` | Tháng |
| `quarter` | Quý |
| `year` | Năm |
| `week_of_year` | Tuần trong năm |
| `weekday` | Thứ trong tuần |
| `is_trading_day` | Có phải ngày giao dịch không |
| `created_at` | Ngày tạo record |

### Mục đích

- Phân tích giá theo ngày / tuần / tháng / quý / năm.
- Hỗ trợ thống kê dữ liệu thị trường theo thời gian.

### Ví dụ

- Top cổ phiếu tăng mạnh nhất trong tuần.
- Tổng volume theo tháng.
- So sánh P/E theo quý.

---

## 3.2. `dim_markets`

Bảng lưu thông tin sàn giao dịch.

| Field | Mô tả |
|---|---|
| `market_id` | Khóa chính |
| `code` | Mã sàn, ví dụ `HOSE`, `HNX`, `UPCOM` |
| `name` | Tên đầy đủ của sàn |
| `country` | Quốc gia |
| `timezone` | Múi giờ |
| `status` | Trạng thái hoạt động |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Ràng buộc

- `code` unique.

### MVP hiện tại

- Chỉ cần lưu **HOSE** trước.

---

## 3.3. `dim_industries`

Bảng lưu ngành/lĩnh vực của cổ phiếu.

| Field | Mô tả |
|---|---|
| `industry_id` | Khóa chính |
| `industry_name` | Tên ngành, ví dụ Banking, Technology, Real Estate |
| `sector_name` | Nhóm lĩnh vực lớn |
| `description` | Mô tả ngành |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Mục đích

- Phân tích cổ phiếu theo ngành.

### Ví dụ

- Ngành ngân hàng có ROE trung bình bao nhiêu?
- Ngành bất động sản có volume giao dịch thế nào?
- Ngành công nghệ tăng trưởng ra sao?

---

## 3.4. `dim_stocks`

Bảng lưu thông tin mã cổ phiếu.

| Field | Mô tả |
|---|---|
| `stock_id` | Khóa chính |
| `market_id` | Khóa ngoại sang `dim_markets` |
| `industry_id` | Khóa ngoại sang `dim_industries` |
| `symbol` | Mã cổ phiếu, ví dụ `FPT`, `VNM`, `HPG` |
| `company_name` | Tên công ty |
| `exchange_code` | Mã sàn giao dịch |
| `status` | Trạng thái cổ phiếu: active, delisted, suspended |
| `listed_date` | Ngày niêm yết |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Ràng buộc

- `unique(market_id, symbol)`

### Mục đích

- Lưu master data của cổ phiếu.
- Không lưu giá trong bảng này.

---

## 3.5. `dim_data_sources`

Bảng lưu nguồn dữ liệu crawl.

| Field | Mô tả |
|---|---|
| `data_source_id` | Khóa chính |
| `name` | Tên nguồn dữ liệu, ví dụ `vnstock`, `cafef`, `ssi` |
| `provider_type` | Loại nguồn: API, crawler, file import, python library |
| `base_url` | URL hoặc thông tin nguồn |
| `description` | Mô tả nguồn |
| `status` | Trạng thái nguồn |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Ràng buộc

- `name` unique.

### Mục đích

- Biết dữ liệu được lấy từ đâu.
- Theo dõi nguồn nào ổn định hoặc lỗi nhiều.

---

## 3.6. `dim_report_periods`

Bảng lưu kỳ báo cáo tài chính.

| Field | Mô tả |
|---|---|
| `report_period_id` | Khóa chính |
| `fiscal_year` | Năm tài chính |
| `fiscal_quarter` | Quý tài chính |
| `period_name` | Tên kỳ, ví dụ `Q1/2026` |
| `period_start_date` | Ngày bắt đầu kỳ |
| `period_end_date` | Ngày kết thúc kỳ |
| `is_latest` | Có phải kỳ mới nhất không |
| `created_at` | Ngày tạo |

### Mục đích

- Phân tích báo cáo tài chính theo quý/năm.

---

# 4. Nhóm Fact Tables

## 4.1. `fact_market_prices`

Bảng lưu dữ liệu thị trường hằng ngày của từng mã cổ phiếu. Đây là bảng quan trọng nhất cho dashboard giá.

| Field | Mô tả |
|---|---|
| `market_price_id` | Khóa chính |
| `stock_id` | Khóa ngoại sang `dim_stocks` |
| `market_id` | Khóa ngoại sang `dim_markets` |
| `industry_id` | Khóa ngoại sang `dim_industries` |
| `data_source_id` | Khóa ngoại sang `dim_data_sources` |
| `time_id` | Khóa ngoại sang `dim_time` |
| `open_price` | Giá mở cửa |
| `high_price` | Giá cao nhất trong ngày |
| `low_price` | Giá thấp nhất trong ngày |
| `close_price` | Giá đóng cửa |
| `volume` | Tổng khối lượng giao dịch |
| `bid_volume` | Khối lượng đặt mua |
| `ask_volume` | Khối lượng đặt bán |
| `foreign_buy` | Khối ngoại mua |
| `foreign_sell` | Khối ngoại bán |
| `foreign_net` | Chênh lệch mua/bán khối ngoại |
| `market_cap` | Vốn hóa thị trường |
| `eps` | Thu nhập trên mỗi cổ phiếu |
| `pe` | Chỉ số P/E |
| `forward_pe` | P/E dự phóng |
| `bvps` | Giá trị sổ sách mỗi cổ phiếu |
| `pb` | Chỉ số P/B |
| `beta` | Mức biến động so với thị trường |
| `ros` | Tỷ suất lợi nhuận trên doanh thu |
| `roe` | Tỷ suất lợi nhuận trên vốn chủ sở hữu |
| `roaa` | Tỷ suất lợi nhuận trên tổng tài sản bình quân |
| `price_change` | Mức thay đổi giá |
| `price_change_percent` | % thay đổi giá |
| `crawled_at` | Thời điểm crawl |
| `created_at` | Thời điểm lưu vào DB |

### Ràng buộc

- `unique(stock_id, time_id, data_source_id)`

### Mục đích

Dùng cho:

- Chart giá
- Volume
- Khối ngoại
- Market cap
- P/E
- P/B
- ROE
- ROA

---

## 4.2. `fact_financial_statements`

Bảng lưu báo cáo tài chính theo quý của từng mã cổ phiếu.

| Field | Mô tả |
|---|---|
| `financial_statement_id` | Khóa chính |
| `stock_id` | Khóa ngoại sang `dim_stocks` |
| `report_period_id` | Khóa ngoại sang `dim_report_periods` |
| `data_source_id` | Khóa ngoại sang `dim_data_sources` |
| `net_revenue` | Doanh thu thuần |
| `gross_profit` | Lợi nhuận gộp |
| `net_profit_from_operating_activities` | Lợi nhuận thuần từ hoạt động kinh doanh |
| `corporate_income_tax` | Thuế thu nhập doanh nghiệp |
| `profit_before_tax` | Lợi nhuận trước thuế |
| `profit_after_tax` | Lợi nhuận sau thuế |
| `parent_company_profit` | Lợi nhuận thuộc về công ty mẹ |
| `current_assets` | Tài sản ngắn hạn |
| `total_assets` | Tổng tài sản |
| `liabilities` | Tổng nợ phải trả |
| `current_liabilities` | Nợ ngắn hạn |
| `equity` | Vốn chủ sở hữu |
| `net_interest_income` | Thu nhập lãi thuần, dùng cho ngân hàng |
| `operating_expense` | Chi phí hoạt động, dùng cho ngân hàng |
| `total_operating_income` | Tổng thu nhập hoạt động, dùng cho ngân hàng |
| `customer_loans` | Dư nợ cho vay khách hàng, dùng cho ngân hàng |
| `customer_deposits` | Tiền gửi khách hàng, dùng cho ngân hàng |
| `retained_earnings` | Lợi nhuận giữ lại |
| `eps` | EPS |
| `pe` | P/E |
| `forward_pe` | Forward P/E |
| `bvps` | BVPS |
| `pb` | P/B |
| `beta` | Beta |
| `ros` | ROS |
| `roe` | ROE |
| `roaa` | ROAA |
| `crawled_at` | Thời điểm crawl |
| `created_at` | Thời điểm lưu |

### Ràng buộc

- `unique(stock_id, report_period_id, data_source_id)`

### Mục đích

- Phân tích sức khỏe tài chính doanh nghiệp theo quý.

---

## 4.3. `fact_financial_report_sources`

Bảng lưu thông tin và dữ liệu lấy từ báo cáo tài chính gốc.

| Field | Mô tả |
|---|---|
| `report_source_id` | Khóa chính |
| `stock_id` | Khóa ngoại sang `dim_stocks` |
| `report_period_id` | Khóa ngoại sang `dim_report_periods` |
| `data_source_id` | Khóa ngoại sang `dim_data_sources` |
| `source_url` | Link báo cáo tài chính gốc |
| `is_valid_url` | Link có hợp lệ không |
| `report_file_type` | Loại file, ví dụ PDF, HTML, XLSX |
| `report_status` | Trạng thái báo cáo |
| `bctt_net_revenue` | Doanh thu thuần từ BCTT gốc |
| `bctt_cost_of_goods_sold` | Giá vốn hàng bán |
| `bctt_gross_profit` | Lợi nhuận gộp |
| `bctt_financial_income` | Doanh thu tài chính |
| `bctt_financial_expense` | Chi phí tài chính |
| `bctt_selling_expense` | Chi phí bán hàng |
| `bctt_admin_expense` | Chi phí quản lý doanh nghiệp |
| `bctt_net_operating_profit` | Lợi nhuận thuần từ hoạt động kinh doanh |
| `bctt_other_profit` | Lợi nhuận khác |
| `bctt_associate_jv_profit` | Lãi/lỗ từ công ty liên doanh, liên kết |
| `bctt_profit_before_tax` | Lợi nhuận trước thuế |
| `bctt_profit_after_tax` | Lợi nhuận sau thuế |
| `bctt_parent_company_profit` | Lợi nhuận thuộc về công ty mẹ |
| `bctt_basic_eps` | EPS cơ bản |
| `crawled_at` | Thời điểm crawl |
| `created_at` | Thời điểm lưu |

### Ràng buộc

- `unique(stock_id, report_period_id, source_url)`

### Mục đích

- Lưu nguồn báo cáo gốc để kiểm chứng dữ liệu tài chính.

---

## 4.4. `fact_crawl_quality`

Bảng lưu chất lượng crawl theo từng job, từng nguồn, từng ngày.

| Field | Mô tả |
|---|---|
| `crawl_quality_id` | Khóa chính |
| `crawl_job_id` | Khóa ngoại sang `crawl_jobs` |
| `data_source_id` | Khóa ngoại sang `dim_data_sources` |
| `market_id` | Khóa ngoại sang `dim_markets` |
| `time_id` | Khóa ngoại sang `dim_time` |
| `records_fetched` | Số record lấy được |
| `records_inserted` | Số record thêm mới |
| `records_updated` | Số record cập nhật |
| `records_failed` | Số record lỗi |
| `success_rate` | Tỷ lệ thành công |
| `status` | Trạng thái crawl |
| `created_at` | Thời điểm tạo |

### Mục đích

- Theo dõi chất lượng dữ liệu: crawl đủ chưa, lỗi bao nhiêu, nguồn nào ổn định.

---

## 4.5. `fact_market_overview`

Bảng lưu dữ liệu tổng quan thị trường theo từng ngày, từng sàn và từng nguồn dữ liệu.

| Field | Ý nghĩa |
|---|---|
| `market_overview_id` | Khóa chính |
| `market_id` | FK → `dim_markets` |
| `time_id` | FK → `dim_time` |
| `data_source_id` | FK → `dim_data_sources` |
| `total_volume` | Tổng khối lượng giao dịch toàn thị trường |
| `total_value` | Tổng giá trị giao dịch toàn thị trường |
| `total_deal_volume` | Tổng khối lượng giao dịch thỏa thuận |
| `total_deal_value` | Tổng giá trị giao dịch thỏa thuận |
| `market_cap` | Tổng vốn hóa thị trường |
| `number_of_stocks` | Số lượng mã trên thị trường |
| `listed_volume` | Tổng khối lượng niêm yết |
| `circulating_volume` | Tổng khối lượng lưu hành |
| `foreign_buy` | Tổng khối ngoại mua |
| `foreign_sell` | Tổng khối ngoại bán |
| `foreign_net` | Chênh lệch mua/bán khối ngoại |
| `bid_volume` | Tổng khối lượng đặt mua |
| `ask_volume` | Tổng khối lượng đặt bán |
| `created_at` | Ngày tạo |

### Mục đích

- Phục vụ dashboard đánh giá sức khỏe thị trường.
- Không phải dữ liệu riêng của từng mã cổ phiếu.

---

# 5. Nhóm Crawl Management

## 5.1. `crawl_jobs`

Bảng lưu cấu hình lịch crawl.

| Field | Mô tả |
|---|---|
| `crawl_job_id` | Khóa chính |
| `data_source_id` | Khóa ngoại sang `dim_data_sources` |
| `market_id` | Khóa ngoại sang `dim_markets` |
| `job_name` | Tên job |
| `data_type` | Loại dữ liệu crawl |
| `cron_expression` | Lịch chạy cron |
| `crawl_mode` | Chế độ crawl: scheduled, manual |
| `status` | Trạng thái job |
| `last_run_at` | Lần chạy gần nhất |
| `next_run_at` | Lần chạy tiếp theo |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Ví dụ `data_type`

- `DAILY_MARKET_PRICE`
- `QUARTERLY_FINANCIAL_STATEMENT`
- `FINANCIAL_REPORT_SOURCE`

### Mục đích

- Staff/Admin quản lý lịch crawl dữ liệu.

---

## 5.2. `crawl_logs`

Bảng lưu log tổng quan cho mỗi lần chạy crawl.

| Field | Mô tả |
|---|---|
| `crawl_log_id` | Khóa chính |
| `crawl_job_id` | Khóa ngoại sang `crawl_jobs` |
| `started_at` | Thời điểm bắt đầu |
| `ended_at` | Thời điểm kết thúc |
| `status` | `SUCCESS`, `FAILED`, `PARTIAL_SUCCESS` |
| `records_fetched` | Số record lấy được |
| `records_inserted` | Số record thêm mới |
| `records_updated` | Số record cập nhật |
| `records_failed` | Số record lỗi |
| `error_message` | Nội dung lỗi tổng quan |
| `created_at` | Ngày tạo log |

### Mục đích

- Biết mỗi lần crawl có chạy thành công không.

---

## 5.3. `crawl_log_details`

Bảng lưu chi tiết từng mã trong một lần crawl.

| Field | Mô tả |
|---|---|
| `crawl_log_detail_id` | Khóa chính |
| `crawl_log_id` | Khóa ngoại sang `crawl_logs` |
| `stock_id` | Khóa ngoại sang `dim_stocks` |
| `symbol` | Mã cổ phiếu |
| `data_type` | Loại dữ liệu bị ảnh hưởng |
| `status` | `SUCCESS`, `FAILED`, `SKIPPED` |
| `message` | Mô tả lỗi hoặc kết quả |
| `created_at` | Ngày tạo |

### Mục đích

- Biết cụ thể mã nào crawl thành công, mã nào lỗi, lỗi vì sao.

---

# 6. Nhóm User Management

## 6.1. `roles`

Bảng lưu vai trò người dùng.

| Field | Mô tả |
|---|---|
| `role_id` | Khóa chính |
| `name` | Tên role |
| `description` | Mô tả role |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Role đề xuất

- `USER`
- `STAFF`
- `ADMIN`

### Ràng buộc

- `name` unique.

---

## 6.2. `users`

Bảng lưu tài khoản người dùng.

| Field | Mô tả |
|---|---|
| `user_id` | Khóa chính |
| `role_id` | Khóa ngoại sang `roles` |
| `full_name` | Họ tên |
| `email` | Email đăng nhập |
| `password_hash` | Mật khẩu đã mã hóa |
| `status` | Trạng thái tài khoản |
| `created_at` | Ngày tạo |
| `updated_at` | Ngày cập nhật |

### Ràng buộc

- `email` unique.

### Mục đích

- Quản lý user, staff, admin.

---

## 6.3. `watchlists`

Bảng lưu danh sách cổ phiếu yêu thích của user.

| Field | Mô tả |
|---|---|
| `watchlist_id` | Khóa chính |
| `user_id` | Khóa ngoại sang `users` |
| `stock_id` | Khóa ngoại sang `dim_stocks` |
| `created_at` | Thời điểm thêm vào watchlist |

### Ràng buộc

- `unique(user_id, stock_id)`

### MVP có thể giới hạn

- Mỗi user theo dõi tối đa 5 mã cổ phiếu.

---

# 7. Chốt vai trò từng bảng

| Bảng | Vai trò |
|---|---|
| `dim_time` | Dimension thời gian |
| `dim_markets` | Dimension sàn giao dịch |
| `dim_industries` | Dimension ngành |
| `dim_stocks` | Dimension mã cổ phiếu |
| `dim_data_sources` | Dimension nguồn dữ liệu |
| `dim_report_periods` | Dimension kỳ báo cáo |
| `fact_market_prices` | Fact dữ liệu thị trường hằng ngày |
| `fact_financial_statements` | Fact báo cáo tài chính theo quý |
| `fact_financial_report_sources` | Fact nguồn báo cáo gốc |
| `fact_crawl_quality` | Fact chất lượng crawl |
| `fact_market_overview` | Fact tổng quan thị trường |
| `crawl_jobs` | Cấu hình job crawl |
| `crawl_logs` | Log tổng quan mỗi lần crawl |
| `crawl_log_details` | Log chi tiết từng mã |
| `roles` | Phân quyền |
| `users` | Người dùng |
| `watchlists` | Cổ phiếu yêu thích |

---

# 8. Kết luận

ERD này không chỉ lưu **OHLCV** mà đã đi theo hướng **Data Warehouse đa chiều**.

Hệ thống có thể phân tích theo:

- Thời gian
- Mã cổ phiếu
- Ngành
- Sàn giao dịch
- Nguồn dữ liệu
- Kỳ báo cáo
- Chất lượng crawl

Thiết kế này phù hợp để xây dựng dashboard ở MVP và làm nền tảng dữ liệu cho AI phase sau.

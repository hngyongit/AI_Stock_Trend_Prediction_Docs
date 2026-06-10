# System Structure

## 1. Tổng quan kiến trúc hệ thống

Dự án **AI-Based Stock Trend Prediction** được xây dựng theo mô hình **Monorepo Architecture**, quản lý tập trung toàn bộ source code của:

- Web Application
- Mobile Application
- Backend API
- Python Crawler Service
- Shared Libraries
- Documentation
- Infrastructure / Deployment

MVP tập trung theo định hướng:

```text
Data First → AI Later
```

Ưu tiên hiện tại:

1. Thu thập dữ liệu chứng khoán.
2. Lưu dữ liệu vào MongoDB.
3. Hiển thị dashboard.
4. Quản lý watchlist.
5. Quản lý crawl job và crawl logs.
6. Chuẩn bị nền tảng cho AI phase sau.

---

# 2. Tech Stack chính

| Layer           | Tech                                     |
| --------------- | ---------------------------------------- |
| Web             | ReactJS + Vite + JavaScript              |
| Mobile          | React Native + Expo + JavaScript         |
| Backend         | Node.js + Express.js + JavaScript        |
| Database        | MongoDB + Mongoose                       |
| Data Warehouse  | MongoDB Collections theo Dim/Fact        |
| Crawler         | Python + vnstock + pandas                |
| Cron            | Node-cron                                |
| Queue           | BullMQ + Redis nếu cần                   |
| Chart           | TradingView Lightweight Charts + ECharts |
| Auth            | JWT + Bcrypt + RBAC                      |
| API Docs        | Swagger / OpenAPI                        |
| Logging         | Winston + Morgan                         |
| Deploy          | DigitalOcean                             |
| Version Control | GitHub                                   |
| AI Later        | Python + Scikit-learn/XGBoost/PyTorch    |

---

# 3. Root Structure

```text
ai-stock-trend-prediction/
│
├── apps/
│   ├── web/
│   └── mobile/
│
├── services/
│   ├── api/
│   └── crawler/
│
├── packages/
│   ├── shared-types/
│   ├── shared-utils/
│   └── api-client/
│
├── docs/
├── infra/
├── scripts/
│
├── .env.example
├── .gitignore
├── README.md
└── package.json
```

---

# 4. apps/web

Web Application dùng cho User, Staff và Admin (React 19 + Vite 8 + TypeScript).

## Tech Stack

- React 19
- Vite 8
- TypeScript 6
- Tailwind CSS v4
- shadcn/ui primitives (Button, Input, Avatar, Badge, Dialog, Sonner)
- ECharts 6 + echarts-for-react
- Axios
- React Router v6
- Zustand v5 (auth state)
- Formik + Yup (forms)
- sonner (toasts)

## Structure

```text
web/
│
├── public/
│
├── src/
│   ├── main.tsx                  # Entry point
│   ├── App.tsx                   # Root: AuthProvider → BrowserRouter → AppRoutes → Toaster
│   ├── index.css                 # Tailwind v4 + shadcn CSS variables + Geist font + dark theme
│   │
│   ├── components/
│   │   ├── ui/                   # shadcn primitives (button, input, avatar, badge, dialog, sonner)
│   │   ├── topbar/               # Shared topbar (BrandLogo, SearchInput, NotificationIcon, UserMenu)
│   │   ├── admin-shell/          # Admin layout sidebar + shell
│   │   └── staff-shell/          # Staff layout sidebar + shell
│   │
│   ├── layouts/
│   │   ├── UserLayout.tsx        # Sidebar + Topbar + content area
│   │   ├── StaffLayout.tsx       # Staff shell + Topbar
│   │   └── AdminLayout.tsx       # Admin shell + Topbar
│   │
│   ├── pages/
│   │   ├── LandingPage/
│   │   ├── LoginPage/
│   │   ├── Register/
│   │   ├── ForgotPassword/
│   │   ├── UserDashboard/
│   │   ├── StockListPage/
│   │   ├── StockDetailPage/
│   │   ├── WatchlistPage/
│   │   ├── AlertsPage/
│   │   ├── ComparisonPage/
│   │   ├── HistoricalAnalysisPage/
│   │   ├── SettingsPage/
│   │   ├── UserProfilePage/
│   │   ├── StaffDashboardPage/
│   │   ├── CrawlJobsPage/
│   │   ├── CrawlLogsPage/
│   │   ├── DataSourcesPage/
│   │   ├── DataValidationPage/
│   │   ├── EtlMonitorPage/
│   │   ├── ImportHistoryPage/
│   │   ├── StockDataMonitorPage/
│   │   ├── Admin/                # AdminUserManagement, AdminStockList, AdminShell
│   │   ├── UserPlaceholderPage/
│   │   └── NothingHere/          # 404 / access denied
│   │
│   ├── routes/
│   │   ├── AppRoutes.tsx         # Route tree (public + USER + STAFF + ADMIN + fallback)
│   │   ├── layoutRoutes.tsx      # USER_ROUTES, STAFF_ROUTES, ADMIN_ROUTES arrays
│   │   ├── RequireAuth.tsx       # Auth guard + role check
│   │   └── renderProtectedLayoutRoute.tsx  # Route → Layout → RequireAuth wrapper
│   │
│   ├── providers/
│   │   └── AuthProvider.tsx      # Auth context provider (compatibility wrapper around useAuthStore)
│   │
│   ├── services/
│   │   └── auth.service.ts       # Auth API + Axios instance + refresh interceptor
│   │
│   ├── stores/
│   │   └── auth.store.ts         # Zustand: accessToken, refreshToken, user, isAuthenticated
│   │
│   ├── lib/
│   │   ├── role-routes.ts        # Route helpers by role (getDefaultHomeRouteByRole, etc.)
│   │   └── utils.ts              # cn() helper (clsx + tailwind-merge)
│   │
│   ├── types/                    # Shared TS types
│   └── shared/                   # Shared utilities
│
├── components.json               # shadcn config
├── eslint.config.js
├── package.json
├── tsconfig.json / tsconfig.app.json / tsconfig.node.json
└── vite.config.ts
```

## Chức năng chính

- Đăng nhập / đăng ký / quên mật khẩu
- Dashboard USER (tổng quan thị trường)
- Stock list + Stock detail (candlestick chart, volume)
- Historical analysis + Comparison
- Watchlist cá nhân
- Alerts management
- Staff: quản lý crawl job, crawl logs, data sources, ETL monitor, data validation
- Admin: quản lý user, stock, market coverage

---

# 5. apps/mobile

Mobile Application dành cho User (React Native + Expo).

## Tech Stack

- React Native 0.85
- Expo ~56
- TypeScript
- @react-navigation/native (stack + bottom-tabs)
- Zustand v5
- Axios
- Formik + Yup
- nativewind ^4
- tailwind-variants
- react-native-gesture-handler + react-native-reanimated
- lucide-react-native
- react-native-svg
- react-native-chart-kit

## Structure

```text
mobile/
│
├── App.tsx                    # Root: GestureHandlerRootView → ThemeProvider → NavigationContainer → RootNavigator
├── app.config.js
├── app.json
├── babel.config.js
├── metro.config.js
├── tailwind.config.js
├── tsconfig.json
├── package.json
│
├── assets/
│   ├── images/
│   │   └── tabIcons/
│   └── fonts/
│
└── src/
    ├── global.css
    │
    ├── app/
    │   ├── navigation/
    │   │   ├── RootNavigator.tsx
    │   │   ├── MainTabNavigator.tsx
    │   │   ├── AppTabBar.tsx
    │   │   └── navigation.types.ts
    │   └── config/
    │
    ├── features/
    │   ├── auth/
    │   ├── dashboard/
    │   ├── stocks/
    │   ├── watchlist/
    │   ├── alerts/
    │   ├── search/
    │   ├── profile/
    │   └── startup/
    │
    ├── shared/
    │   ├── design/
    │   │   └── tokens.ts
    │   ├── services/
    │   │   ├── api.service.ts
    │   │   └── tokenStorage.ts
    │   ├── ui/
    │   │   ├── primitives/          # Pure RN components (no gluestack)
    │   │   ├── components/
    │   │   ├── feedback/
    │   │   ├── forms/
    │   │   ├── layout/
    │   │   └── utils/
    │   │       └── ThemeProvider.tsx
    │   ├── hooks/
    │   ├── utils/
    │   ├── types/
    │   └── constants/
    │
    └── stores/
        ├── auth.store.ts
        ├── market.store.ts
        ├── app-shell.store.ts
        └── startup.store.ts
```

## Chức năng chính

- Đăng nhập / đăng ký
- Dashboard tổng quan thị trường
- Stock detail (OHLCV charts)
- Watchlist cá nhân (swipeable rows, filter chips)
- Alerts management
- Search stock
- Profile settings

---

# 6. services/api

Backend API chính của hệ thống.

## Tech Stack

- Node.js
- Express.js
- JavaScript
- MongoDB
- Mongoose
- JWT Authentication
- Bcrypt
- Swagger
- Node-cron
- Winston
- Morgan

## Structure

```text
services/api/
│
├── src/
│   ├── app.js
│   ├── server.js
│   │
│   ├── config/
│   │   ├── app.config.js
│   │   ├── database.config.js
│   │   ├── jwt.config.js
│   │   └── swagger.config.js
│   │
│   ├── common/
│   │   ├── middlewares/
│   │   ├── guards/
│   │   ├── validators/
│   │   ├── utils/
│   │   └── constants/
│   │
│   ├── database/
│   │   ├── models/
│   │   ├── seeds/
│   │   └── indexes/
│   │
│   ├── modules/
│   │   ├── auth/
│   │   ├── users/
│   │   ├── roles/
│   │   ├── markets/
│   │   ├── industries/
│   │   ├── stocks/
│   │   ├── watchlists/
│   │   ├── dashboard/
│   │   ├── financials/
│   │   ├── data-sources/
│   │   ├── crawl-jobs/
│   │   ├── crawl-logs/
│   │   └── market-overview/
│   │
│   ├── jobs/
│   │   ├── market-price.job.js
│   │   ├── financial-statement.job.js
│   │   └── market-overview.job.js
│   │
│   └── integrations/
│       └── crawler-client/
│           └── crawler-client.service.js
│
├── tests/
├── package.json
└── README.md
```

---

# 7. Backend Module Structure

Mỗi module trong Express nên tổ chức theo dạng:

```text
module-name/
│
├── module-name.routes.js
├── module-name.controller.js
├── module-name.service.js
├── module-name.repository.js
├── module-name.validation.js
└── module-name.constants.js
```

Ví dụ:

```text
stocks/
│
├── stocks.routes.js
├── stocks.controller.js
├── stocks.service.js
├── stocks.repository.js
├── stocks.validation.js
└── stocks.constants.js
```

---

# 8. Backend Modules

## 8.1. auth/

Quản lý xác thực.

```text
auth/
├── auth.routes.js
├── auth.controller.js
├── auth.service.js
├── auth.validation.js
└── auth.constants.js
```

Chức năng:

- Register
- Login
- JWT Access Token
- Refresh Token
- Password Hash bằng Bcrypt

---

## 8.2. users/

Quản lý người dùng.

```text
users/
├── users.routes.js
├── users.controller.js
├── users.service.js
├── users.repository.js
└── users.validation.js
```

Chức năng:

- Quản lý User
- Quản lý Staff
- Quản lý Admin
- Cập nhật trạng thái tài khoản

---

## 8.3. roles/

Quản lý phân quyền.

```text
roles/
├── roles.routes.js
├── roles.controller.js
├── roles.service.js
├── roles.repository.js
└── roles.validation.js
```

Role chính:

- USER
- STAFF
- ADMIN

---

## 8.4. markets/

Quản lý sàn giao dịch.

```text
markets/
├── markets.routes.js
├── markets.controller.js
├── markets.service.js
├── markets.repository.js
└── markets.validation.js
```

MVP chỉ cần HOSE trước.

---

## 8.5. industries/

Quản lý ngành/lĩnh vực cổ phiếu.

```text
industries/
├── industries.routes.js
├── industries.controller.js
├── industries.service.js
├── industries.repository.js
└── industries.validation.js
```

---

## 8.6. stocks/

Quản lý danh sách cổ phiếu.

```text
stocks/
├── stocks.routes.js
├── stocks.controller.js
├── stocks.service.js
├── stocks.repository.js
└── stocks.validation.js
```

Chức năng:

- Danh sách mã cổ phiếu
- Chi tiết cổ phiếu
- Tìm kiếm cổ phiếu
- Lọc theo ngành/sàn

---

## 8.7. watchlists/

Quản lý cổ phiếu yêu thích.

```text
watchlists/
├── watchlists.routes.js
├── watchlists.controller.js
├── watchlists.service.js
├── watchlists.repository.js
└── watchlists.validation.js
```

Chức năng:

- Follow cổ phiếu
- Unfollow cổ phiếu
- Xem danh sách cổ phiếu đang theo dõi
- MVP giới hạn mỗi user theo dõi tối đa 5 mã

---

## 8.8. dashboard/

Tổng hợp dữ liệu dashboard.

```text
dashboard/
├── dashboard.routes.js
├── dashboard.controller.js
├── dashboard.service.js
└── dashboard.repository.js
```

Chức năng:

- Price dashboard
- Volume dashboard
- Financial dashboard
- Market overview dashboard

---

## 8.9. financials/

Quản lý dữ liệu tài chính doanh nghiệp.

```text
financials/
├── financials.routes.js
├── financials.controller.js
├── financials.service.js
├── financials.repository.js
└── financials.validation.js
```

Chức năng:

- Báo cáo tài chính theo quý
- Chỉ số EPS, P/E, P/B, ROE, ROAA
- Link báo cáo tài chính gốc

---

## 8.10. data-sources/

Quản lý nguồn dữ liệu.

```text
data-sources/
├── data-sources.routes.js
├── data-sources.controller.js
├── data-sources.service.js
├── data-sources.repository.js
└── data-sources.validation.js
```

Ví dụ nguồn:

- vnstock
- cafef
- ssi

---

## 8.11. crawl-jobs/

Quản lý lịch crawl.

```text
crawl-jobs/
├── crawl-jobs.routes.js
├── crawl-jobs.controller.js
├── crawl-jobs.service.js
├── crawl-jobs.repository.js
├── crawl-jobs.validation.js
└── crawl-jobs.scheduler.js
```

Chức năng:

- Tạo crawl job
- Cập nhật lịch cron
- Bật/tắt job
- Chạy thủ công

---

## 8.12. crawl-logs/

Theo dõi log crawl.

```text
crawl-logs/
├── crawl-logs.routes.js
├── crawl-logs.controller.js
├── crawl-logs.service.js
└── crawl-logs.repository.js
```

Chức năng:

- Xem lịch sử crawl
- Xem mã nào crawl lỗi
- Xem tỷ lệ thành công
- Theo dõi records inserted/updated/failed

---

# 9. Database Structure

Database dùng MongoDB + Mongoose.

Database schema được đặt trong Backend API.

```text
services/api/src/database/
│
├── models/
│   ├── dim-time.model.js
│   ├── dim-market.model.js
│   ├── dim-industry.model.js
│   ├── dim-stock.model.js
│   ├── dim-data-source.model.js
│   ├── dim-report-period.model.js
│   │
│   ├── fact-market-price.model.js
│   ├── fact-market-overview.model.js
│   ├── fact-financial-statement.model.js
│   ├── fact-financial-report-source.model.js
│   ├── fact-crawl-quality.model.js
│   │
│   ├── user.model.js
│   ├── role.model.js
│   ├── watchlist.model.js
│   ├── crawl-job.model.js
│   ├── crawl-log.model.js
│   └── crawl-log-detail.model.js
│
├── seeds/
│   ├── seed-roles.js
│   ├── seed-markets.js
│   ├── seed-industries.js
│   └── seed-data-sources.js
│
└── indexes/
    ├── stock.index.js
    ├── market-price.index.js
    ├── financial-statement.index.js
    └── watchlist.index.js
```

---

# 10. MongoDB Collections

## 10.1. Dimension Collections

```text
dim_time
dim_markets
dim_industries
dim_stocks
dim_data_sources
dim_report_periods
```

## 10.2. Fact Collections

```text
fact_market_prices
fact_market_overview
fact_financial_statements
fact_financial_report_sources
fact_crawl_quality
```

## 10.3. Operational Collections

```text
users
roles
watchlists
crawl_jobs
crawl_logs
crawl_log_details
```

---

# 11. services/crawler

Python Crawler Service dùng để lấy dữ liệu chứng khoán.

## Tech Stack

- Python
- vnstock
- pandas
- requests
- beautifulsoup4
- pydantic

## Structure

```text
services/crawler/
│
├── src/
│   ├── crawlers/
│   │   ├── market_price_crawler.py
│   │   ├── market_overview_crawler.py
│   │   ├── financial_statement_crawler.py
│   │   └── report_source_crawler.py
│   │
│   ├── processors/
│   │   ├── market_price_processor.py
│   │   ├── market_overview_processor.py
│   │   └── financial_statement_processor.py
│   │
│   ├── validators/
│   │   ├── market_price_validator.py
│   │   └── financial_statement_validator.py
│   │
│   ├── exporters/
│   │   └── json_exporter.py
│   │
│   ├── utils/
│   └── main.py
│
├── requirements.txt
└── README.md
```

## Nhiệm vụ

- Crawl OHLCV
- Crawl dữ liệu tổng quan thị trường
- Crawl báo cáo tài chính
- Chuẩn hóa dữ liệu
- Validate missing value
- Trả JSON cho Backend API

---

# 12. Cron Job Flow

MVP sử dụng Node-cron trong Backend Express.

```text
Node-cron trong Express Backend
        │
        ▼
Gọi Python Crawler Service
        │
        ▼
Crawler lấy dữ liệu từ vnstock
        │
        ▼
Crawler trả JSON
        │
        ▼
Backend validate data
        │
        ▼
Backend lưu MongoDB
        │
        ▼
Backend ghi crawl_logs
```

Sau này nếu dữ liệu lớn hơn, có thể nâng cấp sang:

```text
BullMQ + Redis
```

---

# 13. packages/

Chứa code dùng chung giữa Web, Mobile và Backend.

```text
packages/
│
├── shared-types/
│   ├── user.type.js
│   ├── stock.type.js
│   ├── market-price.type.js
│   ├── financial-statement.type.js
│   └── watchlist.type.js
│
├── shared-utils/
│   ├── format-date.js
│   ├── format-currency.js
│   ├── calculate-percent.js
│   └── validate-symbol.js
│
└── api-client/
    ├── auth.api.js
    ├── stock.api.js
    ├── dashboard.api.js
    ├── watchlist.api.js
    └── crawl.api.js
```

---

# 14. docs/

Tài liệu dự án.

```text
docs/
│
├── BE/                  # Tài liệu chi tiết module Backend API
├── FE/                  # Tài liệu chi tiết phát triển Frontend & Mobile
├── 01-project-overview.md
├── 02-system-architecture-techstack.md
├── 03-erd.md
├── 04-project-structure.md
└── 05-SRS_V1.md
```

---

# 15. infra/

Hạ tầng và triển khai trên DigitalOcean.

```text
infra/
│
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.web
│   ├── Dockerfile.crawler
│   └── docker-compose.yml
│
├── nginx/
│   ├── nginx.conf
│   └── sites-available/
│       └── ai-stock.conf
│
├── digitalocean/
│   ├── droplet-setup.md
│   ├── deploy-api.md
│   ├── deploy-web.md
│   ├── deploy-crawler.md
│   └── firewall.md
│
├── github-actions/
│   ├── web-ci.yml
│   ├── api-ci.yml
│   └── crawler-ci.yml
│
└── monitoring/
    ├── pm2.md
    ├── nginx-log.md
    └── sentry.md
```

---

# 16. scripts/

Script hỗ trợ vận hành.

```text
scripts/
│
├── seed-db.sh
├── backup-db.sh
├── restore-db.sh
├── run-crawler.sh
├── create-indexes.sh
├── deploy-api.sh
├── deploy-web.sh
└── restart-services.sh
```

---

# 17. Deployment Structure

Triển khai trên **DigitalOcean Droplet**.

```text
User
 │
 ▼

Domain
 │
 ▼

Nginx Reverse Proxy
 │
 ├── Web App - React build
 │
 ├── Backend API - Node Express
 │
 └── Crawler Service - Python
 │
 ▼

MongoDB Atlas
```

Khuyến nghị:

- Web build bằng Vite rồi serve qua Nginx.
- Backend chạy bằng PM2.
- Crawler có thể chạy bằng Python command hoặc gọi từ Backend.
- MongoDB nên dùng MongoDB Atlas để giảm tải vận hành.
- DigitalOcean Droplet dùng để chạy Web, API, Crawler, Nginx.

---

# 18. DigitalOcean Deployment Flow

## 18.1. Thành phần deploy

| Thành phần      | Nơi deploy                          |
| --------------- | ----------------------------------- |
| Web React       | DigitalOcean Droplet + Nginx        |
| Backend Express | DigitalOcean Droplet + PM2          |
| Python Crawler  | DigitalOcean Droplet                |
| Database        | MongoDB Atlas                       |
| Reverse Proxy   | Nginx                               |
| Process Manager | PM2                                 |
| SSL             | Certbot / Let’s Encrypt             |
| CI/CD           | GitHub Actions hoặc deploy thủ công |

---

## 18.2. Cấu trúc trên server

```text
/var/www/ai-stock/
│
├── web/
│   └── dist/
│
├── api/
│   ├── src/
│   ├── package.json
│   └── ecosystem.config.js
│
├── crawler/
│   ├── src/
│   ├── requirements.txt
│   └── venv/
│
├── logs/
└── backups/
```

---

## 18.3. PM2 Process

```text
pm2 start services/api/src/server.js --name ai-stock-api
pm2 save
pm2 startup
```

Hoặc dùng file:

```text
ecosystem.config.js
```

Ví dụ:

```javascript
module.exports = {
  apps: [
    {
      name: "ai-stock-api",
      script: "src/server.js",
      cwd: "/var/www/ai-stock/api",
      env: {
        NODE_ENV: "production",
      },
    },
  ],
};
```

---

## 18.4. Nginx Routing

```text
https://domain.com              → React Web
https://domain.com/api          → Express API
https://domain.com/api/docs     → Swagger Docs
```

Ví dụ config:

```nginx
server {
    listen 80;
    server_name domain.com www.domain.com;

    root /var/www/ai-stock/web/dist;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:5000/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

# 19. System Flow

```text
User / Staff / Admin
        │
        ▼

Web App / Mobile App
        │
        ▼

Express Backend API
        │
        ├── Auth Module
        ├── Stock Module
        ├── Dashboard Module
        ├── Watchlist Module
        ├── Crawl Job Module
        └── Crawl Log Module
        │
        ▼

MongoDB Data Warehouse
        ▲
        │

Python Crawler Service
        │
        ▼

VNStock / CafeF / SSI
```

---

# 20. API Flow

```text
Frontend / Mobile
        │
        ▼
Axios gọi RESTful API
        │
        ▼
Express Routes
        │
        ▼
Controller
        │
        ▼
Service
        │
        ▼
Repository
        │
        ▼
Mongoose Model
        │
        ▼
MongoDB Atlas
```

---

# 21. Crawler Flow

```text
Node-cron
   │
   ▼
Run Python Crawler
   │
   ▼
Crawl data từ vnstock
   │
   ▼
Process bằng pandas
   │
   ▼
Validate bằng pydantic
   │
   ▼
Export JSON
   │
   ▼
Express Backend nhận JSON
   │
   ▼
Validate lần cuối
   │
   ▼
Save MongoDB
   │
   ▼
Write crawl_logs
```

---

# 22. AI Phase Future Structure

AI chưa triển khai sâu trong MVP. Sau khi dữ liệu ổn định, có thể bổ sung:

```text
services/
│
└── ai/
    ├── trend-prediction/
    ├── feature-engineering/
    ├── backtesting/
    ├── sentiment-analysis/
    └── portfolio-analytics/
```

## Tech Stack AI Phase

- Python
- Scikit-learn
- XGBoost
- LightGBM
- PyTorch
- TensorFlow
- FastAPI
- Pandas
- NumPy

---

# 23. Kết luận

Cấu trúc hệ thống này phù hợp với MVP vì:

- Bám sát stack JavaScript của team.
- Backend dùng Node.js + Express.js, dễ học và dễ triển khai.
- Database MongoDB được quản lý trong Backend bằng Mongoose.
- Python chỉ tập trung crawl và xử lý dữ liệu.
- Backend chịu trách nhiệm validate, lưu DB và cung cấp API.
- Web/Mobile chỉ tập trung hiển thị dữ liệu.
- DigitalOcean phù hợp để deploy Web, Backend, Crawler trên cùng một VPS.
- Có thể mở rộng sang AI phase sau mà không phải đổi toàn bộ kiến trúc.

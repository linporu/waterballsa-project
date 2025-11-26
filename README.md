# Waterball Software Academy - 課程平台重製版

這是水球軟體學院課程平台的重製專案，使用 Next.js + Spring Boot + PostgreSQL + Nginx 技術棧。

## 技術棧

- **前端**: Next.js 15 (React 19) + TypeScript + Tailwind CSS
- **後端**: Spring Boot 3.5 + Java 17 + JPA
- **資料庫**: PostgreSQL 16 + Liquibase
- **反向代理**: Nginx (支援 HTTP/HTTPS)
- **容器化**: Docker & Docker Compose
- **測試**: Playwright (E2E), REST Assured + Testcontainers (API)

## 專案結構

```
waterballsa-project/
├── docker-compose.yml          # Docker Compose 服務編排
├── .env                        # 環境變數（本地開發用）
├── .env.example                # 環境變數範例
├── Makefile                    # 常用指令（如 Swagger 合併）
│
├── www_root/                   # 應用程式碼
│   ├── waterballsa-frontend/  # Next.js 前端專案
│   │   ├── src/
│   │   │   ├── app/           # Next.js App Router
│   │   │   ├── components/    # React 元件
│   │   │   ├── hooks/         # 自定義 Hooks
│   │   │   ├── contexts/      # React Context
│   │   │   ├── lib/           # 工具函式與 API 客戶端
│   │   │   └── types/         # TypeScript 類型定義
│   │   ├── package.json
│   │   └── tests/             # Playwright E2E 測試
│   │
│   └── waterballsa-backend/   # Spring Boot 後端專案
│       ├── src/main/java/waterballsa/
│       │   ├── controller/    # REST Controllers
│       │   ├── service/       # 業務邏輯層
│       │   ├── repository/    # JPA Repositories
│       │   ├── entity/        # JPA Entities
│       │   ├── dto/           # Data Transfer Objects
│       │   ├── config/        # Spring 配置
│       │   └── exception/     # 異常處理
│       ├── src/main/resources/
│       │   └── db/changelog/  # Liquibase 遷移檔案
│       └── pom.xml
│
├── web_service/                # Nginx 配置
│   ├── etc/nginx/
│   │   ├── nginx.conf         # 主配置檔
│   │   ├── conf.d/            # 站點配置
│   │   └── ssl/               # SSL 憑證
│   └── logrotate.d/           # 日誌輪轉配置
│
├── java/docker_file/           # Java 應用 Dockerfile
├── next/docker_file/           # Next.js 應用 Dockerfile
│
├── db/                         # 資料庫持久化
│   └── postgres_data/
│
└── docs/                       # 文檔
    ├── Requirement.md          # 需求文檔
    ├── Release-1-Spec.md       # Release 1 規格
    ├── Release-2-Spec.md       # Release 2 規格
    ├── Strategy.md             # 開發策略
    └── api-docs/               # OpenAPI 文檔
        └── openapi/            # 模組化 API 規範
```

## 快速開始

### 前置需求

- Docker Desktop
- Docker Compose V2

### 啟動服務

1. 複製環境變數範例檔案：
   ```bash
   cp .env.example .env
   ```

2. （可選）修改 `.env` 中的配置

3. 啟動所有服務：
   ```bash
   docker compose up
   ```

   首次啟動會自動：
   - 建置前後端 Docker 映像
   - 初始化 PostgreSQL 資料庫
   - 執行 Liquibase 資料庫遷移
   - 生成自簽 SSL 憑證
   - 啟動所有服務

4. 訪問服務：
   - **前端**: https://localhost (HTTPS) 或 http://localhost (HTTP)
   - **後端 API**: https://localhost/api 或直接訪問 http://localhost:8080
   - **健康檢查**: http://localhost:8080/actuator/health
   - **資料庫**: localhost:5432 (需要 PostgreSQL 客戶端)

### 常用指令

```bash
# 啟動服務（重建映像）
docker compose up --build

# 背景執行
docker compose up -d

# 停止服務
docker compose down

# 停止服務並移除所有容器、網路
docker compose down -v

# 查看日誌
docker compose logs -f [service_name]

# 進入容器
docker compose exec [service_name] sh

# 清除資料庫資料
docker compose down -v
rm -rf db/postgres_data/*
```

## 開發說明

### 前端開發 (Next.js)

**技術細節**：
- Next.js 15 (App Router)
- React 19
- TypeScript 5
- Tailwind CSS 4
- SWR (資料獲取)
- React Hook Form + Zod (表單驗證)
- Framer Motion (動畫)

**本地開發**：
```bash
cd www_root/waterballsa-frontend
npm install
npm run dev        # 啟動開發伺服器 (http://localhost:3000)
npm run build      # 建置生產版本
npm run lint       # 執行 ESLint
npm run format     # 格式化程式碼
npm run test:e2e   # 執行 Playwright E2E 測試
```

**重要配置**：
- Hot reload 透過 Docker volume 掛載實現
- `node_modules` 和 `.next` 使用獨立 volume，避免跨平台相容性問題

### 後端開發 (Spring Boot)

**技術細節**：
- Spring Boot 3.5.7
- Java 17
- Spring Data JPA
- Spring Security + JWT
- Liquibase (資料庫遷移)
- PostgreSQL 16
- Bucket4j (限流)
- Caffeine (快取)

**本地開發**：
```bash
cd www_root/waterballsa-backend

# 編譯專案
./mvnw clean package

# 執行測試
./mvnw test

# 程式碼格式化（Google Java Format）
./mvnw spotless:apply

# 程式碼品質檢查（Checkstyle）
./mvnw checkstyle:check

# 執行 Liquibase 遷移
./mvnw liquibase:update
```

**資料庫遷移**：
- 使用 Liquibase 管理資料庫 schema
- 遷移檔案位於 `src/main/resources/db/changelog/`
- 容器啟動時自動執行遷移

### 資料庫

**連線資訊**：
```
Host: localhost
Port: 5432
Database: waterballsa
Username: admin
Password: change_me_in_production
```

**管理工具**：
```bash
# 使用 psql 連線
docker compose exec db psql -U admin -d waterballsa

# 匯出資料
docker compose exec db pg_dump -U admin waterballsa > backup.sql

# 匯入資料
docker compose exec -T db psql -U admin waterballsa < backup.sql
```

### Web Service (Nginx)

- 監聽 80 (HTTP) 和 443 (HTTPS)
- 反向代理到前端 (/) 和後端 (/api)
- 自動生成自簽 SSL 憑證（開發用）
- 日誌存放於 `web_service/log/`

## API 文檔

API 文檔使用 OpenAPI 3.0 規範，採用模組化結構管理。

### 查看 API 文檔

1. 合併模組化文件：
   ```bash
   make swagger
   ```

2. 生成的文件位於：`docs/api-docs/bundled-swagger.yaml`

3. 使用 Swagger UI 預覽：
   - 上傳到 https://editor.swagger.io/
   - 或使用本地 Swagger UI

詳細說明請參考 [docs/api-docs/openapi/README.md](docs/api-docs/openapi/README.md)

## 已實作功能

### Release 1（已完成）

#### 1. 權限模組
- ✅ 使用者註冊與登入
- ✅ JWT 身份驗證
- ✅ 登入狀態保持
- ✅ 使用者登出

#### 2. 看課模組
- ✅ 課程列表瀏覽
- ✅ 課程詳情頁面
- ✅ 影片播放（YouTube 整合）
- ✅ 觀看進度追蹤
- ✅ 單元完成判定
- ✅ 交付功能與經驗值系統

### Release 2（進行中）
- 詳見 [docs/Release-2-Spec.md](docs/Release-2-Spec.md)

## 安全性

- 密碼使用 BCrypt 雜湊
- JWT Token 身份驗證
- HTTPS 支援（自簽憑證，生產環境需更換）
- API 限流保護
- SQL Injection 防護（JPA Prepared Statements）
- XSS 防護（React 自動轉義）

## 測試

### 前端測試
```bash
cd www_root/waterballsa-frontend
npm run test:e2e          # 執行 Playwright E2E 測試
npm run test:e2e:ui       # UI 模式
npm run test:e2e:headed   # 顯示瀏覽器
```

### 後端測試
```bash
cd www_root/waterballsa-backend
./mvnw test               # 執行所有測試
./mvnw test -Dtest=UserControllerTest  # 執行特定測試
```

後端使用 Testcontainers 進行整合測試，會自動啟動 PostgreSQL 容器。

## 故障排除

### 常見問題

**1. 前端 Hot Reload 不工作**
- 確認 Docker Desktop 的檔案共享設定已啟用
- 重啟 Docker Compose

**2. 資料庫連線失敗**
```bash
# 檢查資料庫健康狀態
docker compose exec db pg_isready -U admin

# 查看資料庫日誌
docker compose logs db
```

**3. 後端啟動失敗**
```bash
# 查看後端日誌
docker compose logs backend

# 常見原因：資料庫未就緒，等待健康檢查通過後會自動重試
```

**4. SSL 憑證錯誤**
- 瀏覽器會顯示「不安全的連線」，這是正常的（使用自簽憑證）
- 點擊「進階」→「繼續前往」即可

**5. 埠號衝突**
- 修改 `.env` 中的 `FRONTEND_PORT`、`BACKEND_PORT` 或 `HOST_POSTGRES_PORT`

## 參考資料

- **原始平台**: https://world.waterballsa.tw
- **需求文檔**: [docs/Requirement.md](docs/Requirement.md)
- **開發策略**: [docs/Strategy.md](docs/Strategy.md)
- **技術債**: [docs/Tech-Debt.md](docs/Tech-Debt.md)
- **Next.js 文檔**: https://nextjs.org/docs
- **Spring Boot 文檔**: https://spring.io/projects/spring-boot
- **PostgreSQL 文檔**: https://www.postgresql.org/docs/

# Waterball Software Academy - 課程平台重製版

這是水球軟體學院課程平台的重製專案,使用 Next.js + Spring Boot + PostgreSQL 技術棧。

## 技術棧

- **前端**: Next.js (React)
- **後端**: Java Spring Boot
- **資料庫**: PostgreSQL
- **容器化**: Docker & Docker Compose

## 專案結構

```
waterballsa/
├── docker-compose.yml      # Docker Compose 配置
├── .env                    # 環境變數（本地開發用）
├── .env.example            # 環境變數範例
├── frontend/               # Next.js 前端專案
│   └── Dockerfile
├── backend/                # Spring Boot 後端專案
│   └── Dockerfile
├── db/                     # 資料庫資料目錄
│   └── postgres_data/
└── docs/                   # 文檔
```

## 快速開始

### 前置需求

- Docker
- Docker Compose

### 啟動服務

1. 複製環境變數範例檔案：
   ```bash
   cp .env.example .env
   ```

2. （可選）修改 `.env` 中的配置

3. 建立並啟動所有服務：
   ```bash
   docker-compose up --build
   ```

4. 訪問服務：
   - 前端：http://localhost:3000
   - 後端 API：http://localhost:8080
   - 資料庫：localhost:5432

### 停止服務

```bash
docker-compose down
```

### 清除所有資料（包含資料庫）

```bash
docker-compose down -v
rm -rf db/postgres_data/*
```

## 開發說明

### 前端開發（Next.js）

前端專案位於 `frontend/` 目錄，需要：
- Node.js 20+
- 配置 `next.config.js` 中的 `output: 'standalone'` 以支援 Docker 部署

### 後端開發（Spring Boot）

後端專案位於 `backend/` 目錄，需要：
- Java 21+
- Maven 3.9+
- 添加 Spring Boot Actuator 以支援健康檢查

### 資料庫

使用 PostgreSQL 16，資料持久化在 `db/postgres_data/` 目錄。

## 下一步

1. 在 `frontend/` 建立 Next.js 專案
2. 在 `backend/` 建立 Spring Boot 專案
3. 根據需求調整 Dockerfile
4. 開始實作功能

## 參考資料

- 原始課程平台：https://world.waterballsa.tw
- 需求文檔：docs/Requirement.md

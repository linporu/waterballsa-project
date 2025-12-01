# 基礎設施面試題 - Docker、Nginx 與容器編排

> **適用職位**：中階全端軟體工程師
> **難度分布**：Junior (1/3) + Mid-level (2/3)
> **專案背景**：水球軟體學院課程平台 - Next.js + Spring Boot + PostgreSQL + Nginx

---

## 第一部分：Docker Compose 與容器編排（10 題）

### 1. 服務依賴與啟動順序

觀察 `docker-compose.yml` 中的服務依賴配置：

```yaml
backend:
  depends_on:
    db:
      condition: service_healthy

frontend:
  depends_on:
    backend:
      condition: service_healthy

web_service:
  depends_on:
    ssl-init:
      condition: service_completed_successfully
    frontend:
      condition: service_healthy
    backend:
      condition: service_healthy
```

**問題**：為什麼 `backend` 需要等待 `db` 的 `service_healthy` 狀態，而不是使用預設的 `service_started`？這樣的設計解決了什麼問題？如果移除 `condition: service_healthy`，可能會發生什麼情況？

---

### 2. 健康檢查機制的設計

檢視 `docker-compose.yml:21-27` 的後端健康檢查配置：

```yaml
healthcheck:
  test: ['CMD-SHELL', 'curl -f http://localhost:8080/actuator/health || exit 1']
  interval: 15s
  timeout: 15s
  retries: 10
  start_period: 30s
```

**問題**：為什麼 `retries` 設定為 10 次，而 `start_period` 是 30 秒？這些參數是如何搭配運作的？在什麼情況下你會調整這些數值？

---

### 3. Volume 掛載策略

分析 `docker-compose.yml:41-47` 前端服務的 volume 配置：

```yaml
volumes:
  # Mount entire directory for hot reload - automatically includes all files
  - ./www_root/waterballsa-frontend:/app
  # Use named volumes to exclude node_modules and .next from host mount
  # This ensures container uses Linux-compiled dependencies and build cache
  - frontend_node_modules:/app/node_modules
  - frontend_next_cache:/app/.next
```

**問題**：為什麼需要使用 named volumes 來排除 `node_modules` 和 `.next` 目錄？如果不這樣做，在 macOS 或 Windows 上開發會遇到什麼問題？

---

### 4. 環境變數的注入方式

觀察後端服務的環境變數配置（`docker-compose.yml:9-14`）：

```yaml
environment:
  - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/${POSTGRES_DB}
  - SPRING_DATASOURCE_USERNAME=${POSTGRES_USER}
  - SPRING_DATASOURCE_PASSWORD=${POSTGRES_PASSWORD}
  - SPRING_JPA_HIBERNATE_DDL_AUTO=update
  - SERVER_PORT=8080
```

**問題**：這裡混用了 `${POSTGRES_DB}` 這種來自 `.env` 檔案的變數，以及直接寫死的 `SERVER_PORT=8080`。這兩種方式有什麼差異？在什麼情境下應該使用哪一種？

---

### 5. 資料庫持久化

檢視資料庫服務的 volume 配置（`docker-compose.yml:105-106`）：

```yaml
volumes:
  - ./db/postgres_data:/var/lib/postgresql/data
```

**問題**：這個 volume 使用了 bind mount（本機路徑映射）而不是 named volume。這樣做有什麼優缺點？在什麼情況下你會選擇改用 named volume？

---

### 6. SSL 憑證初始化服務

分析 `docker-compose.yml:55-70` 的 `ssl-init` 服務：

```yaml
ssl-init:
  image: alpine:latest
  volumes:
    - ./web_service/etc/nginx/ssl:/ssl
  command:
    - sh
    - -c
    - |
      apk add --no-cache openssl
      if [ ! -f /ssl/localhost.crt ]; then
        echo 'Generating self-signed SSL certificate...'
        openssl req -x509 -out /ssl/localhost.crt -keyout /ssl/localhost.key ...
      else
        echo 'SSL certificate already exists, skipping generation.'
      fi
```

**問題**：為什麼要設計一個獨立的 `ssl-init` 服務，而不是直接在 Nginx 容器啟動時生成憑證？這種設計有什麼好處？

---

### 7. 網路隔離與服務通訊

在專案中，所有服務都在同一個預設網路中互相通訊（例如 `backend:8080`、`frontend:3000`）。

**問題**：如果你想提升安全性，將資料庫隔離到一個獨立的網路中（只有 backend 可以存取），你會如何修改 `docker-compose.yml`？請描述需要新增的配置。

---

### 8. Port Mapping 的設計考量

觀察各服務的 port mapping：

- `backend: "8080:8080"`
- `frontend: "3000:3000"`
- `web_service: "80:80" 和 "443:443"`
- `db: "5432:5432"`

**問題**：為什麼 `web_service` 的 port mapping 沒有使用環境變數（如 `${WEB_PORT:-80}:80`），而 backend 和 frontend 都有？這樣的設計是否合理？

---

### 9. 容器重啟策略

注意到資料庫服務有 `restart: always`（`docker-compose.yml:98`），但其他服務沒有設定 restart policy。

**問題**：為什麼只有資料庫需要 `restart: always`？在什麼情況下你會為前後端服務也加上 restart policy？不同的 restart policy（`no`、`always`、`on-failure`、`unless-stopped`）分別適用於什麼場景？

---

### 10. 多階段建置 vs 單階段建置

比較前端 Dockerfile（`next/docker_file/Dockerfile`）和後端 Dockerfile（`java/docker_file/Dockerfile`）：

**前端**：

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```

**後端**：

```dockerfile
FROM maven:3.9-eclipse-temurin-17
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
ENTRYPOINT ["mvn", "spring-boot:run"]
```

**問題**：這兩個 Dockerfile 都是用於開發環境。如果要改為生產環境的多階段建置（multi-stage build），你會如何設計？為什麼生產環境需要多階段建置？

---

## 第二部分：Nginx 配置與反向代理（10 題）

### 11. Worker Processes 配置

檢視 `web_service/etc/nginx/nginx.conf:2`：

```nginx
worker_processes  auto;
```

**問題**：`worker_processes auto;` 的 `auto` 是根據什麼來決定 worker 數量的？在一個 4 核心的伺服器上，這個設定會產生幾個 worker？如果改成固定數字（例如 `worker_processes 2;`），什麼情況下會比 `auto` 更好？

---

### 12. Worker Connections 的計算

觀察 `nginx.conf:8-10`：

```nginx
events {
    worker_connections  1024;
}
```

**問題**：假設你的伺服器有 4 個 worker processes，每個 worker 有 1024 個 connections，理論上最大併發連線數是多少？但實際上為什麼無法達到這個數字？在作為反向代理時，connection 的計算方式有什麼不同？

---

### 13. 反向代理路徑重寫

分析 `web_service/etc/nginx/conf.d/www.conf:32-38`：

```nginx
location /api/ {
    proxy_pass http://backend:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**問題**：注意 `proxy_pass` 的 URL 結尾有一個斜線（`http://backend:8080/`）。這個斜線的存在與否會如何影響路徑轉發？舉例說明：當使用者請求 `https://localhost/api/users/123` 時，後端實際收到的路徑是什麼？

---

### 14. Proxy Headers 的作用

在上述的 API 代理配置中，設定了多個 `proxy_set_header`。

**問題**：`X-Forwarded-For` 和 `X-Real-IP` 看起來都是傳遞客戶端 IP，它們有什麼差別？`$proxy_add_x_forwarded_for` 和 `$remote_addr` 的值在什麼情況下會不同？

---

### 15. WebSocket 支援

檢視前端代理配置（`www.conf:41-48`）：

```nginx
location / {
    proxy_pass http://frontend:3000;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
    ...
}
```

**問題**：`Upgrade` 和 `Connection "Upgrade"` 這兩個 header 是用來支援什麼協定的？為什麼前端代理需要這些 header，但 API 代理不需要？如果沒有這些設定，Next.js 的哪些功能會失效？

---

### 16. SSL/TLS 協定版本

觀察 SSL 配置（`www.conf:26`）：

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

**問題**：為什麼不支援 TLSv1.0 和 TLSv1.1？這樣的設定會影響哪些舊版瀏覽器或客戶端？如果你的目標使用者中有人使用 Windows 7 + IE11，你需要如何調整？

---

### 17. SSL Cipher Suite 選擇

檢視 cipher 配置（`www.conf:27-28`）：

```nginx
ssl_ciphers TLS-CHACHA20-POLY1305-SHA256:TLS-AES-256-GCM-SHA384:...;
ssl_prefer_server_ciphers on;
```

**問題**：`ssl_prefer_server_ciphers on;` 的作用是什麼？為什麼要讓伺服器決定使用哪個 cipher，而不是讓客戶端決定？這樣做的安全性考量是什麼？

---

### 18. HSTS Header

分析 `www.conf:20`：

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

**問題**：HSTS 的 `max-age=31536000` 代表什麼意思？`includeSubDomains` 和 `preload` 分別有什麼作用？如果這是一個測試環境，設定 `preload` 可能會造成什麼問題？

---

### 19. Client Max Body Size

檢視 `www.conf:13`：

```nginx
client_max_body_size 16M;
```

**問題**：這個設定限制了什麼？如果使用者上傳一個 20MB 的檔案會發生什麼事？在設計課程平台時，如何決定這個數值？如果後端 Spring Boot 也有檔案大小限制（例如 `spring.servlet.multipart.max-file-size`），兩者的關係是什麼？

---

### 20. 日誌管理

觀察日誌配置：

**nginx.conf:4**：

```nginx
error_log  /var/log/nginx/error.log notice;
```

**www.conf:10-11**：

```nginx
access_log /var/log/nginx/www-access.log;
error_log /var/log/nginx/www-error.log debug;
```

**問題**：為什麼全域的 error_log 是 `notice` 層級，但 virtual host 的是 `debug` 層級？在生產環境中，使用 `debug` 層級會有什麼風險？你會如何設計一個「可以動態調整日誌層級」的機制？

---

## 第三部分：開發流程與除錯（5 題）

### 21. Hot Reload 機制

前端容器使用 volume mount 實現 hot reload（`docker-compose.yml:43`）：

```yaml
- ./www_root/waterballsa-frontend:/app
```

後端容器使用 Spring Boot DevTools（`java/docker_file/Dockerfile:19`）：

```dockerfile
ENTRYPOINT ["mvn", "spring-boot:run"]
```

**問題**：這兩種 hot reload 的實現方式有什麼不同？為什麼前端可以直接透過 volume mount 達成，而後端需要依賴 Spring Boot DevTools？如果你修改了 `package.json` 或 `pom.xml`，分別需要做什麼操作才能讓改動生效？

---

### 22. 容器內除錯

假設你的後端服務啟動失敗，`docker compose logs backend` 顯示一些 Java 異常。

**問題**：請描述至少三種方式可以進入容器內部進行更深入的除錯。如果容器已經因為錯誤而停止（無法用 `docker compose exec` 進入），你會怎麼辦？

---

### 23. 依賴快取策略

觀察後端 Dockerfile 的分層（`java/docker_file/Dockerfile:9-10`）：

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline -B
```

**問題**：為什麼要先單獨複製 `pom.xml` 並下載依賴，而不是直接 `COPY . .` 然後一起處理？這樣做如何提升建置速度？在什麼情況下這個快取會失效？

---

### 24. .dockerignore 的作用

檢視 `www_root/waterballsa-frontend/.dockerignore` 中排除了 `node_modules/` 和 `.next/`。

**問題**：這些目錄明明會在 Dockerfile 中被重新生成（`RUN npm install`），為什麼還要在 `.dockerignore` 中排除？如果不排除會發生什麼事？對建置時間和映像檔大小有什麼影響?

---

### 25. 日誌聚合與查看

在這個多容器環境中，你需要同時查看前後端和 Nginx 的日誌。

**問題**：描述至少三種方式可以有效地查看和管理多個容器的日誌。如果你想要將所有日誌集中到一個檔案中以便搜尋，你會如何實現？在生產環境中，你會推薦什麼樣的日誌管理方案（考慮到這是單伺服器部署）？

---

## 第四部分：生產環境與維運（5 題）

### 26. 生產環境的 SSL 憑證

目前使用自簽憑證（`ssl-init` 服務生成）。

**問題**：如果要將這個應用部署到生產環境（使用真實網域，例如 `courses.waterballsa.tw`），你會如何取得和管理 SSL 憑證？請說明使用 Let's Encrypt 的完整流程，包括憑證自動更新機制。你會如何修改 `docker-compose.yml` 來支援這個需求？

---

### 27. 密碼與 Secret 管理

檢視 `.env.example` 中的敏感資訊：

```env
POSTGRES_PASSWORD=change_me_in_production
JWT_SECRET=your_jwt_secret_key_change_in_production
```

**問題**：在生產環境中，直接使用 `.env` 檔案存放密碼有什麼風險？如果這個專案要部署到 AWS EC2 單一伺服器上，你會如何管理這些 secrets？請提出至少兩種改進方案。

---

### 28. 效能瓶頸排查

假設你的課程平台上線後，在高峰時段（500 併發使用者）出現回應緩慢的問題。

**問題**：請描述你會如何系統性地排查效能瓶頸。從 Nginx、前端容器、後端容器到資料庫，每一層你會檢查哪些指標？你會使用什麼工具？如何判斷瓶頸在哪一層？

---

### 29. 零停機部署

目前的部署方式是 `docker compose down` 然後 `docker compose up`，這會造成服務中斷。

**問題**：在單伺服器環境下，如何實現零停機部署（或至少最小化停機時間）？請說明你的方案，包括需要對 `docker-compose.yml` 做哪些修改，以及部署流程的步驟。

---

### 30. 備份與災難恢復

考慮這個專案的資料持久化策略：

- 資料庫資料：`./db/postgres_data`
- SSL 憑證：`./web_service/etc/nginx/ssl`
- 上傳的檔案（如果有）：目前沒有特別處理

**問題**：請設計一個完整的備份與災難恢復計畫。包括：(1) 需要備份哪些內容？(2) 備份頻率和保留策略？(3) 如何驗證備份可用性？(4) 當伺服器硬碟損壞時，如何快速恢復服務？請描述具體的腳本或工具選擇。

---

## 面試評分參考

### 優秀的回答應該包含：

1. **語法理解**：準確解釋配置語法的作用
2. **設計理由**：說明為什麼這樣設計，解決了什麼問題
3. **權衡考量**：不同方案的優缺點比較
4. **實務經驗**：能舉出實際遇到的問題和解決方式
5. **系統思維**：從整體架構角度思考，而非單點優化
6. **生產意識**：考慮安全性、效能、可維護性、可監控性

### 加分項：

- 能指出現有配置的潛在問題
- 提出具體的改進建議
- 了解相關的最佳實踐和業界標準
- 能快速定位問題並提出驗證方法

---

**祝面試順利！**

# API 設計面試題 - WaterballSA 課程平台

## 說明

本面試題目基於 WaterballSA 課程平台的 API 設計（參考 `docs/api-docs/swagger.yaml`），涵蓋權限模組、看課模組、進度追蹤、購課模組等核心功能。

題目難度分為 Junior 和 Mid-level 兩個層級（10 題 + 20 題），著重於：

- RESTful API 設計原則與最佳實踐
- HTTP 方法與狀態碼的正確使用
- 認證與授權機制設計
- API 安全性考量
- 業務邏輯在 API 層的實作
- 單一伺服器架構下的效能優化

**架構說明**：本專案採用單一伺服器架構（Next.js + Spring Boot + PostgreSQL），題目聚焦於此架構下的實務問題，不涉及分散式系統、微服務拆分、資料庫分片等複雜系統設計。

所有問題皆為開放式問答，請根據專案實際需求進行思考與回答。

---

## Junior Level (基礎題 1-10)

### 1. [Junior] RESTful 資源設計

在這個專案中，有以下兩個 API 端點：

- `GET /journeys` - 取得課程列表
- `GET /journeys/{journeyId}` - 取得課程詳情

請說明這種 URL 設計符合 RESTful 的哪些原則？相較於 `GET /getJourneys` 或 `GET /journey/detail?id=1` 這類設計，有什麼優勢？

---

### 2. [Junior] HTTP 方法選擇

專案中有以下三個 API 端點來處理使用者任務進度：

- `GET /users/{userId}/missions/{missionId}/progress` - 取得進度
- `PUT /users/{userId}/missions/{missionId}/progress` - 更新進度
- `POST /users/{userId}/missions/{missionId}/progress/deliver` - 交付任務

請說明為什麼「更新進度」使用 PUT 方法，而「交付任務」使用 POST 方法？這兩個操作的本質差異是什麼？

---

### 3. [Junior] HTTP 狀態碼理解

觀察以下三個 API 端點的成功回應狀態碼：

- `POST /auth/register` 回傳 `201 Created`
- `POST /auth/login` 回傳 `200 OK`
- `POST /orders` 建立新訂單時回傳 `201 Created`，但如果使用者已有該課程的未付款訂單時回傳 `200 OK`

請說明：

1. 為什麼註冊成功用 201 而登入成功用 200？
2. 為什麼建立訂單 API 在某些情況下回傳 200 而不是 201？

---

### 4. [Junior] 認證機制基礎

這個專案使用 JWT Token 進行身份驗證，Token 的有效期限為 1 天。請說明：

1. 什麼是 JWT？它與傳統 Session-Cookie 機制有什麼不同？
2. 為什麼前端需要在每次請求時將 Token 放在 `Authorization: Bearer <token>` Header 中？

---

### 5. [Junior] 路徑參數 vs 查詢參數

比較以下兩個 API 端點的參數設計：

- `GET /journeys/{journeyId}` - 使用路徑參數
- `GET /users/{userId}/orders?page=1&limit=20` - 混合使用路徑參數和查詢參數

請說明什麼情況下應該使用路徑參數（Path Parameter），什麼情況下應該使用查詢參數（Query Parameter）？

---

### 6. [Junior] 必填與選填欄位

觀察 `POST /orders` 建立訂單 API 的請求體：

```json
{
	"items": [
		{
			"journeyId": 17,
			"quantity": 1
		}
	]
}
```

其中 `items` 和 `journeyId` 是必填欄位，而 `quantity` 有預設值 1（選填）。請說明：

1. 在 API 設計中，如何決定哪些欄位應該必填、哪些應該選填？
2. 選填欄位使用預設值有什麼好處？

---

### 7. [Junior] 錯誤回應結構

專案中所有錯誤回應都使用統一的結構：

```json
{
	"error": "錯誤訊息"
}
```

請說明：

1. 為什麼需要統一的錯誤回應格式？
2. 除了 `error` 欄位，還可以加入哪些資訊來幫助前端處理錯誤？（例如錯誤代碼、欄位驗證錯誤等）

---

### 8. [Junior] 認證 vs 授權

觀察以下兩個錯誤狀態碼：

- `401 Unauthorized` - 未登入或 Token 無效
- `403 Forbidden` - 已登入但無權限執行此操作

`GET /users/{userId}/missions/{missionId}/progress` API 在以下情境會回傳什麼狀態碼？

1. 使用者未提供 JWT Token
2. 使用者已登入，但嘗試查看其他使用者（userId 不是自己）的進度

請說明 401 和 403 的使用時機差異。

---

### 9. [Junior] 巢狀資源路徑

專案中任務詳情 API 使用了巢狀路徑：
`GET /journeys/{journeyId}/missions/{missionId}`

請說明：

1. 這種巢狀路徑設計表達了什麼樣的資源關係？
2. 相較於 `GET /missions/{missionId}`（不包含 journeyId），巢狀路徑有什麼優缺點？

---

### 10. [Junior] 時間戳格式

觀察 API 回應中的時間欄位：

```json
{
	"createdAt": 1732420800000,
	"paidAt": 1732507200000
}
```

這些時間欄位使用 Unix Timestamp（毫秒數）格式。請說明：

1. 為什麼 API 通常使用 Unix Timestamp 而不是 ISO 8601 格式（如 `"2023-11-24T10:00:00Z"`）？
2. 前端收到這種格式的時間資料後，通常如何處理？

---

## Mid-level (進階題 11-30)

### 11. [Mid-level] 速率限制的錯誤訊息設計

觀察 `POST /auth/login` API 的速率限制設計：

- 每個 IP 每 15 分鐘最多 5 次失敗嘗試
- 超過速率限制時，**不回傳「速率限制」相關訊息**，而是回傳與「密碼錯誤」相同的錯誤訊息：`401 Unauthorized` + `"使用者名稱或密碼無效"`

請分析：

1. 為什麼要隱藏速率限制的真實原因，使用與密碼錯誤相同的錯誤訊息？
2. 這種設計在安全性上有什麼好處？
3. 這種設計對使用者體驗有什麼影響？如何平衡安全性與使用者體驗？

---

### 12. [Mid-level] Token 黑名單機制

`POST /auth/logout` API 的文件說明：

> Token 會被加入黑名單（儲存在資料庫的 `access_tokens` 表中），黑名單項目會在 Token 過期時自動清理。

請思考：

1. 為什麼需要 Token 黑名單機制？JWT 本身不是無狀態（stateless）的設計嗎？
2. 如果不實作黑名單，使用者登出後會有什麼安全性問題？
3. Token 黑名單表需要建立哪些索引來優化「檢查 Token 是否在黑名單」的查詢效能？

---

### 13. [Mid-level] 登出 API 的必要性

有些開發者認為：「JWT 是無狀態的，前端直接刪除 Token 就好，不需要呼叫後端的登出 API」。

但這個專案仍然實作了 `POST /auth/logout` API。請分析：

1. 前端登出 vs 後端登出有什麼差異？
2. 在什麼情境下，單純前端刪除 Token 是不夠安全的？
3. 如果使用者在多個裝置登入，後端登出 API 應該如何設計？（登出單一裝置 vs 登出所有裝置）

---

### 14. [Mid-level] JWT 過期時間的權衡

這個專案的 JWT Token 有效期設定為 **1 天**。請分析：

1. Token 有效期設定過長（如 30 天）和過短（如 15 分鐘）各有什麼問題？
2. 如果要在「安全性」和「使用者體驗」之間取得平衡，你會建議什麼樣的有效期限？
3. 實務上，「Access Token + Refresh Token」雙 Token 機制如何解決這個兩難問題？（本專案的延後執行規格中有提到）

---

### 15. [Mid-level] 訂單編號生成規則

`POST /orders` API 文件說明訂單編號格式：

> `{timestamp}{userId}{randomCode}`
> 範例：`20251121011117cd5` = `2025112101`（時間戳記）+ `11`（使用者 ID）+ `17cd5`（隨機碼）

請分析這種設計：

1. 為什麼不使用單純的流水號（1, 2, 3...）或 UUID？
2. 將 userId 包含在訂單編號中有什麼好處和風險？
3. 這種格式能否保證訂單編號的唯一性？如果不能，應該如何改進？

---

### 16. [Mid-level] 價格快照機制

觀察訂單資料結構設計：

- `orders` 表中的 `order_items` 包含 `originalPrice`、`discount`、`price`
- 這些價格是在訂單建立時從 `journeys.price` **快照（snapshot）** 過來的
- 即使課程之後調整價格，訂單中記錄的價格也不會改變

請分析：

1. 為什麼需要價格快照機制？直接關聯到 `journeys.price` 有什麼問題？
2. 這種設計屬於資料庫的「正規化」還是「反正規化」？
3. 如果未來需要支援「訂單建立後課程降價，使用者要求退差價」功能，這個設計是否足夠？需要增加什麼欄位？

---

### 17. [Mid-level] 訂單狀態管理

訂單有三種狀態：

- `UNPAID` - 訂單已建立但尚未付款
- `PAID` - 付款完成
- `EXPIRED` - 三天後未付款自動過期

請設計：

1. 訂單狀態轉換的有限狀態機（Finite State Machine），哪些狀態之間可以轉換？
2. `POST /orders/{orderId}/action/pay` 付款 API 應該檢查哪些前置條件（Preconditions）？
3. 如果使用者嘗試支付已過期的訂單，API 應該回傳什麼狀態碼和錯誤訊息？

---

### 18. [Mid-level] 重複購買檢查

`POST /orders` 建立訂單 API 的業務邏輯：

- 如果使用者**已購買**該課程 → 回傳 `409 Conflict` + `"你已經購買此課程"`
- 如果使用者**已有未付款訂單** → 回傳 `200 OK` + 現有訂單資料

請分析：

1. 為什麼已購買回傳 409，而已有未付款訂單回傳 200？這兩種情況的本質差異是什麼？
2. 「回傳現有訂單」是一種冪等性設計嗎？
3. 如果未來要支援「重複購買送給別人」功能，API 設計需要如何調整？

---

### 19. [Mid-level] PUT 方法的 Upsert 語意

`PUT /users/{userId}/missions/{missionId}/progress` API 文件說明：

> **Upsert 操作**：如果進度記錄不存在，建立新記錄；如果存在，更新記錄。

請分析：

1. 為什麼「更新進度」適合使用 PUT 搭配 Upsert 語意，而不是 POST（建立）+ PATCH（更新）分開設計？
2. Upsert 操作的冪等性（Idempotency）是什麼？多次呼叫相同的 PUT 請求會有什麼結果？
3. 在資料庫層級，如何實作 Upsert？（提示：PostgreSQL 的 `INSERT ... ON CONFLICT`）

---

### 20. [Mid-level] 進度更新頻率設計

專案規格要求：「前端在影片播放期間每 10 秒呼叫一次進度更新 API」。

請分析：

1. 為什麼選擇 10 秒而不是即時更新（每秒）或更長間隔（每分鐘）？
2. 如果 1000 名學生同時觀看影片，每 10 秒更新一次會對資料庫造成什麼壓力？
3. 如何優化這個設計來減輕後端負擔？（提示：考慮批次更新、非同步處理、快取等策略）

---

### 21. [Mid-level] 交付與完成的分離設計

專案中「完成任務」和「交付任務」是兩個分開的操作：

1. **完成（COMPLETED）**：影片觀看進度達到 100%，狀態自動變為 COMPLETED
2. **交付（DELIVERED）**：使用者主動呼叫 `POST .../progress/deliver` API，獲得經驗值

請分析：

1. 為什麼要將「完成」和「交付」分開設計？為什麼不是影片看完就自動給經驗值？
2. 這種設計對使用者體驗和遊戲化機制有什麼影響？
3. 如果要防止「使用者快轉影片快速刷經驗值」，應該如何改進這個設計？

---

### 22. [Mid-level] 防止重複交付

`POST /users/{userId}/missions/{missionId}/progress/deliver` API 的邏輯：

- 如果任務已經是 `DELIVERED` 狀態，回傳 `409 Conflict` + `"任務已經交付過了"`

請分析：

1. 為什麼需要防止重複交付？如果沒有這個檢查會發生什麼問題？
2. 這個檢查應該在應用層實作還是資料庫層實作？（提示：考慮 unique constraint）
3. 在高併發情境下，如果使用者快速點擊兩次「交付」按鈕，如何確保經驗值不會重複發放？（提示：考慮交易隔離層級、樂觀鎖等）

---

### 23. [Mid-level] 路徑參數中的 userId 安全性

許多 API 的路徑中包含 `userId`，例如：

- `PUT /users/{userId}/missions/{missionId}/progress`
- `GET /users/{userId}/orders`

但後端會檢查「JWT Token 中的 userId」是否與「路徑參數中的 userId」一致，不一致則回傳 403 或 404。

請分析：

1. 既然後端會檢查權限，為什麼路徑中還要包含 userId？直接設計成 `/users/me/orders` 不是更安全嗎？
2. 從 RESTful 資源設計的角度，`/users/{userId}/orders` 和 `/users/me/orders` 哪個更符合 REST 原則？
3. 如果未來要支援「管理員查看所有使用者的訂單」功能，這兩種設計哪個更容易擴展？

---

### 24. [Mid-level] 404 vs 403 的安全性權衡

觀察以下兩個 API 的權限錯誤處理：

- `GET /users/{userId}/missions/{missionId}/progress` - 如果 userId 不是自己，回傳 `403 Forbidden`
- `GET /orders/{orderId}` - 如果 orderId 不屬於自己，回傳 `404 Not Found`

請分析：

1. 為什麼訂單 API 使用 404 而不是 403？這種設計有什麼安全性考量？
2. 這種「用 404 隱藏權限錯誤」的做法（Security by Obscurity）有什麼優缺點？
3. 在什麼情況下應該使用 403，什麼情況下應該使用 404 來隱藏資源存在性？

---

### 25. [Mid-level] Optional JWT 設計

`GET /journeys/{journeyId}` 課程詳情 API 的認證設計：

- **不需要登入**也可以查看課程基本資訊
- **如果有提供 JWT**，回應會額外包含使用者相關資訊：
  - `userStatus.hasPurchased` - 是否已購買
  - `userStatus.hasUnpaidOrder` - 是否有未付款訂單
  - `missions[].status` - 任務完成狀態

請分析：

1. 為什麼要設計成「可選認證」而不是強制登入？
2. 在實作上，後端如何判斷「JWT 存在但無效」和「使用者未提供 JWT」？錯誤回應應該有什麼差異？
3. 前端如何根據回應內容動態顯示「繼續學習」或「立即購買」按鈕？

---

### 26. [Mid-level] 使用者只能查看自己資料的實作策略

專案中許多 API 都有「使用者只能查看自己的資料」限制，例如：

- 查看進度、查看訂單、查看已購買課程

請設計：

1. 在 Spring Boot 中，應該在哪一層進行權限檢查？（Controller、Service、還是使用 Spring Security 的 @PreAuthorize？）
2. 如何避免在每個 API 都重複寫權限檢查邏輯？（提示：考慮 Interceptor、AOP 等機制）
3. 如果未來要支援「家長查看子女的學習進度」功能，現有的權限檢查邏輯如何擴展？

---

### 27. [Mid-level] 冪等性設計分析

請分析專案中以下 API 的冪等性：

1. `POST /auth/login` - 多次登入
2. `POST /orders` - 多次建立相同課程的訂單（回傳現有訂單）
3. `PUT /users/{userId}/missions/{missionId}/progress` - 多次更新進度
4. `POST /users/{userId}/missions/{missionId}/progress/deliver` - 多次交付（會回傳 409）

請說明：

1. 哪些 API 是冪等的？哪些不是？
2. 為什麼 PUT 通常是冪等的，而 POST 通常不是？
3. 如何設計 POST API 來實現冪等性？（提示：考慮 Idempotency Key）

---

### 28. [Mid-level] 分頁參數設計

`GET /users/{userId}/orders` API 支援分頁：

- `page` - 頁碼（預設 1，從 1 開始計數）
- `limit` - 每頁筆數（預設 20，最大 100）

回應包含分頁資訊：

```json
{
  "orders": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 5
  }
}
```

請分析：

1. 為什麼選擇 `page/limit` 而不是 `offset/limit`？各有什麼優缺點？
2. 分頁回應中，除了 `page`, `limit`, `total`，還可以提供哪些資訊方便前端實作分頁 UI？
3. 當資料量很大時（如百萬筆訂單），傳統的 offset/limit 分頁會有效能問題，應該改用什麼方式？（提示：Cursor-based Pagination）

---

### 29. [Mid-level] Nullable 欄位設計原則

觀察訂單回應中的時間欄位設計：

```json
{
	"createdAt": 1732420800000,
	"expiredAt": 1732680000000,
	"paidAt": null
}
```

- `createdAt` - 永遠有值（訂單建立時間）
- `expiredAt` - 未付款訂單有值，已付款訂單為 `null`
- `paidAt` - 未付款訂單為 `null`，已付款訂單有值

請分析：

1. 為什麼 `paidAt` 和 `expiredAt` 設計成 nullable？為什麼不用預設值（如 0 或 -1）表示「尚未發生」？
2. 前端如何根據這些欄位的 null 狀態來判斷訂單狀態和顯示邏輯？
3. API 文件中應該如何清楚標註欄位的 nullable 特性？

---

### 30. [Mid-level] API 演進與向後相容

假設專案上線後，訂單功能需要新增「優惠券」欄位。請設計 API 演進策略：

**情境**：原本的訂單回應：

```json
{
	"id": 123,
	"price": 7599.0,
	"discount": 0.0
}
```

現在要加入優惠券支援：

```json
{
	"id": 123,
	"price": 7599.0,
	"discount": 0.0,
	"couponCode": "WELCOME2024",
	"couponDiscount": 500.0
}
```

請分析：

1. 這種「新增欄位」的修改是否符合向後相容（Backward Compatibility）原則？為什麼？
2. 如果是「修改欄位結構」（如將 `price` 從數字改成物件 `{"amount": 7599, "currency": "TWD"}`），應該如何處理以避免破壞現有客戶端？
3. 在單一版本 API 維護的策略下（本專案沒有 /v1, /v2 版本劃分），應該如何確保 API 演進不影響現有使用者？

---

## 結語

以上 30 題涵蓋了 API 設計的基礎概念到實務決策，建議在回答時：

1. **結合業務場景**：思考 WaterballSA 平台的實際使用情境
2. **權衡取捨**：API 設計沒有完美方案，重要的是理解每個選擇的優缺點
3. **安全優先**：考慮各種潛在的安全性風險
4. **使用者體驗**：好的 API 設計讓前端開發更輕鬆，使用者體驗更好
5. **可維護性**：清晰的 API 設計和文件降低維護成本

祝你面試順利！🚀

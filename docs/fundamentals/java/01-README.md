# Java & Spring Boot 基礎知識面試準備

> 適合從 Python/Laravel/Go 轉換的中階開發者快速參考指南
>
> 目標:自信回答 50 個後端面試問題

## 📚 概述

本目錄包含 **7 個專注的指南**,涵蓋後端開發面試所需的核心 Java 和 Spring Boot 概念。每個指南都使用**本專案的實際程式碼**,並附有檔案參考和行號。

**完成時間:** 3-4 小時(快速參考節奏)

## 🎯 適用對象

你是一位中階開發者,具備以下條件:

- ✅ 熟悉 Python/Laravel/Go 後端框架
- ✅ 理解後端單體架構概念
- ✅ 需要快速學習 Java/Spring Boot 語法
- ✅ 正在準備技術面試

你不需要:

- ❌ 深入的 Java 語言理論
- ❌ 微服務/分散式系統知識
- ❌ 多年的 Java 經驗

## 🗺️ 學習路徑

### 建議順序

按照以下順序以獲得最佳學習效果:

```
1. Java 現代語法(30 分鐘)              ← 從這裡開始 - 基礎
   ↓
2. Spring Boot 註解(20 分鐘)           ← 基於 Java 知識
   ↓
3. JPA & Hibernate(40 分鐘)            ← ORM 基礎
   ↓
4. Spring Data JPA(25 分鐘)            ← Repository 層
   ↓
5. 交易管理(25 分鐘)                   ← 業務邏輯模式
   ↓
6. Spring Security & JWT(35 分鐘)      ← 身份驗證/授權
   ↓
7. 錯誤處理(20 分鐘)                   ← 橫切關注點
```

**總計: ~3.5 小時**

### 替代路徑

**面試準備(快速通道 - 2 小時):**

1. Java 現代語法(專注於 sealed interfaces、records、Optional)
2. Spring Boot 註解(專注於 @RestController、DI、@Valid)
3. JPA & Hibernate(專注於 entity 註解、關係)
4. 錯誤處理(專注於 @RestControllerAdvice)

**除錯現有程式碼:**

1. Spring Boot 註解
2. JPA & Hibernate
3. 交易管理

## 📖 指南

### [02. Java 現代語法](02-java-modern-syntax.md)

**主題:** Sealed interfaces、records、pattern matching、Stream API、Optional

**何時閱讀:** 你需要理解本專案使用的現代 Java 特性(Java 16+)

**核心概念:**

- `sealed interface` 用於類型安全的結果類型
- `record` 類別用於不可變的 DTO
- 使用 `instanceof` 的 Pattern matching
- 用於集合處理的 Stream API
- `Optional` 用於 null 安全

**相關面試問題:** #1, #2, #3, #5

---

### [03. Spring Boot 註解](03-spring-boot-annotations.md)

**主題:** Controllers、依賴注入、驗證、日誌

**何時閱讀:** 你需要理解 Spring Boot 如何組裝元件

**核心概念:**

- `@RestController` 和請求映射註解
- 建構函式注入(推薦的 DI 模式)
- `@Valid` 用於請求驗證
- 使用 SLF4J 的 Logger 設置
- Bean 驗證註解

**相關面試問題:** #6, #7, #9, #10

---

### [04. JPA & Hibernate](04-jpa-hibernate.md)

**主題:** Entities、生命週期、關係、enum 映射

**何時閱讀:** 你需要理解 Java 物件如何映射到資料庫表

**核心概念:**

- Entity 註解(`@Entity`、`@Table`、`@Column`)
- 生命週期回調(`@PrePersist`、`@PreUpdate`)
- 關係映射(`@OneToMany`、`@ManyToOne`)
- `FetchType.LAZY` vs `EAGER`
- 使用 `@Enumerated` 和 `@JdbcTypeCode` 的 Enum 映射

**相關面試問題:** #4, #11, #12, #15, #26, #27

---

### [05. Spring Data JPA](05-spring-data-jpa.md)

**主題:** Repositories、查詢方法、JPQL、分頁

**何時閱讀:** 你需要理解如何在 Spring 中查詢資料庫

**核心概念:**

- `JpaRepository` 介面
- 方法命名慣例(`findBy`、`existsBy`)
- 使用 JPQL 的自訂 `@Query`
- 使用 `Pageable` 和 `Page<T>` 的分頁
- 使用 `@Lock` 的悲觀鎖定

**相關面試問題:** #13, #14, #28

---

### [06. 交易管理](06-transaction-management.md)

**主題:** @Transactional、傳播、排程任務

**何時閱讀:** 你需要理解交易邊界和一致性

**核心概念:**

- `@Transactional` 註解使用
- 交易傳播層級
- 例外狀況的回滾行為
- 用於 cron 工作的 `@Scheduled`
- Upsert 模式

**相關面試問題:** #16, #17, #18, #19

---

### [07. Spring Security & JWT](07-spring-security-jwt.md)

**主題:** SecurityContext、JWT、filters、CORS

**何時閱讀:** 你需要理解身份驗證和授權

**核心概念:**

- `SecurityContextHolder` 用於當前使用者
- JWT token 建立和驗證
- `OncePerRequestFilter` 用於身份驗證
- CORS 配置
- Token 黑名單模式

**相關面試問題:** #8, #21, #22, #23, #24, #25

---

### [08. 錯誤處理](08-error-handling.md)

**主題:** Exception handlers、自訂例外、ResponseEntity

**何時閱讀:** 你需要理解如何全域處理錯誤

**核心概念:**

- `@RestControllerAdvice` 用於全域例外處理
- `@ExceptionHandler` 用於特定例外
- 自訂例外類別
- `ResponseEntity<T>` 用於錯誤回應
- HTTP 狀態碼選擇

**相關面試問題:** #31, #32

---

## 🔄 框架比較快速參考

如果你來自其他框架,這裡是快速的轉換指南:

| 概念              | Laravel                          | Python/Django                 | Go                   | Spring Boot                |
| ----------------- | -------------------------------- | ----------------------------- | -------------------- | -------------------------- |
| **路由**          | `Route::get()` 在 routes/web.php | `urlpatterns` 在 urls.py      | `http.HandleFunc()`  | `@GetMapping` 註解         |
| **依賴注入**      | 建構函式注入                     | Manual 或 dependency_injector | Manual 或 wire       | `@Autowired` 或建構函式    |
| **ORM**           | Eloquent models                  | Django ORM / SQLAlchemy       | GORM                 | JPA/Hibernate entities     |
| **驗證**          | Form Requests                    | Forms / Serializers           | validator package    | `@Valid` + Bean Validation |
| **中介軟體**      | Middleware classes               | Middleware classes            | Middleware functions | Filters / Interceptors     |
| **日誌**          | `Log::info()`                    | `logging.info()`              | `log.Println()`      | `logger.info()`            |
| **DB Migrations** | Migrations                       | Migrations                    | migrate/goose        | Liquibase/Flyway           |
| **驗證**          | Laravel Passport/Sanctum         | Django REST Framework         | JWT libraries        | Spring Security + JWT      |
| **錯誤處理**      | Exception Handler                | Exception middleware          | Error handlers       | `@RestControllerAdvice`    |

## 🎓 面試問題索引

這些指南中引用了所有 50 個後端面試問題:

**Java 語法(問題 1-5)**

- 參見:[02-java-modern-syntax.md](02-java-modern-syntax.md)

**Spring Boot 架構(問題 6-10)**

- 參見:[03-spring-boot-annotations.md](03-spring-boot-annotations.md)

**JPA & Hibernate(問題 11-15, 26-27)**

- 參見:[04-jpa-hibernate.md](04-jpa-hibernate.md)

**Spring Data JPA(問題 13-14, 28)**

- 參見:[05-spring-data-jpa.md](05-spring-data-jpa.md)

**交易管理(問題 16-19)**

- 參見:[06-transaction-management.md](06-transaction-management.md)

**Security & JWT(問題 8, 21-25)**

- 參見:[07-spring-security-jwt.md](07-spring-security-jwt.md)

**錯誤處理(問題 31-32)**

- 參見:[08-error-handling.md](08-error-handling.md)

**完整面試問題:** [../../interview/interview-backend.md](../../interview/interview-backend.md)

## 💡 如何使用這些指南

### 面試準備

1. **循序閱讀** 按照上述學習路徑
2. **執行程式碼** - 所有範例都來自本專案,你可以找到並執行它們
3. **練習** - 閱讀每個指南後,嘗試回答連結的面試問題
4. **比較** - 使用框架比較表格與你現有的知識關聯

### 在職參考

1. **跳到特定主題** 使用上面的指南索引
2. **搜尋語法** - 每個指南都有「快速語法速查表」部分
3. **找專案範例** - 查找帶有檔案參考的「實際專案範例」部分
4. **避免常見錯誤** - 查看「常見陷阱」部分

### 程式碼範例格式

所有程式碼範例遵循以下格式:

```markdown
### 範例:描述性標題

**檔案:** `www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java:56-65`

[程式碼片段]

**說明:** 這段程式碼的作用
**重點:**

- 重要細節 1
- 重要細節 2
```

你可以點擊檔案路徑直接跳轉到 IDE 中的原始程式碼。

## 🚀 下一步

1. **開始於:** [02-java-modern-syntax.md](02-java-modern-syntax.md)
2. **有問題?** 查看 [interview-backend.md](../../interview/interview-backend.md) 以獲得詳細的 Q&A
3. **想要練習?** 複製專案並探索每個指南中參考的實際程式碼檔案

## 📦 專案結構參考

```
www_root/waterballsa-backend/src/main/java/waterballsa/
├── controller/          # @RestController 類別 (HTTP 層)
├── service/            # 業務邏輯與 @Transactional
├── repository/         # JpaRepository 介面 (資料層)
├── entity/             # JPA entities (資料庫模型)
├── dto/                # Record 類別用於 API 請求/回應
├── exception/          # 自訂例外 + GlobalExceptionHandler
├── filter/             # JWT 身份驗證過濾器
├── config/             # Spring 配置 (Security、CORS)
├── util/               # 工具類別 (JwtUtil 等)
└── validator/          # 自訂驗證邏輯
```

## 🔗 其他資源

- [WaterballSA 後端面試問題](../../interview/interview-backend.md) - 所有 50 個問題及詳細答案
- [WaterballSA API 文件](../../api-docs/openapi/README.md) - API 規格和範例
- [專案 README](../../../README.md) - 設置和架構概述

---

**準備好開始了嗎?** → [02. Java 現代語法](02-java-modern-syntax.md)

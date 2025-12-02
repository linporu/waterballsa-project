# Spring Data JPA

> 給來自 Python/Laravel/Go 背景的中階開發者的快速參考
> 📚 相關面試題：[#13](../../interview/interview-backend.md#13-pessimistic-locking-mid)、[#14](../../interview/interview-backend.md#14-jpql-query-mid)、[#28](../../interview/interview-backend.md#28-pagination-實作-mid)

## 從 Python/Laravel/Go 轉換過來

| 概念           | Laravel/Eloquent      | Python/SQLAlchemy            | Go/GORM                | Spring Data JPA            |
| -------------- | --------------------- | ---------------------------- | ---------------------- | -------------------------- |
| **Repository** | Model class           | Session queries              | DB methods             | `JpaRepository` interface  |
| **依 ID 查詢** | `Model::find($id)`    | `session.get(Model, id)`     | `db.First(&model, id)` | `repository.findById(id)`  |
| **查詢全部**   | `Model::all()`        | `session.query(Model).all()` | `db.Find(&models)`     | `repository.findAll()`     |
| **自訂查詢**   | Query builder         | `session.query().filter()`   | `db.Where()`           | `@Query` JPQL              |
| **儲存**       | `$model->save()`      | `session.add(model)`         | `db.Create(&model)`    | `repository.save(model)`   |
| **刪除**       | `$model->delete()`    | `session.delete(model)`      | `db.Delete(&model)`    | `repository.delete(model)` |
| **分頁**       | `Model::paginate(20)` | `query.limit().offset()`     | `db.Limit().Offset()`  | `Pageable` + `Page<T>`     |
| **鎖定**       | `lockForUpdate()`     | `with_for_update()`          | Gorm locking           | `@Lock(PESSIMISTIC_WRITE)` |

## 快速語法速查表

### 1. Repository 介面

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // 內建可用方法：
    // - findById(Long id): Optional<Order>
    // - findAll(): List<Order>
    // - save(Order entity): Order
    // - delete(Order entity): void
    // - count(): long
    // - existsById(Long id): boolean
}
```

**Repository 階層架構：**

- `Repository<T, ID>` - 標記介面
- `CrudRepository<T, ID>` - 基本 CRUD 操作
- `PagingAndSortingRepository<T, ID>` - 新增分頁和排序功能
- `JpaRepository<T, ID>` - 新增 JPA 專屬操作（最常用）

---

### 2. 方法命名慣例（衍生查詢）

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // 依單一欄位查詢
    Optional<Order> findByOrderNumber(String orderNumber);

    // 依多個欄位查詢
    Optional<Order> findByIdAndUserId(Long id, Long userId);

    // 布林查詢
    List<Order> existsByUserId(Long userId);

    // 計數
    long countByStatus(OrderStatus status);

    // 刪除
    void deleteByUserId(Long userId);

    // 帶排序
    List<Order> findByUserIdOrderByCreatedAtDesc(Long userId);
}
```

**方法命名關鍵字：**

- `findBy` / `getBy` / `queryBy` / `readBy`
- `countBy`
- `existsBy`
- `deleteBy` / `removeBy`
- `And`、`Or`
- `OrderBy...Asc` / `OrderBy...Desc`
- `Top` / `First`（例如：`findTop10By`）

---

### 3. 使用 JPQL 自訂 @Query

```java
@Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.deletedAt IS NULL")
List<Order> findByUserIdNotDeleted(@Param("userId") Long userId);

@Query("SELECT o FROM Order o JOIN o.items oi WHERE oi.journeyId = :journeyId")
List<Order> findOrdersByJourneyId(@Param("journeyId") Long journeyId);

// 原生 SQL（謹慎使用）
@Query(value = "SELECT * FROM orders WHERE user_id = ?1", nativeQuery = true)
List<Order> findByUserIdNative(Long userId);
```

**JPQL vs SQL：**

- JPQL 使用實體名稱（Order）而非資料表名稱（orders）
- JPQL 使用欄位名稱（userId）而非資料庫欄位名稱（user_id）
- JPQL 與資料庫無關

---

### 4. 使用 Pageable 進行分頁

```java
// Repository 方法
Page<Order> findByUserId(Long userId, Pageable pageable);

// 在 service 中使用
Pageable pageable = PageRequest.of(page - 1, limit);  // page 是從 0 開始
Page<Order> orderPage = orderRepository.findByUserId(userId, pageable);

// 提取資料
List<Order> orders = orderPage.getContent();
long totalElements = orderPage.getTotalElements();
int totalPages = orderPage.getTotalPages();
```

---

### 5. 悲觀鎖定

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id AND o.userId = :userId")
Optional<Order> findByIdAndUserIdForUpdate(@Param("id") Long id, @Param("userId") Long userId);
```

**鎖定類型：**

- `PESSIMISTIC_WRITE` - 獨占鎖（SELECT FOR UPDATE）
- `PESSIMISTIC_READ` - 共享鎖（SELECT FOR SHARE）
- `OPTIMISTIC` - 基於版本的鎖定

---

## 實際專案範例

### 範例 1：基本 Repository 介面

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:17-18`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L17-L18)

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // 從 JpaRepository 繼承所有 CRUD 方法
}
```

**說明：** 擴展 JpaRepository 的最小 repository。

**免費獲得的內建方法：**

```java
// 查詢
Optional<Order> findById(Long id);
List<Order> findAll();
List<Order> findAllById(Iterable<Long> ids);

// 儲存
Order save(Order order);  // INSERT 或 UPDATE
List<Order> saveAll(Iterable<Order> orders);

// 刪除
void delete(Order order);
void deleteById(Long id);
void deleteAll();

// 計數與存在性檢查
long count();
boolean existsById(Long id);
```

---

### 範例 2：方法命名慣例

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:26-35`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L26-L35)

```java
/**
 * 依訂單編號查詢訂單。
 */
Optional<Order> findByOrderNumber(String orderNumber);

/**
 * 依 ID 和使用者 ID 查詢訂單（用於擁有權驗證）。
 */
Optional<Order> findByIdAndUserId(Long id, Long userId);
```

**說明：** Spring Data JPA 從方法名稱生成查詢。

**生成的 SQL：**

```sql
-- findByOrderNumber
SELECT * FROM orders WHERE order_number = ?

-- findByIdAndUserId
SELECT * FROM orders WHERE id = ? AND user_id = ?
```

**方法命名模式：**

```java
// 單一欄位
findByUsername(String username)
// → WHERE username = ?

// 多欄位搭配 AND
findByUsernameAndEmail(String username, String email)
// → WHERE username = ? AND email = ?

// 多欄位搭配 OR
findByUsernameOrEmail(String username, String email)
// → WHERE username = ? OR email = ?

// 排序
findByUserIdOrderByCreatedAtDesc(Long userId)
// → WHERE user_id = ? ORDER BY created_at DESC

// LIKE
findByUsernameLike(String pattern)
// → WHERE username LIKE ?

// IN
findByIdIn(List<Long> ids)
// → WHERE id IN (?, ?, ?)

// 大於 / 小於
findByCreatedAtAfter(LocalDateTime date)
// → WHERE created_at > ?
```

---

### 範例 3：使用 JPQL 自訂 @Query

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:46-57`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L46-L57)

```java
/**
 * 依使用者 ID 和旅程 ID 查詢未付款訂單。
 * 如有多筆訂單則回傳最近的一筆。
 */
@Query(
    "SELECT o FROM Order o "
        + "JOIN o.items oi "
        + "WHERE o.userId = :userId "
        + "AND o.status = :status "
        + "AND oi.journeyId = :journeyId "
        + "AND o.deletedAt IS NULL "
        + "ORDER BY o.createdAt DESC LIMIT 1")
Optional<Order> findByUserIdAndStatusAndJourneyId(
    @Param("userId") Long userId,
    @Param("status") OrderStatus status,
    @Param("journeyId") Long journeyId);
```

**說明：** 帶有 JOIN 和多個條件的自訂 JPQL 查詢。

**重點：**

- `JOIN o.items oi` - 使用 JPA 關聯進行 JOIN（不是資料表名稱）
- `@Param` - 使用具名參數以提高可讀性
- JPQL 實體名稱：`Order`、`items`（不是資料表名稱）
- `LIMIT 1` - 回傳最近的訂單

**生成的 SQL（約略）：**

```sql
SELECT o.*
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = ?
  AND o.status = ?
  AND oi.journey_id = ?
  AND o.deleted_at IS NULL
ORDER BY o.created_at DESC
LIMIT 1
```

---

### 範例 4：使用 @Query 進行分頁

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:66-71`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L66-L71)

```java
/**
 * 依使用者 ID 查詢所有訂單，並進行分頁，依建立時間降序排序。
 */
@Query(
    "SELECT o FROM Order o "
        + "WHERE o.userId = :userId "
        + "AND o.deletedAt IS NULL "
        + "ORDER BY o.createdAt DESC")
Page<Order> findByUserIdOrderByCreatedAtDesc(@Param("userId") Long userId, Pageable pageable);
```

**在 Service 中使用：**

```java
// page = 1（來自使用者），limit = 20
Pageable pageable = PageRequest.of(page - 1, limit);  // 轉換為從 0 開始
Page<Order> orderPage = orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageable);

// 提取結果
List<Order> orders = orderPage.getContent();
long totalElements = orderPage.getTotalElements();
int totalPages = orderPage.getTotalPages();
boolean hasNext = orderPage.hasNext();
```

**重點：**

- 回傳型別為 `Page<Order>`（不是 `List<Order>`）
- `Pageable` 參數會自動加上 LIMIT 和 OFFSET
- 如果在 Pageable 中使用 `Sort`，就不需要在 JPQL 中指定 ORDER BY
- 頁面索引從 0 開始（第 1 頁 → 索引 0）

**回應結構：**

```java
{
  "orders": [...],  // getContent()
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,           // getTotalElements()
    "totalPages": 3,       // getTotalPages()
    "hasNext": true,       // hasNext()
    "hasPrevious": false   // hasPrevious()
  }
}
```

---

### 範例 5：用於並發控制的悲觀鎖定

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:93-95`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L93-L95)

```java
/**
 * 使用悲觀寫入鎖依 ID 和使用者 ID 查詢訂單，用於付款處理。
 * 這可以防止對同一筆訂單進行並發付款嘗試。
 */
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id AND o.userId = :userId AND o.deletedAt IS NULL")
Optional<Order> findByIdAndUserIdForUpdate(@Param("id") Long id, @Param("userId") Long userId);
```

**在 Service 中使用：**

```java
@Transactional
public PayOrderResponse payOrder(Long orderId, Long userId) {
    // 在訂單上取得獨占鎖
    Order order = orderRepository.findByIdAndUserIdForUpdate(orderId, userId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));

    // 現在只有此交易可以修改訂單
    // 其他交易會等待
    orderValidator.validateOrderNotPaid(order);
    order.markAsPaid();
    orderRepository.save(order);

    return response;
}
```

**說明：** 防止付款處理期間的競態條件。

**生成的 SQL：**

```sql
SELECT * FROM orders
WHERE id = ? AND user_id = ? AND deleted_at IS NULL
FOR UPDATE  -- 獨占鎖
```

**為什麼需要：**

```java
// 沒有鎖定 - 競態條件！
// 使用者快速點擊兩次「付款」
執行緒 1：讀取訂單（status = UNPAID）
執行緒 2：讀取訂單（status = UNPAID）
執行緒 1：標記為 PAID，儲存
執行緒 2：標記為 PAID，儲存  // 重複付款！

// 使用悲觀鎖定 - 安全
執行緒 1：讀取訂單 FOR UPDATE（取得鎖）
執行緒 2：讀取訂單 FOR UPDATE（等待鎖）
執行緒 1：標記為 PAID，儲存，提交（釋放鎖）
執行緒 2：取得鎖，讀取（status = PAID），驗證失敗
```

**鎖定類型：**

- `PESSIMISTIC_WRITE` - 獨占鎖，其他人等待
- `PESSIMISTIC_READ` - 共享讀取鎖
- `OPTIMISTIC` - 基於版本，在提交時檢查

---

### 範例 6：批次處理的排程查詢

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:81`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L81)

```java
/**
 * 依狀態查詢到期時間已過的訂單。
 * 由排程任務使用，用於在 3 天後讓未付款訂單過期。
 */
List<Order> findByStatusAndExpiredAtBefore(OrderStatus status, LocalDateTime now);
```

**在排程任務中使用：**

```java
@Scheduled(cron = "0 0 * * * *")  // 每小時執行
public void expireUnpaidOrders() {
    LocalDateTime now = LocalDateTime.now();
    List<Order> expiredOrders = orderRepository
        .findByStatusAndExpiredAtBefore(OrderStatus.UNPAID, now);

    for (Order order : expiredOrders) {
        order.markAsExpired();
        orderRepository.save(order);
    }

    logger.info("已過期 {} 筆未付款訂單", expiredOrders.size());
}
```

**生成的 SQL：**

```sql
SELECT * FROM orders
WHERE status = 'UNPAID'
  AND expired_at < ?
```

---

## 常見陷阱

### ❌ 不要：從 repository 回傳 null

```java
// 不要這樣做
public Order findById(Long id) {
    return orderRepository.findById(id).orElse(null);  // 不好！
}
```

### ✅ 要：回傳 Optional

```java
// 要這樣做
public Order findById(Long id) {
    return orderRepository.findById(id)
        .orElseThrow(() -> new OrderNotFoundException(id));
}
```

**為什麼：** Optional 讓 null 處理變得明確。呼叫者知道值可能不存在。

---

### ❌ 不要：對複雜邏輯使用衍生查詢

```java
// 方法名稱過長且難以閱讀
List<Order> findByUserIdAndStatusAndCreatedAtBetweenAndDeletedAtIsNullOrderByCreatedAtDesc(
    Long userId, OrderStatus status, LocalDateTime start, LocalDateTime end);
```

### ✅ 要：對複雜查詢使用 @Query

```java
@Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.status = :status " +
       "AND o.createdAt BETWEEN :start AND :end AND o.deletedAt IS NULL " +
       "ORDER BY o.createdAt DESC")
List<Order> findOrdersByUserAndDateRange(...);
```

**為什麼：** 對於複雜條件，@Query 更易讀。

---

### ❌ 不要：對大型資料集忘記分頁

```java
List<Order> findByUserId(Long userId);  // 可能會回傳 10,000+ 筆訂單！
```

### ✅ 要：使用分頁

```java
Page<Order> findByUserId(Long userId, Pageable pageable);
```

**為什麼：** 防止記憶體問題和慢查詢。

---

### ❌ 不要：對 LAZY 關聯使用 N+1 查詢

```java
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    // 每次迴圈都會觸發對 items 的單獨查詢！
    order.getItems().size();  // N+1 問題
}
```

### ✅ 要：使用 JOIN FETCH

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
List<Order> findByUserIdWithItems(@Param("userId") Long userId);
```

**為什麼：** 在一次查詢中取得所有資料。

---

### ❌ 不要：混用 JPQL 實體名稱和 SQL 資料表名稱

```java
@Query("SELECT o FROM orders o WHERE o.user_id = :userId")  // 錯誤！
// 'orders' 是資料表名稱，'user_id' 是欄位名稱
```

### ✅ 要：使用實體和欄位名稱

```java
@Query("SELECT o FROM Order o WHERE o.userId = :userId")  // 正確！
// 'Order' 是實體名稱，'userId' 是欄位名稱
```

---

## Repository 方法快速參考

### 內建方法（來自 JpaRepository）

| 方法                   | 回傳型別      | 說明             |
| ---------------------- | ------------- | ---------------- |
| `findById(ID)`         | `Optional<T>` | 依主鍵查詢       |
| `findAll()`            | `List<T>`     | 查詢所有實體     |
| `findAll(Pageable)`    | `Page<T>`     | 分頁查詢所有實體 |
| `save(T)`              | `T`           | 儲存或更新實體   |
| `saveAll(Iterable<T>)` | `List<T>`     | 儲存多筆資料     |
| `delete(T)`            | `void`        | 刪除實體         |
| `deleteById(ID)`       | `void`        | 依 ID 刪除       |
| `count()`              | `long`        | 計算總數         |
| `existsById(ID)`       | `boolean`     | 檢查是否存在     |

### 方法命名關鍵字

| 關鍵字           | SQL 範例                 | JPQL 範例        |
| ---------------- | ------------------------ | ---------------- |
| `findBy`         | `SELECT *`               | `SELECT e`       |
| `And`            | `WHERE x = ? AND y = ?`  | 相同             |
| `Or`             | `WHERE x = ? OR y = ?`   | 相同             |
| `Is`, `Equals`   | `WHERE x = ?`            | 相同             |
| `LessThan`       | `WHERE x < ?`            | 相同             |
| `GreaterThan`    | `WHERE x > ?`            | 相同             |
| `Before`         | `WHERE x < ?`            | 相同（用於日期） |
| `After`          | `WHERE x > ?`            | 相同（用於日期） |
| `Like`           | `WHERE x LIKE ?`         | 相同             |
| `NotLike`        | `WHERE x NOT LIKE ?`     | 相同             |
| `StartingWith`   | `WHERE x LIKE 'value%'`  | 相同             |
| `EndingWith`     | `WHERE x LIKE '%value'`  | 相同             |
| `Containing`     | `WHERE x LIKE '%value%'` | 相同             |
| `In`             | `WHERE x IN (...)`       | 相同             |
| `NotIn`          | `WHERE x NOT IN (...)`   | 相同             |
| `IsNull`         | `WHERE x IS NULL`        | 相同             |
| `IsNotNull`      | `WHERE x IS NOT NULL`    | 相同             |
| `True`           | `WHERE x = TRUE`         | 相同             |
| `False`          | `WHERE x = FALSE`        | 相同             |
| `OrderBy...Asc`  | `ORDER BY x ASC`         | 相同             |
| `OrderBy...Desc` | `ORDER BY x DESC`        | 相同             |

### 分頁方法

```java
// 建立 Pageable
Pageable pageable = PageRequest.of(page, size);
Pageable withSort = PageRequest.of(page, size, Sort.by("createdAt").descending());

// Page 方法
Page<Order> page = repository.findAll(pageable);
List<Order> content = page.getContent();
long total = page.getTotalElements();
int totalPages = page.getTotalPages();
boolean hasNext = page.hasNext();
```

---

## 透過面試題練習

- 📝 [題目 #13：悲觀鎖定](../../interview/interview-backend.md#13-pessimistic-locking-mid)
- 📝 [題目 #14：JPQL 查詢](../../interview/interview-backend.md#14-jpql-query-mid)
- 📝 [題目 #28：分頁實作](../../interview/interview-backend.md#28-pagination-實作-mid)

---

**上一篇：** [← 04. JPA & Hibernate](04-jpa-hibernate.md)

**下一篇：** [06. Transaction Management](06-transaction-management.md) →

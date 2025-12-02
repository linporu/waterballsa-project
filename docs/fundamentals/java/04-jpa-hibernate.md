# JPA & Hibernate

> 從 Python/Laravel/Go 轉來的中階開發者快速參考
> 📚 相關面試問題：[#4](../../interview/interview-backend.md#4-enum-與-jpa-整合-junior), [#11](../../interview/interview-backend.md#11-entity-lifecycle-callbacks-junior), [#12](../../interview/interview-backend.md#12-onetomany-relationship-junior), [#15](../../interview/interview-backend.md#15-bigdecimal-與-precision-junior), [#26](../../interview/interview-backend.md#26-bidirectional-relationship-mid), [#27](../../interview/interview-backend.md#27-cascadetype-與-orphanremoval-mid)

## 從 Python/Laravel/Go 轉來

| 概念           | Laravel/Eloquent           | Python/SQLAlchemy          | Go/GORM                  | JPA/Hibernate              |
| -------------- | -------------------------- | -------------------------- | ------------------------ | -------------------------- |
| **Model 類別** | `extends Model`            | `declarative_base()`       | `type User struct`       | `@Entity`                  |
| **主鍵**       | `protected $primaryKey`    | `Column(primary_key=True)` | `gorm:"primaryKey"`      | `@Id`                      |
| **資料表名稱** | `protected $table`         | `__tablename__`            | `TableName()`            | `@Table(name="...")`       |
| **欄位映射**   | 慣例或 `$casts`            | `Column()`                 | `gorm:"column:..."`      | `@Column`                  |
| **關聯**       | `hasMany()`, `belongsTo()` | `relationship()`           | `Belongs To`, `Has Many` | `@OneToMany`, `@ManyToOne` |
| **時間戳記**   | `$timestamps = true`       | `default=datetime.now`     | `gorm:"autoCreateTime"`  | `@PrePersist` 回調         |
| **軟刪除**     | `SoftDeletes` trait        | `deleted_at` 欄位          | `gorm:"softDelete"`      | 自訂 `deletedAt` 欄位      |
| **列舉**       | Cast to enum               | `Enum` 類型                | 自訂類型                 | `@Enumerated`              |

## 快速語法備忘錄

### 1. Entity 基礎

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "price", nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    protected Order() {  // JPA 需要無參數建構子
    }

    public Order(Long userId, BigDecimal price) {
        this.userId = userId;
        this.price = price;
    }

    // Getters 和 setters...
}
```

**關鍵註解：**

- `@Entity` - 標記類別為 JPA entity
- `@Table(name="...")` - 映射到資料庫資料表
- `@Id` - 主鍵欄位
- `@GeneratedValue` - 自動遞增策略
- `@Column` - 欄位映射與約束

---

### 2. 欄位約束

```java
@Column(
    name = "email",              // 資料庫欄位名稱
    nullable = false,            // NOT NULL 約束
    unique = true,               // UNIQUE 約束
    length = 255,                // VARCHAR(255)
    columnDefinition = "TEXT",   // 自訂 SQL 類型
    updatable = false,           // 建立後無法更新
    insertable = true,           // 可以插入
    precision = 10,              // BigDecimal：總位數
    scale = 2                    // BigDecimal：小數位數
)
private String email;
```

---

### 3. Entity 生命週期回調

```java
@Entity
public class Order {

    @PrePersist  // INSERT 之前
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }

    @PreUpdate  // UPDATE 之前
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }

    @PreRemove  // DELETE 之前
    protected void onDelete() {
        // 清理邏輯
    }

    @PostLoad  // 從資料庫載入 entity 之後
    protected void onLoad() {
        // 載入後處理
    }
}
```

**生命週期事件：**

- `@PrePersist` - 儲存新 entity 之前
- `@PostPersist` - 儲存新 entity 之後
- `@PreUpdate` - 更新 entity 之前
- `@PostUpdate` - 更新 entity 之後
- `@PreRemove` - 刪除 entity 之前
- `@PostRemove` - 刪除 entity 之後
- `@PostLoad` - 從資料庫載入之後

---

### 4. 列舉映射

```java
public enum OrderStatus {
    UNPAID, PAID, EXPIRED
}

@Entity
public class Order {

    @Enumerated(EnumType.STRING)  // 以字串儲存："UNPAID", "PAID"
    @JdbcTypeCode(SqlTypes.NAMED_ENUM)  // 使用 PostgreSQL ENUM 類型
    @Column(name = "status", nullable = false, columnDefinition = "order_status")
    private OrderStatus status;
}
```

**@Enumerated 選項：**

- `EnumType.STRING` - 儲存列舉名稱（例如 "UNPAID"）
- `EnumType.ORDINAL` - 儲存列舉索引（例如 0, 1, 2）- 不建議使用

**為什麼使用 STRING 而非 ORDINAL：**

- 順序獨立：可以安全地重新排序列舉值
- 可讀性：資料庫儲存 "PAID" 而非 "1"
- 可維護性：可以在列舉中任意位置新增新值

---

### 5. 關聯映射

#### One-to-Many（擁有方）

```java
@Entity
public class Order {

    @OneToMany(
        mappedBy = "order",              // OrderItem 類別中的欄位名稱
        cascade = CascadeType.ALL,        // 串聯操作
        orphanRemoval = true,             // 刪除孤兒項目
        fetch = FetchType.LAZY            // 延遲載入（預設）
    )
    private List<OrderItem> items = new ArrayList<>();
}
```

#### Many-to-One（反向方）

```java
@Entity
public class OrderItem {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
}
```

---

### 6. FetchType.LAZY vs EAGER

```java
@ManyToOne(fetch = FetchType.LAZY)   // 存取時才取得（ManyToOne 的預設值）
private Order order;

@ManyToOne(fetch = FetchType.EAGER)  // 與主要 entity 一起立即取得
private Order order;

@OneToMany(fetch = FetchType.LAZY)   // OneToMany 的預設值
private List<OrderItem> items;
```

**LAZY（建議）：**

- 只在存取時載入相關 entities
- 防止 N+1 查詢問題
- 效能較佳

**EAGER：**

- 立即載入相關 entities
- 可能造成效能問題
- 謹慎使用

---

### 7. CascadeType 選項

```java
@OneToMany(cascade = CascadeType.ALL)  // 所有操作都串聯
@OneToMany(cascade = CascadeType.PERSIST)  // 只串聯 INSERT
@OneToMany(cascade = CascadeType.MERGE)  // 只串聯 UPDATE
@OneToMany(cascade = CascadeType.REMOVE)  // 只串聯 DELETE
@OneToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})  // 多個
```

**CascadeType.ALL：** 包含 PERSIST、MERGE、REMOVE、REFRESH、DETACH

---

### 8. 軟刪除模式

```java
@Entity
public class Order {

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    public void softDelete() {
        this.deletedAt = LocalDateTime.now();
    }

    public boolean isDeleted() {
        return this.deletedAt != null;
    }
}
```

---

## 實際專案範例

### 範例 1：完整的 Entity 與生命週期回調

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:12-87`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L12-L87)

```java
@Entity
@Table(name = "orders")
public class Order {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(name = "order_number", nullable = false, unique = true, length = 50)
  private String orderNumber;

  @NonNull
  @Column(name = "user_id", nullable = false)
  private Long userId;

  @Enumerated(EnumType.STRING)
  @JdbcTypeCode(SqlTypes.NAMED_ENUM)
  @Column(name = "status", nullable = false, columnDefinition = "order_status")
  private OrderStatus status;

  @Column(name = "price", nullable = false, precision = 10, scale = 2)
  private BigDecimal price;

  @Column(name = "created_at", nullable = false, updatable = false)
  private LocalDateTime createdAt;

  @Column(name = "paid_at")
  private LocalDateTime paidAt;

  @Column(name = "expired_at")
  private LocalDateTime expiredAt;

  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<OrderItem> items = new ArrayList<>();

  protected Order() {
    // JPA 需要無參數建構子
  }

  @PrePersist
  protected void onCreate() {
    this.createdAt = LocalDateTime.now();
    this.updatedAt = LocalDateTime.now();
    if (this.status == OrderStatus.UNPAID) {
      this.expiredAt = this.createdAt.plusDays(3);
    }
  }

  @PreUpdate
  protected void onUpdate() {
    this.updatedAt = LocalDateTime.now();
  }
}
```

**說明：** 包含所有常見 JPA 模式的完整訂單 entity。

**重點：**

- **主鍵：** 自動遞增的 `id`
- **唯一性約束：** `order_number` 必須唯一
- **列舉映射：** `status` 以字串形式儲存在 PostgreSQL 列舉類型中
- **BigDecimal：** `price` 精度 10，小數位數 2（總共 10 位數，小數點後 2 位）
- **不可變欄位：** `created_at` 設定 `updatable = false`
- **可為 null 的欄位：** `paid_at`、`expired_at` 可以為 null
- **@PrePersist：** 建立時自動設定時間戳記和過期時間
- **@PreUpdate：** 修改時自動更新 `updated_at`
- **關聯：** 與 OrderItem 的一對多關聯

---

### 範例 2：與 PostgreSQL 的列舉整合

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:27-30`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L27-L30)

```java
@Enumerated(EnumType.STRING)
@JdbcTypeCode(SqlTypes.NAMED_ENUM)
@Column(name = "status", nullable = false, columnDefinition = "order_status")
private OrderStatus status;
```

**列舉定義：**

```java
public enum OrderStatus {
  UNPAID,
  PAID,
  EXPIRED
}
```

**說明：** 將 Java 列舉映射到 PostgreSQL 自訂列舉類型。

**重點：**

- `@Enumerated(EnumType.STRING)` - 儲存列舉名稱，而非序數
- `@JdbcTypeCode(SqlTypes.NAMED_ENUM)` - 使用 PostgreSQL 原生 ENUM
- `columnDefinition = "order_status"` - PostgreSQL 列舉類型名稱

**為什麼使用這種方法：**

```sql
-- PostgreSQL schema
CREATE TYPE order_status AS ENUM ('UNPAID', 'PAID', 'EXPIRED');

CREATE TABLE orders (
    status order_status NOT NULL
);
```

**如果新增列舉值：**

```java
public enum OrderStatus {
  UNPAID,
  PAID,
  EXPIRED,
  REFUNDED  // 新增的值
}
```

需要更新 PostgreSQL：

```sql
ALTER TYPE order_status ADD VALUE 'REFUNDED';
```

**如果刪除列舉值：**

- 無法從 PostgreSQL 列舉中移除（限制）
- 解決方案：在 Java 中標記為已棄用，保留在資料庫中

---

### 範例 3：@PrePersist 和 @PreUpdate 生命週期

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:74-87`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L74-L87)

```java
@PrePersist
protected void onCreate() {
  this.createdAt = LocalDateTime.now();
  this.updatedAt = LocalDateTime.now();
  // 為未付款訂單設定建立後 3 天過期
  if (this.status == OrderStatus.UNPAID) {
    this.expiredAt = this.createdAt.plusDays(3);
  }
}

@PreUpdate
protected void onUpdate() {
  this.updatedAt = LocalDateTime.now();
}
```

**說明：** Entity 儲存/更新時的自動時間戳記和業務邏輯。

**重點：**

- `@PrePersist` 在 `INSERT` SQL 之前執行
- `@PreUpdate` 在 `UPDATE` SQL 之前執行
- 業務邏輯：未付款訂單 3 天後過期
- 無需手動管理時間戳記

**執行流程：**

```java
// 建立新訂單
Order order = new Order(...);
orderRepository.save(order);
// → @PrePersist 執行 → 設定 createdAt、updatedAt、expiredAt → INSERT SQL

// 更新訂單
order.markAsPaid();
orderRepository.save(order);
// → @PreUpdate 執行 → 更新 updatedAt → UPDATE SQL
```

---

### 範例 4：@OneToMany 與 Cascade 和 orphanRemoval

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:56-57`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L56-L57)

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

**反向方（OrderItem）：**

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
private Order order;
```

**說明：** 雙向一對多關聯。

**重點：**

- `mappedBy = "order"` - OrderItem.order 欄位是關聯的擁有者
- `cascade = CascadeType.ALL` - 儲存/刪除操作串聯到項目
- `orphanRemoval = true` - 從集合中移除時刪除項目
- 雙向：Order 知道 items，OrderItem 知道 order

**串聯行為：**

```java
// 建立包含項目的訂單
Order order = new Order(...);
order.addItem(new OrderItem(...));  // 輔助方法設定雙向連結
orderRepository.save(order);
// → Order 儲存 → Items 自動儲存（串聯）

// 刪除訂單
orderRepository.delete(order);
// → Items 自動刪除（串聯）

// 從集合中移除項目
order.getItems().remove(item);
orderRepository.save(order);
// → Item 從資料庫中刪除（orphanRemoval）
```

---

### 範例 5：雙向關聯輔助方法

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:89-93`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L89-L93)

```java
// 業務方法
public void addItem(OrderItem item) {
  items.add(item);
  item.setOrder(this);  // 設定雙向連結
}
```

**說明：** 輔助方法維護雙向關聯的一致性。

**重點：**

- 確保關聯的兩邊同步
- 防止不一致的狀態
- 比手動設定雙方更簡潔

**用法：**

```java
// ✅ 正確：使用輔助方法
Order order = new Order(...);
OrderItem item = new OrderItem(...);
order.addItem(item);  // 設定關聯的雙方

// ❌ 錯誤：手動設定（容易忘記一邊）
order.getItems().add(item);
item.setOrder(order);  // 容易忘記這個！
```

---

### 範例 6：@ManyToOne 與 FetchType.LAZY

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/OrderItem.java:15-17`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/OrderItem.java#L15-L17)

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
private Order order;
```

**說明：** 多個訂單項目屬於一個訂單，使用延遲載入。

**重點：**

- `fetch = FetchType.LAZY` - 存取前不載入 Order
- `@JoinColumn(name = "order_id")` - 外鍵欄位名稱
- `nullable = false` - Order 為必填（NOT NULL 約束）

**LAZY vs EAGER：**

```java
// LAZY（建議）
@ManyToOne(fetch = FetchType.LAZY)
private Order order;

OrderItem item = orderItemRepository.findById(1L).get();
// SELECT * FROM order_items WHERE id = 1  (無 JOIN)

item.getOrder().getOrderNumber();
// SELECT * FROM orders WHERE id = ?  (存取時才執行獨立查詢)

// EAGER（不建議）
@ManyToOne(fetch = FetchType.EAGER)
private Order order;

OrderItem item = orderItemRepository.findById(1L).get();
// SELECT * FROM order_items oi LEFT JOIN orders o ON oi.order_id = o.id WHERE oi.id = 1
// (總是 join，即使不需要 order 資料)
```

---

### 範例 7：BigDecimal 與精度和小數位數

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:32-39`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L32-L39)

```java
@Column(name = "original_price", nullable = false, precision = 10, scale = 2)
private BigDecimal originalPrice;

@Column(name = "discount", nullable = false, precision = 10, scale = 2)
private BigDecimal discount;

@Column(name = "price", nullable = false, precision = 10, scale = 2)
private BigDecimal price;
```

**說明：** 使用 BigDecimal 處理金額以確保精度。

**重點：**

- `precision = 10` - 總位數（例如 12345678.90）
- `scale = 2` - 小數點後位數（例如 XX.12）
- `BigDecimal` - 精確的小數表示（無浮點數誤差）

**為什麼金額要用 BigDecimal：**

```java
// ❌ 不要用 double 處理金額
double price = 0.1 + 0.2;  // = 0.30000000000000004 (浮點數誤差！)

// ✅ 要用 BigDecimal
BigDecimal price = new BigDecimal("0.1").add(new BigDecimal("0.2"));
// = 0.3 (精確)
```

**SQL 映射：**

```sql
CREATE TABLE orders (
    original_price DECIMAL(10, 2) NOT NULL,  -- 最大值：99999999.99
    discount DECIMAL(10, 2) NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);
```

---

### 範例 8：軟刪除實作

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:53-54`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L53-L54)

```java
@Column(name = "deleted_at")
private LocalDateTime deletedAt;
```

**業務方法：**

```java
public void softDelete() {
  this.deletedAt = LocalDateTime.now();
}

public boolean isDeleted() {
  return this.deletedAt != null;
}
```

**說明：** 軟刪除模式 - 標記為已刪除而非真正移除。

**重點：**

- `deleted_at` 可為 null - null 表示未刪除
- 查詢必須過濾掉已刪除的記錄
- 允許資料復原和稽核軌跡

**在查詢中使用：**

```java
// Repository 方法
@Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.deletedAt IS NULL")
List<Order> findByUserIdNotDeleted(@Param("userId") Long userId);
```

**Laravel 對應：**

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Order extends Model {
    use SoftDeletes;  // 自動處理軟刪除
}

// 查詢自動排除軟刪除的記錄
Order::where('user_id', $userId)->get();
```

---

### 範例 9：可為 Null 的時間戳記

**File:** [`www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java:41-48`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/entity/Order.java#L41-L48)

```java
@Column(name = "created_at", nullable = false, updatable = false)
private LocalDateTime createdAt;

@Column(name = "paid_at")
private LocalDateTime paidAt;

@Column(name = "expired_at")
private LocalDateTime expiredAt;
```

**說明：** 不同的時間戳記欄位有不同的可為 null 規則。

**重點：**

- `created_at` - 必填、不可變（`updatable = false`）
- `paid_at` - 可為 null（null = 尚未付款）
- `expired_at` - 可為 null（null = 無過期時間或已付款）

**業務邏輯：**

```java
public void markAsPaid() {
  this.status = OrderStatus.PAID;
  this.paidAt = LocalDateTime.now();
  this.expiredAt = null;  // 付款時清除過期時間
}

public boolean isExpired() {
  return expiredAt != null && LocalDateTime.now().isAfter(expiredAt);
}
```

---

## 常見陷阱

### ❌ 不要：忘記無參數建構子

```java
@Entity
public class Order {
    private Long id;
    private String orderNumber;

    // 遺漏無參數建構子！
    public Order(Long id, String orderNumber) {
        this.id = id;
        this.orderNumber = orderNumber;
    }
}
```

### ✅ 要：包含 protected 無參數建構子

```java
@Entity
public class Order {
    private Long id;
    private String orderNumber;

    protected Order() {
        // JPA 需要無參數建構子
    }

    public Order(Long id, String orderNumber) {
        this.id = id;
        this.orderNumber = orderNumber;
    }
}
```

**原因：** JPA 使用反射來實例化 entities。設為 `protected` 可防止直接實例化。

---

### ❌ 不要：到處使用 EAGER 載入

```java
@ManyToOne(fetch = FetchType.EAGER)  // 不好！
private Order order;

@OneToMany(fetch = FetchType.EAGER)  // 不好！
private List<OrderItem> items;
```

### ✅ 要：預設使用 LAZY，選擇性使用 EAGER

```java
@ManyToOne(fetch = FetchType.LAZY)  // 正確！
private Order order;

@OneToMany(fetch = FetchType.LAZY)  // 正確！
private List<OrderItem> items;
```

**原因：** EAGER 會造成 N+1 問題並載入不必要的資料。需要即時載入時使用 `@EntityGraph` 或查詢中的 JOIN FETCH。

---

### ❌ 不要：使用 EnumType.ORDINAL

```java
@Enumerated(EnumType.ORDINAL)  // 不好！儲存 0, 1, 2
private OrderStatus status;

public enum OrderStatus {
    UNPAID,   // 0
    PAID,     // 1
    EXPIRED   // 2
}
```

### ✅ 要：使用 EnumType.STRING

```java
@Enumerated(EnumType.STRING)  // 正確！儲存 "UNPAID", "PAID", "EXPIRED"
private OrderStatus status;
```

**原因：** ORDINAL 在重新排序列舉值時會出問題。STRING 安全且可讀。

---

### ❌ 不要：使用 double/float 處理金額

```java
@Column(name = "price")
private double price;  // 不好！浮點數誤差
```

### ✅ 要：使用 BigDecimal 搭配 precision/scale

```java
@Column(name = "price", precision = 10, scale = 2)
private BigDecimal price;  // 正確！精確的小數
```

**原因：** 浮點數運算有四捨五入誤差。BigDecimal 是精確的。

---

### ❌ 不要：忘記設定雙向關聯的兩邊

```java
Order order = new Order(...);
OrderItem item = new OrderItem(...);
order.getItems().add(item);
// 遺漏：item.setOrder(order)
orderRepository.save(order);  // item.order 在資料庫中是 NULL！
```

### ✅ 要：使用輔助方法維護一致性

```java
Order order = new Order(...);
OrderItem item = new OrderItem(...);
order.addItem(item);  // 設定雙方
orderRepository.save(order);
```

**原因：** 雙向關聯需要雙方同步。

---

### ❌ 不要：在 @PrePersist/@PreUpdate 中直接修改 entities

```java
@PrePersist
protected void onCreate() {
    // 不要呼叫 repository 或修改其他 entities！
    otherEntityRepository.save(...);  // 不好！
}
```

### ✅ 要：只設定當前 entity 的欄位

```java
@PrePersist
protected void onCreate() {
    this.createdAt = LocalDateTime.now();
    this.updatedAt = LocalDateTime.now();
}
```

**原因：** 生命週期回調在交易期間執行。Repository 呼叫可能造成死鎖或非預期行為。

---

## JPA 註解快速參考

### Entity 映射

| 註解              | 用途              | 範例                                 |
| ----------------- | ----------------- | ------------------------------------ |
| `@Entity`         | 標記為 JPA entity | 類別層級                             |
| `@Table`          | 資料表名稱映射    | `@Table(name="orders")`              |
| `@Id`             | 主鍵              | 欄位層級                             |
| `@GeneratedValue` | 自動遞增策略      | `@GeneratedValue(strategy=IDENTITY)` |
| `@Column`         | 欄位映射          | `@Column(name="user_id")`            |

### 關聯

| 註解          | 基數     | 範例                           |
| ------------- | -------- | ------------------------------ |
| `@OneToOne`   | 1:1      | User ↔ Profile                 |
| `@OneToMany`  | 1:N      | Order → OrderItems             |
| `@ManyToOne`  | N:1      | OrderItem → Order              |
| `@ManyToMany` | N:M      | Student ↔ Course               |
| `@JoinColumn` | 外鍵欄位 | `@JoinColumn(name="order_id")` |

### 生命週期

| 註解           | 何時執行    | 使用情境       |
| -------------- | ----------- | -------------- |
| `@PrePersist`  | INSERT 之前 | 設定 createdAt |
| `@PostPersist` | INSERT 之後 | 發送事件       |
| `@PreUpdate`   | UPDATE 之前 | 設定 updatedAt |
| `@PostUpdate`  | UPDATE 之後 | 快取失效       |
| `@PreRemove`   | DELETE 之前 | 清理           |
| `@PostRemove`  | DELETE 之後 | 記錄刪除       |
| `@PostLoad`    | SELECT 之後 | 解密欄位       |

---

## 透過面試問題練習

- 📝 [問題 #4：Enum 與 JPA 整合](../../interview/interview-backend.md#4-enum-與-jpa-整合-junior)
- 📝 [問題 #11：Entity Lifecycle Callbacks](../../interview/interview-backend.md#11-entity-lifecycle-callbacks-junior)
- 📝 [問題 #12：@OneToMany Relationship](../../interview/interview-backend.md#12-onetomany-relationship-junior)
- 📝 [問題 #15：BigDecimal 與 Precision](../../interview/interview-backend.md#15-bigdecimal-與-precision-junior)
- 📝 [問題 #26：Bidirectional Relationship](../../interview/interview-backend.md#26-bidirectional-relationship-mid)
- 📝 [問題 #27：CascadeType 與 orphanRemoval](../../interview/interview-backend.md#27-cascadetype-與-orphanremoval-mid)

---

**上一篇：** [← 03. Spring Boot 註解](03-spring-boot-annotations.md)

**下一篇：** [05. Spring Data JPA](05-spring-data-jpa.md) →

# 交易管理 (Transaction Management)

> 為來自 Python/Laravel/Go 背景的中階開發者提供的快速參考
> 📚 相關面試問題: [#16](../../interview/interview-backend.md#16-transactional-註解-mid), [#17](../../interview/interview-backend.md#17-transaction-propagation-mid), [#18](../../interview/interview-backend.md#18-rollback-行為-mid), [#19](../../interview/interview-backend.md#19-scheduled-tasks-mid)

## 從 Python/Laravel/Go 轉換過來

| 概念          | Laravel               | Python/Django                       | Go                 | Spring Boot                     |
| ------------- | --------------------- | ----------------------------------- | ------------------ | ------------------------------- |
| **交易**      | `DB::transaction(fn)` | `@transaction.atomic`               | `tx := db.Begin()` | `@Transactional`                |
| **手動提交**  | `DB::commit()`        | `transaction.commit()`              | `tx.Commit()`      | 自動提交                        |
| **手動回滾**  | `DB::rollback()`      | `transaction.rollback()`            | `tx.Rollback()`    | 拋出例外                        |
| **排程任務**  | Task Scheduler        | Celery Beat                         | cron package       | `@Scheduled`                    |
| **Cron 語法** | Laravel cron          | Celery schedule                     | cron spec          | Spring cron                     |
| **唯讀查詢**  | 無內建                | `transaction.atomic(readonly=True)` | 手動               | `@Transactional(readOnly=true)` |

## 快速語法速查表

### 1. @Transactional 基礎

```java
@Service
public class OrderService {

    @Transactional  // 方法在交易中執行
    public OrderCreationResult createOrder(Long userId, CreateOrderRequest request) {
        // 此方法中的所有資料庫操作都在一個交易中
        Order order = orderRepository.save(new Order(...));
        OrderItem item = orderItemRepository.save(new OrderItem(...));

        // 如果拋出例外,兩個操作都會回滾
        if (someCondition) {
            throw new RuntimeException("Rollback everything!");
        }

        return result;
        // 如果沒有例外,交易在此提交
    }
}
```

**作用:**

- 將方法包裝在資料庫交易中
- 成功時自動提交
- 遇到未檢查例外 (RuntimeException) 時自動回滾
- 遇到已檢查例外時不會回滾 (除非特別設定)

**使用位置:**

- 服務層方法 (不是 Controller 或 Repository)
- 修改多個實體的方法
- 需要原子性 (全有或全無) 的方法

---

### 2. 交易傳播 (Transaction Propagation)

```java
@Transactional(propagation = Propagation.REQUIRED)  // 預設:加入現有交易或建立新交易
public void methodA() {
    methodB();  // 加入此交易
}

@Transactional(propagation = Propagation.REQUIRES_NEW)  // 總是建立新交易
public void methodB() {
    // 在獨立的交易中執行,獨立提交
}

@Transactional(propagation = Propagation.MANDATORY)  // 必須有現有交易
public void methodC() {
    // 如果在沒有活動交易的情況下被呼叫,會拋出例外
}

@Transactional(propagation = Propagation.NEVER)  // 不能有交易
public void methodD() {
    // 如果在交易內被呼叫,會拋出例外
}
```

**傳播類型:**

- `REQUIRED` (預設) - 使用現有交易或建立新交易
- `REQUIRES_NEW` - 總是建立新交易 (暫停當前交易)
- `MANDATORY` - 必須有現有交易
- `SUPPORTS` - 如果有現有交易則使用,否則以非交易方式執行
- `NOT_SUPPORTED` - 以非交易方式執行 (暫停當前交易)
- `NEVER` - 以非交易方式執行 (如果有交易則報錯)
- `NESTED` - 建立巢狀交易 (保存點)

---

### 3. 唯讀交易 (Read-Only Transactions)

```java
@Transactional(readOnly = true)
public JourneyDetailResponse getJourneyDetail(Long journeyId) {
    // 優化:無 flush,無髒檢查
    Journey journey = journeyRepository.findById(journeyId).orElseThrow();
    return mapToResponse(journey);
}
```

**好處:**

- 效能優化 (無 flush 開銷)
- 資料庫可以優化 (不需要鎖定)
- 防止意外寫入
- 明確表達意圖

---

### 4. 回滾規則 (Rollback Rules)

```java
@Transactional(
    rollbackFor = Exception.class,              // 任何 Exception 都回滾
    noRollbackFor = CustomBusinessException.class  // 遇到此例外不回滾
)
public void processPayment() {
    // 自訂回滾行為
}
```

**預設行為:**

- 遇到 `RuntimeException` 和 `Error` 時回滾
- 遇到已檢查例外時不回滾
- 可以用 `rollbackFor` 和 `noRollbackFor` 自訂

---

### 5. 排程任務 (Scheduled Tasks)

```java
@Component
public class OrderExpirationScheduler {

    @Scheduled(cron = "0 0 * * * *")  // 每小時
    public void expireUnpaidOrders() {
        // 排程任務邏輯
    }

    @Scheduled(fixedRate = 60000)  // 每 60 秒
    public void cleanupTask() {
        // 固定頻率任務
    }

    @Scheduled(fixedDelay = 30000)  // 上次完成後 30 秒
    public void monitoringTask() {
        // 固定延遲任務
    }
}
```

**Cron 表達式格式:**

```
┌───────────── 秒 (0-59)
│ ┌───────────── 分 (0-59)
│ │ ┌───────────── 時 (0-23)
│ │ │ ┌───────────── 日 (1-31)
│ │ │ │ ┌───────────── 月 (1-12 或 JAN-DEC)
│ │ │ │ │ ┌───────────── 星期 (0-7 或 MON-SUN, 0 和 7 都是星期日)
│ │ │ │ │ │
* * * * * *
```

**常見模式:**

- `"0 0 * * * *"` - 每小時
- `"0 */15 * * * *"` - 每 15 分鐘
- `"0 0 0 * * *"` - 每天午夜
- `"0 0 9 * * MON-FRI"` - 平日早上 9 點

---

## 實際專案範例

### 範例 1: 使用 @Transactional 確保原子性

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java:77-117`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java#L77-L117)

```java
@Transactional
public OrderCreationResult createOrder(
    @NonNull Long userId, CreateOrderRequest request) {
  logger.debug("Creating order for user: {}", userId);

  orderValidator.validateAndLockUser(userId);
  orderValidator.validateOrderRequest(request);

  Long journeyId = request.items().get(0).journeyId();
  orderValidator.validateJourneyNotPurchased(userId, journeyId);

  // 檢查使用者是否已有此旅程的未付款訂單
  var existingOrder = orderRepository.findByUserIdAndStatusAndJourneyId(
      userId, OrderStatus.UNPAID, journeyId);
  if (existingOrder.isPresent()) {
    return new OrderCreationResult.Existing(mapToOrderResponse(existingOrder.get()));
  }

  // 建立訂單和訂單項目
  Journey journey = orderValidator.validateAndGetJourney(journeyId);
  String orderNumber = OrderNumberGenerator.generate(userId);
  Order order = new Order(orderNumber, userId, journey.getPrice(), BigDecimal.ZERO);

  OrderItem orderItem = new OrderItem(journeyId, 1, journey.getPrice(), BigDecimal.ZERO);
  order.addItem(orderItem);

  order = orderRepository.save(order);  // Order 和 OrderItem 都儲存 (cascade)

  logger.info("Successfully created order {} for user {}", order.getId(), userId);
  return new OrderCreationResult.Created(mapToOrderResponse(order));
}
```

**說明:** 單一交易確保訂單建立的原子性。

**關鍵點:**

- 所有操作在一個交易中 (驗證、查詢、插入)
- 如果任何操作失敗,全部回滾
- 不會有部分狀態 (例如有 Order 但沒有 OrderItem)
- `@Transactional` 在服務層,不在控制器

**交易邊界:**

```
交易開始
├─ 鎖定使用者 (悲觀鎖)
├─ 驗證訂單請求
├─ 檢查現有訂單
├─ 驗證旅程
├─ 產生訂單編號
├─ 儲存訂單 (INSERT orders)
├─ 儲存訂單項目 (INSERT order_items, cascade)
└─ 提交 (如果成功) 或回滾 (如果有例外)
```

---

### 範例 2: 查詢操作使用唯讀交易

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java:37-49`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java#L37-L49)

```java
@Transactional(readOnly = true)
public JourneyListResponse getJourneys() {
  logger.debug("Fetching all journeys");

  List<Journey> journeys = journeyRepository.findAllNotDeleted();

  logger.info("Successfully fetched {} journeys", journeys.size());

  List<JourneyListItemDTO> journeyList =
      journeys.stream().map(this::mapToJourneyListItemDTO).collect(Collectors.toList());

  return new JourneyListResponse(journeyList);
}
```

**說明:** 查詢操作使用唯讀交易。

**關鍵點:**

- `readOnly = true` - 效能優化
- 不需要 flush (Hibernate 不追蹤變更)
- 資料庫可在需要時套用讀取鎖
- 防止意外修改

**效能優勢:**

```java
// 不使用 readOnly
@Transactional
public List<Journey> getJourneys() {
    List<Journey> journeys = repository.findAll();
    // Hibernate 追蹤所有實體以進行髒檢查
    // 提交前 flush 變更
    return journeys;
}

// 使用 readOnly
@Transactional(readOnly = true)
public List<Journey> getJourneys() {
    List<Journey> journeys = repository.findAll();
    // 無髒檢查,無 flush 開銷
    return journeys;
}
```

---

### 範例 3: 帶有悲觀鎖的交易

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java:150-186`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java#L150-L186)

```java
@Transactional
public PayOrderResponse payOrder(Long orderId, Long userId) {
  logger.debug("Processing payment for order {} by user {}", orderId, userId);

  // 以悲觀鎖取得訂單
  Order order = orderRepository.findByIdAndUserIdForUpdate(orderId, userId)
      .orElseThrow(() -> new OrderNotFoundException(orderId));

  orderValidator.validateOrderNotPaid(order);
  orderValidator.validateOrderNotExpired(order);

  // 標記訂單為已付款
  order.markAsPaid();
  order = orderRepository.save(order);

  // 插入購買記錄以授予存取權限
  for (OrderItem item : order.getItems()) {
    UserJourney userJourney = new UserJourney(
        userId, item.getJourneyId(), orderId, order.getPaidAt());
    userJourneyRepository.save(userJourney);
    logger.info("Granted journey {} access to user {}", item.getJourneyId(), userId);
  }

  logger.info("Successfully completed payment for order {}", orderId);
  return new PayOrderResponse(...);
}
```

**說明:** 帶有悲觀鎖的交易防止同時付款。

**關鍵點:**

- `findByIdAndUserIdForUpdate` 取得資料庫鎖 (SELECT FOR UPDATE)
- 其他交易會等待直到此交易提交
- 防止重複付款的競爭條件
- 交易提交時釋放鎖

**並發情境:**

```java
// 使用者快速連按兩次「付款」按鈕

執行緒 1: @Transactional 開始
執行緒 1: SELECT ... FOR UPDATE (取得鎖)
執行緒 2: @Transactional 開始
執行緒 2: SELECT ... FOR UPDATE (等待鎖)
執行緒 1: 驗證、標記已付款、儲存
執行緒 1: 提交 (釋放鎖)
執行緒 2: 取得鎖,讀取訂單 (已經是 PAID)
執行緒 2: 驗證失敗,拋出例外
執行緒 2: 回滾
```

---

### 範例 4: 帶有交易的排程任務

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java:220-240`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java#L220-L240)

```java
/**
 * 排程任務:3 天後讓未付款訂單過期。
 * 每小時執行。
 */
@Scheduled(cron = "0 0 * * * *")
@Transactional
public void expireUnpaidOrders() {
  logger.debug("Running scheduled task to expire unpaid orders");

  LocalDateTime now = LocalDateTime.now();
  List<Order> expiredOrders = orderRepository
      .findByStatusAndExpiredAtBefore(OrderStatus.UNPAID, now);

  for (Order order : expiredOrders) {
    order.markAsExpired();
    orderRepository.save(order);
    logger.info("Expired order: {}", order.getId());
  }

  logger.info("Expired {} unpaid orders", expiredOrders.size());
}
```

**說明:** 排程任務每小時執行以讓未付款訂單過期。

**關鍵點:**

- `@Scheduled(cron = "0 0 * * * *")` - 每小時整點執行
- `@Transactional` - 所有更新在一個交易中
- 查詢 `expired_at < now()` 的訂單
- 更新狀態為 EXPIRED

**Cron 排程:**

```
"0 0 * * * *" 意思是:
 │ │ │ │ │ │
 │ │ │ │ │ └─ 任何星期 (*)
 │ │ │ │ └─── 任何月份 (*)
 │ │ │ └───── 任何日期 (*)
 │ │ └─────── 每小時 (*)
 │ └───────── 分鐘 0
 └─────────── 秒 0

結果: 每天在 00:00, 01:00, 02:00, ..., 23:00 執行
```

**啟用排程:**

```java
@SpringBootApplication
@EnableScheduling  // @Scheduled 需要此設定才能運作
public class WaterballsaBackendApplication {
    public static void main(String[] args) {
        SpringApplication.run(WaterballsaBackendApplication.class, args);
    }
}
```

---

### 範例 5: 實際應用中的交易傳播

**情境:** 訂單建立呼叫多個服務

```java
@Service
public class OrderService {

    @Transactional  // Propagation.REQUIRED (預設)
    public void createOrder(OrderRequest request) {
        Order order = saveOrder(request);  // 加入此交易
        sendEmail(order);                   // 加入此交易
        updateInventory(order);             // 加入此交易
        // 所有操作一起提交
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendEmail(Order order) {
        // 在新交易中執行 (獨立)
        // 如果此操作失敗,訂單建立仍然成功
        emailService.send(order);
    }
    // 即使訂單稍後回滾,電子郵件仍已發送並提交
}
```

**不同傳播方式的使用情境:**

```java
// REQUIRED (預設) - 最常見
@Transactional
public void businessLogic() {
    // 使用現有交易或建立新交易
}

// REQUIRES_NEW - 獨立操作
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String action) {
    // 總是提交,即使父交易回滾
    // 用於:日誌記錄、通知、指標
}

// MANDATORY - 必須在交易內被呼叫
@Transactional(propagation = Propagation.MANDATORY)
public void validateWithinTransaction() {
    // 確保呼叫者有交易上下文
    // 如果沒有活動交易則拋出例外
}

// NOT_SUPPORTED - 不使用交易執行
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void readConfiguration() {
    // 如果有當前交易則暫停
    // 用於:外部 API 呼叫、檔案 I/O
}
```

---

## 常見陷阱

### ❌ 不要:在 Controller 中使用 @Transactional

```java
@RestController
public class OrderController {

    @Transactional  // 錯誤!層級不對
    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(...) {
        // 交易範圍太廣,包含了 HTTP 處理
    }
}
```

### ✅ 要:在 Service 中使用 @Transactional

```java
@Service
public class OrderService {

    @Transactional  // 正確!服務層
    public OrderCreationResult createOrder(...) {
        // 交易只涵蓋業務邏輯
    }
}
```

**原因:** 交易應該在服務層,不是控制器或資料庫層。

---

### ❌ 不要:忘記已檢查例外預設不會回滾

```java
@Transactional
public void processOrder() throws IOException {
    orderRepository.save(order);
    throw new IOException("File error");  // 交易會提交!(不會回滾)
}
```

### ✅ 要:為已檢查例外設定回滾

```java
@Transactional(rollbackFor = Exception.class)
public void processOrder() throws IOException {
    orderRepository.save(order);
    throw new IOException("File error");  // 現在會回滾
}
```

**原因:** 預設只有 RuntimeException 和 Error 會觸發回滾。

---

### ❌ 不要:從同一個類別呼叫 @Transactional 方法 (自我呼叫)

```java
@Service
public class OrderService {

    public void publicMethod() {
        this.transactionalMethod();  // 交易不會啟動!(自我呼叫)
    }

    @Transactional
    private void transactionalMethod() {
        // 從同一個類別呼叫時 @Transactional 會被忽略
    }
}
```

### ✅ 要:從外部類別呼叫或設為 public

```java
@Service
public class OrderService {

    @Transactional
    public void publicTransactionalMethod() {  // Public 且從外部呼叫
        // 交易正常運作
    }
}

// 外部呼叫者
@Service
public class PaymentService {
    private final OrderService orderService;

    public void process() {
        orderService.publicTransactionalMethod();  // 交易啟動
    }
}
```

**原因:** Spring 使用代理來處理 @Transactional。自我呼叫會繞過代理。

---

### ❌ 不要:使用長時間執行的交易

```java
@Transactional
public void processLargeDataset() {
    List<Order> orders = orderRepository.findAll();  // 100,000 筆訂單
    for (Order order : orders) {
        // 處理每筆訂單 (耗時 30 分鐘)
        processOrder(order);
    }
    // 交易持有鎖 30 分鐘!
}
```

### ✅ 要:使用較小交易的批次處理

```java
public void processLargeDataset() {
    int batchSize = 100;
    Pageable pageable = PageRequest.of(0, batchSize);

    Page<Order> page;
    do {
        page = orderRepository.findAll(pageable);
        processBatch(page.getContent());  // 每批次使用獨立交易
        pageable = page.nextPageable();
    } while (page.hasNext());
}

@Transactional
public void processBatch(List<Order> orders) {
    // 每批次使用小交易
    orders.forEach(this::processOrder);
}
```

**原因:** 長交易會鎖定資源並有逾時/死鎖風險。

---

## 交易快速參考

### @Transactional 屬性

| 屬性            | 值                         | 預設                    | 用途               |
| --------------- | -------------------------- | ----------------------- | ------------------ |
| `propagation`   | REQUIRED, REQUIRES_NEW 等  | REQUIRED                | 交易傳播行為       |
| `isolation`     | DEFAULT, READ_COMMITTED 等 | DEFAULT                 | 隔離等級           |
| `timeout`       | 秒數                       | -1 (無限制)             | 交易逾時           |
| `readOnly`      | true/false                 | false                   | 唯讀操作的優化     |
| `rollbackFor`   | Exception 類別             | RuntimeException, Error | 哪些例外會導致回滾 |
| `noRollbackFor` | Exception 類別             | 無                      | 哪些例外不會回滾   |

### 隔離等級

| 等級               | 髒讀    | 不可重複讀取 | 幻讀    |
| ------------------ | ------- | ------------ | ------- |
| `READ_UNCOMMITTED` | ✅ 可能 | ✅ 可能      | ✅ 可能 |
| `READ_COMMITTED`   | ❌ 防止 | ✅ 可能      | ✅ 可能 |
| `REPEATABLE_READ`  | ❌ 防止 | ❌ 防止      | ✅ 可能 |
| `SERIALIZABLE`     | ❌ 防止 | ❌ 防止      | ❌ 防止 |

### 排程任務選項

```java
// Cron 表達式
@Scheduled(cron = "0 0 * * * *")  // 每小時

// 固定頻率 (開始到開始)
@Scheduled(fixedRate = 60000)  // 每 60 秒

// 固定延遲 (結束到開始)
@Scheduled(fixedDelay = 30000)  // 完成後 30 秒

// 初始延遲
@Scheduled(initialDelay = 5000, fixedRate = 60000)  // 5 秒後開始,然後每 60 秒
```

---

## 練習面試問題

- 📝 [問題 #16: @Transactional 註解](../../interview/interview-backend.md#16-transactional-註解-mid)
- 📝 [問題 #17: Transaction Propagation](../../interview/interview-backend.md#17-transaction-propagation-mid)
- 📝 [問題 #18: Rollback 行為](../../interview/interview-backend.md#18-rollback-行為-mid)
- 📝 [問題 #19: Scheduled Tasks](../../interview/interview-backend.md#19-scheduled-tasks-mid)

---

**上一篇:** [← 05. Spring Data JPA](05-spring-data-jpa.md)

**下一篇:** [07. Spring Security & JWT](07-spring-security-jwt.md) →

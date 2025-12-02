# Java 現代語法

> 適合從 Python/Laravel/Go 轉換的中階開發者快速參考
> 📚 相關面試問題: [#1](../../interview/interview-backend.md#1-sealed-interface-語法解析-junior), [#2](../../interview/interview-backend.md#2-pattern-matching-與-instanceof-mid), [#3](../../interview/interview-backend.md#3-stream-api-與-lambda-表達式-junior), [#5](../../interview/interview-backend.md#5-optional-的使用時機-junior)

## 從 Python/Laravel/Go 轉換

| 概念               | Python                    | Laravel/PHP                 | Go                | Java/Spring Boot          |
| ------------------ | ------------------------- | --------------------------- | ----------------- | ------------------------- |
| **不可變資料類別** | `@dataclass(frozen=True)` | Array/stdClass              | `struct`          | `record`                  |
| **型別聯合**       | `Union[TypeA, TypeB]`     | 無內建                      | `interface{}`     | `sealed interface`        |
| **模式匹配**       | `match` (3.10+)           | `match` (8.0+)              | 型別斷言          | `instanceof` with pattern |
| **列表操作**       | List comprehension        | `array_map`, `array_filter` | `for` loops       | Stream API                |
| **Null 安全**      | `Optional` / `None`       | `??`, `?->`                 | `nil` check       | `Optional<T>`             |
| **Lambda**         | `lambda x: x * 2`         | `fn($x) => $x * 2`          | `func(x int) int` | `x -> x * 2`              |

## 快速語法速查表

### 1. Sealed Interfaces

```java
public sealed interface OrderCreationResult {
    OrderResponse orderResponse();

    record Created(OrderResponse orderResponse) implements OrderCreationResult {}
    record Existing(OrderResponse orderResponse) implements OrderCreationResult {}
}
```

**作用:** 限制哪些類別可以實作介面(封閉型別階層)

**使用時機:**

- 具有多種可能結果的回傳型別(成功/錯誤、已建立/已存在)
- 型別安全的狀態機
- 窮舉式模式匹配

**為何使用 sealed:** 編譯器確保你處理所有情況,防止外部實作

---

### 2. Record 類別

```java
public record OrderResponse(
    Long id,
    String orderNumber,
    String status,
    BigDecimal price
) {}
```

**作用:** 建立不可變的資料類別,自動提供:

- 建構函式
- Getter(沒有 `get` 前綴: `orderResponse.id()` 而非 `.getId()`)
- `equals()`、`hashCode()`、`toString()`

**使用時機:** DTOs、API 回應、值物件

**為何不用 class:** 更少樣板程式碼,預設不可變,意圖更清晰

---

### 3. Pattern Matching 與 instanceof

```java
// 舊寫法(Java 16 之前)
if (result instanceof OrderCreationResult.Created) {
    OrderCreationResult.Created created = (OrderCreationResult.Created) result;
    return created.orderResponse();
}

// 新寫法(Java 16+)
if (result instanceof OrderCreationResult.Created created) {
    return created.orderResponse();  // 'created' 已經轉型
}
```

**作用:** 在一行中結合型別檢查 + 轉型 + 變數宣告

**使用時機:** 型別檢查並立即使用轉型後的變數

---

### 4. Stream API

```java
List<MissionDTO> missions = chapter.getMissions().stream()
    .filter(mission -> !mission.isDeleted())
    .sorted(Comparator.comparing(Mission::getOrderIndex))
    .map(this::mapToMissionDTO)
    .collect(Collectors.toList());
```

**作用:** 函數式風格的集合處理(類似 Python 列表推導式)

**常用操作:**

- `filter(predicate)` - 保留符合條件的元素
- `map(function)` - 轉換每個元素
- `sorted(comparator)` - 排序元素
- `collect(collector)` - 轉換為集合

---

### 5. Lambda 表達式

```java
// 語法: (參數) -> 表達式
x -> x * 2                          // 單一參數,無需括號
(x, y) -> x + y                     // 多個參數
() -> System.out.println("Hi")      // 無參數
(x) -> { return x * 2; }           // 區塊主體與明確回傳
```

**使用時機:** 回調函式、Stream 操作、事件處理器

---

### 6. 方法參考

```java
// 實例方法參考
.map(this::mapToMissionDTO)
// 等同於: .map(mission -> this.mapToMissionDTO(mission))

// 靜態方法參考
.map(Mission::getOrderIndex)
// 等同於: .map(mission -> mission.getOrderIndex())

// 建構函式參考
.map(MissionDTO::new)
// 等同於: .map(data -> new MissionDTO(data))
```

**使用時機:** 當 lambda 只呼叫單一方法時

**為何使用:** 更簡潔,意圖更清晰

---

### 7. Optional API

```java
// 建立 Optional
Optional<Order> order = orderRepository.findById(id);

// 使用 Optional
Order result = order
    .orElseThrow(() -> new OrderNotFoundException(id));

// 其他方法
order.isPresent()                    // boolean: 是否有值?
order.orElse(defaultOrder)           // 值或預設值
order.orElseGet(() -> createOrder()) // 值或供應商結果
order.map(Order::getPrice)           // 如果存在則轉換
order.ifPresent(o -> logger.info())  // 如果存在則執行
```

**作用:** 可能為 null 值的容器

**使用時機:** Repository 回傳型別、可選的方法參數

**何時不使用:** Entity/DTO 中的欄位(改用 null)

---

## 實際專案範例

### 範例 1: 型別安全回傳值的 Sealed Interface

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java:56-65`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/OrderService.java#L56-L65)

```java
public sealed interface OrderCreationResult {
  OrderResponse orderResponse();

  /** 成功建立新訂單。 */
  record Created(OrderResponse orderResponse) implements OrderCreationResult {}

  /** 回傳現有未付款訂單(冪等行為)。 */
  record Existing(OrderResponse orderResponse) implements OrderCreationResult {}
}
```

**說明:** `OrderService.createOrder()` 可以回傳新建立的訂單或現有的未付款訂單。使用 sealed interface 使其型別安全。

**重點:**

- `sealed` 確保只有 `Created` 和 `Existing` 可以實作此介面
- 兩者都是 `record` 類別(不可變、最少樣板)
- 每個都有單一欄位 `orderResponse` - 實際的訂單資料
- JavaDoc 解釋 Created 和 Existing 之間的語義差異

**為何採用此設計:**

- 型別安全:無法忘記處理某個情況
- 自我文件化:回傳型別清楚顯示兩種可能性
- 編譯器檢查:模式匹配確保窮盡性

---

### 範例 2: Controller 中的 Pattern Matching

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java:43-49`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java#L43-L49)

```java
var result = orderService.createOrder(currentUserId, request);

if (result instanceof OrderService.OrderCreationResult.Created created) {
  logger.info("Successfully created order for user {}", currentUserId);
  return ResponseEntity.status(HttpStatus.CREATED).body(created.orderResponse());
} else if (result instanceof OrderService.OrderCreationResult.Existing existing) {
  logger.info("Successfully returned existing unpaid order for user {}", currentUserId);
  return ResponseEntity.ok(existing.orderResponse());
}
```

**說明:** Controller 使用模式匹配來處理 sealed interface 回傳型別並回傳適當的 HTTP 狀態碼。

**重點:**

- `Created` 情況 → `201 CREATED` 狀態
- `Existing` 情況 → `200 OK` 狀態(冪等行為)
- `created` 和 `existing` 變數自動轉型並限定作用域
- 不需要手動型別轉換

**為何使用不同狀態碼:**

- 201: 建立了新資源
- 200: 請求成功但沒有新資源(回傳現有的)

---

### 範例 3: 集合處理的 Stream API

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java:94-99`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java#L94-L99)

```java
List<MissionSummaryDTO> missions =
    chapter.getMissions().stream()
        .filter(mission -> !mission.isDeleted())
        .sorted(Comparator.comparing(Mission::getOrderIndex))
        .map(this::mapToMissionSummaryDTO)
        .collect(Collectors.toList());
```

**說明:** 將 Mission entities 列表轉換為 MissionSummaryDTO 物件,過濾掉已刪除的並按順序索引排序。

**重點:**

- `.filter()` - 移除軟刪除的任務
- `.sorted()` - 按 `orderIndex` 欄位排序
- `.map()` - 轉換 Mission → MissionSummaryDTO
- `.collect()` - 將 stream 轉回 List

**Python 等價寫法:**

```python
missions = [
    self.map_to_mission_summary_dto(mission)
    for mission in sorted(
        [m for m in chapter.get_missions() if not m.is_deleted()],
        key=lambda m: m.order_index
    )
]
```

**Laravel 等價寫法:**

```php
$missions = $chapter->missions
    ->filter(fn($m) => !$m->is_deleted)
    ->sortBy('order_index')
    ->map(fn($m) => $this->mapToMissionSummaryDTO($m))
    ->values()
    ->all();
```

---

### 範例 4: DTOs 的 Record 類別

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/dto/OrderResponse.java:6-27`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/dto/OrderResponse.java#L6-L27)

```java
public record OrderResponse(
    Long id,
    String orderNumber,
    Long userId,
    String username,
    String status,
    BigDecimal originalPrice,
    BigDecimal discount,
    BigDecimal price,
    List<OrderItemResponse> items,
    Long createdAt,
    Long paidAt,
    Long expiredAt) {

  public record OrderItemResponse(
      Long journeyId,
      String journeyTitle,
      Integer quantity,
      BigDecimal originalPrice,
      BigDecimal discount,
      BigDecimal price) {}
}
```

**說明:** 使用 record 類別的 API 回應 DTO。注意巢狀的 record `OrderItemResponse`。

**重點:**

- 預設不可變(所有欄位都是 `final`)
- 自動 getter: `orderResponse.id()`,而非 `.getId()`
- 允許巢狀 record
- 非常適合 JSON 序列化/反序列化

**免費獲得的功能:**

```java
// 自動建構函式
new OrderResponse(1L, "ORD123", 1L, "user", ...);

// 自動 getter(無 'get' 前綴)
Long id = orderResponse.id();

// 自動 equals、hashCode、toString
orderResponse.equals(anotherOrder);
```

**Python 等價寫法:**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class OrderResponse:
    id: int
    order_number: str
    # ...
```

---

### 範例 5: Repository 中的 Optional

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java:35`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/repository/OrderRepository.java#L35)

```java
Optional<Order> findByIdAndUserId(Long id, Long userId);
```

**說明:** Repository 方法回傳 `Optional<T>` 以明確指示值可能不存在。

**在 Service 層的使用:**

```java
Order order = orderRepository.findByIdAndUserId(orderId, userId)
    .orElseThrow(() -> new OrderNotFoundException(orderId));
```

**重點:**

- 明確:回傳型別顯示值可能不存在
- 型別安全:無法忘記處理 null 情況
- 流暢的 API:鏈接方法如 `map`、`filter`、`orElse`

**常用模式:**

```java
// 如果找不到則拋出例外
.orElseThrow(() -> new NotFoundException())

// 回傳預設值
.orElse(defaultValue)

// 回傳延遲計算的預設值
.orElseGet(() -> createDefault())

// 僅在存在時執行
.ifPresent(order -> logger.info("Found: {}", order))

// 如果存在則轉換
.map(Order::getPrice)
```

---

### 範例 6: 方法參考 vs Lambda

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java:46`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java#L46)

```java
// 方法參考(較佳)
journeys.stream()
    .map(this::mapToJourneyListItemDTO)
    .collect(Collectors.toList());

// 等價的 lambda(較冗長)
journeys.stream()
    .map(journey -> this.mapToJourneyListItemDTO(journey))
    .collect(Collectors.toList());
```

**何時使用方法參考:** Lambda 主體只呼叫單一方法

**何時使用 lambda:** 需要額外邏輯或多個陳述式

---

### 範例 7: 使用方法參考的 Comparator

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java:79`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/service/JourneyService.java#L79)

```java
.sorted(Comparator.comparing(Chapter::getOrderIndex))
```

**說明:** 按 chapters 的 `orderIndex` 欄位排序。

**其他 Comparator 模式:**

```java
// 反向排序
.sorted(Comparator.comparing(Chapter::getOrderIndex).reversed())

// 多個欄位
.sorted(Comparator.comparing(Chapter::getOrderIndex)
                  .thenComparing(Chapter::getTitle))

// Null 安全
.sorted(Comparator.comparing(Chapter::getOrderIndex,
                             Comparator.nullsLast(Comparator.naturalOrder())))
```

---

## 常見陷阱

### ❌ 別這樣: 將 Optional 用作欄位型別

```java
public class Order {
    private Optional<String> description;  // 不好!
}
```

### ✅ 這樣做: 僅將 Optional 用作回傳型別

```java
public class Order {
    private String description;  // 可以是 null

    public Optional<String> getDescription() {
        return Optional.ofNullable(description);
    }
}
```

**原因:** Optional 設計用於回傳型別,而非欄位。它增加開銷且不可序列化。

---

### ❌ 別這樣: 不檢查就呼叫 get()

```java
Optional<Order> orderOpt = repository.findById(id);
Order order = orderOpt.get();  // 不好! 如果為空會拋出 NoSuchElementException
```

### ✅ 這樣做: 使用 orElseThrow 或其他安全方法

```java
Order order = repository.findById(id)
    .orElseThrow(() -> new OrderNotFoundException(id));
```

**原因:** `.get()` 違背了 Optional 的目的。總是使用安全方法。

---

### ❌ 別這樣: 在串流時修改集合

```java
List<Mission> missions = chapter.getMissions();
missions.stream()
    .forEach(mission -> missions.remove(mission));  // 不好! ConcurrentModificationException
```

### ✅ 這樣做: 收集到新列表

```java
List<Mission> activeMissions = chapter.getMissions().stream()
    .filter(mission -> !mission.isDeleted())
    .collect(Collectors.toList());
```

**原因:** Stream 應該無副作用。修改來源集合會導致例外。

---

### ❌ 別這樣: 簡單操作過度使用 stream

```java
// 對簡單操作來說太複雜
list.stream().forEach(System.out::println);
```

### ✅ 這樣做: 簡單時使用傳統迴圈

```java
// 對於簡單迭代更易讀
for (var item : list) {
    System.out.println(item);
}
```

**原因:** Stream 有開銷。在它們增加價值時使用(filter、map、reduce)。

---

### ❌ 別這樣: 嘗試修改 record 欄位

```java
public record OrderResponse(Long id, String status) {}

OrderResponse order = new OrderResponse(1L, "PAID");
// order.id = 2L;  // 編譯錯誤! Record 是不可變的
```

### ✅ 這樣做: 建立具有不同值的新實例

```java
OrderResponse updatedOrder = new OrderResponse(order.id(), "EXPIRED");
```

**原因:** Record 設計上是不可變的。如果需要,使用建造者模式或複製方法。

---

## Stream API 快速參考

### 終端操作(觸發處理)

| 操作          | 描述               | 範例                            |
| ------------- | ------------------ | ------------------------------- |
| `collect()`   | 轉換為集合         | `.collect(Collectors.toList())` |
| `forEach()`   | 對每個元素執行     | `.forEach(System.out::println)` |
| `count()`     | 計數元素           | `.count()`                      |
| `findFirst()` | 取得第一個元素     | `.findFirst().orElse(null)`     |
| `anyMatch()`  | 檢查是否有任何匹配 | `.anyMatch(m -> m.isDeleted())` |
| `allMatch()`  | 檢查是否全部匹配   | `.allMatch(m -> m.isValid())`   |
| `reduce()`    | 合併元素           | `.reduce(0, Integer::sum)`      |

### 中間操作(惰性、可鏈接)

| 操作         | 描述           | 範例                                 |
| ------------ | -------------- | ------------------------------------ |
| `filter()`   | 保留匹配的元素 | `.filter(m -> !m.isDeleted())`       |
| `map()`      | 轉換元素       | `.map(Mission::getTitle)`            |
| `sorted()`   | 排序元素       | `.sorted(Comparator.comparing(...))` |
| `distinct()` | 移除重複       | `.distinct()`                        |
| `limit()`    | 取前 N 個      | `.limit(10)`                         |
| `skip()`     | 跳過前 N 個    | `.skip(5)`                           |

---

## 面試問題練習

- 📝 [問題 #1: Sealed Interface 語法解析](../../interview/interview-backend.md#1-sealed-interface-語法解析-junior)
- 📝 [問題 #2: Pattern Matching 與 instanceof](../../interview/interview-backend.md#2-pattern-matching-與-instanceof-mid)
- 📝 [問題 #3: Stream API 與 Lambda 表達式](../../interview/interview-backend.md#3-stream-api-與-lambda-表達式-junior)
- 📝 [問題 #5: Optional 的使用時機](../../interview/interview-backend.md#5-optional-的使用時機-junior)

---

**下一步:** [03. Spring Boot 註解](03-spring-boot-annotations.md) →

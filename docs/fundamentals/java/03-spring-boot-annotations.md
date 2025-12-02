# Spring Boot 註解

> 給從 Python/Laravel/Go 轉過來的中階開發者的快速參考
> 📚 相關面試問題：[#6](../../interview/interview-backend.md#6-controller-層的職責-junior), [#7](../../interview/interview-backend.md#7-依賴注入的方式-junior), [#9](../../interview/interview-backend.md#9-bean-validation-註解-junior), [#10](../../interview/interview-backend.md#10-logger-使用-junior)

## 從 Python/Laravel/Go 轉過來

| 概念          | Laravel                                  | Python/Django                    | Go                    | Spring Boot                     |
| ------------- | ---------------------------------------- | -------------------------------- | --------------------- | ------------------------------- |
| **路由定義**  | `Route::post('/auth')` in routes/web.php | `path('auth/', view)` in urls.py | `http.HandleFunc()`   | `@PostMapping("/auth")`         |
| **依賴注入**  | Constructor injection                    | `@inject` decorator              | Manual initialization | `@Autowired` or constructor     |
| **請求驗證**  | Form Requests                            | Serializers                      | validator package     | `@Valid` + Bean Validation      |
| **JSON 回應** | `return response()->json()`              | `return JsonResponse()`          | `json.NewEncoder()`   | Return object (auto-serialized) |
| **服務層**    | Service classes                          | Services/Managers                | Service structs       | `@Service` classes              |
| **日誌記錄**  | `Log::info()`                            | `logging.info()`                 | `log.Println()`       | `logger.info()` (SLF4J)         |

## 快速語法速查表

### 1. Controller 註解

```java
@RestController                    // 標記類別為 REST controller (@Controller + @ResponseBody)
@RequestMapping("/orders")         // 此 controller 中所有方法的基礎路徑
public class OrderController {

    @PostMapping                   // POST /orders
    @GetMapping("/{id}")          // GET /orders/{id}
    @PutMapping("/{id}")          // PUT /orders/{id}
    @DeleteMapping("/{id}")       // DELETE /orders/{id}
    @PatchMapping("/{id}")        // PATCH /orders/{id}
}
```

**@RestController vs @Controller:**

- `@RestController` = `@Controller` + `@ResponseBody`
- 使用 `@RestController` 於 REST API（回傳 JSON）
- 使用 `@Controller` 於傳統 MVC（回傳視圖）

---

### 2. 請求參數註解

```java
@GetMapping("/users/{userId}/orders")
public OrderListResponse getOrders(
    @PathVariable Long userId,              // 從 URL 路徑取得：/users/123/orders
    @RequestParam(defaultValue = "1") int page,  // 從查詢字串取得：?page=2
    @RequestParam(defaultValue = "20") int limit // 從查詢字串取得：&limit=50
) { }

@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
    @RequestBody @Valid CreateOrderRequest request  // 從 JSON body 取得
) { }
```

**@PathVariable:** 從 URL 路徑中提取值（例如 `/users/{id}`）
**@RequestParam:** 從查詢字串中提取值（例如 `?page=1&limit=20`）
**@RequestBody:** 提取並反序列化請求主體（JSON → Java 物件）

---

### 3. 依賴注入（建構子注入 - 建議使用）

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;  // final = 不可變

    // 建構子注入（現代 Spring Boot 不需要 @Autowired）
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

**為什麼使用建構子注入？**

- 不可變的依賴（可以標記為 `final`）
- 容易測試（在單元測試中可以傳入 mock）
- 明確的依賴（在建構子簽名中可見）
- 空值安全（如果缺少依賴，Spring 會快速失敗）

**替代方案：欄位注入（不建議使用）**

```java
@Autowired  // 不要這樣做！
private OrderService orderService;
```

---

### 4. Bean Validation 註解

```java
public record RegisterRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$")
    String username,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 72)
    String password
) {}
```

**常見的驗證註解：**

- `@NotNull` - 值不可為 null
- `@NotBlank` - 字串不可為 null、空字串或只有空白
- `@NotEmpty` - 集合/陣列不可為空
- `@Size(min=x, max=y)` - 字串/集合大小限制
- `@Min(value)` / `@Max(value)` - 數值限制
- `@Pattern(regexp)` - 正則表達式驗證
- `@Email` - 電子郵件格式驗證

**觸發驗證：** 在 controller 中使用 `@Valid` 或 `@Validated`

---

### 5. Service 層註解

```java
@Service  // 標記為 Spring 管理的 service bean
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional  // 將在交易管理指南中介紹
    public OrderCreationResult createOrder(Long userId, CreateOrderRequest request) {
        // 業務邏輯在這裡
    }
}
```

**@Service:**

- 語義化註解（技術上與 `@Component` 相同）
- 表示此類別包含業務邏輯
- Spring 自動建立並管理 bean 生命週期

---

### 6. Logger 設定（SLF4J）

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestController
public class OrderController {

    private static final Logger logger = LoggerFactory.getLogger(OrderController.class);

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(...) {
        logger.debug("POST /orders request received");  // DEBUG 層級
        logger.info("Successfully created order");      // INFO 層級
        logger.warn("Rate limit exceeded");             // WARN 層級
        logger.error("Unexpected error", exception);     // ERROR 層級
    }
}
```

**日誌層級（從低到高）：**

1. TRACE - 非常詳細
2. DEBUG - 開發資訊
3. INFO - 重要事件
4. WARN - 警告情況
5. ERROR - 錯誤情況

**參數化日誌記錄（建議使用）：**

```java
logger.info("Created order {} for user {}", orderId, userId);
// 優於：logger.info("Created order " + orderId + " for user " + userId);
```

---

### 7. 使用 ResponseEntity 回傳 HTTP 回應

```java
// 回傳 201 CREATED 含 body
return ResponseEntity.status(HttpStatus.CREATED).body(orderResponse);

// 回傳 200 OK 含 body
return ResponseEntity.ok(orderResponse);

// 回傳 204 NO CONTENT（無 body）
return ResponseEntity.noContent().build();

// 回傳 404 NOT FOUND 含 body
return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse);
```

---

## 實際專案範例

### 範例 1：使用建構子注入的 Controller

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java:18-28`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java#L18-L28)

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

  private static final Logger logger = LoggerFactory.getLogger(OrderController.class);

  private final OrderService orderService;

  public OrderController(OrderService orderService) {
    this.orderService = orderService;
  }
}
```

**說明：** 標準的 controller 設定，使用建構子注入和 logger。

**重點：**

- `@RestController` - 自動回傳 JSON（不需要 `@ResponseBody`）
- `@RequestMapping("/orders")` - 所有方法都以 `/orders` 路徑開始
- 建構子注入 - 不需要 `@Autowired` 註解
- `final` 關鍵字 - 確保 orderService 在建構後不可變
- Logger 是 `static final` - 每個類別一個 logger 實例

---

### 範例 2：使用 @Valid 進行請求驗證

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java:36-44`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java#L36-L44)

```java
@PostMapping("/register")
public ResponseEntity<RegisterResponse> register(@Valid @RequestBody RegisterRequest request) {
    logger.debug("Register request received for username: {}", request.username());

    RegisterResponse response = authService.register(request);

    logger.info("User registration successful: {}", response.userId());

    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

**說明：** `@Valid` 註解會使用 Bean Validation 註解來觸發 `RegisterRequest` 的驗證。

**重點：**

- `@Valid` - 在方法執行前觸發驗證
- `@RequestBody` - 將 JSON 反序列化為 RegisterRequest
- 如果驗證失敗，Spring 會自動回傳 400 Bad Request
- `ResponseEntity.status(HttpStatus.CREATED)` - 回傳 201 狀態碼
- 參數化日誌記錄 - 使用 `{}` 佔位符來記錄 username 和 userId

**驗證失敗時會發生什麼：**

1. Spring 捕獲 `MethodArgumentNotValidException`
2. GlobalExceptionHandler 處理它（參見錯誤處理指南）
3. 回傳 400 Bad Request 及錯誤詳情

---

### 範例 3：DTO 中的 Bean Validation

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/dto/RegisterRequest.java:7-21`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/dto/RegisterRequest.java#L7-L21)

```java
public record RegisterRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Pattern(
        regexp = "^[a-zA-Z0-9_]+$",
        message = "Username must contain only alphanumeric characters and underscores")
    String username,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 72, message = "Password must be between 8 and 72 characters")
    @Pattern(
        regexp = "^[a-zA-Z0-9@$!%*?&#]+$",
        message = "Password must contain only alphanumeric characters and special characters...")
    String password
) {}
```

**說明：** 驗證規則直接定義在 record 欄位上。

**重點：**

- `@NotBlank` - 拒絕 null、空字串或只有空白
- `@Size` - 強制長度限制
- `@Pattern` - 正則表達式驗證允許的字元
- 自訂訊息 - 在錯誤回應中回傳
- 宣告式驗證 - 不需要手動 if/else 檢查

**驗證流程：**

```
Client POST → controller 中的 @Valid → 驗證引擎檢查註解
           → 如果無效：400 Bad Request 及錯誤訊息
           → 如果有效：執行 controller 方法
```

**Laravel 對應寫法：**

```php
// 在 FormRequest 中
public function rules() {
    return [
        'username' => 'required|min:3|max:50|regex:/^[a-zA-Z0-9_]+$/',
        'password' => 'required|min:8|max:72',
    ];
}
```

---

### 範例 4：Logger 使用模式

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java:47-73`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java#L47-L73)

```java
@PostMapping("/login")
public ResponseEntity<LoginResponse> login(
    @Valid @RequestBody LoginRequest request, HttpServletRequest httpRequest) {
  String ipAddress = getClientIp(httpRequest);
  logger.debug("Login request received for username: {} from IP: {}",
               request.username(), ipAddress);

  if (!rateLimitService.tryConsume(ipAddress)) {
    logger.warn("Rate limit exceeded for IP: {}", ipAddress);
    throw new InvalidCredentialsException();
  }

  try {
    LoginResponse response = authService.login(request);
    rateLimitService.reset(ipAddress);
    logger.info("User login successful for username: {}", request.username());
    return ResponseEntity.ok(response);
  } catch (InvalidCredentialsException e) {
    throw e;
  }
}
```

**說明：** 針對不同情境使用不同的日誌層級。

**重點：**

- `logger.debug()` - 請求詳情（生產環境中停用）
- `logger.warn()` - 速率限制違規（潛在的安全問題）
- `logger.info()` - 成功的操作（稽核軌跡）
- 參數化日誌記錄 - `{}` 避免字串串接的開銷
- IP 位址記錄 - 對安全監控很有用

**最佳實踐：**

- DEBUG：請求/回應詳情、方法進入/退出
- INFO：業務事件（使用者註冊、訂單建立）
- WARN：可恢復的錯誤、安全事件
- ERROR：不可恢復的錯誤及堆疊追蹤

---

### 範例 5：請求映射的變體

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java:36-52`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java#L36-L52)

```java
@PostMapping  // POST /orders
public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
    // ...
}

@GetMapping("/{orderId}")  // GET /orders/{orderId}
public ResponseEntity<OrderResponse> getOrder(@PathVariable Long orderId) {
    // ...
}

@PostMapping("/{orderId}/action/pay")  // POST /orders/{orderId}/action/pay
public ResponseEntity<PayOrderResponse> payOrder(@PathVariable Long orderId) {
    // ...
}
```

**說明：** 不同的 HTTP 方法和路徑模式。

**重點：**

- `@PostMapping` 繼承類別層級的 `@RequestMapping` 中的 `/orders`
- `@GetMapping("/{orderId}")` → 完整路徑是 `/orders/{orderId}`
- `@PathVariable` 從 URL 中提取 `{orderId}`
- 動作端點 - RESTful 方式處理操作：`/orders/{id}/action/pay`

**RESTful 路由模式：**

- `POST /orders` - 建立訂單
- `GET /orders/{id}` - 取得訂單詳情
- `GET /orders` - 列出訂單
- `PUT /orders/{id}` - 更新訂單（完整替換）
- `PATCH /orders/{id}` - 更新訂單（部分更新）
- `DELETE /orders/{id}` - 刪除訂單
- `POST /orders/{id}/action/pay` - 對訂單執行動作

---

### 範例 6：多個依賴

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java:28-34`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/AuthController.java#L28-L34)

```java
private final AuthService authService;
private final RateLimitService rateLimitService;

public AuthController(AuthService authService, RateLimitService rateLimitService) {
  this.authService = authService;
  this.rateLimitService = rateLimitService;
}
```

**說明：** 具有多個 service 依賴的 controller。

**重點：**

- 兩個 service 都透過建構子注入
- 所有依賴都是 `final`（不可變）
- 不需要 `@Autowired` 註解（現代 Spring Boot）
- 順序不重要 - Spring 會自動解析依賴

**測試優勢：**

```java
// 在單元測試中容易 mock
AuthService mockAuthService = mock(AuthService.class);
RateLimitService mockRateLimitService = mock(RateLimitService.class);
AuthController controller = new AuthController(mockAuthService, mockRateLimitService);
```

---

## 常見陷阱

### ❌ 不要：使用欄位注入

```java
@RestController
public class OrderController {
    @Autowired  // 不好！難以測試，允許 null
    private OrderService orderService;
}
```

### ✅ 要：使用建構子注入

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

**原因：** 建構子注入可測試、空值安全，並且使依賴明確。

---

### ❌ 不要：忘記使用 @Valid 進行驗證

```java
@PostMapping("/register")
public ResponseEntity<RegisterResponse> register(@RequestBody RegisterRequest request) {
    // RegisterRequest 中的驗證註解會被忽略！
}
```

### ✅ 要：使用 @Valid 觸發驗證

```java
@PostMapping("/register")
public ResponseEntity<RegisterResponse> register(@Valid @RequestBody RegisterRequest request) {
    // 方法執行前會執行驗證
}
```

**原因：** 沒有 `@Valid`，Bean Validation 註解不會被處理。

---

### ❌ 不要：在日誌中使用字串串接

```java
logger.info("Created order " + orderId + " for user " + userId);  // 不好！
```

### ✅ 要：使用參數化日誌記錄

```java
logger.info("Created order {} for user {}", orderId, userId);
```

**原因：** 參數化日誌記錄在日誌停用時可避免字串串接的開銷。

---

### ❌ 不要：混用 @PathVariable 和 @RequestParam

```java
// 令人困惑：id 是從路徑還是查詢字串來的？
@GetMapping("/orders")
public Order getOrder(@PathVariable Long id) { }  // 錯誤！
```

### ✅ 要：根據來源使用正確的註解

```java
@GetMapping("/orders/{id}")  // id 來自路徑
public Order getOrder(@PathVariable Long id) { }

@GetMapping("/orders")  // id 來自查詢字串 (?id=123)
public Order getOrder(@RequestParam Long id) { }
```

**原因：** `@PathVariable` 用於 URL 路徑段，`@RequestParam` 用於查詢參數。

---

### ❌ 不要：在 controller 中放置業務邏輯

```java
@RestController
public class OrderController {
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(...) {
        // 計算價格、檢查庫存、發送郵件... 不好！
    }
}
```

### ✅ 要：保持 controller 簡潔，業務邏輯放在 service 中

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(...) {
        var result = orderService.createOrder(userId, request);  // 委派給 service
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }
}
```

**原因：** Controller 應該只處理 HTTP 相關事項（請求/回應）。業務邏輯屬於 service。

---

## 註解快速參考

### Controller 層

| 註解              | 用途                       | 範例                            |
| ----------------- | -------------------------- | ------------------------------- |
| `@RestController` | REST API controller        | 類別層級                        |
| `@Controller`     | MVC controller（回傳視圖） | 類別層級                        |
| `@RequestMapping` | 所有方法的基礎路徑         | `@RequestMapping("/api")`       |
| `@GetMapping`     | GET 端點                   | `@GetMapping("/users")`         |
| `@PostMapping`    | POST 端點                  | `@PostMapping("/users")`        |
| `@PutMapping`     | PUT 端點                   | `@PutMapping("/users/{id}")`    |
| `@PatchMapping`   | PATCH 端點                 | `@PatchMapping("/users/{id}")`  |
| `@DeleteMapping`  | DELETE 端點                | `@DeleteMapping("/users/{id}")` |

### 請求參數

| 註解             | 來源             | 範例                                           |
| ---------------- | ---------------- | ---------------------------------------------- |
| `@PathVariable`  | URL 路徑         | `/users/{id}` → `@PathVariable Long id`        |
| `@RequestParam`  | 查詢字串         | `?page=1` → `@RequestParam int page`           |
| `@RequestBody`   | 請求主體（JSON） | `@RequestBody CreateOrderRequest req`          |
| `@RequestHeader` | HTTP 標頭        | `@RequestHeader("Authorization") String token` |

### 驗證

| 註解            | 限制                    | 範例                                      |
| --------------- | ----------------------- | ----------------------------------------- |
| `@Valid`        | 觸發驗證                | `@Valid @RequestBody RegisterRequest req` |
| `@NotNull`      | 值不可為 null           | `@NotNull String name`                    |
| `@NotBlank`     | 字串不可為 null/空/空白 | `@NotBlank String username`               |
| `@NotEmpty`     | 集合/陣列不可為空       | `@NotEmpty List<String> items`            |
| `@Size`         | 大小限制                | `@Size(min=3, max=50) String username`    |
| `@Min` / `@Max` | 數值最小/最大值         | `@Min(1) int quantity`                    |
| `@Pattern`      | 正則表達式驗證          | `@Pattern(regexp="^[A-Z]+$")`             |
| `@Email`        | 電子郵件格式            | `@Email String email`                     |

### 元件註冊

| 註解              | 用途                | 典型用途       |
| ----------------- | ------------------- | -------------- |
| `@Component`      | 通用 Spring bean    | 工具類別       |
| `@Service`        | Service 層 bean     | 業務邏輯       |
| `@Repository`     | DAO/repository bean | 資料存取       |
| `@Controller`     | MVC controller      | Web controller |
| `@RestController` | REST controller     | REST API       |
| `@Configuration`  | 配置類別            | Java 配置      |

---

## 練習面試問題

- 📝 [問題 #6：Controller 層的職責](../../interview/interview-backend.md#6-controller-層的職責-junior)
- 📝 [問題 #7：依賴注入的方式](../../interview/interview-backend.md#7-依賴注入的方式-junior)
- 📝 [問題 #9：Bean Validation 註解](../../interview/interview-backend.md#9-bean-validation-註解-junior)
- 📝 [問題 #10：Logger 使用](../../interview/interview-backend.md#10-logger-使用-junior)

---

**上一篇：** [← 02. Java Modern Syntax](02-java-modern-syntax.md)

**下一篇：** [04. JPA & Hibernate](04-jpa-hibernate.md) →

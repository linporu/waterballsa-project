# 錯誤處理

> 給來自 Python/Laravel/Go 的中階開發者的快速參考
> 📚 相關面試題目：[#31](../../interview/interview-backend.md#31-global-exception-handler-mid), [#32](../../interview/interview-backend.md#32-custom-exceptions-mid)

## 來自 Python/Laravel/Go 的開發者

| 概念               | Laravel               | Python/Django         | Go                   | Spring Boot                       |
| ------------------ | --------------------- | --------------------- | -------------------- | --------------------------------- |
| **全域例外處理器** | `Handler.php`         | Exception middleware  | Custom middleware    | `@RestControllerAdvice`           |
| **處理特定例外**   | `render()` method     | `exception_handler()` | Type switch          | `@ExceptionHandler`               |
| **自訂例外**       | Extend `Exception`    | Extend `Exception`    | Custom `error` type  | Extend `RuntimeException`         |
| **驗證錯誤**       | `ValidationException` | `ValidationError`     | Validator errors     | `MethodArgumentNotValidException` |
| **錯誤回應**       | `response()->json()`  | `JsonResponse`        | `json.Marshal()`     | `ResponseEntity<ErrorResponse>`   |
| **HTTP 狀態碼**    | `abort(404)`          | `raise Http404`       | `w.WriteHeader(404)` | `HttpStatus.NOT_FOUND`            |

## 語法速查表

### 1. 全域例外處理器

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
        logger.warn("Order not found error: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("訂單不存在"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneralException(Exception ex) {
        logger.error("Unexpected error occurred: ", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("系統發生錯誤，請稍後再試"));
    }
}
```

**@RestControllerAdvice：**

- 為所有控制器提供全域例外處理
- 自動將回傳值序列化為 JSON
- 捕捉任何 `@RestController` 拋出的例外

---

### 2. 自訂例外類別

```java
// 基礎例外
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long orderId) {
        super("Order not found: " + orderId);
    }
}

// 另一個自訂例外
public class OrderAlreadyPaidException extends RuntimeException {
    public OrderAlreadyPaidException(Long orderId) {
        super("Order already paid: " + orderId);
    }
}
```

**為什麼要繼承 RuntimeException：**

- 非檢查例外（不需要 `throws` 宣告）
- 自動觸發 `@Transactional` 回滾
- 程式碼更簡潔（不會有混亂的 try-catch）

---

### 3. 錯誤回應 DTO

```java
public record ErrorResponse(String error) {}

// 使用方式：
return ResponseEntity
    .status(HttpStatus.BAD_REQUEST)
    .body(new ErrorResponse("輸入資料格式錯誤"));
```

---

### 4. 驗證錯誤處理

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationErrors(
    MethodArgumentNotValidException ex) {

    Map<String, String> errors = new HashMap<>();
    ex.getBindingResult().getAllErrors().forEach(error -> {
        String fieldName = ((FieldError) error).getField();
        String errorMessage = error.getDefaultMessage();
        errors.put(fieldName, errorMessage);
    });

    logger.warn("Validation errors: {}", errors);
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse("資料驗證失敗，請檢查輸入內容"));
}
```

---

### 5. HTTP 狀態碼選擇

```java
// 400 Bad Request - 客戶端錯誤，無效輸入
throw new InvalidInputException("Invalid order request");
→ ResponseEntity.status(HttpStatus.BAD_REQUEST)

// 401 Unauthorized - 需要身份驗證
throw new UnauthorizedException("Token expired");
→ ResponseEntity.status(HttpStatus.UNAUTHORIZED)

// 403 Forbidden - 已驗證但無權限
throw new ForbiddenException("Access denied");
→ ResponseEntity.status(HttpStatus.FORBIDDEN)

// 404 Not Found - 資源不存在
throw new OrderNotFoundException(orderId);
→ ResponseEntity.status(HttpStatus.NOT_FOUND)

// 409 Conflict - 違反業務規則
throw new OrderAlreadyPaidException(orderId);
→ ResponseEntity.status(HttpStatus.CONFLICT)

// 500 Internal Server Error - 非預期的伺服器錯誤
throw new Exception("Database connection failed");
→ ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
```

---

## 實際專案範例

### 範例 1：完整的全域例外處理器

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/exception/GlobalExceptionHandler.java:15-156`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/exception/GlobalExceptionHandler.java#L15-L156)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

  private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

  @ExceptionHandler(DuplicateUsernameException.class)
  public ResponseEntity<ErrorResponse> handleDuplicateUsername(DuplicateUsernameException ex) {
    logger.warn("Duplicate username error: {}", ex.getMessage());
    return ResponseEntity.status(HttpStatus.CONFLICT).body(new ErrorResponse("使用者名稱已存在"));
  }

  @ExceptionHandler(InvalidCredentialsException.class)
  public ResponseEntity<ErrorResponse> handleInvalidCredentials(InvalidCredentialsException ex) {
    logger.warn("Invalid credentials error: {}", ex.getMessage());
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(new ErrorResponse("帳號或密碼錯誤"));
  }

  @ExceptionHandler(OrderNotFoundException.class)
  public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
    logger.warn("Order not found error: {}", ex.getMessage());
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse("訂單不存在"));
  }

  @ExceptionHandler(OrderAlreadyPaidException.class)
  public ResponseEntity<ErrorResponse> handleOrderAlreadyPaid(OrderAlreadyPaidException ex) {
    logger.warn("Order already paid error: {}", ex.getMessage());
    return ResponseEntity.status(HttpStatus.CONFLICT).body(new ErrorResponse("訂單已經付款"));
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ErrorResponse> handleValidationErrors(MethodArgumentNotValidException ex) {
    Map<String, String> errors = new HashMap<>();
    ex.getBindingResult().getAllErrors().forEach(error -> {
        String fieldName = ((FieldError) error).getField();
        String errorMessage = error.getDefaultMessage();
        errors.put(fieldName, errorMessage);
    });

    logger.warn("Validation errors: {}", errors);
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse("資料驗證失敗，請檢查輸入內容"));
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorResponse> handleGeneralException(Exception ex) {
    logger.error("Unexpected error occurred: ", ex);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("系統發生錯誤，請稍後再試"));
  }
}
```

**說明：** 為所有控制器提供集中式例外處理。

**重點：**

- `@RestControllerAdvice` - 套用於所有 `@RestController` 類別
- 每個 `@ExceptionHandler` 處理特定的例外類型
- 回傳帶有適當 HTTP 狀態碼的 `ResponseEntity<ErrorResponse>`
- 記錄錯誤以便監控和除錯
- 捕捉所有非預期例外的處理器

**例外處理流程：**

```
控制器拋出例外
    ↓
Spring 搜尋符合的 @ExceptionHandler
    ↓
找到特定處理器（例如 OrderNotFoundException）
    ↓
執行處理器方法
    ↓
回傳 ResponseEntity 給客戶端
    ↓
客戶端收到 JSON 錯誤回應
```

---

### 範例 2：自訂例外類別

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/exception/OrderNotFoundException.java`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/exception/OrderNotFoundException.java)

```java
public class OrderNotFoundException extends RuntimeException {
  public OrderNotFoundException(Long orderId) {
    super("Order not found with ID: " + orderId);
  }
}
```

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/exception/OrderAlreadyPaidException.java`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/exception/OrderAlreadyPaidException.java)

```java
public class OrderAlreadyPaidException extends RuntimeException {
  public OrderAlreadyPaidException(Long orderId) {
    super("Order already paid: " + orderId);
  }
}
```

**說明：** 針對特定業務錯誤的自訂例外類別。

**重點：**

- 繼承 `RuntimeException`（非檢查例外）
- 描述性名稱指出錯誤類型
- 建構子接受上下文資訊（例如 orderId）
- 方法簽章中不需要 `throws` 宣告

**在服務中的使用方式：**

```java
@Transactional
public PayOrderResponse payOrder(Long orderId, Long userId) {
    Order order = orderRepository.findByIdAndUserId(orderId, userId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));

    if (order.isPaid()) {
        throw new OrderAlreadyPaidException(orderId);
    }

    // 處理付款...
}
```

---

### 範例 3：錯誤回應 DTO

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/dto/ErrorResponse.java`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/dto/ErrorResponse.java)

```java
public record ErrorResponse(String error) {}
```

**說明：** 簡單的錯誤回應 DTO。

**JSON 回應：**

```json
{
	"error": "訂單不存在"
}
```

**增強版錯誤回應（選用）：**

```java
public record ErrorResponse(
    String error,
    String code,        // 給前端處理用的錯誤代碼
    String timestamp,   // 錯誤發生時間
    String path         // 造成錯誤的請求路徑
) {}
```

**增強版 JSON：**

```json
{
	"error": "訂單不存在",
	"code": "ORDER_NOT_FOUND",
	"timestamp": "2025-12-02T10:30:00Z",
	"path": "/orders/123"
}
```

---

### 範例 4：驗證錯誤處理

**檔案：** [`www_root/waterballsa-backend/src/main/java/waterballsa/exception/GlobalExceptionHandler.java:133-148`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/exception/GlobalExceptionHandler.java#L133-L148)

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationErrors(
    MethodArgumentNotValidException ex) {
  Map<String, String> errors = new HashMap<>();
  ex.getBindingResult()
      .getAllErrors()
      .forEach(error -> {
          String fieldName = ((FieldError) error).getField();
          String errorMessage = error.getDefaultMessage();
          errors.put(fieldName, errorMessage);
      });

  logger.warn("Validation errors: {}", errors);

  return ResponseEntity.status(HttpStatus.BAD_REQUEST)
      .body(new ErrorResponse("資料驗證失敗，請檢查輸入內容"));
}
```

**說明：** 處理 `@Valid` 驗證失敗。

**驗證錯誤流程：**

```
客戶端發送：POST /auth/register
{
  "username": "ab",      // 太短（最少 3 個字元）
  "password": "short"    // 太短（最少 8 個字元）
}
    ↓
@Valid 觸發驗證
    ↓
驗證失敗 → 拋出 MethodArgumentNotValidException
    ↓
GlobalExceptionHandler 捕捉
    ↓
提取欄位錯誤：{username: "...", password: "..."}
    ↓
記錄錯誤以便除錯
    ↓
回傳 400 Bad Request 及錯誤訊息
```

**詳細錯誤回應（替代方案）：**

```java
public record ValidationErrorResponse(
    String message,
    Map<String, String> fieldErrors
) {}

return ResponseEntity.status(HttpStatus.BAD_REQUEST)
    .body(new ValidationErrorResponse("資料驗證失敗", errors));
```

**JSON：**

```json
{
	"message": "資料驗證失敗",
	"fieldErrors": {
		"username": "Username must be between 3 and 50 characters",
		"password": "Password must be between 8 and 72 characters"
	}
}
```

---

### 範例 5：HTTP 狀態碼對應

**404 Not Found：**

```java
@ExceptionHandler(OrderNotFoundException.class)
public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse("訂單不存在"));
}
```

**409 Conflict：**

```java
@ExceptionHandler(OrderAlreadyPaidException.class)
public ResponseEntity<ErrorResponse> handleOrderAlreadyPaid(OrderAlreadyPaidException ex) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(new ErrorResponse("訂單已經付款"));
}

@ExceptionHandler(DuplicateUsernameException.class)
public ResponseEntity<ErrorResponse> handleDuplicateUsername(DuplicateUsernameException ex) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(new ErrorResponse("使用者名稱已存在"));
}
```

**401 Unauthorized：**

```java
@ExceptionHandler(InvalidCredentialsException.class)
public ResponseEntity<ErrorResponse> handleInvalidCredentials(InvalidCredentialsException ex) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
        .body(new ErrorResponse("帳號或密碼錯誤"));
}

@ExceptionHandler(UnauthorizedException.class)
public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException ex) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
        .body(new ErrorResponse("登入資料已過期"));
}
```

**403 Forbidden：**

```java
@ExceptionHandler(ForbiddenException.class)
public ResponseEntity<ErrorResponse> handleForbidden(ForbiddenException ex) {
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
        .body(new ErrorResponse("禁止訪問"));
}

@ExceptionHandler(ProgressAccessDeniedException.class)
public ResponseEntity<ErrorResponse> handleProgressAccessDenied(ProgressAccessDeniedException ex) {
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
        .body(new ErrorResponse("無法存取其他使用者的進度"));
}
```

---

## 常見錯誤

### ❌ 不要：向客戶端暴露內部錯誤細節

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception ex) {
    // 錯誤！暴露堆疊追蹤和內部細節
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse(ex.getMessage() + "\n" + ex.getStackTrace()));
}
```

### ✅ 要：回傳通用訊息，記錄詳細資訊

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception ex) {
    logger.error("Unexpected error occurred: ", ex);  // 記錄完整細節
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("系統發生錯誤，請稍後再試"));  // 通用訊息
}
```

**原因：** 安全性 - 不要暴露實作細節。使用日誌進行除錯。

---

### ❌ 不要：為業務邏輯使用檢查例外

```java
public void processOrder(Long orderId) throws OrderNotFoundException {  // 錯誤！
    // 檢查例外，強制到處使用 try-catch
}
```

### ✅ 要：使用非檢查例外（RuntimeException）

```java
public void processOrder(Long orderId) {  // 正確！
    throw new OrderNotFoundException(orderId);  // RuntimeException
}
```

**原因：** 非檢查例外不會讓程式碼充斥 try-catch，且會觸發 @Transactional 回滾。

---

### ❌ 不要：捕捉並忽略例外

```java
try {
    orderService.createOrder(userId, request);
} catch (Exception e) {
    // 靜默忽略 - 錯誤！
}
```

### ✅ 要：讓例外向上傳遞到 GlobalExceptionHandler

```java
// Controller
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
    // 不要捕捉 - 讓 GlobalExceptionHandler 處理
    var result = orderService.createOrder(getCurrentUserId(), request);
    return ResponseEntity.ok(result);
}
```

**原因：** 集中式錯誤處理確保一致的錯誤回應。

---

### ❌ 不要：為錯誤回傳 200 OK

```java
@PostMapping("/login")
public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
    try {
        return ResponseEntity.ok(authService.login(request));
    } catch (InvalidCredentialsException e) {
        // 錯誤！為認證失敗回傳 200 OK
        return ResponseEntity.ok(new LoginResponse(null, "Login failed"));
    }
}
```

### ✅ 要：回傳適當的 HTTP 狀態碼

```java
@PostMapping("/login")
public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
    // 讓 GlobalExceptionHandler 為 InvalidCredentialsException 回傳 401
    return ResponseEntity.ok(authService.login(request));
}
```

**原因：** HTTP 狀態碼用於溝通成功/失敗。200 OK 應該代表成功。

---

## 例外階層快速參考

### Spring Boot 例外處理

```
Throwable
├── Error (系統錯誤，不要捕捉)
└── Exception
    ├── RuntimeException (非檢查)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── OrderNotFoundException      ← 自訂
    │   └── OrderAlreadyPaidException   ← 自訂
    └── Checked Exceptions (需要 throws 宣告)
        ├── IOException
        ├── SQLException
        └── ...
```

### 自訂例外最佳實踐

```java
// 領域的基礎例外
public abstract class OrderException extends RuntimeException {
    public OrderException(String message) {
        super(message);
    }
}

// 特定例外
public class OrderNotFoundException extends OrderException {
    public OrderNotFoundException(Long orderId) {
        super("Order not found: " + orderId);
    }
}

public class OrderAlreadyPaidException extends OrderException {
    public OrderAlreadyPaidException(Long orderId) {
        super("Order already paid: " + orderId);
    }
}
```

### 常見 HTTP 狀態碼

| 狀態碼                        | 意義           | 何時使用                 |
| ----------------------------- | -------------- | ------------------------ |
| **200 OK**                    | 成功           | 成功的 GET、PUT、DELETE  |
| **201 Created**               | 資源已建立     | 成功建立新資源的 POST    |
| **204 No Content**            | 成功，無內容   | 成功的 DELETE            |
| **400 Bad Request**           | 無效輸入       | 驗證錯誤、格式錯誤的請求 |
| **401 Unauthorized**          | 需要驗證       | 缺少/無效的 token        |
| **403 Forbidden**             | 已驗證，無權限 | 使用者無法存取資源       |
| **404 Not Found**             | 資源不存在     | 訂單/使用者找不到        |
| **409 Conflict**              | 違反業務規則   | 重複的使用者名稱、已付款 |
| **500 Internal Server Error** | 伺服器錯誤     | 非預期例外、資料庫錯誤   |

---

## 面試題目練習

- 📝 [題目 #31：全域例外處理器](../../interview/interview-backend.md#31-global-exception-handler-mid)
- 📝 [題目 #32：自訂例外](../../interview/interview-backend.md#32-custom-exceptions-mid)

---

**上一篇：** [← 07. Spring Security & JWT](07-spring-security-jwt.md)

**回到目錄：** [📚 所有指南](01-README.md)

# 後端面試題 - 50 題

> 針對中階全端軟體工程師職位準備
> 涵蓋：語法細節、分層架構、併發控制、資料完整性、交易管理

---

## 第一部分：Java 語法基礎與設計模式 (Junior)

### 1. Sealed Interface 語法解析 (Junior)

**檔案：OrderService.java (Line 56-65)**

請看以下程式碼：

```java
public sealed interface OrderCreationResult {
  OrderResponse orderResponse();

  record Created(OrderResponse orderResponse) implements OrderCreationResult {}
  record Existing(OrderResponse orderResponse) implements OrderCreationResult {}
}
```

請解釋：

- 什麼是 `sealed interface`？它與普通 interface 有什麼不同？
- 為什麼要使用 `record` 而不是 `class`？
- 這個設計模式解決了什麼問題？
- 在 Controller 層是如何使用這個 sealed interface 的？（提示：看 OrderController.java 的 pattern matching）

---

### 2. Pattern Matching 與 instanceof (Mid)

**檔案：OrderController.java (Line 43-52)**

請看以下程式碼：

```java
if (result instanceof OrderService.OrderCreationResult.Created created) {
  return ResponseEntity.status(HttpStatus.CREATED).body(created.orderResponse());
} else if (result instanceof OrderService.OrderCreationResult.Existing existing) {
  return ResponseEntity.ok(existing.orderResponse());
}
```

請解釋：

- 這種語法跟傳統的 `instanceof` + 型別轉換有什麼不同？
- 為什麼這裡回傳 `201 CREATED` vs `200 OK` 的 HTTP status code？
- 這樣的設計對前端有什麼好處？

---

### 3. Stream API 與 Lambda 表達式 (Junior)

**檔案：JourneyService.java (Line 94-99)**

請看以下程式碼：

```java
List<MissionSummaryDTO> missions =
    chapter.getMissions().stream()
        .filter(mission -> !mission.isDeleted())
        .sorted(Comparator.comparing(Mission::getOrderIndex))
        .map(this::mapToMissionSummaryDTO)
        .collect(Collectors.toList());
```

請解釋：

- 每一個 Stream 操作的作用是什麼？
- `Comparator.comparing(Mission::getOrderIndex)` 這種寫法叫什麼？
- `this::mapToMissionSummaryDTO` 是什麼語法？
- 如果不用 Stream API，你會怎麼寫？哪種寫法更好？

---

### 4. Enum 與 JPA 整合 (Junior)

**檔案：Order.java (Line 27-30)**

請看以下程式碼：

```java
@Enumerated(EnumType.STRING)
@JdbcTypeCode(SqlTypes.NAMED_ENUM)
@Column(name = "status", nullable = false, columnDefinition = "order_status")
private OrderStatus status;
```

請解釋：

- `@Enumerated(EnumType.STRING)` vs `EnumType.ORDINAL` 的差異？
- 為什麼要使用 `@JdbcTypeCode(SqlTypes.NAMED_ENUM)`？
- 如果資料庫已經有資料，你新增一個 Enum 值會發生什麼事？
- 如果你刪除一個 Enum 值會怎樣？

---

### 5. Optional 的使用時機 (Junior)

**檔案：OrderRepository.java (Line 35)**

請看以下程式碼：

```java
Optional<Order> findByIdAndUserId(Long id, Long userId);
```

請解釋：

- 為什麼 Repository 方法要回傳 `Optional<Order>` 而不是直接回傳 `Order`？
- 在 Service 層如何正確地使用這個 Optional？
- 什麼情況下不應該使用 Optional？

---

## 第二部分：Spring Boot 架構與注入 (Junior/Mid)

### 6. Controller 層的職責 (Junior)

**檔案：OrderController.java**

請解釋：

- Controller 層應該負責什麼？不應該負責什麼？
- 為什麼所有的 business logic 都在 Service 層而不是 Controller？
- `@RestController` vs `@Controller` 的差異？
- `@RequestMapping("/orders")` 的作用是什麼？

---

### 7. 依賴注入的方式 (Junior)

**檔案：OrderController.java (Line 26-28)**

請看以下程式碼：

```java
public OrderController(OrderService orderService) {
  this.orderService = orderService;
}
```

請解釋：

- 這是哪一種依賴注入方式？
- 為什麼不用 `@Autowired` 註解？
- 這種方式有什麼優點？
- 還有哪些依賴注入的方式？各有什麼優缺點？

---

### 8. SecurityContextHolder 的運作原理 (Mid)

**檔案：OrderController.java (Line 92-95)**

請看以下程式碼：

```java
private Long getCurrentUserId() {
  Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
  return Objects.requireNonNull((Long) authentication.getPrincipal(), "User ID must not be null");
}
```

請解釋：

- `SecurityContextHolder` 是什麼？它的資料是如何設定的？
- `Authentication.getPrincipal()` 通常會回傳什麼型別？為什麼這裡是 `Long`？
- 這個方法在多執行緒環境下安全嗎？為什麼？
- 如果使用者未登入，這個方法會回傳什麼？

---

### 9. @Valid 與 Bean Validation (Junior)

**檔案：OrderController.java (Line 37)**

請看以下程式碼：

```java
@PostMapping
public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request)
```

請解釋：

- `@Valid` 註解的作用是什麼？
- 驗證失敗時會發生什麼事？
- 錯誤訊息是如何被處理的？（提示：看 GlobalExceptionHandler）
- 如何在 DTO 上定義驗證規則？

---

### 10. Logger 的使用原則 (Junior)

**檔案：OrderService.java (Line 25, 80, 115)**

請看以下程式碼：

```java
private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
logger.debug("Creating order for user: {}", userId);
logger.info("Successfully created order {} for user {}", order.getId(), userId);
```

請解釋：

- 為什麼 Logger 要宣告為 `private static final`？
- `logger.debug()` vs `logger.info()` vs `logger.warn()` vs `logger.error()` 的使用時機？
- 為什麼不用 `String.format()` 或字串相加來組合 log 訊息？
- 什麼情況下應該 log Exception 的 stack trace？

---

## 第三部分：JPA 與資料庫操作 (Mid)

### 11. Entity Lifecycle Callbacks (Mid)

**檔案：Order.java (Line 74-87)**

請看以下程式碼：

```java
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
```

請解釋：

- `@PrePersist` 和 `@PreUpdate` 是什麼時候被呼叫的？
- 為什麼要在這裡設定 `createdAt` 和 `updatedAt`，而不是在建構子或 setter？
- 如果在 `@PrePersist` 中拋出例外會怎樣？
- JPA 還有哪些其他的 Lifecycle Callbacks？

---

### 12. FetchType.LAZY 的陷阱 (Mid)

**檔案：Mission.java (Line 18-20)**

請看以下程式碼：

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "chapter_id", nullable = false)
private Chapter chapter;
```

請解釋：

- `FetchType.LAZY` vs `FetchType.EAGER` 的差異？
- 什麼是 N+1 Query Problem？如何避免？
- 如果在 `@Transactional` 外面存取 `chapter` 會發生什麼事？
- 什麼情況下應該使用 LAZY？什麼情況下應該使用 EAGER？

---

### 13. Pessimistic Locking 實作 (Mid)

**檔案：OrderRepository.java (Line 93-95)**

請看以下程式碼：

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id AND o.userId = :userId AND o.deletedAt IS NULL")
Optional<Order> findByIdAndUserIdForUpdate(@Param("id") Long id, @Param("userId") Long userId);
```

請解釋：

- `PESSIMISTIC_WRITE` lock 是什麼？它在資料庫層面是如何實現的？
- 為什麼付款時需要使用 pessimistic lock？（提示：想想併發付款的情況）
- `PESSIMISTIC_WRITE` vs `PESSIMISTIC_READ` vs `OPTIMISTIC` 的差異？
- 如果兩個 transaction 同時嘗試 lock 同一筆資料會發生什麼事？
- Pessimistic lock 有什麼缺點？

---

### 14. Custom JPQL Query (Mid)

**檔案：OrderRepository.java (Line 46-57)**

請看以下程式碼：

```java
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

請解釋：

- 為什麼要寫自訂 JPQL 而不用 Spring Data JPA 的方法命名規則？
- `JOIN o.items oi` 是什麼意思？這是 SQL JOIN 還是 JPA 的關聯？
- `LIMIT 1` 在不同資料庫中是否都能用？
- 這個查詢的目的是什麼？（提示：看 OrderService 的 idempotence 設計）

---

### 15. Soft Delete 設計 (Mid)

**檔案：Order.java (Line 54, 112-118)**

請看以下程式碼：

```java
@Column(name = "deleted_at")
private LocalDateTime deletedAt;

public void softDelete() {
  this.deletedAt = LocalDateTime.now();
}

public boolean isDeleted() {
  return this.deletedAt != null;
}
```

請解釋：

- 什麼是 Soft Delete？為什麼要用 Soft Delete 而不直接刪除資料？
- 如何確保查詢時不會查到已刪除的資料？（提示：看 Repository 的 Query）
- Soft Delete 有什麼缺點？
- 如果要實作「永久刪除」功能，應該怎麼做？

---

## 第四部分：交易管理與併發控制 (Mid)

### 16. @Transactional 的範圍 (Mid)

**檔案：OrderService.java (Line 77, 150)**

請看以下程式碼：

```java
@Transactional
public OrderCreationResult createOrder(@NonNull Long userId, CreateOrderRequest request) {
  // ... business logic
}

@Transactional
public PayOrderResponse payOrder(Long orderId, Long userId) {
  // ... business logic
}
```

請解釋：

- `@Transactional` 的作用是什麼？
- 為什麼要在 Service 層加 `@Transactional` 而不是 Repository 層？
- 在 `@Transactional` 方法中如果拋出例外會發生什麼事？
- `@Transactional(readOnly = true)` 有什麼用？什麼時候應該使用？
- 如果一個 `@Transactional` 方法呼叫另一個 `@Transactional` 方法，transaction 會如何傳播？

---

### 17. Idempotence 設計實作 (Mid)

**檔案：OrderService.java (Line 89-95)**

請看以下程式碼：

```java
// Check if user already has an unpaid order for this journey
var existingOrder =
    orderRepository.findByUserIdAndStatusAndJourneyId(userId, OrderStatus.UNPAID, journeyId);
if (existingOrder.isPresent()) {
  logger.info("Returning existing unpaid order for user {} and journey {}", userId, journeyId);
  return new OrderCreationResult.Existing(mapToOrderResponse(existingOrder.get()));
}
```

請解釋：

- 什麼是 Idempotence？為什麼 API 設計要考慮 idempotence？
- 這段程式碼如何實現 idempotence？
- 如果使用者連續點兩次「建立訂單」按鈕會發生什麼事？
- 這個設計是否完全 thread-safe？如果兩個請求同時進來會怎樣？
- 如何改進這個設計讓它更安全？（提示：想想 database constraint 或 lock）

---

### 18. 併發付款的處理 (Mid)

**檔案：OrderService.java (Line 154-161)**

請看以下程式碼：

```java
Order order =
    orderRepository
        .findByIdAndUserIdForUpdate(orderId, userId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));

orderValidator.validateOrderNotPaid(order);
orderValidator.validateOrderNotExpired(order);

order.markAsPaid();
```

請解釋：

- 如果兩個使用者同時對同一個訂單發起付款請求會發生什麼事？
- `findByIdAndUserIdForUpdate()` 的 pessimistic lock 如何防止 race condition？
- 為什麼要先檢查 `validateOrderNotPaid()`？這不是多此一舉嗎？
- 如果 lock 發生 timeout 會拋出什麼例外？如何處理？

---

### 19. Scheduled Task 的實作 (Mid)

**檔案：OrderService.java (Line 244-263)**

請看以下程式碼：

```java
@Scheduled(cron = "0 */10 * * * *")
@Transactional
public void expireOrders() {
  LocalDateTime now = LocalDateTime.now();
  List<Order> expiredOrders =
      orderRepository.findByStatusAndExpiredAtBefore(OrderStatus.UNPAID, now);

  if (!expiredOrders.isEmpty()) {
    for (Order order : expiredOrders) {
      order.markAsExpired();
      orderRepository.save(order);
    }
  }
}
```

請解釋：

- `@Scheduled(cron = "0 */10 * * * *")` 是什麼意思？
- 為什麼這個方法也需要 `@Transactional`？
- 如果在 loop 中處理某個 order 時拋出例外會怎樣？整個 transaction 會 rollback 嗎？
- 如果這個方法執行時間超過 10 分鐘會怎樣？
- 如何改進這個設計？（提示：batch processing, pagination）

---

### 20. Upsert 操作的實作 (Mid)

**檔案：ProgressService.java (Line 102-113)**

請看以下程式碼：

```java
UserMissionProgress progress =
    progressRepository
        .findByUserIdAndMissionIdAndDeletedAtIsNull(pathUserId, missionId)
        .orElseGet(
            () -> {
              User user = userRepository.findById(pathUserId).orElseThrow(...);
              return new UserMissionProgress(user, mission);
            });

progress.updateWatchPosition(cappedPosition);
progressRepository.save(progress);
```

請解釋：

- 什麼是 Upsert？這段程式碼如何實現 upsert？
- 為什麼要用 `orElseGet()` 而不是 `orElse()`？
- 如果 progress 不存在，`progressRepository.save()` 會執行 INSERT；如果存在，會執行 UPDATE。JPA 是如何判斷的？
- 這個設計是否 thread-safe？如果兩個請求同時更新進度會怎樣？

---

## 第五部分：Spring Security 與 JWT (Mid)

### 21. JWT Token 的生成 (Mid)

**檔案：JwtUtil.java (Line 38-56)**

請看以下程式碼：

```java
public String generateToken(User user) {
  Instant now = Instant.now();
  Instant expiration = now.plusMillis(expirationMs);
  String jti = UUID.randomUUID().toString();

  String token =
      Jwts.builder()
          .id(jti)
          .subject(user.getId().toString())
          .claim("username", user.getUsername())
          .issuedAt(Date.from(now))
          .expiration(Date.from(expiration))
          .signWith(secretKey)
          .compact();

  return token;
}
```

請解釋：

- JWT 的結構是什麼？（Header, Payload, Signature）
- 什麼是 JTI？為什麼需要它？
- JWT 中應該放哪些資訊？不應該放哪些資訊？
- `signWith(secretKey)` 使用什麼演算法？為什麼要簽名？
- JWT 的優點和缺點是什麼？

---

### 22. JWT Filter 的執行流程 (Mid)

**檔案：JwtAuthenticationFilter.java (Line 50-104)**

請看以下程式碼：

```java
protected void doFilterInternal(
    @NonNull HttpServletRequest request,
    @NonNull HttpServletResponse response,
    @NonNull FilterChain filterChain)
    throws ServletException, IOException {

  String authHeader = request.getHeader(AUTHORIZATION_HEADER);

  if (authHeader == null || !authHeader.startsWith(BEARER_PREFIX)) {
    filterChain.doFilter(request, response);
    return;
  }

  String token = authHeader.substring(BEARER_PREFIX.length());
  // ... validation and authentication setup

  filterChain.doFilter(request, response);
}
```

請解釋：

- `OncePerRequestFilter` 是什麼？為什麼要繼承它？
- Filter 在 Spring Security 中的執行順序是什麼？
- 如果 token 驗證失敗，為什麼還要繼續執行 `filterChain.doFilter()`？
- `SecurityContextHolder.getContext().setAuthentication()` 的作用是什麼？
- 為什麼要把 `userId` 放在 `authentication.getPrincipal()` 中？

---

### 23. Token Blacklist 機制 (Mid)

**檔案：AuthService.java (Line 98-126)**

請看以下程式碼：

```java
public LogoutResponse logout(String token) {
  if (!jwtUtil.validateToken(token)) {
    throw new UnauthorizedException();
  }

  String jti = jwtUtil.getJtiFromToken(token);
  LocalDateTime expiresAt = jwtUtil.getExpirationFromToken(token);
  Long userId = jwtUtil.getUserIdFromToken(token);

  if (accessTokenRepository.existsByTokenJti(jti)) {
    throw new UnauthorizedException();
  }

  AccessToken accessToken = new AccessToken(jti, userId, expiresAt);
  accessTokenRepository.save(accessToken);

  return new LogoutResponse("Logout successful");
}
```

請解釋：

- 為什麼 JWT 需要 blacklist？JWT 不是 stateless 的嗎？
- Blacklist 的資料表應該如何設計？（提示：需要哪些欄位？需要 index 嗎？）
- 如何避免 blacklist 資料表無限成長？
- 有沒有更好的方式實作 logout 功能？（提示：refresh token, short-lived access token）

---

### 24. Rate Limiting 的實作 (Mid)

**檔案：AuthController.java (Line 54-58)**

請看以下程式碼：

```java
if (!rateLimitService.tryConsume(ipAddress)) {
  logger.warn("Rate limit exceeded for IP: {}", ipAddress);
  throw new InvalidCredentialsException();
}
```

請解釋：

- Rate limiting 是什麼？為什麼需要它？
- 為什麼登入失敗時不重置 rate limit，但成功時要重置？
- 如何實作 rate limiting？（提示：看 pom.xml 的 Bucket4j）
- 用 IP 來做 rate limiting 有什麼問題？如何改進？
- 如果 rate limit 設定太嚴格或太寬鬆會有什麼影響？

---

### 25. CORS 配置 (Mid)

**檔案：SecurityConfig.java (Line 37-56)**

請看以下程式碼：

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration configuration = new CorsConfiguration();
  configuration.setAllowedOriginPatterns(
      Arrays.asList(
          "http://localhost:3000",
          "http://localhost:80",
          "https://localhost:443",
          "https://*.ngrok-free.dev"
      ));
  configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
  configuration.setAllowedHeaders(Arrays.asList("*"));
  configuration.setAllowCredentials(true);
  configuration.setMaxAge(3600L);

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/**", configuration);
  return source;
}
```

請解釋：

- 什麼是 CORS？為什麼需要 CORS 配置？
- `setAllowCredentials(true)` 是什麼意思？
- 為什麼要設定 `OPTIONS` method？它在 CORS 中的作用是什麼？
- `setMaxAge(3600L)` 的作用是什麼？
- 生產環境的 CORS 配置應該怎麼設定？

---

## 第六部分：資料庫設計與查詢優化 (Mid)

### 26. 關聯關係的設計 (Mid)

**檔案：Order.java (Line 56-57)**

請看以下程式碼：

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

請解釋：

- `mappedBy = "order"` 是什麼意思？哪一邊是關聯的擁有者？
- `CascadeType.ALL` 包含哪些操作？
- `orphanRemoval = true` 是什麼意思？跟 `CascadeType.REMOVE` 有什麼不同？
- 為什麼要初始化 `new ArrayList<>()`？

---

### 27. 雙向關聯的管理 (Mid)

**檔案：Order.java (Line 90-93)**

請看以下程式碼：

```java
public void addItem(OrderItem item) {
  items.add(item);
  item.setOrder(this);
}
```

請解釋：

- 為什麼需要這個 helper method？
- 如果只呼叫 `items.add(item)` 而不呼叫 `item.setOrder(this)` 會怎樣？
- JPA 如何處理雙向關聯？
- 這種設計有什麼優點？

---

### 28. Pagination 實作 (Mid)

**檔案：OrderRepository.java (Line 66-71)**

請看以下程式碼：

```java
@Query(
    "SELECT o FROM Order o "
        + "WHERE o.userId = :userId "
        + "AND o.deletedAt IS NULL "
        + "ORDER BY o.createdAt DESC")
Page<Order> findByUserIdOrderByCreatedAtDesc(@Param("userId") Long userId, Pageable pageable);
```

請解釋：

- `Page<Order>` vs `List<Order>` 的差異？
- `Pageable` 包含哪些資訊？
- 如何在 Service 層使用這個方法？（提示：看 UserService）
- Pagination 有什麼好處？為什麼不直接查全部資料？
- Offset-based pagination vs Cursor-based pagination 的差異？

---

### 29. N+1 Query 問題 (Mid)

**檔案：MissionService.java**

假設你要列出所有 Journey 以及每個 Journey 的所有 Chapter。如果使用以下程式碼：

```java
List<Journey> journeys = journeyRepository.findAll();
for (Journey journey : journeys) {
  System.out.println(journey.getChapters().size()); // 觸發 lazy loading
}
```

請解釋：

- 這段程式碼會執行幾次 SQL query？
- 什麼是 N+1 Query Problem？
- 如何解決這個問題？（提示：`@EntityGraph`, `JOIN FETCH`）
- 在什麼情況下 lazy loading 反而是好的？

---

### 30. 資料完整性約束 (Mid)

**檔案：OrderRepository.java (Line 46-57)**

在 `createOrder` 時，我們需要確保：

1. 使用者沒有購買過該課程
2. 使用者沒有該課程的未付款訂單

請解釋：

- 這些檢查是在應用層做的，如果有 race condition 會發生什麼事？
- 如何在資料庫層面保證資料完整性？（提示：unique constraint）
- `user_journeys` 表應該有什麼 unique constraint？
- 如果有 unique constraint，當違反時會拋出什麼例外？如何處理？

---

## 第七部分：錯誤處理與例外設計 (Mid)

### 31. 全域例外處理器 (Mid)

**檔案：GlobalExceptionHandler.java**

請解釋：

- `@RestControllerAdvice` vs `@ControllerAdvice` 的差異？
- 為什麼要使用全域例外處理器而不是在每個 Controller 中 try-catch？
- 例外處理的順序是什麼？（提示：從特定到一般）
- 如何處理 Spring 框架本身拋出的例外？（如 `MethodArgumentNotValidException`）

---

### 32. 自訂例外的設計 (Mid)

**檔案：DuplicateUsernameException, InvalidCredentialsException 等**

請解釋：

- 什麼時候應該自訂例外？
- Checked Exception vs Unchecked Exception 的差異？這個專案使用哪一種？
- 為什麼要為每種錯誤情況建立不同的例外類別？
- 自訂例外應該繼承 `Exception` 還是 `RuntimeException`？

---

### 33. 錯誤訊息的設計 (Junior)

**檔案：GlobalExceptionHandler.java (Line 34-36)**

請看以下程式碼：

```java
@ExceptionHandler(InvalidCredentialsException.class)
public ResponseEntity<ErrorResponse> handleInvalidCredentials(InvalidCredentialsException ex) {
  logger.warn("Invalid credentials error: {}", ex.getMessage());
  return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(new ErrorResponse("帳號或密碼錯誤"));
}
```

請解釋：

- 為什麼錯誤訊息要回傳「帳號或密碼錯誤」而不是「密碼錯誤」或「帳號不存在」？
- 什麼資訊應該回傳給前端？什麼資訊應該只記錄在 log？
- HTTP status code 的選擇原則是什麼？
- 如何設計多語系的錯誤訊息？

---

## 第八部分：測試與品質保證 (Junior/Mid)

### 34. 單元測試 vs 整合測試 (Junior)

請解釋：

- 單元測試和整合測試的差異？
- Service 層的單元測試應該怎麼寫？需要 mock 什麼？
- 什麼情況下應該寫整合測試？
- 這個專案使用 Testcontainers 來做什麼？（提示：看 pom.xml）

---

### 35. 測試資料的準備 (Mid)

假設你要測試 `payOrder` 方法，你需要準備：

- 一個已登入的使用者
- 一個未付款的訂單
- 訂單包含一個課程

請解釋：

- 如何準備這些測試資料？
- 測試完成後如何清理資料？
- 如何確保測試之間不會互相影響？
- 什麼是 Test Fixture？

---

## 第九部分：效能優化 (Mid)

### 36. 查詢效能優化 (Mid)

**檔案：OrderService.java (Line 190-242)**

請看 `mapToOrderResponse` 方法，它需要：

1. 查詢 User 來取得 username
2. 對每個 OrderItem 查詢 Journey 來取得 title

請解釋：

- 這段程式碼可能會有什麼效能問題？
- 如何優化這些查詢？（提示：JOIN FETCH, DTO projection）
- 什麼時候應該用 `@EntityGraph`？什麼時候應該用 DTO projection？
- 如果資料量很大，應該考慮什麼？

---

### 37. Cache 的使用時機 (Mid)

**檔案：pom.xml (Line 108-111) - Caffeine**

請解釋：

- 什麼資料適合放入 cache？什麼資料不適合？
- Cache 有哪些策略？（Cache-Aside, Read-Through, Write-Through, Write-Behind）
- 如何避免 cache 與資料庫不一致？
- 如何設定 cache 的過期時間？
- 分散式環境下的 cache 應該怎麼處理？

---

### 38. 批次操作的優化 (Mid)

**檔案：OrderService.java (Line 254-259)**

請看以下程式碼：

```java
for (Order order : expiredOrders) {
  order.markAsExpired();
  orderRepository.save(order);
}
```

請解釋：

- 這段程式碼有什麼效能問題？
- 如何改成批次更新？（提示：`saveAll`, native query）
- 批次大小應該如何設定？
- 批次操作時如何處理部分失敗的情況？

---

## 第十部分：架構設計與重構 (Mid)

### 39. 分層架構的設計原則 (Mid)

請解釋：

- Controller、Service、Repository 三層的職責分別是什麼？
- DTO vs Entity 的差異？為什麼要分開？
- 如果 Service 層的方法變得很長（>100 行），應該如何重構？
- 什麼是 Domain Driven Design？跟這個專案的架構有什麼差異？

---

### 40. Validator 的設計 (Mid)

**檔案：OrderValidator.java, ProgressValidator.java**

請解釋：

- 為什麼要獨立出 Validator 層？
- Validator 跟 Service 的差異是什麼？
- 什麼樣的邏輯應該放在 Validator？什麼應該放在 Service？
- Validator 的方法應該回傳 boolean 還是直接拋出例外？

---

### 41. DTO 的轉換時機 (Mid)

**檔案：OrderService.java (Line 190-242)**

請解釋：

- 為什麼要在 Service 層轉換 Entity to DTO，而不是直接回傳 Entity？
- 如果直接回傳 Entity 會有什麼問題？（提示：Jackson serialization, lazy loading）
- 使用 MapStruct 或 ModelMapper 這類工具的優缺點是什麼？
- DTO 應該是 immutable 的嗎？（record vs class）

---

### 42. Business Logic 的位置 (Mid)

**檔案：Order.java, User.java**

請看以下兩種設計：

設計 A：在 Entity 中

```java
public void markAsPaid() {
  this.status = OrderStatus.PAID;
  this.paidAt = LocalDateTime.now();
}
```

設計 B：在 Service 中

```java
order.setStatus(OrderStatus.PAID);
order.setPaidAt(LocalDateTime.now());
```

請解釋：

- 哪一種設計比較好？為什麼？
- Entity 應該是 Anemic Domain Model 還是 Rich Domain Model？
- 什麼樣的 business logic 應該放在 Entity？什麼應該放在 Service？

---

## 第十一部分：程式碼品質與慣例 (Junior/Mid)

### 43. Immutability 的好處 (Mid)

**檔案：各種 DTO (record 類別)**

請解釋：

- 為什麼大部分 DTO 使用 `record` 而不是 `class`？
- Immutable 物件有什麼好處？
- 在多執行緒環境下，immutable 物件如何幫助避免 race condition？
- Entity 應該是 immutable 的嗎？為什麼？

---

### 44. Null Safety (Junior)

**檔案：OrderController.java (Line 94), ProgressService.java (Line 85)**

請解釋：

- `@NonNull` 註解的作用是什麼？
- 如何避免 NullPointerException？
- `Optional` vs 直接回傳 null 的差異？
- Java 有沒有類似 Kotlin 的 null safety？

---

### 45. Magic Number 的處理 (Junior)

**檔案：ProgressService.java (Line 25), Order.java (Line 80)**

請看以下程式碼：

```java
private static final Integer DEFAULT_EXPERIENCE_REWARD = 100;

this.expiredAt = this.createdAt.plusDays(3);
```

請解釋：

- 為什麼要定義 `DEFAULT_EXPERIENCE_REWARD` 常數？
- `plusDays(3)` 是否也應該定義成常數？
- 如何管理這些設定值？（提示：application.properties, database configuration table）
- 什麼是 Magic Number？如何避免？

---

## 第十二部分：部署與維運 (Mid)

### 46. Profile 的使用 (Junior)

**檔案：application.properties (需額外查看)**

請解釋：

- Spring Boot 的 Profile 是什麼？
- 如何為不同環境（dev, staging, production）設定不同的 configuration？
- 如何在部署時指定要使用哪個 profile？
- Database connection pool 的設定應該放在哪裡？

---

### 47. Logging 策略 (Mid)

請解釋：

- 生產環境的 log level 應該設定為什麼？
- 如何避免 log 檔案無限成長？
- 什麼資訊應該記錄在 log？什麼不應該？（提示：隱私、安全性）
- 如何實作 Distributed Tracing？（提示：Correlation ID）

---

### 48. Health Check 的實作 (Junior)

**檔案：HealthController.java**

請解釋：

- Health check endpoint 的作用是什麼？
- 應該檢查哪些項目？（database, redis, external API）
- Kubernetes 的 liveness probe vs readiness probe 的差異？
- Health check 失敗時應該回傳什麼 HTTP status code？

---

## 第十三部分：情境題與設計決策 (Mid)

### 49. 設計一個「限時優惠」功能 (Mid)

需求：

- 課程可以設定限時優惠價格
- 優惠期間內下單要使用優惠價格
- 優惠結束後建立的訂單要使用原價
- 優惠期間內建立但尚未付款的訂單，即使優惠結束也要保持優惠價格

請設計：

- Database schema 應該如何設計？
- 如何確保價格鎖定的邏輯正確？
- 如何處理時區問題？
- 如何避免使用者惡意囤積未付款訂單？

---

### 50. 架構重構決策 (Mid)

當專案規模成長，你發現：

- Service 類別變得太大（>1000 行）
- 很多 business logic 重複
- 測試變得很難寫
- 修改一個功能會影響很多地方

請提出：

- 你會如何重構？
- 應該引入哪些設計模式？
- 如何確保重構不會破壞現有功能？
- 如何說服團隊進行重構？

---

## 總結

這 50 道面試題涵蓋了：

- ✅ Java 語法細節（sealed interface, pattern matching, Stream API, enum）
- ✅ Spring Boot 框架（DI, annotations, filter, security）
- ✅ JPA 與資料庫（entity lifecycle, lazy loading, locking, query optimization）
- ✅ 併發控制（pessimistic lock, transaction, idempotence）
- ✅ 資料完整性（soft delete, cascade, orphan removal）
- ✅ JWT 與認證（token generation, validation, blacklist）
- ✅ 錯誤處理（global exception handler, custom exceptions）
- ✅ 分層架構（Controller-Service-Repository）
- ✅ 程式碼品質（immutability, null safety, constants）
- ✅ 效能優化（N+1 query, caching, batch processing）

準備建議：

1. 先能清楚解釋每段程式碼的作用
2. 理解為什麼這樣設計（trade-offs）
3. 能提出替代方案和改進建議
4. 能夠結合實際情境說明

祝面試順利！🎯

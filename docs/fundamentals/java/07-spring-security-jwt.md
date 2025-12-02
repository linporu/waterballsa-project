# Spring Security & JWT

> 給來自 Python/Laravel/Go 的中階開發者的快速參考
> 📚 相關面試題目: [#8](../../interview/interview-backend.md#8-securitycontextholder-mid), [#21](../../interview/interview-backend.md#21-jwt-token-structure-mid), [#22](../../interview/interview-backend.md#22-onceper requestfilter-mid), [#23](../../interview/interview-backend.md#23-jwt-blacklist-mid), [#24](../../interview/interview-backend.md#24-token-validation-mid), [#25](../../interview/interview-backend.md#25-cors-configuration-mid)

## 從 Python/Laravel/Go 轉換過來

| 概念               | Laravel                 | Python/Django         | Go                              | Spring Security          |
| ------------------ | ----------------------- | --------------------- | ------------------------------- | ------------------------ |
| **JWT 函式庫**     | `tymon/jwt-auth`        | `PyJWT`               | `golang-jwt/jwt`                | `jjwt` (io.jsonwebtoken) |
| **建立 Token**     | `JWTAuth::fromUser()`   | `jwt.encode()`        | `jwt.NewWithClaims()`           | `Jwts.builder()`         |
| **驗證 Token**     | `JWTAuth::parseToken()` | `jwt.decode()`        | `jwt.Parse()`                   | `Jwts.parser()`          |
| **取得當前使用者** | `Auth::user()`          | `request.user`        | Context value                   | `SecurityContextHolder`  |
| **中介層**         | Middleware classes      | Middleware classes    | Middleware func                 | `OncePerRequestFilter`   |
| **CORS**           | `cors` middleware       | `django-cors-headers` | `rs/cors`                       | `CorsConfiguration`      |
| **密碼雜湊**       | `Hash::make()`          | `make_password()`     | `bcrypt.GenerateFromPassword()` | `BCryptPasswordEncoder`  |

## 快速語法速查表

### 1. JWT 結構

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

┌─────────────────────────────────────┐
│ Header (Base64URL 編碼)             │
│ {"alg": "HS256", "typ": "JWT"}     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Payload (Base64URL 編碼)            │
│ {                                    │
│   "sub": "123",     // Subject (使用者 ID)
│   "jti": "uuid",    // JWT ID
│   "username": "john"// 自訂宣告
│   "iat": 1516239022,// 發行時間
│   "exp": 1516325422 // 過期時間
│ }                                    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Signature                            │
│ HMACSHA256(                          │
│   base64UrlEncode(header) + "." +   │
│   base64UrlEncode(payload),         │
│   secret                             │
│ )                                    │
└─────────────────────────────────────┘
```

---

### 2. 建立 JWT Token

```java
@Component
public class JwtUtil {

    private final SecretKey secretKey;
    private final long expirationMs;

    public String generateToken(User user) {
        Instant now = Instant.now();
        Instant expiration = now.plusMillis(expirationMs);

        return Jwts.builder()
            .id(UUID.randomUUID().toString())        // jti 宣告
            .subject(user.getId().toString())         // sub 宣告 (使用者 ID)
            .claim("username", user.getUsername())    // 自訂宣告
            .issuedAt(Date.from(now))                 // iat 宣告
            .expiration(Date.from(expiration))        // exp 宣告
            .signWith(secretKey)                      // 用密鑰簽章
            .compact();                               // 建立 token 字串
    }
}
```

---

### 3. 驗證 JWT Token

```java
public boolean validateToken(String token) {
    try {
        Jwts.parser()
            .verifyWith(secretKey)
            .build()
            .parseSignedClaims(token);
        return true;
    } catch (ExpiredJwtException e) {
        // Token 已過期
        return false;
    } catch (JwtException e) {
        // 無效的 token
        return false;
    }
}
```

---

### 4. OncePerRequestFilter 用於認證

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {

        // 從 Authorization header 提取 token
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);

            if (jwtUtil.validateToken(token)) {
                Long userId = jwtUtil.getUserIdFromToken(token);

                // 在 SecurityContext 中設定認證
                Authentication auth = new UsernamePasswordAuthenticationToken(
                    userId, null, Collections.emptyList());
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }

        // 繼續過濾鏈
        filterChain.doFilter(request, response);
    }
}
```

---

### 5. 取得當前使用者

```java
@RestController
public class OrderController {

    private Long getCurrentUserId() {
        Authentication authentication = SecurityContextHolder
            .getContext()
            .getAuthentication();
        return (Long) authentication.getPrincipal();
    }

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        Long userId = getCurrentUserId();  // 從 SecurityContext 取得
        var result = orderService.createOrder(userId, request);
        return ResponseEntity.ok(result);
    }
}
```

---

### 6. CORS 設定

```java
@Configuration
public class SecurityConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(Arrays.asList("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);  // Preflight 快取時間

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 實際專案範例

### 範例 1: JWT Token 產生

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java:38-56`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java#L38-L56)

```java
public String generateToken(User user) {
  Instant now = Instant.now();
  Instant expiration = now.plusMillis(expirationMs);  // 1 天
  String jti = UUID.randomUUID().toString();

  String token = Jwts.builder()
      .id(jti)                              // JWT ID 用於黑名單
      .subject(user.getId().toString())      // 使用者 ID
      .claim("username", user.getUsername()) // 自訂宣告
      .issuedAt(Date.from(now))              // 發行時間
      .expiration(Date.from(expiration))     // 過期時間
      .signWith(secretKey)                   // HMAC-SHA256 簽章
      .compact();

  logger.debug("Generated JWT token for user: {} with JTI: {}", user.getId(), jti);
  return token;
}
```

**說明:** 建立包含標準和自訂宣告的 JWT token。

**重點:**

- `jti` (JWT ID) - 用於黑名單追蹤的唯一識別碼
- `sub` (Subject) - 以字串儲存的使用者 ID
- 自訂宣告 `username` - 額外的使用者資訊
- `iat` (Issued At) - Token 建立時間
- `exp` (Expiration) - Token 過期時間 (1 天)
- `signWith(secretKey)` - HMAC-SHA256 簽章

**產生的 token 結構:**

```json
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload
{
  "jti": "a1b2c3d4-...",
  "sub": "123",
  "username": "john_doe",
  "iat": 1732420800,
  "exp": 1732507200
}

// Signature (用密鑰計算)
```

---

### 範例 2: JWT Token 驗證

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java:64-72`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java#L64-L72)

```java
public boolean validateToken(String token) {
  try {
    Jwts.parser()
        .verifyWith(secretKey)  // 用密鑰驗證簽章
        .build()
        .parseSignedClaims(token);  // 解析並驗證
    return true;
  } catch (Exception e) {
    logger.debug("JWT validation failed: {}", e.getMessage());
    return false;
  }
}
```

**說明:** 驗證 JWT token 簽章和過期時間。

**執行的驗證檢查:**

1. 簽章驗證 (防止竄改)
2. 過期檢查 (`exp` 宣告)
3. 格式驗證

**常見例外:**

- `ExpiredJwtException` - Token 已過期
- `SignatureException` - 無效簽章 (被竄改)
- `MalformedJwtException` - 無效的 token 格式
- `UnsupportedJwtException` - 不支援的 token

---

### 範例 3: 從 Token 提取宣告

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java:80-96`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/util/JwtUtil.java#L80-L96)

```java
public Long getUserIdFromToken(String token) {
  Claims claims = Jwts.parser()
      .verifyWith(secretKey)
      .build()
      .parseSignedClaims(token)
      .getPayload();
  return Long.parseLong(claims.getSubject());
}

public String getJtiFromToken(String token) {
  Claims claims = Jwts.parser()
      .verifyWith(secretKey)
      .build()
      .parseSignedClaims(token)
      .getPayload();
  return claims.getId();
}
```

**說明:** 從驗證過的 token 中提取特定宣告。

**可用的宣告取得方法:**

- `claims.getId()` - jti 宣告
- `claims.getSubject()` - sub 宣告
- `claims.get("username", String.class)` - 自訂宣告
- `claims.getIssuedAt()` - iat 宣告
- `claims.getExpiration()` - exp 宣告

---

### 範例 4: 認證過濾器

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/filter/JwtAuthenticationFilter.java:50-90`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/filter/JwtAuthenticationFilter.java#L50-L90)

```java
@Override
protected void doFilterInternal(
    @NonNull HttpServletRequest request,
    @NonNull HttpServletResponse response,
    @NonNull FilterChain filterChain) throws ServletException, IOException {

  String authHeader = request.getHeader(AUTHORIZATION_HEADER);

  // 檢查 Authorization header 是否存在且以 "Bearer " 開頭
  if (authHeader == null || !authHeader.startsWith(BEARER_PREFIX)) {
    filterChain.doFilter(request, response);
    return;
  }

  String token = authHeader.substring(BEARER_PREFIX.length());

  // 驗證 token
  if (!jwtUtil.validateToken(token)) {
    logger.debug("Invalid JWT token");
    filterChain.doFilter(request, response);
    return;
  }

  try {
    // 檢查 token 是否在黑名單中
    String jti = jwtUtil.getJtiFromToken(token);
    if (accessTokenRepository.existsByTokenJti(jti)) {
      logger.debug("Token is blacklisted (logged out): {}", jti);
      filterChain.doFilter(request, response);
      return;
    }

    // 提取使用者 ID 並驗證使用者存在
    Long userId = jwtUtil.getUserIdFromToken(token);
    User user = userRepository.findById(userId).orElse(null);

    if (user == null || user.isDeleted()) {
      logger.debug("User not found or deleted: {}", userId);
      filterChain.doFilter(request, response);
      return;
    }

    // 在 SecurityContext 中設定認證
    UsernamePasswordAuthenticationToken authentication =
        new UsernamePasswordAuthenticationToken(userId, null, Collections.emptyList());
    SecurityContextHolder.getContext().setAuthentication(authentication);

    logger.debug("Authenticated user: {}", userId);

  } catch (Exception e) {
    logger.error("Error processing JWT token", e);
  }

  filterChain.doFilter(request, response);
}
```

**說明:** 每個請求執行一次的過濾器,用於透過 JWT 進行認證。

**重點:**

- 繼承 `OncePerRequestFilter` - 保證每個請求只執行一次
- 從 `Authorization: Bearer <token>` header 提取 token
- 驗證 token 簽章和過期時間
- 檢查 token 黑名單 (用於登出)
- 驗證使用者存在且未被刪除
- 在 `SecurityContextHolder` 中設定認證
- 無論認證結果如何都繼續過濾鏈

**過濾器執行流程:**

```
請求 → JwtAuthenticationFilter
  ├─ 無 Authorization header → 繼續 (未認證)
  ├─ 無效的 token → 繼續 (未認證)
  ├─ 黑名單中的 token → 繼續 (未認證)
  ├─ 找不到使用者 → 繼續 (未認證)
  └─ 有效 token + 使用者存在 → 設定認證 → 繼續 (已認證)
    ↓
下一個過濾器 → Controller
    ↓
如果有 @PreAuthorize 或受保護端點:
  ├─ 已認證 → 允許
  └─ 未認證 → 401 Unauthorized
```

---

### 範例 5: Controller 中使用 SecurityContext

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java:70-77`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/controller/OrderController.java#L70-L77)

```java
private Long getCurrentUserId() {
  Authentication authentication = SecurityContextHolder
      .getContext()
      .getAuthentication();

  return (Long) authentication.getPrincipal();
}

@PostMapping
public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
  Long currentUserId = getCurrentUserId();
  var result = orderService.createOrder(currentUserId, request);
  // ...
}
```

**說明:** 從 SecurityContext 取得已認證的使用者 ID。

**重點:**

- `SecurityContextHolder.getContext()` - 執行緒本地儲存的安全上下文
- `getAuthentication()` - 當前認證物件
- `getPrincipal()` - 使用者 ID (在過濾器中設定)
- 在 controller 中呼叫,將使用者 ID 傳遞給 service

**SecurityContext 生命週期:**

```
請求開始
  ↓
JwtAuthenticationFilter 設定: SecurityContextHolder.getContext().setAuthentication(userId)
  ↓
Controller 呼叫: getCurrentUserId() → 從 SecurityContextHolder 讀取
  ↓
Service 層接收 userId
  ↓
請求結束 → SecurityContext 清除 (執行緒本地清理)
```

---

### 範例 6: Token 黑名單 (登出)

**黑名單實體:**

```java
@Entity
@Table(name = "access_tokens")
public class AccessToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "token_jti", nullable = false, unique = true)
    private String tokenJti;  // JWT ID

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```

**登出服務:**

```java
@Transactional
public LogoutResponse logout(String token) {
    String jti = jwtUtil.getJtiFromToken(token);
    Long userId = jwtUtil.getUserIdFromToken(token);
    LocalDateTime expiresAt = jwtUtil.getExpirationFromToken(token);

    // 將 token 加入黑名單
    AccessToken accessToken = new AccessToken(jti, userId, expiresAt);
    accessTokenRepository.save(accessToken);

    logger.info("User {} logged out, token blacklisted: {}", userId, jti);
    return new LogoutResponse("登出成功");
}
```

**過濾器中的黑名單檢查:**

```java
String jti = jwtUtil.getJtiFromToken(token);
if (accessTokenRepository.existsByTokenJti(jti)) {
    // Token 在黑名單中,拒絕存取
    filterChain.doFilter(request, response);
    return;
}
```

**說明:** Token 黑名單防止已登出的 token 被重複使用。

**為何需要:**

- JWT 是無狀態的,通常無法被「撤銷」
- 登出必須在伺服器端使 token 失效
- 每個請求都會檢查黑名單

**最佳化:**

```java
// 排程清理過期的 token
@Scheduled(cron = "0 0 * * * *")  // 每小時
@Transactional
public void cleanupExpiredTokens() {
    LocalDateTime now = LocalDateTime.now();
    accessTokenRepository.deleteByExpiresAtBefore(now);
    logger.info("Cleaned up expired tokens");
}
```

---

### 範例 7: CORS 設定

**檔案:** [`www_root/waterballsa-backend/src/main/java/waterballsa/config/SecurityConfig.java:37-56`](../../../www_root/waterballsa-backend/src/main/java/waterballsa/config/SecurityConfig.java#L37-L56)

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration configuration = new CorsConfiguration();
  configuration.setAllowedOriginPatterns(Arrays.asList(
      "http://localhost:3000",    // 前端開發
      "http://localhost:80",      // Nginx HTTP
      "https://localhost:443",    // Nginx HTTPS
      "https://*.ngrok-free.dev"  // Ngrok 網域
  ));
  configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
  configuration.setAllowedHeaders(Arrays.asList("*"));
  configuration.setAllowCredentials(true);  // 允許 cookies/auth headers
  configuration.setMaxAge(3600L);           // Preflight 快取 1 小時

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/**", configuration);  // 套用到所有路徑
  return source;
}
```

**說明:** CORS 設定允許前端呼叫後端 API。

**主要設定:**

- `setAllowedOriginPatterns` - 允許的前端來源 (支援萬用字元)
- `setAllowedMethods` - 允許的 HTTP 方法
- `setAllowedHeaders` - 允許的請求 headers (\*)
- `setAllowCredentials(true)` - 允許 Authorization header 和 cookies
- `setMaxAge` - Preflight 回應快取時間

**CORS preflight 請求:**

```http
OPTIONS /orders HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

→ 回應:
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: *
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
```

---

## 常見陷阱

### ❌ 不要: 在 JWT payload 中儲存敏感資料

```java
Jwts.builder()
    .claim("password", user.getPassword())  // 錯誤! JWT 沒有加密
    .claim("creditCard", "1234-5678-...")   // 錯誤! Payload 是可讀的
```

### ✅ 要: 只儲存非敏感的識別碼

```java
Jwts.builder()
    .subject(user.getId().toString())
    .claim("username", user.getUsername())
    .claim("role", user.getRole())
```

**原因:** JWT payload 是 Base64 編碼,不是加密。任何人都可以解碼並讀取它。

---

### ❌ 不要: 使用弱的密鑰

```java
String secret = "secret";  // 錯誤! 太短且太弱
SecretKey key = Keys.hmacShaKeyFor(secret.getBytes());
```

### ✅ 要: 使用強大的隨機密鑰

```java
// application.properties
jwt.secret=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6...

// HS256 最少需要 256 位元 (32 位元組)
@Value("${jwt.secret}")
private String secret;

SecretKey key = Keys.hmacShaKeyFor(secret.getBytes());
```

**原因:** HMAC-SHA256 需要至少 256 位元的密鑰才能確保安全性。

---

### ❌ 不要: 忘記檢查 token 黑名單

```java
if (jwtUtil.validateToken(token)) {
    // 遺漏黑名單檢查!
    setAuthentication(token);
}
```

### ✅ 要: 登出後永遠檢查黑名單

```java
if (jwtUtil.validateToken(token)) {
    String jti = jwtUtil.getJtiFromToken(token);
    if (!accessTokenRepository.existsByTokenJti(jti)) {
        setAuthentication(token);
    }
}
```

---

### ❌ 不要: 設定過長的過期時間

```java
long expirationMs = 30 * 24 * 60 * 60 * 1000;  // 30 天 - 太長了!
```

### ✅ 要: 使用合理的過期時間並搭配更新 token

```java
long expirationMs = 24 * 60 * 60 * 1000;  // 1 天
// 實作更新 token 以延長會話時間
```

**原因:** 長效 token 如果被洩露會增加安全風險。

---

## 安全性快速參考

### JWT 最佳實踐

✅ **要:**

- 使用強大的密鑰 (至少 256 位元)
- 設定合理的過期時間 (小時/天,不是月)
- 實作 token 黑名單用於登出
- 在正式環境使用 HTTPS
- 每個請求都驗證 token
- 檢查 token 簽章和過期時間

❌ **不要:**

- 在 payload 中儲存敏感資料
- 使用可預測的密鑰
- 不設定過期時間
- 不驗證就信任 token
- 在正式環境使用 HTTP

### CORS 常見問題

| 問題                      | 解決方法                                   |
| ------------------------- | ------------------------------------------ |
| "CORS policy blocked"     | 將前端來源加入 `allowedOrigins`            |
| "Credentials not allowed" | 設定 `allowCredentials(true)`              |
| "Method not allowed"      | 將方法加入 `allowedMethods`                |
| "Header not allowed"      | 將 header 加入 `allowedHeaders` 或使用 `*` |
| Preflight 失敗            | 檢查 OPTIONS 方法是否被允許                |

---

## 搭配面試題目練習

- 📝 [題目 #8: SecurityContextHolder](../../interview/interview-backend.md#8-securitycontextholder-mid)
- 📝 [題目 #21: JWT Token 結構](../../interview/interview-backend.md#21-jwt-token-structure-mid)
- 📝 [題目 #22: OncePerRequestFilter](../../interview/interview-backend.md#22-onceperrequestfilter-mid)
- 📝 [題目 #23: JWT 黑名單](../../interview/interview-backend.md#23-jwt-blacklist-mid)
- 📝 [題目 #24: Token 驗證](../../interview/interview-backend.md#24-token-validation-mid)
- 📝 [題目 #25: CORS 設定](../../interview/interview-backend.md#25-cors-configuration-mid)

---

**上一篇:** [← 06. Transaction Management](06-transaction-management.md)

**下一篇:** [08. Error Handling](08-error-handling.md) →

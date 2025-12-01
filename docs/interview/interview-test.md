# E2E 測試面試題 - Waterball Software Academy 課程平台

> **面試準備說明**
> 本文件包含 40 道 E2E 測試相關的面試問題，專為中階全端工程師職位設計。
> 題目難度從 Junior 到 Mid-level，涵蓋前端 Playwright 測試與後端 REST Assured + Testcontainers 測試。

---

## 前端 E2E 測試 (Playwright) - 20 題

### 第一部分：Playwright 基礎配置 (Junior)

#### Q1. Playwright 配置檔案基礎

請解釋 `playwright.config.ts` 中以下配置的作用：

```typescript
fullyParallel: true,
forbidOnly: !!process.env.CI,
retries: process.env.CI ? 2 : 0,
workers: process.env.CI ? 1 : undefined,
```

為什麼 CI 環境和本地開發環境的配置會不同？

#### Q2. WebServer 配置

在 `playwright.config.ts` 中有以下配置：

```typescript
webServer: {
  command: 'echo "Using existing Docker services"',
  url: 'http://localhost',
  reuseExistingServer: true,
  timeout: 120 * 1000,
}
```

為什麼這裡使用 `echo` 命令而不是真的啟動伺服器？`reuseExistingServer: true` 的目的是什麼？

#### Q3. 測試追蹤與截圖設定

請說明以下配置的作用與使用時機：

```typescript
trace: 'on-first-retry',
screenshot: 'only-on-failure',
```

---

### 第二部分：測試結構與組織 (Junior/Mid)

#### Q4. test.describe 與 test.describe.serial 的差異

在 `mission-progress-flow.spec.ts` 中使用了 `test.describe.serial`：

```typescript
test.describe.serial('Complete user journey for video mission', () => {
	let credentials: UserCredentials
	let durationSeconds: number
	// 三個連續的測試...
})
```

為什麼這裡使用 `serial` 而不是一般的 `describe`？什麼情境下應該使用 serial tests？

#### Q5. test.step 的作用

在測試中大量使用了 `test.step`，例如：

```typescript
await test.step('Register and login new user', async () => {
	credentials = await registerAndLogin(request)
	// ...
})
```

使用 `test.step` 有什麼好處？對測試報告有什麼影響？

#### Q6. 測試數據隔離

在 `auth-flow.spec.ts` 中，每次測試都會產生唯一的使用者名稱：

```typescript
const timestamp = Date.now()
const testUsername = `testuser_${timestamp}`
```

為什麼需要這樣做？如果使用固定的測試帳號會有什麼問題？

#### Q7. Helper Functions 的設計

在 `mission-progress-flow.spec.ts` 中定義了多個 helper functions（`registerAndLogin`, `getMissionDuration`, `updateProgress` 等）。請說明這種設計的優缺點，以及何時應該將測試邏輯抽取成 helper function？

---

### 第三部分：選擇器與互動 (Junior/Mid)

#### Q8. 選擇器策略

在 `auth-flow.spec.ts` 中使用了不同的選擇器：

```typescript
page.getByRole('link', { name: '登入' })
page.getByPlaceholder('請輸入使用者名稱')
page.getByRole('button', { name: '註冊' })
page.locator('header')
```

請說明這些選擇器的差異，以及選擇使用哪種選擇器的考量點是什麼？

#### Q9. 避免選擇器衝突

在登入測試中有這段程式碼：

```typescript
await page.locator('form').getByRole('button', { name: '登入' }).click()
```

為什麼不直接使用 `page.getByRole('button', { name: '登入' })`？

#### Q10. 等待機制 - waitForLoadState

請解釋以下程式碼中 `waitForLoadState('networkidle')` 的作用：

```typescript
await page.goto('/')
await page.waitForLoadState('networkidle')
const loginButton = page.getByRole('link', { name: '登入' })
await expect(loginButton).toBeVisible()
```

什麼時候需要使用 `waitForLoadState`？有哪些不同的 load state 可以選擇？

#### Q11. toBeVisible 與 timeout

在測試中經常看到：

```typescript
await expect(page.getByText('註冊成功！')).toBeVisible({ timeout: 10000 })
```

為什麼需要設定 `timeout: 10000`？預設的 timeout 是多少？什麼情況下需要調整 timeout？

#### Q12. Cookie 操作

在 `mission-progress-flow.spec.ts` 中，測試會手動設定 cookies：

```typescript
await context.addCookies([
	{
		name: 'auth_token',
		value: credentials.authToken,
		domain: 'localhost',
		path: '/'
	}
	// ...
])
```

為什麼需要手動設定 cookies？這種做法和透過 UI 登入有什麼差異？

---

### 第四部分：API 整合測試 (Mid)

#### Q13. APIRequestContext 的使用

在 `mission-progress-flow.spec.ts` 中混合使用了 API 請求和 UI 操作：

```typescript
async function registerAndLogin(request: APIRequestContext): Promise<UserCredentials> {
	await request.post(`${API_BASE_URL}/auth/register`, {
		data: { username, password }
	})
	// ...
}
```

為什麼要在 E2E 測試中使用 API 請求？這樣做的優缺點是什麼？

#### Q14. 測試數據準備策略

在任務進度測試中，先透過 API 建立測試數據，再測試 UI：

```typescript
await test.step('Reset progress to 0 via API', async () => {
	await updateProgress(request, credentials.userId, credentials.authToken, 0)
})
await test.step('Navigate to mission page', async () => {
	await page.goto(MISSION_URL)
})
```

這種「API 準備數據 + UI 驗證」的模式有什麼好處？

#### Q15. Interface 型別定義

測試檔案中定義了多個 interfaces：

```typescript
interface UserCredentials {
	userId: number
	username: string
	password: string
	authToken: string
}
```

為什麼要在測試中定義這些 interfaces？對測試的可維護性有什麼幫助？

#### Q16. API Response 驗證

在測試中會驗證 API response：

```typescript
const response = await updateProgress(/* ... */)
const data = await response.json()
expect(data.status).toBe('COMPLETED')
```

為什麼在 E2E 測試中還要驗證 API response？這不是應該由 API 測試負責嗎？

---

### 第五部分：進階測試模式 (Mid)

#### Q17. 處理 Flaky Tests

在 `auth-flow.spec.ts` 的註解中提到：

```typescript
// Clear all cookies to ensure clean state (fixes flaky test)
await context.clearCookies()
```

什麼是 flaky test？為什麼清除 cookies 可以修復 flaky test？

#### Q18. 狀態不可逆測試

在任務進度測試中驗證了狀態不會倒退：

```typescript
// Step 6: Verify status does NOT revert (still COMPLETED)
await test.step('Verify status remains COMPLETED after progress reset', async () => {
	const progress = await getProgress(/* ... */)
	expect(progress.status).toBe('COMPLETED')
})
```

為什麼要測試「重置進度後狀態仍維持 COMPLETED」？這種測試在驗證什麼業務邏輯？

#### Q19. 側邊欄圖示狀態驗證

測試中驗證了不同狀態下的圖示變化：

```typescript
// UNCOMPLETED: dashed circle
const circleIcon = missionLink.locator('svg.lucide-circle')
await expect(circleIcon).toBeVisible()

// COMPLETED: checkmark (no stroke-dasharray)
const dashedCircleIcon = missionLink.locator('svg[stroke-dasharray="3 3"]')
await expect(dashedCircleIcon).not.toBeVisible()

// DELIVERED: green checkmark
const greenCheckIcon = missionLink.locator('svg.text-green-500')
await expect(greenCheckIcon).toBeVisible()
```

請解釋這段測試的邏輯。為什麼要用這種方式驗證圖示狀態？

#### Q20. 完整購買流程測試

在 `purchase-flow.spec.ts` 中，一個測試涵蓋了從註冊到上課的完整流程。請說明這種「長流程測試」的優缺點，以及在什麼情況下應該將測試拆分成多個獨立的測試？

---

## 後端 E2E 測試 (REST Assured + Testcontainers) - 20 題

### 第一部分：Testcontainers 基礎 (Junior)

#### Q21. PostgreSQL Container 的 Singleton Pattern

在 `BaseE2ETest.java` 中使用了 static 初始化區塊：

```java
static final PostgreSQLContainer<?> postgres;

static {
  @SuppressWarnings("resource")
  PostgreSQLContainer<?> container = new PostgreSQLContainer<>("postgres:15-alpine")
      .withDatabaseName("e2e_test")
      .withUsername("test")
      .withPassword("test");
  container.start();
  postgres = container;
}
```

為什麼要使用這種 singleton pattern？如果每個測試類別都建立自己的 container 會有什麼問題？

#### Q22. @SuppressWarnings("resource")

請解釋程式碼中的 `@SuppressWarnings("resource")` 是什麼意思？為什麼需要這個 annotation？

#### Q23. DynamicPropertySource 的作用

```java
@DynamicPropertySource
static void configureProperties(DynamicPropertyRegistry registry) {
  registry.add("spring.datasource.url", postgres::getJdbcUrl);
  registry.add("spring.datasource.username", postgres::getUsername);
  registry.add("spring.datasource.password", postgres::getPassword);
  registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
  registry.add("spring.liquibase.enabled", () -> "true");
}
```

`@DynamicPropertySource` 的作用是什麼？為什麼不能在 `application.properties` 中直接設定這些屬性？

---

### 第二部分：REST Assured 基礎 (Junior/Mid)

#### Q24. Given-When-Then 語法結構

請解釋 REST Assured 的 given-when-then 語法：

```java
given()
    .contentType(ContentType.JSON)
    .body(requestBody)
.when()
    .post("/auth/register")
.then()
    .statusCode(201)
    .body("message", equalTo("Registration successful"));
```

每個部分的職責是什麼？

#### Q25. ContentType 與 Request Body 格式化

在測試中使用了 Text Blocks 來格式化 JSON：

```java
String requestBody = """
    {
        "username": "%s",
        "password": "%s"
    }
    """;
given()
    .contentType(ContentType.JSON)
    .body(String.format(requestBody, username, password))
```

為什麼要設定 `ContentType.JSON`？如果不設定會發生什麼事？

#### Q26. JsonPath 提取 Response 資料

```java
Long userId = given()
    .contentType(ContentType.JSON)
    .body(requestBody)
.when()
    .post("/auth/register")
.then()
    .statusCode(201)
    .extract()
    .jsonPath()
    .getLong("userId");
```

請說明 `.extract().jsonPath().getLong("userId")` 這段程式碼的作用。還有哪些常用的提取方法？

#### Q27. Hamcrest Matchers

測試中使用了各種 Hamcrest matchers：

```java
.body("id", notNullValue())
.body("userId", equalTo(userId.intValue()))
.body("items", hasSize(1))
.body("paidAt", nullValue())
```

請說明這些 matchers 的作用。為什麼要使用 matchers 而不是直接提取值後用 assertEquals 驗證？

---

### 第三部分：測試組織與 Helper Methods (Mid)

#### Q28. @Nested 測試類別的使用

在 `OrderE2ETest.java` 中大量使用了 `@Nested`：

```java
@Nested
@DisplayName("POST /orders")
class CreateOrderTests {
  @Test
  @DisplayName("Should successfully create a new order")
  void shouldCreateNewOrderSuccessfully() { /* ... */ }
}
```

使用 `@Nested` 有什麼好處？對測試報告有什麼影響？

#### Q29. @BeforeEach 與測試隔離

```java
@BeforeEach
void setUp() {
  username = "testuser_" + System.currentTimeMillis();
  String password = "Test1234!";
  userId = registerUser(username, password);
  userToken = loginAndGetToken(username, password);
}
```

為什麼要在 `@BeforeEach` 中建立新使用者？這對測試隔離有什麼幫助？

#### Q30. Helper Methods 的設計原則

在 `BaseE2ETest` 中定義了多個 helper methods：

```java
protected String loginAndGetToken(String username, String password) { /* ... */ }
protected Long registerUser(String username, String password) { /* ... */ }
protected String bearerToken(String token) { return "Bearer " + token; }
```

請說明這些 helper methods 的設計考量。什麼樣的邏輯應該抽取成 helper method？

---

### 第四部分：複雜測試場景 (Mid)

#### Q31. @Sql 測試數據管理

```java
@Sql(
    scripts = {"/test-data/cleanup.sql", "/test-data/orders.sql"},
    executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(
    scripts = "/test-data/cleanup.sql",
    executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class OrderE2ETest extends BaseE2ETest {
```

`@Sql` annotation 的作用是什麼？為什麼要在測試前後都執行 cleanup.sql？

#### Q32. 訂單編號格式驗證

測試中驗證了訂單編號的格式：

```java
// Order number format: {timestamp:10 digits}{userId}{randomCode:5 chars}
String userIdStr = userId.toString();
int expectedMinLength = 10 + userIdStr.length() + 5;

assertThat(orderNumber, containsString(userIdStr));
assertThat(orderNumber.length(), greaterThanOrEqualTo(expectedMinLength));
assertThat(orderNumber.substring(0, 10), matchesPattern("\\d{10}"));
```

為什麼要驗證訂單編號的格式？這種驗證在測試什麼？

#### Q33. 價格鎖定測試

```java
@Test
@DisplayName("Should lock price at order creation time")
void shouldLockPriceAtOrderCreation() {
  // Create order with original price 1999.00
  Response createResponse = given()
      .header("Authorization", bearerToken(userToken))
      .contentType(ContentType.JSON)
      .body(requestBody)
  .when()
      .post("/orders")
  .then()
      .statusCode(201)
      .body("items[0].originalPrice", equalTo(1999.00f))
      .extract()
      .response();

  // Verify order still has locked price
  given()
      .header("Authorization", bearerToken(userToken))
  .when()
      .get("/orders/{orderId}", orderId)
  .then()
      .statusCode(200)
      .body("items[0].originalPrice", equalTo(1999.00f));
}
```

這個測試在驗證什麼業務邏輯？為什麼價格鎖定很重要？

#### Q34. 資料庫直接驗證

在測試中直接查詢資料庫：

```java
@Autowired private JdbcTemplate jdbcTemplate;

Integer count = jdbcTemplate.queryForObject(
    "SELECT COUNT(*) FROM user_journeys WHERE user_id = ? AND journey_id = ? AND order_id = ?",
    Integer.class,
    userId, journeyId, orderId);

assertThat(count, equalTo(1));
```

為什麼要直接查詢資料庫？什麼時候應該用 API 驗證，什麼時候應該查詢資料庫？

#### Q35. 訂單過期測試

```java
private void expireOrderInDatabase(Long orderId) {
  jdbcTemplate.update("UPDATE orders SET status = 'EXPIRED' WHERE id = ?", orderId);
}

@Test
void shouldFailToPayExpiredOrder() {
  // Create order
  Long orderId = /* ... */;

  // Manually expire the order
  expireOrderInDatabase(orderId);

  // Attempt to pay expired order
  given()
      .header("Authorization", bearerToken(userToken))
  .when()
      .post("/orders/{orderId}/action/pay", orderId)
  .then()
      .statusCode(409)
      .body("error", containsString("訂單已過期"));
}
```

這個測試直接修改資料庫狀態來模擬過期。請說明這種測試策略的優缺點。

---

### 第五部分：認證與授權測試 (Mid)

#### Q36. Bearer Token 管理

```java
protected String bearerToken(String token) {
  return "Bearer " + token;
}

given()
    .header("Authorization", bearerToken(token))
.when()
    .post("/auth/logout")
```

為什麼要把 `bearerToken` 抽取成 helper method？JWT token 在測試中應該如何管理？

#### Q37. 多用戶測試場景

```java
@Test
void shouldFailWhenAccessingOthersOrder() {
  // Create order with first user
  Long orderId = /* user1 creates order */;

  // Register second user
  String secondUsername = "testuser2_" + System.currentTimeMillis();
  registerUser(secondUsername, secondPassword);
  String secondUserToken = loginAndGetToken(secondUsername, secondPassword);

  // Try to access first user's order
  given()
      .header("Authorization", bearerToken(secondUserToken))
  .when()
      .get("/orders/{orderId}", orderId)
  .then()
      .statusCode(404);
}
```

這個測試在驗證什麼？為什麼回傳 404 而不是 403？

#### Q38. Token 失效測試

```java
@Test
void shouldFailToLogoutTwiceWithSameToken() {
  String token = loginAndGetToken(username, password);

  // First logout succeeds
  given()
      .header("Authorization", bearerToken(token))
  .when()
      .post("/auth/logout")
  .then()
      .statusCode(200);

  // Second logout with same token should fail
  given()
      .header("Authorization", bearerToken(token))
  .when()
      .post("/auth/logout")
  .then()
      .statusCode(401);
}
```

這個測試在驗證什麼安全機制？為什麼同一個 token 不能重複登出？

#### Q39. 完整購買流程驗證

```java
@Test
@DisplayName("Should complete full purchase flow: create -> query -> pay -> verify")
void shouldCompleteFullPurchaseFlow() {
  // Step 1: Create order
  // Step 2: Query order detail
  // Step 3: Complete payment
  // Step 4: Verify order is paid
  // Step 5: Verify journey access granted
}
```

這個測試包含了 5 個步驟。請說明為什麼要寫成一個測試而不是 5 個獨立的測試？這種設計的優缺點是什麼？

#### Q40. 並發測試場景

```java
@Test
void shouldHandleMultipleUsersConcurrentOrders() {
  // User 1: Purchase journey 1
  Response user1Response = given()
      .header("Authorization", bearerToken(userToken))
      .body(user1RequestBody)
  .when()
      .post("/orders")
  .then()
      .extract().response();

  // User 2: Purchase journey 2
  String user2Token = loginAndGetToken(user2Username, user2Password);
  Response user2Response = given()
      .header("Authorization", bearerToken(user2Token))
      .body(user2RequestBody)
  .when()
      .post("/orders")
  .then()
      .extract().response();

  // Verify order numbers are unique
  assertThat(user1OrderNumber, not(equalTo(user2OrderNumber)));
}
```

這個測試在驗證什麼？在實際的並發場景中，這個測試能發現什麼問題？

---

## 面試準備建議

1. **理解每一行程式碼**：面試時可能會被要求解釋任何一段測試程式碼的細節
2. **準備實際案例**：思考在開發過程中遇到的測試問題和解決方案
3. **比較不同做法**：了解每種測試策略的優缺點和使用時機
4. **關注測試品質**：思考如何寫出可維護、可靠、有意義的測試
5. **連結業務邏輯**：理解測試背後要驗證的業務需求

祝你面試順利！🚀

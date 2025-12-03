# 後端面試題解答

## 第一部分：Java 語法基礎與設計模式 (Junior)

### Q1. Sealed Interface 語法解析 (Junior)

**解答**:

1. `sealed interface` 是 Java 17 引入的新特性,它可以限制哪些類別或介面可以實作或繼承它。跟普通 interface 不同的地方在於,sealed interface 明確定義了「只有這些類別可以實作我」,其他人都不行。像這個例子中,只有 `Created` 和 `Existing` 這兩個 record 可以實作 `OrderCreationResult`,其他任何類別都不能實作。

   這樣做的好處是讓編譯器可以幫我們做窮舉檢查。當我們用 pattern matching 處理這個 interface 時,編譯器知道只會有這兩種可能,如果我們漏了某個 case,編譯器會警告我們。

2. 使用 `record` 而不是 `class` 主要有幾個原因。首先,這兩個類別都只是用來「包裝資料」的,沒有複雜的 business logic,record 簡潔又清楚。其次,record 自動提供了 immutability、`equals()`、`hashCode()`、`toString()` 這些方法,我們不用自己寫。最後,record 的語意很明確:「這就是一個 data carrier」,讓程式碼更容易理解。

3. 這個設計模式解決的問題是「如何優雅地表達兩種不同的結果」。傳統作法可能會用一個 boolean 欄位 `isNewOrder`,或是回傳 null 代表某種情況,但這些方式都不夠清楚。用 sealed interface 搭配 record,每一種結果都有明確的型別,讓 API 的語意非常清楚:「建立訂單可能會產生新訂單,也可能回傳已存在的訂單」。

4. 在 Controller 層使用時,我們會用 pattern matching 的 `instanceof` 來判斷結果型別。如果是 `Created`,就回傳 `201 CREATED` status code;如果是 `Existing`,就回傳 `200 OK`。這種寫法比傳統的 if-else 加上型別轉換簡潔很多,而且因為 sealed interface 的限制,編譯器可以確保我們處理了所有可能的情況。

---

### Q2. Pattern Matching 與 instanceof (Mid)

**解答**:

1. 這種新語法叫做「Pattern Matching for instanceof」,是 Java 16 引入的功能。傳統的寫法需要先用 `instanceof` 檢查型別,然後再手動做型別轉換,像這樣:

   ```java
   if (result instanceof OrderService.OrderCreationResult.Created) {
     OrderService.OrderCreationResult.Created created = (OrderService.OrderCreationResult.Created) result;
     return ResponseEntity.status(HttpStatus.CREATED).body(created.orderResponse());
   }
   ```

   新語法讓我們在檢查型別的同時就完成轉換,並宣告一個新變數 `created`,一行搞定。這樣不僅程式碼更簡潔,也更安全,因為編譯器會確保 `created` 變數只在型別檢查通過的 scope 內有效。

2. 回傳不同的 HTTP status code 是為了遵循 REST API 的語意規範。`201 CREATED` 表示「成功建立了一個新資源」,讓前端知道這是一個全新的訂單;`200 OK` 表示「請求成功,但沒有建立新資源」,代表回傳的是已經存在的訂單。

   這種語意上的區別很重要。比如說,如果前端在處理訂單建立後需要顯示「訂單建立成功」的提示訊息,他們可以根據 status code 來決定要不要顯示。如果是 `201`,就顯示「訂單已建立」;如果是 `200`,可能就顯示「您已有未付款的訂單」。

3. 對前端來說,這樣的設計有幾個好處。首先,前端不需要解析 response body 就能知道發生了什麼事,直接看 status code 就行。其次,這讓 API 的行為更符合預期:多次呼叫同一個請求會得到一致的結果(idempotent),不會重複建立訂單。最後,前端可以根據不同的 status code 做不同的 UI 處理,提升使用者體驗。

---

### Q3. Stream API 與 Lambda 表達式 (Junior)

**解答**:

1. 這段程式碼的每個操作是這樣運作的:

   - `stream()`: 把 List 轉換成 Stream,讓我們可以用函數式的方式處理資料
   - `filter(mission -> !mission.isDeleted())`: 過濾掉已刪除的任務,只保留沒被刪除的
   - `sorted(Comparator.comparing(Mission::getOrderIndex))`: 根據任務的 `orderIndex` 欄位做排序
   - `map(this::mapToMissionSummaryDTO)`: 把每個 Mission entity 轉換成 MissionSummaryDTO
   - `collect(Collectors.toList())`: 把處理後的 Stream 收集成 List

2. `Comparator.comparing(Mission::getOrderIndex)` 這種寫法叫做 Method Reference(方法參照)。`Mission::getOrderIndex` 相當於 `mission -> mission.getOrderIndex()`,是一種更簡潔的寫法。當 lambda 表達式只是呼叫某個方法並回傳結果時,就可以用 method reference 來簡化。

3. `this::mapToMissionSummaryDTO` 也是 method reference,代表「呼叫這個物件的 `mapToMissionSummaryDTO` 方法」。等同於 `mission -> this.mapToMissionSummaryDTO(mission)`,但更簡潔。

4. 如果不用 Stream API,傳統寫法會是這樣:

   ```java
   List<MissionSummaryDTO> missions = new ArrayList<>();
   for (Mission mission : chapter.getMissions()) {
     if (!mission.isDeleted()) {
       missions.add(this.mapToMissionSummaryDTO(mission));
     }
   }
   Collections.sort(missions, Comparator.comparing(MissionSummaryDTO::getOrderIndex));
   ```

   兩種寫法各有優缺點。Stream API 更簡潔、更容易平行化處理,而且鏈式呼叫的風格讓資料處理的流程一目了然。但傳統 for loop 對初學者來說可能更直觀,而且在某些情況下效能會稍微好一點點。以這個例子來說,Stream API 的寫法更好,因為程式碼更清楚,而且效能差異微乎其微。

---

### Q4. Enum 與 JPA 整合 (Junior)

**解答**:

1. `EnumType.STRING` 會把 enum 的名稱(如 "UNPAID", "PAID")存到資料庫;`EnumType.ORDINAL` 則是存 enum 的順序編號(0, 1, 2...)。

   使用 `STRING` 的好處是資料庫裡的值有意義,一看就懂,而且如果你在 enum 中間插入新的值,不會影響既有資料。用 `ORDINAL` 的話,資料庫裡都是數字,很難 debug,更糟的是,如果你在 enum 前面加入新的值,順序就亂了,可能導致資料錯亂。所以除非有特殊理由(如空間限制),都應該用 `STRING`。

2. `@JdbcTypeCode(SqlTypes.NAMED_ENUM)` 是告訴 Hibernate 要使用 PostgreSQL 的 enum 型別,而不是普通的 varchar。PostgreSQL 有專門的 enum type,可以提供更好的型別安全和效能。如果不加這個註解,Hibernate 會把它當成 varchar 來處理,雖然也能用,但就失去了資料庫層的型別檢查。

3. 如果你新增一個 Enum 值,比如加入 `REFUNDED`,只要資料庫的 enum type 也同步更新,就沒問題。但要注意的是,PostgreSQL 的 enum type 需要用 `ALTER TYPE` 來新增值,而且順序很重要。如果沒有正確更新資料庫 schema,應用程式在儲存新的 enum 值時會出錯。

4. 如果你刪除一個 Enum 值,就很麻煩了。如果資料庫裡已經有資料使用這個值,你不能直接從 Java code 刪掉,不然讀取資料時會拋出例外,因為 JPA 無法把資料庫的值對應到 Java enum。正確的做法是:先確保沒有資料使用這個值(或把它們遷移到其他狀態),然後才能刪除。這也是為什麼在設計 enum 時要特別謹慎,一旦有資料使用就很難改。

---

### Q5. Optional 的使用時機 (Junior)

**解答**:

1. Repository 方法回傳 `Optional<Order>` 而不是直接回傳 `Order`,是為了明確表達「這個查詢可能找不到資料」。如果直接回傳 `Order`,當找不到資料時你要回傳什麼?回傳 `null` 嗎?那使用者就必須記得檢查 null,很容易忘記,造成 NullPointerException。

   用 `Optional` 的好處是「強迫」使用者意識到這個值可能不存在,編譯器和 IDE 會提醒你要處理「找不到」的情況。這讓程式碼更安全,也更清楚表達 API 的語意。

2. 在 Service 層使用 Optional 時,通常會用這幾種方式:

   ```java
   // 方式 1: 如果找不到就拋出例外
   Order order = orderRepository.findByIdAndUserId(id, userId)
       .orElseThrow(() -> new OrderNotFoundException(id));

   // 方式 2: 提供預設值
   Order order = orderRepository.findByIdAndUserId(id, userId)
       .orElse(createDefaultOrder());

   // 方式 3: 用 ifPresent 處理
   orderRepository.findByIdAndUserId(id, userId)
       .ifPresent(order -> { /* do something */ });
   ```

   要避免的是用 `get()` 直接取值而不檢查,這跟使用 null 一樣危險。

3. Optional 不應該用在以下情況:

   - 方法參數:不要寫 `void doSomething(Optional<Order> order)`,直接用 `Order` 並允許 null 比較好
   - 類別的欄位:不要在 Entity 或 DTO 中宣告 `Optional<String> name`,直接用 nullable 型別就好
   - 回傳 Collection:不要回傳 `Optional<List<Order>>`,直接回傳空 List 比較自然

   Optional 主要用在「回傳單一值,且可能不存在」的情況,其他情境用 Optional 反而會讓程式碼變得複雜。

---

## 第二部分：Spring Boot 架構與注入 (Junior/Mid)

### Q6. Controller 層的職責 (Junior)

**解答**:

1. Controller 層應該負責的事情包括:接收 HTTP 請求、驗證輸入格式(透過 `@Valid`)、呼叫對應的 Service 方法、把 Service 的結果轉換成 HTTP Response(設定 status code、headers 等)、處理當前使用者的身份認證資訊。簡單來說,Controller 就是「HTTP 層的轉接器」,負責 HTTP 協定相關的事情。

   Controller 不應該負責:business logic 的實作、直接操作 Repository、複雜的資料轉換、資料驗證的 business rule(如「訂單是否過期」這種業務邏輯)。這些都應該在 Service 層處理。

2. 所有 business logic 都放在 Service 層,是為了「關注點分離」。Controller 專注於 HTTP 協定,Service 專注於業務邏輯,Repository 專注於資料存取。這樣做的好處是:程式碼更容易測試(測試 Service 不需要模擬 HTTP request)、更容易維護(改 business logic 不影響 Controller)、更容易重用(同一個 Service 可以被多個 Controller 或排程任務使用)。

3. `@RestController` 是 `@Controller` + `@ResponseBody` 的組合。`@Controller` 表示這是一個 MVC controller,但預設會回傳 View(如 JSP 頁面);`@RestController` 表示這是一個 RESTful API controller,所有方法的回傳值都會直接序列化成 JSON(或 XML)回傳給前端,不會去找 View。

   現代的前後端分離架構都用 `@RestController`,因為後端只負責提供 API,前端用 React/Vue 等框架自己渲染畫面。

4. `@RequestMapping("/orders")` 的作用是為這個 Controller 的所有方法設定共同的 URL 前綴。比如說,裡面有個方法加上 `@PostMapping`,實際的 endpoint 就是 `POST /orders`;如果有個方法加上 `@GetMapping("/{id}")`,實際 endpoint 就是 `GET /orders/{id}`。這樣可以避免在每個方法都重複寫 `/orders`。

---

### Q7. 依賴注入的方式 (Junior)

**解答**:

1. 這是「Constructor Injection」(建構子注入),是 Spring 推薦的依賴注入方式。透過建構子接收依賴,並賦值給 final 欄位。

2. 從 Spring 4.3 開始,如果一個類別只有一個建構子,Spring 會自動進行注入,不需要加 `@Autowired`。而且,省略 `@Autowired` 讓程式碼更簡潔,減少 Spring 相關的 annotation,讓類別更「POJO」,更容易在非 Spring 環境下測試。

3. Constructor Injection 的優點有很多:

   - 可以使用 `final` 欄位,確保依賴不可變,更安全
   - 依賴關係很明確,一看建構子就知道這個類別需要哪些依賴
   - 強迫開發者提供所有必要的依賴,避免遺漏
   - 更容易寫單元測試,直接 `new OrderController(mockService)` 就能測試,不需要 Spring container

4. 其他依賴注入方式包括:

   - **Field Injection**: 直接在欄位上加 `@Autowired`,最簡潔但不推薦,因為無法使用 final、難以測試、隱藏了依賴關係

   ```java
   @Autowired
   private OrderService orderService;
   ```

   - **Setter Injection**: 透過 setter 方法注入,適合 optional 的依賴,但很少用

   ```java
   @Autowired
   public void setOrderService(OrderService orderService) {
     this.orderService = orderService;
   }
   ```

   現代 Spring 開發幾乎都用 Constructor Injection,除非有特殊需求。

---

### Q8. SecurityContextHolder 的運作原理 (Mid)

**解答**:

1. `SecurityContextHolder` 是 Spring Security 用來存放當前使用者認證資訊的地方。它的資料是在「JWT Filter」中設定的。當請求進來時,JwtAuthenticationFilter 會從 HTTP header 取出 JWT token,驗證 token 是否有效,然後把使用者 ID 包裝成 `Authentication` 物件,存到 `SecurityContextHolder` 中。之後在 Controller 或 Service 裡就可以隨時取出當前使用者的資訊。

2. `Authentication.getPrincipal()` 通常會回傳 `UserDetails` 物件或使用者名稱字串,但這個專案自訂了認證流程,直接把使用者的 `Long` ID 當作 principal 存進去。這樣做的好處是效能更好,不需要每次都從資料庫查詢完整的使用者資訊,只要有 ID 就夠了。

   在 JwtAuthenticationFilter 裡,會建立一個 `Authentication` 物件,把 userId 設為 principal,然後存到 SecurityContext 中。

3. 這個方法在多執行緒環境下是安全的。`SecurityContextHolder` 預設使用 `ThreadLocal` 來存放 context,每個執行緒都有自己獨立的 SecurityContext。所以即使多個請求同時進來,每個執行緒都會拿到屬於自己的使用者資訊,不會互相干擾。

   但要注意的是,如果你在 async 方法或手動建立的執行緒中使用,就需要特別處理,因為 ThreadLocal 不會自動傳遞到新的執行緒。

4. 如果使用者未登入,這個方法會拋出 NullPointerException。因為未登入的請求,`authentication` 會是 null,或者 `authentication.getPrincipal()` 會是 null。這就是為什麼程式碼用 `Objects.requireNonNull()` 來確保不是 null,如果是 null 就拋出清楚的錯誤訊息。

   實際上,對於需要登入的 endpoint,Spring Security 的配置會在 filter chain 中就把未登入的請求擋掉,不會讓它到達 Controller,所以這個方法理論上不會遇到 null 的情況。

---

### Q9. @Valid 與 Bean Validation (Junior)

**解答**:

1. `@Valid` 註解的作用是告訴 Spring:「請驗證這個物件的欄位是否符合規則」。Spring 會根據 DTO 類別上定義的 validation annotation(如 `@NotNull`、`@NotBlank`、`@Email` 等)來檢查輸入資料。如果驗證通過,就繼續執行方法;如果驗證失敗,就拋出例外。

2. 驗證失敗時,Spring 會拋出 `MethodArgumentNotValidException` 例外。這個例外不會直接回傳給前端,而是會被 GlobalExceptionHandler 攔截。在 exception handler 裡,我們可以提取出所有驗證錯誤的欄位和訊息,包裝成友善的錯誤回應回傳給前端,通常會用 `400 Bad Request` 的 status code。

3. 錯誤訊息的處理流程是這樣的:

   - Spring 發現驗證失敗,拋出 `MethodArgumentNotValidException`
   - GlobalExceptionHandler 有一個 `@ExceptionHandler(MethodArgumentNotValidException.class)` 方法會接住這個例外
   - Handler 方法會遍歷所有的 `FieldError`,取出欄位名稱和錯誤訊息
   - 把這些錯誤包裝成一個統一的 `ErrorResponse` 格式,回傳給前端

   這樣前端就能知道哪些欄位有問題,並在 UI 上顯示對應的錯誤訊息。

4. 在 DTO 上定義驗證規則很簡單,只要在欄位上加上對應的 annotation:

   ```java
   public record CreateOrderRequest(
       @NotNull(message = "課程 ID 不能為空")
       Long journeyId,

       @NotBlank(message = "使用者名稱不能為空")
       @Size(min = 3, max = 20, message = "使用者名稱長度必須在 3-20 之間")
       String username,

       @Email(message = "Email 格式不正確")
       String email
   ) {}
   ```

   常用的 validation annotation 還有 `@Min`、`@Max`、`@Pattern`、`@Positive` 等,可以涵蓋大部分的驗證需求。

---

### Q10. Logger 的使用原則 (Junior)

**解答**:

1. Logger 宣告為 `private static final` 有幾個原因:

   - `private`: 這個 logger 是給當前類別內部使用的,不應該被外部存取
   - `static`: Logger 本身是 thread-safe 的,而且不需要依賴任何實例狀態,做成 static 可以節省記憶體,所有實例共用同一個 logger
   - `final`: Logger 初始化後不應該被改變,用 final 確保不可變性

   這是業界的標準寫法,幾乎所有 Java 專案都這樣宣告 logger。

2. 不同 log level 的使用時機:

   - `logger.debug()`: 開發時用的詳細資訊,像是「進入某個方法」、「變數的值」,生產環境通常不會記錄
   - `logger.info()`: 重要的業務流程資訊,像是「建立訂單成功」、「使用者登入」,生產環境會記錄,幫助追蹤系統運作
   - `logger.warn()`: 潛在的問題,像是「使用了棄用的功能」、「某個資源快用完了」,不影響當前操作但需要注意
   - `logger.error()`: 發生錯誤了,像是「資料庫連線失敗」、「拋出例外」,需要立即處理

3. 不用 `String.format()` 或字串相加,而是用 `logger.debug("Creating order for user: {}", userId)` 這種語法,主要原因是效能。使用 `{}` placeholder 的方式,如果 log level 不夠(比如生產環境把 debug 關掉了),logger 根本不會執行字串組合,可以省下不必要的運算。

   如果用字串相加 `logger.debug("Creating order for user: " + userId)`,即使 debug 被關掉,字串相加還是會執行,浪費效能。

4. 應該 log Exception stack trace 的時機是:當發生「非預期的錯誤」時。比如資料庫連線失敗、外部 API 呼叫失敗、程式邏輯錯誤等。記錄完整的 stack trace 可以幫助我們快速定位問題發生的位置。

   ```java
   try {
     // some operation
   } catch (Exception e) {
     logger.error("Failed to process order", e);  // 第二個參數傳入例外
   }
   ```

   但對於「預期的錯誤」,像是使用者輸入錯誤、找不到資料這種情況,只需要記錄訊息就好,不需要 stack trace,因為這不是系統錯誤,記錄太多反而會污染 log。

---

## 第三部分：JPA 與資料庫操作 (Mid)

### Q11. Entity Lifecycle Callbacks (Mid)

**解答**:

1. `@PrePersist` 是在 Entity 第一次被持久化(INSERT)到資料庫「之前」被呼叫;`@PreUpdate` 是在 Entity 更新(UPDATE)到資料庫「之前」被呼叫。注意是「之前」,所以你在這些 callback 裡做的修改會被包含在 SQL 語句中一起執行。

   比如說,當你呼叫 `orderRepository.save(order)` 儲存一個新的 order 時,JPA 會先執行 `onCreate()` 方法設定 `createdAt` 和 `updatedAt`,然後才執行 INSERT 語句。

2. 在 lifecycle callback 裡設定這些時間戳記,好處是「集中管理」和「不會忘記」。如果在建構子或 setter 裡設定,開發者可能會忘記呼叫,或是在不同地方有不一致的邏輯。把它放在 lifecycle callback,JPA 會自動幫你處理,無論是透過 Repository 或 EntityManager 操作,時間戳記都會被正確設定。

   而且這樣的設計符合「關注點分離」:Entity 負責維護自己的狀態,不用依賴外部程式碼來設定這些基本欄位。

3. 如果在 `@PrePersist` 中拋出例外,整個 transaction 會被 rollback。JPA 會把這個例外往上拋,最終可能會被 Spring 的 transaction manager 攔截,然後回滾所有資料庫操作。這是一個保護機制,確保不合法的 Entity 不會被儲存到資料庫。

   實務上,如果你在 `@PrePersist` 裡做驗證邏輯(雖然不建議,應該用 Bean Validation),當驗證失敗拋出例外時,資料就不會被寫入。

4. JPA 還有其他 Lifecycle Callbacks:

   - `@PostPersist`: INSERT 之後被呼叫,可以用來發送事件通知
   - `@PostUpdate`: UPDATE 之後被呼叫
   - `@PreRemove`: DELETE 之前被呼叫
   - `@PostRemove`: DELETE 之後被呼叫
   - `@PostLoad`: 從資料庫載入 Entity 之後被呼叫,可以用來做一些初始化或計算

   這些 callback 讓我們可以在 Entity 生命週期的各個階段插入自訂邏輯。

---

### Q12. FetchType.LAZY 的陷阱 (Mid)

**解答**:

1. `FetchType.LAZY` 表示「延遲載入」,只有在真正存取關聯物件時才會去資料庫查詢;`FetchType.EAGER` 表示「立即載入」,在查詢 Entity 的同時就會把關聯物件一起查出來。

   比如說,當你查詢一個 Mission 時,如果 chapter 是 LAZY,一開始只會 SELECT mission 的資料,當你呼叫 `mission.getChapter()` 時才會再執行一次 SELECT 查詢 chapter。如果是 EAGER,就會用 JOIN 一次把兩個都查出來。

2. **N+1 Query Problem** 是一個很常見的效能問題。假設你查詢了 N 個 missions:

   ```java
   List<Mission> missions = missionRepository.findAll();  // 1 次查詢
   for (Mission mission : missions) {
     System.out.println(mission.getChapter().getName());  // N 次查詢
   }
   ```

   第一行執行 1 次 SQL 查所有 missions,然後 loop 裡面每個 mission 存取 chapter 時又各執行 1 次 SQL,總共是 1 + N 次查詢。如果有 100 個 missions,就會執行 101 次查詢,效能非常差。

   避免的方法有兩種:使用 `JOIN FETCH` 在第一次查詢時就把關聯物件一起抓出來,或是用 `@EntityGraph` 註解指定要一起載入的關聯。

3. 如果在 `@Transactional` 外面存取 lazy loading 的關聯物件,會拋出 `LazyInitializationException`。因為 Hibernate 的 session 已經關閉了,無法再從資料庫載入資料。這是新手最常遇到的錯誤之一。

   解決方法是:要嘛確保在 `@Transactional` 方法內完成所有資料存取,要嘛在 Service 層就把需要的資料轉換成 DTO,不要把 Entity 直接傳到 Controller 或前端。

4. 大部分情況應該使用 LAZY,這是最佳實踐。因為很多時候我們不需要關聯物件的資料,LAZY 可以避免不必要的查詢。只有在「幾乎每次都會用到」的關聯才考慮用 EAGER,但要小心 EAGER 可能會連帶載入一大串關聯物件,造成效能問題。

   實務上,`@ManyToOne` 和 `@OneToOne` 預設是 EAGER,`@OneToMany` 和 `@ManyToMany` 預設是 LAZY,但建議全部明確設定為 LAZY,需要時再用 `JOIN FETCH` 指定載入。

---

### Q13. Pessimistic Locking 實作 (Mid)

**解答**:

1. `PESSIMISTIC_WRITE` lock 是一種「悲觀鎖」,它會在查詢資料時立即鎖定該筆資料,不讓其他 transaction 修改,甚至不讓其他人讀取(取決於資料庫的實作)。在資料庫層面,它是用 `SELECT ... FOR UPDATE` 來實現的。

   當你執行這個查詢時,PostgreSQL 會在該筆 order 上加一個 row-level lock,其他 transaction 如果也想用 FOR UPDATE 查詢同一筆資料,就會被 block 住,必須等到第一個 transaction commit 或 rollback 之後才能繼續。

2. 付款時需要 pessimistic lock 是為了防止「並發付款」的問題。想像一下:兩個使用者(或是使用者快速點擊兩次)同時對同一個訂單發起付款請求。如果沒有 lock,兩個 transaction 可能都讀到訂單狀態是 UNPAID,然後都執行付款邏輯,可能會導致重複扣款、重複發送課程權限等問題。

   有了 pessimistic lock,第一個 transaction 會鎖住訂單,第二個 transaction 必須等待,等第一個完成後再檢查訂單狀態,這時就會發現訂單已經是 PAID 了,就不會重複付款。

3. 這三種鎖的差異:

   - `PESSIMISTIC_WRITE`: 最強的鎖,鎖定資料不讓其他人修改或用 FOR UPDATE 讀取
   - `PESSIMISTIC_READ`: 比較弱的鎖,只防止其他人修改,但允許讀取,較少使用
   - `OPTIMISTIC`: 樂觀鎖,不會實際鎖定資料,而是用 version 欄位來檢測衝突,適合衝突機率低的情況

   Pessimistic lock 適合「高衝突」的場景,如付款、扣庫存;Optimistic lock 適合「低衝突」的場景,如編輯文章。

4. 如果兩個 transaction 同時嘗試 lock 同一筆資料,後來的那個會被 block 住,進入等待狀態。它會等到第一個 transaction 結束(commit 或 rollback),然後才能取得 lock 繼續執行。如果等待時間超過設定的 timeout,就會拋出 `PessimisticLockException` 或 `LockTimeoutException`。

5. Pessimistic lock 的缺點包括:

   - 效能較差,因為會 block 其他 transaction,降低並發能力
   - 可能造成 deadlock:如果兩個 transaction 互相等待對方的 lock,就會死鎖
   - 持有 lock 的時間要盡量短,長時間持有會嚴重影響系統效能

   所以使用 pessimistic lock 時要謹慎,確保 transaction 能快速完成,不要在鎖住資料的狀態下做耗時的操作(如呼叫外部 API)。

---

### Q14. Custom JPQL Query (Mid)

**解答**:

1. 這裡需要寫自訂 JPQL 是因為查詢條件比較複雜,用 Spring Data JPA 的方法命名規則會變得很長很難讀。如果用方法命名,可能要寫成:

   ```java
   Optional<Order> findFirstByUserIdAndStatusAndItems_JourneyIdAndDeletedAtIsNullOrderByCreatedAtDesc(...)
   ```

   這樣的方法名稱不僅很醜,而且容易出錯。自訂 JPQL 可以讓查詢邏輯更清晰,更容易維護。

2. `JOIN o.items oi` 是 JPQL 的關聯 JOIN,不是直接寫 SQL JOIN。這裡是利用 Order entity 中定義的 `items` 關聯來做 join,JPA 會根據你在 Entity 中定義的 `@OneToMany` 關係自動產生對應的 SQL JOIN。

   這樣的好處是,如果未來資料庫 schema 改變(比如外鍵欄位名稱改了),只要 Entity 的關聯定義不變,JPQL 就不需要修改。這是 JPA 抽象化的優點之一。

3. `LIMIT 1` 並不是標準的 JPQL 語法,它是 Hibernate 特有的擴展。不同的資料庫對 limit 的語法不同:MySQL 和 PostgreSQL 用 `LIMIT`,SQL Server 用 `TOP`,Oracle 用 `ROWNUM`。

   更好的寫法是在 Spring Data JPA 的方法名稱加上 `First` 關鍵字,或是在 `@Query` 外面用 `Pageable` 參數來控制查詢數量,這樣更有可移植性。但如果你確定只用一種資料庫,用 `LIMIT` 也沒問題。

4. 這個查詢的目的是實現「Idempotence」(冪等性)。在建立訂單時,系統會先檢查:使用者是否已經有一個「未付款」的訂單包含這個課程。如果有,就直接回傳現有的訂單,不建立新的。

   `ORDER BY o.createdAt DESC LIMIT 1` 是為了取得最新的那筆未付款訂單。這樣設計可以防止使用者重複點擊「購買」按鈕時產生多個相同的訂單,提升使用者體驗。

---

### Q15. Soft Delete 設計 (Mid)

**解答**:

1. **Soft Delete** 是「邏輯刪除」,不是真的從資料庫刪掉資料,而是標記一個 `deletedAt` 欄位來表示「這筆資料已被刪除」。跟 Hard Delete(物理刪除)相比,Soft Delete 的好處有:

   - 可以恢復誤刪的資料,不用擔心資料永久遺失
   - 保留歷史記錄,有助於審計和分析
   - 避免外鍵約束的問題,如果其他資料參照到這筆資料,刪除時不會出錯
   - 某些法規要求保留資料一段時間,不能直接刪除

2. 確保查詢時不會查到已刪除的資料,有幾種做法:

   - 在每個 Repository 方法的 `@Query` 中加上 `AND o.deletedAt IS NULL` 條件,就像這個專案的做法
   - 使用 Hibernate 的 `@Where(clause = "deleted_at IS NULL")` 註解在 Entity 上,自動過濾已刪除資料
   - 使用 `@SQLDelete` 和 `@FilterDef` 來自訂刪除和查詢邏輯

   第一種方法最直接但要記得每個查詢都加,第二種方法最省事但較不靈活。

3. Soft Delete 的缺點包括:

   - 資料表會越來越大,因為資料不會真的被刪除,影響查詢效能
   - 每個查詢都要加 `deletedAt IS NULL` 條件,容易遺漏
   - Unique constraint 會變複雜,比如使用者名稱的 unique constraint 要改成「deleted_at IS NULL 的情況下不能重複」
   - 可能會有隱私或法規問題,有些資料法律要求必須真的刪除

4. 如果要實作「永久刪除」功能,可以加一個專門的 Repository 方法:

   ```java
   @Modifying
   @Query("DELETE FROM Order o WHERE o.id = :id")
   void hardDelete(@Param("id") Long id);
   ```

   或是直接用 native SQL:

   ```java
   @Modifying
   @Query(value = "DELETE FROM orders WHERE id = :id", nativeQuery = true)
   void hardDelete(@Param("id") Long id);
   ```

   實務上,永久刪除通常只開放給管理員,或是用定期排程清理「刪除超過 N 天」的資料,平衡資料保留和資料庫效能。

---

## 第四部分：交易管理與併發控制 (Mid)

### Q16. @Transactional 的範圍 (Mid)

**解答**:

1. `@Transactional` 的作用是告訴 Spring:「這個方法裡的所有資料庫操作要在同一個 transaction 裡執行」。Transaction 有 ACID 特性:Atomicity(原子性,全部成功或全部失敗)、Consistency(一致性)、Isolation(隔離性)、Durability(持久性)。簡單說,就是確保一系列操作要嘛全部完成,要嘛全部回滾,不會出現「做一半」的狀態。

2. 要在 Service 層加 `@Transactional`,而不是 Repository 層,是因為一個 business operation 通常會涉及多個 Repository 操作。比如說付款流程:要更新訂單狀態、要新增課程權限記錄,這兩個操作必須在同一個 transaction 裡,確保同時成功或同時失敗。

   如果只在 Repository 層加,每個 Repository 方法都是獨立的 transaction,就無法保證多個操作的一致性。比如訂單更新成功了,但新增權限失敗了,就會造成資料不一致。

3. 在 `@Transactional` 方法中,如果拋出「unchecked exception」(RuntimeException 及其子類),Spring 會自動 rollback 整個 transaction。所有在這個 transaction 中做的資料庫變更都會被撤銷,就像這些操作從來沒發生過。

   但如果拋出「checked exception」(extends Exception 但不是 RuntimeException),預設不會 rollback,除非你明確設定 `@Transactional(rollbackFor = Exception.class)`。

4. `@Transactional(readOnly = true)` 是告訴資料庫這個 transaction 只會讀取資料,不會修改。好處有:

   - 資料庫可以做優化,比如跳過某些鎖定機制,提升效能
   - Hibernate 可以做一些優化,比如不需要 dirty checking
   - 明確表達這個方法的意圖,讓程式碼更易讀

   應該在所有「只查詢不修改」的 Service 方法上使用,比如 `getOrderById`、`listOrders` 這類方法。

5. Transaction 的傳播行為預設是 `PROPAGATION_REQUIRED`:如果外層已經有 transaction,就加入這個 transaction;如果沒有,就建立新的。所以當一個 `@Transactional` 方法呼叫另一個 `@Transactional` 方法時,它們會共用同一個 transaction。

   這表示如果內層方法拋出例外,整個 transaction(包括外層)都會 rollback。你可以用不同的傳播行為改變這個邏輯,比如 `REQUIRES_NEW` 會建立獨立的 transaction,但要小心使用,很容易搞亂。

---

### Q17. Idempotence 設計實作 (Mid)

**解答**:

1. **Idempotence**(冪等性)是指:多次執行同一個操作,結果跟只執行一次是一樣的。在 API 設計中,這非常重要,因為網路不可靠,前端可能會因為 timeout 或使用者重複點擊而重複發送相同的請求。如果 API 不是 idempotent 的,就可能造成重複扣款、重複建立訂單等問題。

2. 這段程式碼實現 idempotence 的方式是:在建立新訂單之前,先檢查使用者是否已經有「未付款」的訂單包含相同的課程。如果有,就直接回傳現有的訂單,而不是建立新的。這樣即使使用者多次點擊「購買」按鈕,也只會有一個訂單。

3. 如果使用者連續點兩次「建立訂單」按鈕,第一次會建立新訂單,第二次會查到剛建立的訂單並直接回傳。前端收到的 HTTP status code 會不同:第一次是 `201 CREATED`,第二次是 `200 OK`,這樣前端就能知道發生了什麼事。

4. 這個設計**不是完全 thread-safe** 的。如果兩個請求「完全同時」進來,都在檢查時沒找到現有訂單,就會各自建立一個新訂單,造成重複。這種情況雖然機率很小(因為人類手速沒那麼快),但在高並發環境下還是有可能發生。

5. 改進方法有幾種:

   - **資料庫層 unique constraint**: 在 `orders` 表加上 unique constraint `(user_id, status, journey_id) WHERE status = 'UNPAID'`,違反時資料庫會拋出例外,Service 層接住這個例外再去查詢現有訂單
   - **分散式鎖**: 用 Redis 的 distributed lock,在檢查和建立之間加鎖
   - **Optimistic locking**: 用 version 欄位檢測衝突
   - **Database lock**: 在查詢使用者資訊時用 `SELECT ... FOR UPDATE` 鎖住,雖然會降低並發效能

   最簡單有效的是第一種,用資料庫 constraint 作為最後一道防線。

---

### Q18. 併發付款的處理 (Mid)

**解答**:

1. 如果兩個使用者(或同一個使用者點兩次)同時對同一個訂單發起付款,沒有 lock 的話會很危險。兩個 transaction 可能都讀到訂單狀態是 UNPAID,然後都執行付款邏輯,可能導致:重複給予課程權限、重複發送通知、重複記錄付款記錄等問題。

   但有了 `findByIdAndUserIdForUpdate()` 的 pessimistic lock,第一個 transaction 會鎖住訂單,第二個必須等待,等第一個完成並 commit 之後,第二個才能讀取訂單,這時訂單已經是 PAID 狀態了,驗證就會失敗,不會重複付款。

2. `findByIdAndUserIdForUpdate()` 使用的 pessimistic lock(資料庫層的 `FOR UPDATE`)會在查詢時立即鎖定該筆資料。第一個 transaction 取得 lock 後,其他想要用 `FOR UPDATE` 查詢同一筆資料的 transaction 會被 block,進入等待佇列。

   等第一個 transaction commit 或 rollback 釋放 lock 之後,第二個才能取得 lock 並繼續執行。這樣就能避免 race condition,確保同一時間只有一個 transaction 在處理這筆訂單。

3. 先檢查 `validateOrderNotPaid()` 並不是多此一舉,有幾個原因:

   - **防止重複付款**: 如果有人刻意規避前端檢查,或是系統有 bug,這是最後一道防線
   - **提供明確的錯誤訊息**: 如果訂單已付款,可以回傳「訂單已支付」而不是「找不到訂單」,使用者體驗更好
   - **程式碼可讀性**: 明確表達「這個方法只處理未付款的訂單」,讓邏輯更清楚

   雖然理論上有 lock 應該不會有並發問題,但防禦性編程總是好的,多一層檢查不會有壞處。

4. 如果 lock 等待超過資料庫設定的 timeout(預設通常是幾秒到幾十秒),會拋出 `PessimisticLockException` 或底層的 `QueryTimeoutException`。

   處理方式通常是:

   - 在 GlobalExceptionHandler 攔截這個例外
   - 回傳友善的錯誤訊息給前端,比如「系統繁忙,請稍後再試」
   - 可能需要記錄 log 來追蹤是不是有效能瓶頸
   - 前端可以讓使用者重試,或自動 retry

   重要的是要確保 transaction 執行速度夠快,不要在持有 lock 的狀態下做耗時操作,這樣才能減少 timeout 的機會。

---

### Q19. Scheduled Task 的實作 (Mid)

**解答**:

1. `@Scheduled(cron = "0 */10 * * * *")` 是 Spring 的排程註解,這個 cron 表達式的意思是:

   - 秒: 0(每分鐘的第 0 秒)
   - 分: \*/10(每 10 分鐘)
   - 時: \*(任何小時)
   - 日: \*(任何日期)
   - 月: \*(任何月份)
   - 週: \*(任何星期幾)

   簡單說就是「每 10 分鐘執行一次」,比如 10:00、10:10、10:20 這樣。

2. 這個方法需要 `@Transactional` 是因為它涉及多筆資料的更新。在 loop 中,每個 order 都會被標記為 EXPIRED 並儲存。如果中間發生錯誤,我們希望前面成功的操作也要 rollback,確保資料的一致性。

   另外,`@Transactional` 確保整個排程任務在同一個 transaction 中執行,避免一些訂單更新成功、一些失敗的情況。

3. 如果在 loop 中處理某個 order 時拋出例外,**整個 transaction 會 rollback**。這表示前面已經標記為 EXPIRED 的訂單也會被還原,所有變更都會被撤銷。下次排程執行時,會再次嘗試處理所有過期訂單。

   這個行為可能不是我們想要的,因為如果某一筆資料有問題,會導致所有訂單都無法被處理。更好的做法是每個 order 用獨立的 transaction,或是在 loop 中 try-catch 處理例外,記錄錯誤但繼續處理其他訂單。

4. 如果執行時間超過 10 分鐘,會跟下一次排程「重疊」,導致有兩個任務同時在跑。Spring 預設沒有防止重疊執行的機制,可能會造成:

   - 同一筆訂單被處理兩次
   - 資料庫 lock 衝突
   - 系統資源耗盡

   可以在 `@Scheduled` 加上 `fixedDelay` 而不是 `cron`,或是加上 distributed lock 來防止重疊執行。

5. 改進方法:

   - **Batch processing with pagination**: 分批處理,每次只處理 100 筆,避免一次查太多資料

   ```java
   PageRequest pageRequest = PageRequest.of(0, 100);
   Page<Order> expiredOrders = orderRepository.findExpiredOrders(now, pageRequest);
   ```

   - **每個 order 用獨立 transaction**: 避免一個失敗影響全部
   - **加入 distributed lock**: 防止多個 instance 同時執行
   - **監控執行時間**: 如果超過預期時間,發出告警
   - **使用 bulk update**: 直接用一條 SQL UPDATE 多筆資料,效能更好

   ```sql
   UPDATE orders SET status = 'EXPIRED' WHERE status = 'UNPAID' AND expired_at < NOW()
   ```

---

### Q20. Upsert 操作的實作 (Mid)

**解答**:

1. **Upsert** 是 "Update or Insert" 的縮寫,表示「如果資料存在就更新,不存在就新增」。這段程式碼用 `orElseGet()` 來實現:先用 `findByUserIdAndMissionIdAndDeletedAtIsNull()` 查詢進度,如果找到就回傳現有的;如果找不到,就執行 `orElseGet` 裡的 lambda,建立一個新的 `UserMissionProgress` 物件。

   然後無論是現有的還是新建的,都呼叫 `updateWatchPosition()` 更新觀看進度,最後 `save()` 儲存。JPA 會根據 entity 的狀態決定執行 INSERT 還是 UPDATE。

2. 要用 `orElseGet()` 而不是 `orElse()`,是因為它們的執行時機不同:

   - `orElse(value)`: 無論 Optional 是否為空,`value` 都會被執行/計算
   - `orElseGet(() -> value)`: 只有在 Optional 為空時,lambda 才會被執行

   如果用 `orElse()`,即使查到了現有進度,也會去查詢 user(執行 `userRepository.findById()`),這是不必要的資料庫查詢。用 `orElseGet()` 可以避免這個浪費,只有在真的需要建立新物件時才查詢 user。

3. JPA 判斷 INSERT 還是 UPDATE 的方式是:檢查 entity 的 `@Id` 欄位。如果 id 是 `null`,或是 entity 是「transient state」(不在 persistence context 中),就執行 INSERT;如果 entity 已經在 persistence context 中,就執行 UPDATE。

   在這個例子中,如果查到現有的 progress,它已經有 id 而且是從資料庫讀出來的(managed state),所以 `save()` 會執行 UPDATE。如果是新建的 progress,id 是 null(transient state),所以 `save()` 會執行 INSERT。

4. 這個設計**不是完全 thread-safe** 的。如果兩個請求同時更新同一個使用者的同一個任務進度,可能會發生:

   - 兩個都查不到現有進度,各自建立新的 progress,第二個儲存時可能會違反 unique constraint(如果有的話)
   - 第一個更新的進度被第二個覆蓋掉,導致資料遺失

   改進方法:

   - 在資料庫加上 `unique(user_id, mission_id, deleted_at)` constraint,防止重複插入
   - 使用 optimistic locking,用 `@Version` 欄位檢測衝突
   - 在 method 層級加上 synchronization 或 distributed lock
   - 把整個 upsert 邏輯包在 pessimistic lock 中

   實務上,如果是「個人進度」這種不太會並發更新的場景,可能不需要這麼嚴格;但如果是「庫存」、「帳戶餘額」這種高並發場景,就一定要處理好 race condition。

---

## 第五部分:Spring Security 與 JWT (Mid)

### Q21. JWT Token 的生成 (Mid)

**解答**:

1. JWT 的結構分成三個部分,用 `.` 分隔:

   - **Header**: 包含 token 的型別(JWT)和簽名演算法(如 HS256)
   - **Payload**: 包含實際的資料,也就是 claims,像是使用者 ID、過期時間等
   - **Signature**: 用來驗證 token 沒有被竄改,由前兩部分加上 secret key 用特定演算法簽名產生

   三個部分都會用 Base64URL 編碼,組合起來就是一個 JWT token。

2. **JTI**(JWT ID)是 token 的唯一識別碼。為什麼需要它呢?主要是為了實作「登出」功能。JWT 本身是 stateless 的,server 不會儲存 token,但這也表示 server 沒辦法主動撤銷 token。有了 JTI,我們可以把登出的 token 的 JTI 加入黑名單(存到資料庫),驗證 token 時檢查 JTI 是否在黑名單中,就能實現登出功能。

   另外,JTI 也可以用來防止 token 被重複使用(replay attack),或是追蹤 token 的使用情況。

3. JWT 中應該放的資訊:使用者 ID(subject)、token 發行時間(issuedAt)、過期時間(expiration)、JTI、一些基本的使用者資訊(如 username)。不應該放的資訊:密碼、敏感個人資訊(如身分證字號)、大量的資料(會讓 token 太大)。

   重點是:JWT 的 payload 只是 Base64 編碼,不是加密,任何人都可以解碼看到內容。所以不要放任何敏感資訊。

4. `signWith(secretKey)` 預設使用 HMAC-SHA256(HS256)演算法,但實際上會根據你提供的 key 的長度自動選擇(HS256, HS384, HS512)。簽名的目的是確保 token 的完整性:只有擁有 secret key 的 server 才能產生有效的簽名,所以如果有人試圖竄改 payload 的內容,簽名就會對不上,驗證時就會失敗。

   這樣可以確保 token 是由我們的 server 發出的,而且內容沒有被竄改過。

5. JWT 的優點:

   - Stateless:server 不需要儲存 session,容易水平擴展
   - 跨域友善:可以在不同的 domain 使用
   - 行動友善:不依賴 cookie,適合 mobile app
   - 自包含:token 本身就包含使用者資訊,不用每次都查資料庫

   JWT 的缺點:

   - 無法主動撤銷:除非實作黑名單,否則 token 在過期前都有效
   - Token 洩漏風險:如果 token 被偷,攻擊者可以冒充使用者直到 token 過期
   - Token 大小:相比 session ID,JWT token 比較大,每次請求都要帶上完整的 token
   - 無法更新資訊:一旦發出,token 的內容就固定了,如果使用者資訊改變(如權限變更),要等到下次重新登入才會生效

---

### Q22. JWT Filter 的執行流程 (Mid)

**解答**:

1. `OncePerRequestFilter` 是 Spring 提供的 Filter 基底類別,它保證「每個請求只執行一次」。為什麼需要這個保證呢?因為在 servlet container 中,同一個請求可能會經過多次 filter chain(比如 forward 或 include 的情況),如果不用 `OncePerRequestFilter`,filter 可能會被執行多次,造成重複的邏輯或效能問題。

   對於認證 filter 來說,我們只想在請求進來的第一時間做一次驗證就好,不需要重複驗證,所以繼承 `OncePerRequestFilter` 是最適合的。

2. Filter 在 Spring Security 中的執行順序是由 SecurityConfig 中的 `addFilterBefore` 或 `addFilterAfter` 決定的。這個專案的設定是:

   ```java
   .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
   ```

   表示 JWT filter 會在 UsernamePasswordAuthenticationFilter 之前執行。完整的順序大概是:CORS filter → JWT filter → Authorization filter → Controller。

3. 如果 token 驗證失敗,還是要繼續執行 `filterChain.doFilter()`,是因為不是所有的 endpoint 都需要登入。像是 `/auth/login`、`/auth/register` 這些公開的 API,即使沒有 token 也應該能存取。

   JWT filter 的職責只是「如果有有效的 token,就設定 authentication」,但不會「擋住沒有 token 的請求」。真正決定「哪些 endpoint 需要登入」的是 SecurityConfig 中的 `authorizeHttpRequests` 設定。如果請求需要登入但 SecurityContext 中沒有 authentication,Spring Security 會在後面的 filter 中攔截,回傳 401 Unauthorized。

4. `SecurityContextHolder.getContext().setAuthentication()` 的作用是把當前使用者的認證資訊存到 Spring Security 的 context 中。一旦設定了,後面的 filter、Controller、Service 都可以透過 `SecurityContextHolder.getContext().getAuthentication()` 取得當前使用者的資訊。

   Spring Security 會根據這個 authentication 來決定:這個使用者是否有權限存取某個 endpoint、是否已登入等。

5. 把 `userId` 放在 `authentication.getPrincipal()` 中,是為了方便在 Controller 或 Service 中快速取得當前使用者的 ID。標準的做法是把整個 `UserDetails` 物件放進去,但這個專案選擇只放 ID,原因是:

   - 效能考量:不需要每次驗證 token 時都從資料庫查詢完整的使用者資訊
   - 簡化邏輯:大部分情況下我們只需要使用者 ID 就夠了
   - 避免 stale data:如果把完整使用者資訊放在 authentication 中,可能會有資料過時的問題

   需要使用者其他資訊時,再用 ID 去查詢即可。

---

### Q23. Token Blacklist 機制 (Mid)

**解答**:

1. JWT 需要 blacklist 主要是為了實作「登出」功能。JWT 的設計理念是 stateless,server 不儲存任何 session 資訊,token 一旦發出,在過期前都是有效的。但這樣就沒辦法實作登出功能了,因為即使使用者點了登出,舊的 token 依然可以用。

   所以我們需要一個 blacklist 機制:當使用者登出時,把 token 的 JTI 加入黑名單,之後驗證 token 時檢查 JTI 是否在黑名單中。這樣就能讓已登出的 token 失效。雖然這有點違背 JWT 的 stateless 理念,但這是必要的 trade-off。

2. Blacklist 的資料表設計應該包含:

   - `token_jti`: JWT 的唯一識別碼,設為 primary key 或 unique key
   - `user_id`: 使用者 ID,方便查詢某個使用者的所有黑名單 token
   - `expires_at`: Token 的過期時間,用來定期清理已過期的 token
   - `created_at`: 加入黑名單的時間

   需要在 `token_jti` 上建立 index(因為每次驗證都要查詢),如果常需要查詢某個使用者的 blacklist,也可以在 `user_id` 上建立 index。

3. 為了避免 blacklist 無限成長,應該定期清理已過期的 token。可以用 scheduled task 定期執行:

   ```sql
   DELETE FROM access_tokens WHERE expires_at < NOW()
   ```

   比如每天半夜執行一次清理任務,刪除所有已過期的 blacklist token。因為 token 過期後就算在 blacklist 中也沒意義了,可以安全刪除。

4. 更好的 logout 實作方式:

   - **Refresh token 機制**: Access token 設定很短的過期時間(如 15 分鐘),搭配長效的 refresh token。登出時只需要刪除 refresh token,access token 很快就會自然過期,blacklist 的大小就可以控制得很小
   - **Short-lived access token**: 讓 access token 的有效期限很短(5-15 分鐘),這樣即使登出了,最多等 15 分鐘 token 就會過期,可能不需要 blacklist
   - **Token version**: 在使用者表加一個 `token_version` 欄位,每次登出時就增加 version,token 中也帶上 version,驗證時比對是否一致

   實務上,refresh token + short-lived access token 是最常見也最推薦的做法。

---

### Q24. Rate Limiting 的實作 (Mid)

**解答**:

1. **Rate limiting** 是限制某個使用者或 IP 在一定時間內能發送的請求數量。為什麼需要它?主要有幾個原因:

   - 防止暴力破解:攻擊者如果想猜密碼,可能會發送大量的登入請求,rate limiting 可以限制嘗試次數
   - 防止 DDoS 攻擊:大量請求會耗盡 server 資源,rate limiting 可以保護系統
   - 公平使用:防止某些使用者佔用過多資源,影響其他使用者
   - API 保護:特別是付費 API,需要限制免費用戶的使用量

2. 登入失敗時不重置 rate limit,但成功時要重置,是為了防止暴力破解攻擊。如果每次登入失敗都重置,攻擊者可以持續嘗試,永遠不會被限制住。只有在「登入成功」的情況下才重置,代表這個 IP 是合法使用者,可以清除之前的失敗記錄,讓他下次登入時不會被影響。

   這種設計可以有效阻止攻擊者暴力猜密碼,同時又不會影響正常使用者。

3. Rate limiting 的實作方式,這個專案用的是 **Bucket4j** library,實作了 Token Bucket 演算法:

   - 想像有一個水桶,最多可以裝 5 個 token
   - 每次請求會消耗 1 個 token
   - Token 會以固定速率補充(這裡是 15 分鐘補充 5 個,相當於每 3 分鐘補充 1 個)
   - 如果桶是空的,請求就會被拒絕

   另外搭配 Caffeine cache 來存放每個 IP 的 bucket,cache 會在 15 分鐘沒有存取後自動清除,節省記憶體。

4. 用 IP 來做 rate limiting 的問題:

   - NAT 問題:同一個公司或學校的很多人可能共用同一個 public IP,一個人被限制會影響到其他人
   - 代理和 VPN:攻擊者可以換 IP 來繞過限制
   - IPv6 問題:攻擊者可能有很多個 IPv6 位址

   改進方法:

   - 結合 IP + User-Agent + 其他 fingerprint 來識別使用者
   - 對已登入的使用者,用 user ID 來限制而不是 IP
   - 使用更複雜的 bot detection 機制,如 CAPTCHA
   - 分層限制:IP 限制 + 使用者限制

5. Rate limit 設定太嚴格的影響:正常使用者可能會被誤判,影響使用者體驗,尤其是同一個網路下的多個使用者。設定太寬鬆的影響:無法有效防止攻擊,系統還是可能被暴力破解或 DDoS。

   需要根據實際情況調整,比如分析正常使用者的行為模式,設定一個合理的上限,並且監控 rate limit 的觸發情況,適時調整。

---

### Q25. CORS 配置 (Mid)

**解答**:

1. **CORS**(Cross-Origin Resource Sharing)是瀏覽器的安全機制。當前端 JavaScript 從某個 domain(如 `http://localhost:3000`)發送請求到另一個 domain(如 `http://localhost:8080`)時,瀏覽器會擋住這個請求,除非 server 明確允許。這是為了防止惡意網站竊取使用者資料。

   為什麼需要 CORS 配置?因為現代前後端分離的架構中,前端和後端通常在不同的 domain 或 port,如果不設定 CORS,前端根本無法呼叫後端 API。

2. `setAllowCredentials(true)` 表示允許前端發送「帶有憑證的請求」,包括 cookies、HTTP authentication、TLS client certificates 等。如果設為 false,即使前端有 cookie,瀏覽器也不會把它帶到 cross-origin 的請求中。

   對於使用 JWT 的系統來說,如果 JWT 存在 cookie 中,就需要設定 `allowCredentials: true`。如果 JWT 是存在 localStorage 並透過 Authorization header 發送,技術上可以不用,但通常還是會設為 true 以備不時之需。

3. `OPTIONS` method 在 CORS 中扮演「預檢請求」的角色。對於「非簡單請求」(比如帶有 Authorization header,或 content-type 是 application/json),瀏覽器會先發送一個 OPTIONS 請求問 server:「我可以發送這個請求嗎?」,這就是 preflight request。

   Server 必須正確回應 OPTIONS 請求,告訴瀏覽器允許哪些 methods、headers,瀏覽器確認允許後才會發送真正的請求。所以 CORS 配置中一定要加上 OPTIONS。

4. `setMaxAge(3600L)` 設定 preflight request 的快取時間,單位是秒。這表示瀏覽器在收到 preflight response 後,可以把結果快取 3600 秒(1 小時),在這段時間內對同一個 endpoint 的請求就不需要再發送 preflight request,可以減少網路請求,提升效能。

5. 生產環境的 CORS 配置應該:

   - **不要用 `*`**: 不要設定 `setAllowedOrigins(Arrays.asList("*"))`,要明確列出允許的 domain
   - **限制 origins**: 只允許實際的前端 domain,比如 `https://www.example.com`,不要包含開發用的 localhost
   - **用 allowedOriginPatterns**: 如果有多個子網域,可以用 pattern 如 `https://*.example.com`
   - **限制 methods**: 只開放需要的 HTTP methods,不要全開
   - **限制 headers**: 如果可以的話,明確指定允許的 headers 而不是用 `*`
   - **設定適當的 maxAge**: 可以設長一點如 86400(24 小時)來減少 preflight 請求

   最重要的是「最小權限原則」:只開放必要的權限,其他都拒絕。

---

## 第六部分:資料庫設計與查詢優化 (Mid)

### Q26. 關聯關係的設計 (Mid)

**解答**:

1. `mappedBy = "order"` 表示這個關聯是由「對方」來管理的,也就是 `OrderItem` entity 中的 `order` 欄位是關聯的擁有者(owner side)。在 JPA 的雙向關聯中,必須有一方是 owner,負責維護外鍵;另一方是 inverse side,只是「映射」到 owner 的關聯。

   在這個例子中,`OrderItem` 是 owner(因為它有 `@JoinColumn` 指定外鍵欄位),`Order` 是 inverse side(用 `mappedBy` 表示)。實際上,資料庫只有一個外鍵欄位在 `order_items` 表中,指向 `orders` 表。

2. `CascadeType.ALL` 包含所有的 cascade 操作:

   - `PERSIST`: 儲存 order 時,自動儲存所有 items
   - `MERGE`: 更新 order 時,自動更新所有 items
   - `REMOVE`: 刪除 order 時,自動刪除所有 items
   - `REFRESH`: 重新載入 order 時,自動重新載入所有 items
   - `DETACH`: 分離 order 時,自動分離所有 items

   這樣的設定很適合「整體-部分」的關係,像是訂單跟訂單項目,它們的生命週期緊密相連。

3. `orphanRemoval = true` 表示:如果一個 item 從 `items` list 中被移除,但沒有被加到其他 order,那它就變成「孤兒」,JPA 會自動把它從資料庫刪除。

   跟 `CascadeType.REMOVE` 的差異:

   - `CascadeType.REMOVE`: 只有在刪除「父」entity 時才會刪除「子」entity
   - `orphanRemoval = true`: 只要「子」entity 脫離了「父」entity 的集合,就會被刪除,即使「父」entity 還存在

   舉例來說,如果你從 order 的 items list 中移除一個 item:

   ```java
   order.getItems().remove(item);  // 有 orphanRemoval,item 會被刪除
   ```

4. 初始化 `new ArrayList<>()` 有幾個原因:

   - 避免 NullPointerException:如果不初始化,直接呼叫 `order.getItems().add()` 會出錯
   - 方便使用:可以立即操作 collection,不需要檢查 null
   - JPA 最佳實踐:JPA 會用這個 collection 來追蹤變更,如果是 null 會有問題

---

### Q27. 雙向關聯的管理 (Mid)

**解答**:

1. 這個 helper method 的目的是「同步雙向關聯」。在 JPA 的雙向關聯中,雖然只有 owner side(OrderItem)的設定會影響資料庫,但為了讓 Java 物件在記憶體中的狀態保持一致,兩邊都要設定。

   如果不用這個 method,開發者可能會忘記設定雙向關聯,導致物件狀態不一致。這個 helper method 確保「一個動作就能同時設定兩邊」,不容易出錯。

2. 如果只呼叫 `items.add(item)` 而不呼叫 `item.setOrder(this)`,會發生幾個問題:

   - 在同一個 transaction 內,如果你呼叫 `item.getOrder()`,會得到 null,因為 Java 物件還沒設定
   - 如果你在 persist 之前需要用到 item 的 order 資訊,會出錯
   - 雖然資料庫會正確儲存(因為 owner side 的設定才重要),但物件狀態不一致可能導致難以追蹤的 bug

   最糟的情況是:如果 item 已經是 managed state,而你只設定了 inverse side,JPA 可能不會偵測到變更,資料庫根本不會更新。

3. JPA 處理雙向關聯的原則是:「只有 owner side 的設定會影響資料庫」。當你儲存一個 Order 時:

   - JPA 會遍歷 `items` 集合
   - 對每個 item,檢查它的 `order` 欄位(owner side 的外鍵)
   - 根據這個欄位來設定資料庫的外鍵值

   所以理論上,只要 owner side 設定正確,資料庫就會正確。但 inverse side 的設定對於「在記憶體中操作物件」很重要。

4. 這種設計的優點:

   - 防止遺漏:確保雙向關聯都有設定,不會因為忘記設定而出錯
   - 封裝:把關聯管理的邏輯封裝在 entity 內,使用者不需要知道內部細節
   - 一致性:保證 Java 物件的狀態跟資料庫一致
   - 可讀性:程式碼更清楚,`order.addItem(item)` 比分開設定兩邊更直觀

---

### Q28. Pagination 實作 (Mid)

**解答**:

1. `Page<Order>` vs `List<Order>` 的差異:

   - `List<Order>`: 只包含資料本身,不知道總共有幾筆、有幾頁
   - `Page<Order>`: 除了資料,還包含分頁資訊,如總筆數(totalElements)、總頁數(totalPages)、當前頁碼(number)、是否有下一頁(hasNext)等

   `Page` 是 Spring Data 提供的介面,專門用來處理分頁查詢的結果,讓前端可以顯示分頁導覽。

2. `Pageable` 是 Spring Data 的介面,包含:

   - `pageNumber`: 第幾頁(從 0 開始)
   - `pageSize`: 每頁幾筆資料
   - `sort`: 排序規則(欄位名稱、升冪或降冪)
   - `offset`: 計算出來的資料起始位置

   建立 Pageable 的方式:

   ```java
   Pageable pageable = PageRequest.of(0, 10);  // 第 0 頁,每頁 10 筆
   Pageable pageable = PageRequest.of(1, 20, Sort.by("createdAt").descending());
   ```

3. 在 Service 層使用這個方法很簡單:

   ```java
   PageRequest pageRequest = PageRequest.of(page, size);
   Page<Order> orderPage = orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageRequest);

   // 取得資料
   List<Order> orders = orderPage.getContent();
   // 取得總筆數
   long total = orderPage.getTotalElements();
   // 是否有下一頁
   boolean hasNext = orderPage.hasNext();
   ```

4. Pagination 的好處:

   - 效能:一次只查詢需要的資料,不會一次載入幾千筆資料到記憶體
   - 使用者體驗:前端可以分頁顯示,不會讓使用者看到超長的列表
   - 頻寬節省:減少網路傳輸的資料量
   - 資料庫負擔:用 `LIMIT` 和 `OFFSET` 可以減輕資料庫壓力

   如果不做 pagination,假設有 10 萬筆訂單,一次查全部會:耗盡記憶體、回應時間超長、可能觸發 OOM。

5. **Offset-based pagination** vs **Cursor-based pagination**:

   - **Offset-based**(這個專案用的):用 page number 和 page size,SQL 會用 `LIMIT 10 OFFSET 20`。缺點是:如果在翻頁過程中有新資料插入或刪除,可能會重複看到資料或漏掉資料;在 offset 很大時效能差(資料庫要跳過前面所有資料)

   - **Cursor-based**:用「上一頁最後一筆資料的 ID」作為起點,SQL 會用 `WHERE id > last_id LIMIT 10`。優點是:效能穩定、不受資料變動影響。缺點是:無法直接跳到第 N 頁、實作較複雜

   實務上,社群媒體的「無限滾動」常用 cursor-based,一般的列表分頁用 offset-based 就夠了。

---

### Q29. N+1 Query 問題 (Mid)

**解答**:

1. 這段程式碼會執行 **1 + N** 次 SQL query。第一次是 `findAll()` 查詢所有 Journey(假設有 N 個),然後在 loop 中,每次存取 `journey.getChapters()` 時,因為 chapters 是 LAZY loading,會觸發一次額外的查詢。所以總共是 1 + N 次。

   如果有 100 個 journeys,就會執行 101 次查詢,非常沒效率。

2. **N+1 Query Problem** 是 ORM 框架常見的效能問題。問題在於:你查詢了 N 筆主資料,然後為了取得每筆資料的關聯物件,又各執行了 1 次查詢,總共 N+1 次。雖然每次查詢很快,但累積起來的網路往返時間(latency)會變得很可觀。

   避免方法:

   - **JOIN FETCH**:在 JPQL 中明確指定要一起載入的關聯

     ```java
     @Query("SELECT j FROM Journey j JOIN FETCH j.chapters WHERE ...")
     List<Journey> findAllWithChapters();
     ```

   - **@EntityGraph**:用 annotation 指定要載入的關聯

     ```java
     @EntityGraph(attributePaths = {"chapters"})
     List<Journey> findAll();
     ```

   - **Batch fetching**:設定 `@BatchSize`,一次查詢多個關聯物件

3. 如果在 `@Transactional` 外面存取 lazy loading 的關聯,會拋出 `LazyInitializationException: could not initialize proxy - no Session`。這是因為 Hibernate 的 session 已經關閉了,無法再執行查詢。

   這是 lazy loading 最大的陷阱,解決方法是:

   - 確保在 `@Transactional` 方法內完成所有資料存取
   - 使用 eager loading 或 JOIN FETCH
   - 把資料轉成 DTO,在 transaction 內就把需要的資料都取出來

4. Lazy loading 反而是好的情況:

   - 大部分情況不需要關聯資料:比如只是顯示 Journey 列表,不需要 chapters
   - 關聯資料很大:如果每個 Journey 有 100 個 chapters,eager loading 會載入很多不必要的資料
   - 條件式載入:根據不同情況決定要不要載入關聯,lazy 讓你有彈性

   EAGER loading 的缺點是:即使不需要,也會把關聯資料都載入,浪費記憶體和頻寬。所以預設用 LAZY,需要時再用 JOIN FETCH,是比較好的策略。

---

### Q30. 資料完整性約束 (Mid)

**解答**:

1. 這些檢查如果只在應用層做,當有 race condition 時會出問題。假設兩個請求同時進來:

   - 請求 A 檢查:使用者沒有這個課程的訂單 ✓
   - 請求 B 檢查:使用者沒有這個課程的訂單 ✓(因為請求 A 還沒寫入)
   - 請求 A 建立訂單並寫入資料庫
   - 請求 B 建立訂單並寫入資料庫
   - 結果:同一個使用者有兩筆相同課程的訂單,違反業務規則

2. 在資料庫層面保證資料完整性,應該用 **unique constraint**:

   ```sql
   CREATE UNIQUE INDEX idx_user_journey_unpaid
   ON orders (user_id, journey_id)
   WHERE status = 'UNPAID' AND deleted_at IS NULL;
   ```

   或是在 `user_journeys` 表(如果有的話):

   ```sql
   ALTER TABLE user_journeys
   ADD CONSTRAINT uk_user_journey
   UNIQUE (user_id, journey_id, deleted_at);
   ```

   這樣即使應用層的檢查有漏洞,資料庫也會保證不會插入重複資料。

3. `user_journeys` 表應該有 unique constraint 確保:「每個使用者對每個課程只能有一筆有效的購買記錄」。

   ```sql
   UNIQUE (user_id, journey_id) WHERE deleted_at IS NULL
   ```

   或是用 composite primary key:

   ```sql
   PRIMARY KEY (user_id, journey_id)
   ```

   如果有 soft delete,constraint 要加上 `WHERE deleted_at IS NULL`,這樣刪除後可以再次購買。

4. 當違反 unique constraint 時,資料庫會拋出例外,JPA 會把它包裝成:

   - `DataIntegrityViolationException`(Spring)
   - `ConstraintViolationException`(JPA)

   處理方式:

   ```java
   try {
     orderRepository.save(order);
   } catch (DataIntegrityViolationException e) {
     // 檢查是不是 unique constraint violation
     // 重新查詢現有訂單並回傳
     Order existing = orderRepository.findByUserIdAndJourneyId(userId, journeyId);
     return new OrderCreationResult.Existing(mapToOrderResponse(existing));
   }
   ```

   這樣的設計結合了「應用層檢查(效能優化)」和「資料庫 constraint(最終防線)」,是最安全的做法。應用層檢查可以在大部分情況避免不必要的錯誤,資料庫 constraint 則確保在任何情況下資料都是正確的。

---

## 第七部分:錯誤處理與例外設計 (Mid)

### Q31. 全域例外處理器 (Mid)

**解答**:

1. `@RestControllerAdvice` 和 `@ControllerAdvice` 的差異在於:

   - `@ControllerAdvice`: 適用於傳統的 MVC controller,回傳 View(如 JSP 頁面)
   - `@RestControllerAdvice`: 等於 `@ControllerAdvice` + `@ResponseBody`,專門用於 RESTful API,所有 handler 方法的回傳值都會自動序列化成 JSON

   既然我們是做前後端分離的 REST API,用 `@RestControllerAdvice` 最合適。

2. 使用全域例外處理器的好處:

   - **避免重複的程式碼**: 不需要在每個 Controller 方法都寫 try-catch,一個地方統一處理
   - **統一的錯誤格式**: 確保所有錯誤回應都有一致的格式,前端更容易處理
   - **關注點分離**: Controller 專注於業務邏輯,錯誤處理集中在一個地方
   - **容易維護**: 如果要改錯誤訊息或 status code,只需要修改一個地方
   - **更乾淨的程式碼**: Controller 的程式碼更簡潔,可讀性更高

3. 例外處理的順序是「從特定到一般」,Spring 會選擇最符合的 handler:

   - 先匹配特定的例外(如 `DuplicateUsernameException`)
   - 如果沒有特定的 handler,再往上找父類別的 handler
   - 最後才用 `Exception.class` 的 handler 作為兜底

   這就是為什麼我們要把 `@ExceptionHandler(Exception.class)` 放在最後,確保所有未預期的錯誤都能被接住,不會讓使用者看到醜陋的 stack trace。順序很重要,要從具體到抽象。

4. Spring 框架拋出的例外處理方式:

   - **`MethodArgumentNotValidException`**: 當 `@Valid` 驗證失敗時拋出,我們可以用 `@ExceptionHandler` 接住,提取 validation errors 並回傳給前端

     ```java
     @ExceptionHandler(MethodArgumentNotValidException.class)
     public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
       Map<String, String> errors = new HashMap<>();
       ex.getBindingResult().getAllErrors().forEach(error -> {
         errors.put(((FieldError) error).getField(), error.getDefaultMessage());
       });
       return ResponseEntity.badRequest().body(new ErrorResponse(errors));
     }
     ```

   - **`HttpMessageNotReadableException`**: 當 JSON 格式錯誤時拋出
   - **`MissingServletRequestParameterException`**: 當必要的 query parameter 缺失時拋出
   - **`HttpRequestMethodNotSupportedException`**: 當 HTTP method 不支援時(如 endpoint 只接受 POST 但送了 GET)

   這些 Spring 的例外我們都可以用 `@ExceptionHandler` 接住,提供友善的錯誤訊息。

---

### Q32. 自訂例外的設計 (Mid)

**解答**:

1. 應該自訂例外的時機:

   - **表達特定的業務錯誤**: 像是「使用者名稱重複」、「訂單已付款」,這些有明確的語意,比用通用的 `RuntimeException` 更清楚
   - **需要不同的處理方式**: 不同的業務錯誤可能要回傳不同的 HTTP status code 或錯誤訊息
   - **提升程式碼可讀性**: 看到 `throw new OrderAlreadyPaidException()` 比 `throw new RuntimeException("Order already paid")` 更容易理解
   - **方便測試**: 測試時可以明確驗證是否拋出特定的例外

2. **Checked Exception** vs **Unchecked Exception**:

   - **Checked Exception**: 繼承自 `Exception`,編譯器會強制你處理(try-catch 或 throws),適合「可預期且可恢復的錯誤」,如檔案不存在、網路錯誤
   - **Unchecked Exception**: 繼承自 `RuntimeException`,不強制處理,適合「程式邏輯錯誤或業務規則違反」,如 NullPointerException、非法參數

   這個專案使用 **Unchecked Exception** (都繼承 `RuntimeException`),這是現代 Spring 開發的主流做法。原因是:

   - Checked exception 會讓程式碼充斥著 try-catch,影響可讀性
   - 業務邏輯的錯誤通常是不可恢復的,沒必要強制處理
   - Spring 的 `@Transactional` 預設只會 rollback unchecked exception

3. 為每種錯誤情況建立不同的例外類別,好處是:

   - **精確的錯誤處理**: 可以針對不同例外回傳不同的 HTTP status code 和訊息
   - **更好的語意**: 一看類別名稱就知道發生什麼錯誤
   - **避免字串比對**: 不需要用 `if (ex.getMessage().contains("..."))` 來判斷錯誤類型
   - **方便擴展**: 新增錯誤類型時只要加新的例外類別,不會影響現有程式碼
   - **測試更精確**: 可以用 `assertThrows(OrderAlreadyPaidException.class, ...)` 來驗證

   如果所有錯誤都用同一個通用例外,就失去了這些好處。

4. 自訂例外應該繼承 **`RuntimeException`**,原因如上面第 2 點所述。繼承 `Exception` 會變成 checked exception,在 Spring 開發中會造成不便:

   - Service 方法都要加 `throws` 宣告,程式碼變得冗長
   - `@Transactional` 預設不會 rollback checked exception
   - 大部分業務邏輯錯誤都是不可恢復的,強制處理沒有意義

---

### Q33. 錯誤訊息的設計 (Junior)

**解答**:

1. 為什麼要回傳「帳號或密碼錯誤」而不是「密碼錯誤」或「帳號不存在」?這是為了**安全考量**:

   - 如果回傳「帳號不存在」,攻擊者就可以列舉出系統中有哪些有效的帳號,然後專門針對這些帳號暴力破解密碼
   - 如果回傳「密碼錯誤」,攻擊者就知道這個帳號存在,減少了需要猜測的變數
   - 統一回傳模糊的訊息「帳號或密碼錯誤」,攻擊者無法判斷到底是哪個欄位錯誤,增加破解難度

   這是資安的基本原則:不要洩漏多餘的資訊給潛在攻擊者。

2. 回傳給前端 vs 只記錄在 log 的原則:

   - **回傳給前端**: 使用者能理解且能採取行動的資訊,如「帳號或密碼錯誤」、「訂單已付款」,用友善的中文訊息
   - **只記錄在 log**: 技術細節、內部狀態、stack trace、資料庫錯誤訊息等,前端使用者不需要也不應該看到

   不要把 exception.getMessage() 直接回傳給前端,因為可能包含敏感資訊(如 SQL 查詢、檔案路徑)或讓使用者困惑的技術術語。

3. HTTP status code 的選擇原則:

   - **400 Bad Request**: 輸入格式錯誤、validation 失敗、業務規則違反(如「訂單已付款」)
   - **401 Unauthorized**: 未登入或 token 無效
   - **403 Forbidden**: 已登入但沒有權限(如存取別人的資料)
   - **404 Not Found**: 資源不存在
   - **409 Conflict**: 資源衝突(如「使用者名稱已存在」)
   - **500 Internal Server Error**: 伺服器內部錯誤,未預期的例外

   選擇正確的 status code 很重要,前端可以根據 status code 做不同的處理。

4. 多語系錯誤訊息的設計:

   - **用 i18n (internationalization)**: Spring 提供 `MessageSource` 來管理多語系訊息

     ```java
     @Autowired
     private MessageSource messageSource;

     String message = messageSource.getMessage("error.duplicate.username",
                                                null,
                                                LocaleContextHolder.getLocale());
     ```

   - **訊息檔案**: 建立 `messages_zh_TW.properties`、`messages_en_US.properties` 等

     ```
     error.duplicate.username=使用者名稱已存在
     error.invalid.credentials=帳號或密碼錯誤
     ```

   - **前端處理**: 或是更簡單的方式,後端只回傳 error code,前端根據 code 顯示對應語言的訊息

     ```json
     {
     	"errorCode": "DUPLICATE_USERNAME",
     	"message": "使用者名稱已存在"
     }
     ```

---

## 第八部分:測試與品質保證 (Junior/Mid)

### Q34. 單元測試 vs 整合測試 (Junior)

**解答**:

1. **單元測試** vs **整合測試** 的差異:

   - **單元測試**:

     - 測試最小的程式單位(通常是一個方法)
     - 隔離測試,用 mock 取代外部依賴(如 Repository、外部 API)
     - 執行速度快,秒級完成
     - 專注於邏輯正確性
     - 例:測試 Service 的某個方法,mock Repository 的回傳值

   - **整合測試**:
     - 測試多個元件整合在一起的行為
     - 使用真實的依賴(如真實的資料庫)
     - 執行速度較慢,可能需要幾秒到幾分鐘
     - 專注於元件之間的互動
     - 例:測試從 Controller 到 Repository 的完整流程

2. Service 層的單元測試寫法:

   ```java
   @ExtendWith(MockitoExtension.class)
   class OrderServiceTest {

     @Mock
     private OrderRepository orderRepository;

     @Mock
     private UserRepository userRepository;

     @InjectMocks
     private OrderService orderService;

     @Test
     void testCreateOrder_Success() {
       // Arrange
       Long userId = 1L;
       CreateOrderRequest request = new CreateOrderRequest(100L);
       when(userRepository.findById(userId)).thenReturn(Optional.of(new User()));
       when(orderRepository.findByUserIdAndJourneyId(...)).thenReturn(Optional.empty());

       // Act
       OrderCreationResult result = orderService.createOrder(userId, request);

       // Assert
       assertThat(result).isInstanceOf(OrderCreationResult.Created.class);
       verify(orderRepository).save(any(Order.class));
     }
   }
   ```

   需要 mock 的東西:所有外部依賴,包括 Repository、其他 Service、外部 API client 等。

3. 什麼情況下應該寫整合測試:

   - 測試資料庫查詢邏輯:確保 JPQL/SQL 語法正確
   - 測試 transaction 行為:驗證 rollback 機制
   - 測試完整的 API 流程:從 HTTP request 到 database
   - 測試安全配置:JWT filter、CORS 等
   - 測試複雜的資料關聯:cascade、lazy loading 等

   整合測試更慢但更接近實際使用情境,可以抓到單元測試抓不到的問題。

4. **Testcontainers** 的用途:

   這個專案的 pom.xml 包含 Testcontainers,它可以在測試時啟動真實的 Docker container(如 PostgreSQL、Redis),讓整合測試使用真實的資料庫而不是 H2 in-memory database。

   好處:

   - 測試環境跟生產環境一致(都用 PostgreSQL)
   - 可以測試資料庫特定的功能(如 PostgreSQL 的 enum type)
   - 每次測試都是乾淨的資料庫,測試之間互不影響
   - 測試完自動清理 container

   使用方式:

   ```java
   @Testcontainers
   @SpringBootTest
   class OrderServiceIntegrationTest {

     @Container
     static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

     @DynamicPropertySource
     static void configureProperties(DynamicPropertyRegistry registry) {
       registry.add("spring.datasource.url", postgres::getJdbcUrl);
     }

     @Test
     void testPayOrder() {
       // 使用真實的 PostgreSQL 資料庫
     }
   }
   ```

---

### Q35. 測試資料的準備 (Mid)

**解答**:

1. 準備測試資料的方法:

   - **直接在測試中建立**:

     ```java
     @Test
     void testPayOrder() {
       // Arrange: 建立測試資料
       User user = new User("testuser", "password");
       user = userRepository.save(user);

       Journey journey = new Journey("Test Journey", ...);
       journey = journeyRepository.save(journey);

       Order order = new Order("ORD-001", user.getId(), ...);
       order = orderRepository.save(order);

       // Act: 執行測試
       PayOrderResponse response = orderService.payOrder(order.getId(), user.getId());

       // Assert: 驗證結果
       assertThat(response.getStatus()).isEqualTo("PAID");
     }
     ```

   - **使用 Test Fixture 類別**:

     ```java
     public class TestDataBuilder {
       public static User createTestUser(String username) {
         return new User(username, "password");
       }

       public static Order createUnpaidOrder(Long userId, Long journeyId) {
         return new Order("ORD-001", userId, ...);
       }
     }
     ```

   - **使用 SQL script**:

     ```java
     @Sql("/test-data.sql")  // 在測試前執行 SQL
     @Test
     void testPayOrder() { ... }
     ```

2. 測試完成後如何清理資料:

   - **@Transactional + Rollback**: 最簡單的方式

     ```java
     @SpringBootTest
     @Transactional  // 測試結束後自動 rollback
     class OrderServiceIntegrationTest {
       @Test
       void testPayOrder() {
         // 所有資料操作都會在測試結束後自動回滾
       }
     }
     ```

   - **@DirtiesContext**: 重新啟動 Spring context,確保乾淨的環境(但很慢)

   - **手動清理**:

     ```java
     @AfterEach
     void cleanup() {
       orderRepository.deleteAll();
       userRepository.deleteAll();
     }
     ```

3. 如何確保測試之間不會互相影響:

   - **使用 `@Transactional` rollback**: 每個測試都在獨立的 transaction 中,結束後自動回滾
   - **測試資料隔離**: 每個測試用不同的資料(如不同的 username、order number)
   - **隨機資料**: 用 UUID 或 random number 產生測試資料,避免衝突
   - **Testcontainers**: 每次測試都啟動全新的資料庫 container
   - **避免共用狀態**: 不要在測試類別中用 static 變數儲存測試資料

4. **Test Fixture** 是什麼:

   Test Fixture 是「測試固定裝置」,指測試所需的前置條件和環境,包括:

   - 測試資料:預先建立的 entities
   - 測試環境:資料庫、伺服器、檔案系統
   - Mock 物件:模擬的外部依賴

   在 JUnit 中,fixture 相關的 annotations:

   - `@BeforeEach`: 每個測試前執行,準備 fixture
   - `@BeforeAll`: 所有測試前執行一次
   - `@AfterEach`: 每個測試後執行,清理 fixture
   - `@AfterAll`: 所有測試後執行一次

   好的 test fixture 設計可以讓測試更容易維護,減少重複的程式碼。

---

## 第九部分:效能優化 (Mid)

### Q36. 查詢效能優化 (Mid)

**解答**:

1. **效能問題在哪**:

   這段程式碼最大的問題就是 N+1 Query。你看 `mapToOrderResponse` 這個方法,它要先查一次訂單,然後再去查使用者的 username,接著對訂單裡的每個 item 都要再查一次 journey。假設一個訂單有 5 個課程,那就要執行 1 + 1 + 5 = 7 次 query。如果你要顯示 100 個訂單,那就是 700 次 query,這效能肯定爆炸。

   而且這還是在 Controller 層做轉換的時候才發現問題,這時候 transaction 可能都已經關閉了,lazy loading 也會炸。

2. **優化方式**:

   有幾種做法。第一種是用 `JOIN FETCH` 在查詢訂單的時候就一次把相關的資料都抓回來。在 Repository 寫一個 query 像是 `SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id`,這樣就能一次把 order 跟 items 都拉回來。

   第二種是用 `@EntityGraph` 註解,在 Repository 方法上標註要 eager fetch 哪些關聯。這個比較方便,不用寫 JPQL。

   第三種是用 DTO projection,直接在 query 裡面就組好要回傳的資料,像是用 JPQL 的 `new` constructor expression,或是用 native query。這個效能最好,因為只查需要的欄位,不會拿到一堆用不到的資料。

3. **何時用哪種**:

   如果你只是要 eager load 關聯的 entity,用 `@EntityGraph` 最簡單。如果你需要跨很多張表的複雜查詢,或是只要某幾個欄位,那就用 DTO projection。一般來說,API 回傳給前端的資料最好都用 DTO projection,因為你通常不需要完整的 entity,只要部分欄位。

4. **資料量大的考量**:

   如果資料量很大,還要考慮 pagination。不能一次查全部,要分頁查詢。另外也可以考慮加 index,確保查詢效能。如果是很常查的資料,也可以考慮加 cache。

   還有一個重點是要避免在 loop 裡面查資料庫。像這個例子,如果有 10 個 order items,就要查 10 次 journey。應該要先把所有的 journey IDs 收集起來,用 `WHERE id IN (...)` 一次查回來,然後在記憶體裡面做 mapping。

---

### Q37. Cache 的使用時機 (Mid)

**解答**:

1. **什麼適合放 cache**:

   適合放 cache 的資料通常有幾個特性:讀多寫少、不常變動、查詢成本高。像是課程資訊、章節列表這種基本上建立後就不太會改的,就很適合。使用者的個人資料、權限設定這些也可以 cache。

   不適合 cache 的是那種即時性要求高、常常變動的資料,像是訂單狀態、庫存數量、使用者的學習進度。如果 cache 這些資料,很容易就會跟資料庫不一致,造成使用者看到過期的資訊。

2. **Cache 策略**:

   常見的有幾種。**Cache-Aside** 是最常用的,應用程式自己管理 cache,查詢時先看 cache 有沒有,沒有才去查資料庫,然後把結果放進 cache。

   **Read-Through** 是 cache layer 自己去資料庫查,應用程式不用管。**Write-Through** 是寫入時同時寫 cache 和資料庫。**Write-Behind** 是先寫 cache,非同步再寫資料庫,這個效能最好但風險也最高,萬一 cache 掛了資料就不見了。

   我們專案如果要用,Cache-Aside 最保險,用 Spring 的 `@Cacheable` 註解就能搞定。

3. **避免不一致**:

   最簡單的方式就是設定過期時間(TTL),讓 cache 自動失效。或是在資料更新時,主動去 invalidate 對應的 cache,用 `@CacheEvict` 就可以。

   但要小心的是,如果是分散式系統,可能多台機器都有 local cache,這時候 invalidate 就很麻煩,需要用 Redis 這類集中式的 cache,或是用 pub/sub 來同步 cache 失效的訊息。

4. **過期時間設定**:

   這要看資料的特性。像課程標題這種幾乎不變的,可以設長一點,比如 1 小時或甚至 1 天。使用者資訊可能 5-10 分鐘。如果是比較即時但又不是超級即時的,像是討論區的回覆數,可以設 1-2 分鐘。

   太短會讓 cache 形同虛設,太長又會有資料不一致的問題,要找個平衡。

5. **分散式環境**:

   如果是多台 server,就不能用 local cache(像 Caffeine),要用 Redis、Memcached 這類集中式的。這樣所有 server 看到的 cache 都是一致的。

   但要注意的是,網路延遲會比 local cache 慢,所以可以考慮用兩層 cache:local cache 作為第一層,Redis 作為第二層。這樣常用的資料可以直接從記憶體拿,不用每次都打網路。

---

### Q38. 批次操作的優化 (Mid)

**解答**:

1. **效能問題**:

   這段程式碼的問題是在 loop 裡面一筆一筆 save,假設有 1000 個過期的訂單,就要執行 1000 次 UPDATE query。每次 query 都有網路往返的成本,而且資料庫也要一直 parse SQL、執行、commit,這樣非常沒效率。

   更慘的是,這些都在同一個 transaction 裡面,如果中間某一筆出錯,整個 transaction 都要 rollback,前面的努力都白費了。

2. **改成批次更新**:

   最簡單的方式是用 `saveAll()`,把所有要更新的 entities 一次傳進去。JPA 會自動做 batch update,一次送多筆到資料庫。但要注意,要在 application.properties 設定 `spring.jpa.properties.hibernate.jdbc.batch_size=50` 這種參數,不然 Hibernate 預設不會真的做 batch。

   更激進的做法是用 native query 或 JPQL 的 bulk update,直接寫一個 `UPDATE orders SET status = 'EXPIRED' WHERE status = 'UNPAID' AND expired_at < :now`,這樣資料庫一次就搞定所有的訂單,不用一筆一筆處理。但缺點是這樣就繞過了 JPA 的 lifecycle callbacks,`@PreUpdate` 那些就不會被呼叫了。

3. **批次大小設定**:

   通常設定在 50-100 之間比較合適。太小的話,效能提升不明顯;太大的話,可能會超過資料庫的限制,或是佔用太多記憶體。

   如果資料量真的很大,比如要更新 10 萬筆,那就要考慮分批處理,用 pagination 的方式,每次處理 1000 筆,處理完再處理下一批。這樣可以避免 OOM,也不會讓某一個 transaction 執行太久。

4. **處理部分失敗**:

   如果用 bulk update,通常是 all-or-nothing,要嘛全部成功,要嘛全部失敗。

   如果想要更細緻的控制,可以考慮把 `@Transactional` 範圍縮小,每一批用一個獨立的 transaction。或是把整個 batch 分成多個小 batch,每個 batch 用自己的 transaction,這樣即使某一批失敗,其他批次的資料還是會被儲存。

   不過這要小心處理,要確保系統的一致性不會因此受影響。通常在 scheduled task 裡面,如果某筆失敗了,記錄 log 然後跳過,下一次再處理也是可以接受的。

---

## 第十部分:架構設計與重構 (Mid)

### Q39. 分層架構的設計原則 (Mid)

**解答**:

1. **三層職責**:

   **Controller** 主要負責 HTTP 層的東西,像是接收請求、驗證基本的格式、呼叫 Service、把結果轉成 HTTP response。它不應該有任何 business logic,也不應該直接操作 Repository。就是一個薄薄的適配層,把 HTTP 世界跟應用程式的 domain 世界連接起來。

   **Service** 是 business logic 的核心,所有的商業邏輯都在這裡。像是建立訂單要檢查使用者有沒有買過、計算價格、處理付款流程,這些都是 Service 的工作。Service 也負責協調多個 Repository,控制 transaction 的範圍。

   **Repository** 就是專門負責資料存取,提供 CRUD 操作。它不應該有 business logic,只是單純地跟資料庫溝通。查詢條件可以寫在這裡,但判斷「要不要查」、「查完要幹嘛」這些都不是 Repository 該管的。

2. **DTO vs Entity**:

   Entity 是資料庫的映射,跟資料庫的 table 結構一致,它的職責是表達 domain model。DTO 則是專門用來在不同層之間傳遞資料,特別是跟外部世界(像前端)溝通時用的。

   為什麼要分開?第一,安全性。Entity 可能有一些敏感欄位像 passwordHash,你不想回傳給前端。第二,解耦合。如果資料庫 schema 改了,不代表 API response 也要跟著改。第三,效能。Entity 可能有一堆關聯,如果直接序列化可能會觸發 lazy loading,產生一堆額外的 query。

3. **Service 太長怎麼重構**:

   首先可以看看是不是可以拆成多個小方法,每個方法做一件事。然後看看有沒有重複的邏輯可以抽出來。

   如果整個 Service 還是太大,可以考慮按照功能拆成多個 Service。比如 OrderService 太大了,可以拆成 OrderCreationService、OrderPaymentService、OrderQueryService 這樣。

   或是引入一些設計模式,像是 Strategy Pattern 來處理不同的付款方式,或是 Chain of Responsibility 來處理一連串的驗證。也可以把一些邏輯抽到 Validator 或 Helper 類別。

4. **DDD 的差異**:

   Domain Driven Design 強調 domain model 的豐富性,Entity 不只是資料容器,還包含 business logic。像 `order.markAsPaid()` 這種就是 DDD 的思維,把行為放在 entity 裡面。

   傳統的三層架構比較傾向 Anemic Domain Model,Entity 只有 getter/setter,所有邏輯都在 Service。這個專案算是混合式的,有一些簡單的 business method 放在 Entity,但主要的協調邏輯還是在 Service。

   完整的 DDD 還會有 Aggregate、Value Object、Repository 的嚴格定義,以及 Domain Event 這些概念。對於中小型專案來說,傳統三層架構已經夠用,不用過度設計。

---

### Q40. Validator 的設計 (Mid)

**解答**:

1. **為什麼獨立 Validator**:

   把驗證邏輯獨立出來有幾個好處。第一是 Single Responsibility Principle,Service 已經夠忙了,不要再塞一堆驗證的 if-else。第二是可重用性,像是「檢查訂單是否已付款」這個邏輯,可能在好幾個地方都會用到,抽成 Validator 就可以共用。

   第三是可測試性,Validator 通常是 stateless 的,很容易寫單元測試。第四是可讀性,當你看到 `orderValidator.validateOrderNotExpired(order)` 這種,一看就知道在做什麼,不用去翻一大段 if 條件。

2. **Validator vs Service**:

   Validator 負責的是「檢查某個條件是否滿足」,它不改變系統狀態,只是檢查然後決定要不要拋出例外。Service 則是「執行某個操作」,會改變系統狀態,像是建立訂單、更新資料。

   簡單來說,Validator 是 guard,Service 是 worker。Validator 確保前提條件都滿足了,Service 才真正去做事情。

3. **什麼邏輯放哪裡**:

   Validator 適合放那種「檢查型」的邏輯,像是「使用者是否存在」、「訂單是否已過期」、「課程是否已購買」。這些都是檢查某個條件,失敗就拋例外。

   Service 適合放那種「協調型」的邏輯,像是「呼叫多個 Repository」、「計算價格」、「發送通知」。這些不只是檢查,還有實際的操作。

   有時候界線不是很清楚。一個判斷原則是:如果這個邏輯需要查詢資料庫或呼叫外部服務,然後基於結果做判斷,那通常放 Validator。如果這個邏輯需要修改資料或協調多個操作,那就放 Service。

4. **回傳 boolean 還是拋例外**:

   我比較推薦拋例外。原因是如果回傳 boolean,呼叫者還要自己判斷然後決定要做什麼,很容易忘記處理或處理不一致。拋例外的話,驗證失敗就直接中斷,不用擔心後面的程式碼會繼續執行。

   而且拋例外可以攜帶更多資訊,像是「訂單 123 已過期」,這樣 Global Exception Handler 可以統一處理,轉成適當的 HTTP response 給前端。

   如果有些情況真的需要根據驗證結果做不同的處理(而不是直接失敗),那可以考慮用 sealed interface 回傳 Success/Failure,但這種情況比較少見。

---

### Q41. DTO 的轉換時機 (Mid)

**解答**:

1. **為什麼在 Service 層轉換**:

   主要原因是要保持分層架構的清晰。Service 層處理完 business logic 後,回傳的應該是適合給外部使用的資料格式,而不是內部的 domain model。這樣 Controller 就變得很單純,只要把 Service 回傳的 DTO 包裝成 HTTP response 就好,不用管轉換的邏輯。

   而且在 Service 層轉換,可以確保轉換是在 transaction 內完成的,如果有 lazy loading 的關聯需要存取,不會出錯。如果等到 Controller 才轉換,transaction 可能已經關閉了,一存取 lazy loading 就會拋 LazyInitializationException。

2. **直接回傳 Entity 的問題**:

   第一個問題是**安全性**。Entity 可能包含敏感資料,像是 passwordHash、內部的 ID、audit 欄位這些,你不希望暴露給前端。直接回傳 entity 很容易不小心洩漏這些資訊。

   第二是 **Jackson 序列化的問題**。Entity 通常有很多關聯,像是 Order 有 items,如果直接序列化可能會觸發 lazy loading,產生一堆額外的 query。更糟的是,如果有雙向關聯,還會造成 circular reference,序列化會失敗或無限迴圈。

   第三是**耦合問題**。如果你直接回傳 entity,前端就跟你的資料庫 schema 綁死了。哪天你想改 entity 的結構,可能會破壞 API 的相容性。用 DTO 就可以把內部結構跟外部 API 解耦。

3. **MapStruct/ModelMapper 的優缺點**:

   **優點**是可以自動處理很多重複的 mapping 邏輯,特別是那種欄位名稱一樣、一對一對應的情況,可以省很多 boilerplate code。而且 MapStruct 是在編譯時期生成程式碼,效能跟手寫差不多。

   **缺點**是學習曲線,要理解它的慣例和設定方式。而且當 mapping 邏輯比較複雜時,像是需要呼叫其他 service 或 repository、要做條件判斷,這些工具反而會變得很難用。有時候手寫一個 `mapToDTO()` 方法,雖然多寫幾行程式碼,但更清楚明白。

   我個人的經驗是,對於簡單的 mapping 可以考慮用這些工具,但複雜的還是手寫比較實在。

4. **DTO 應該是 immutable 嗎**:

   我認為**應該是 immutable**,特別是用 `record` 就是個很好的選擇。原因有幾個:

   第一,DTO 的職責就是傳遞資料,不是用來修改的。如果它是 mutable,可能會被不小心改到,造成預期外的行為。

   第二,immutable 物件天生 thread-safe,不用擔心併發問題。在多執行緒環境下,你可以安心地共用這些物件。

   第三,`record` 自動提供了 `equals()`、`hashCode()`、`toString()`,這些在測試和 debug 時超方便。

   唯一可能不用 immutable 的情況是,如果 DTO 很大,而且需要 builder pattern 來建構,那可能用 class 配合 Lombok 的 `@Builder` 會比較方便。但這種情況不多見。

---

### Q42. Business Logic 的位置 (Mid)

**解答**:

1. **哪種設計比較好**:

   我認為**設計 A 比較好**,也就是把 business logic 放在 Entity 裡面。`markAsPaid()` 這種方法封裝了「標記為已付款」這個業務概念,它不只是改兩個欄位,而是表達了一個有意義的操作。

   設計 B 的問題是,呼叫者必須知道「標記為已付款」需要改哪些欄位,這些細節不應該暴露出來。如果哪天我們決定付款時還要記錄其他資訊,設計 B 就要到處修改,但設計 A 只要改 `markAsPaid()` 方法就好。

   而且設計 A 更好測試。你可以寫一個單元測試,呼叫 `markAsPaid()`,然後驗證 status 和 paidAt 都被正確設定,不用管 Service 層的複雜邏輯。

2. **Anemic vs Rich Domain Model**:

   **Anemic Domain Model** 就是 entity 只有 getter/setter,沒有行為,所有邏輯都在 Service。這樣的 entity 就像是個資料容器,沒有自己的智慧。

   **Rich Domain Model** 則是 entity 本身包含 business logic,知道怎麼操作自己的狀態。像是 `order.markAsPaid()`、`user.addExperience()`、`mission.isCompleted()` 這種,entity 有自己的行為。

   我比較偏好 Rich Domain Model,因為它更符合物件導向的精神,資料跟行為應該放在一起。但也不是說要把所有邏輯都塞到 entity,要看情況。

3. **什麼邏輯放哪裡**:

   放在 **Entity** 的邏輯通常是:

   - 跟單一 entity 狀態相關的操作,像是 `markAsPaid()`、`softDelete()`
   - 不需要依賴其他 service 或 repository 的商業規則
   - Entity 內部的驗證邏輯,像是 `isExpired()`、`isPaid()`

   放在 **Service** 的邏輯通常是:

   - 需要協調多個 entity 的操作,像是建立訂單要處理 User、Order、OrderItem、Journey
   - 需要呼叫其他 service 或 repository
   - 涉及 transaction 管理的邏輯
   - 需要複雜計算或呼叫外部 API 的操作

   簡單的判斷原則:如果這個操作「只跟這個 entity 有關」,放 entity;如果需要「協調多個物件」或「依賴外部資源」,放 service。

---

## 第十一部分:程式碼品質與慣例 (Junior/Mid)

### Q43. Immutability 的好處 (Mid)

**解答**:

1. **為什麼用 record**:

   `record` 是 Java 14 引入的新特性,專門用來表示「不可變的資料載體」。相比 class,它有幾個優勢:首先是簡潔,一行就能定義完整的 DTO,不用寫一堆 getter、constructor、equals、hashCode。

   其次是語意清楚,當你看到 `record`,就知道這是個 immutable 的資料容器,不會有複雜的行為。如果看到 `class`,你不確定它是 mutable 還是 immutable,要去看 field 是不是 final 才知道。

   第三是編譯器保證,`record` 的所有 field 都自動是 `final`,你沒辦法不小心把它變成 mutable。

2. **Immutable 的好處**:

   最大的好處是**簡化推理**。當你拿到一個 immutable 物件,你知道它永遠不會變,不用擔心誰會偷改它。這讓程式碼更容易理解,也更容易 debug。

   第二是**線程安全**。Immutable 物件可以安全地在多個執行緒間共用,不需要 synchronization。這在高併發的系統中非常重要。

   第三是**可以作為 Map 的 key**。如果物件是 mutable,放到 Map 後又被改了,hashCode 就變了,會造成找不到資料。Immutable 物件就沒這個問題。

   第四是**防止意外修改**。特別是當物件在多個層之間傳遞時,你不希望某一層改了它而影響其他層。

3. **多執行緒環境的幫助**:

   在多執行緒環境下,最常見的問題就是 race condition,多個執行緒同時讀寫同一個物件,造成資料不一致。

   如果物件是 immutable,根本就不存在「寫」的操作,自然就沒有 race condition。每個執行緒拿到的都是獨立的物件(或是安全地共用同一個不變的物件),不會互相影響。

   這也是為什麼在設計 DTO 時,強烈建議用 immutable。DTO 在 Controller、Service、Repository 之間傳遞,如果是 mutable,很容易被不小心改到,造成詭異的 bug。

4. **Entity 應該是 immutable 嗎**:

   **Entity 不應該是 immutable**,因為它的職責就是表達「可變的狀態」。像是訂單從未付款變成已付款、使用者的經驗值增加,這些都是狀態的改變。

   如果 entity 是 immutable,每次狀態改變都要建立新物件,這樣 JPA 就無法追蹤它的變化,也沒辦法正確地更新資料庫。

   所以正確的做法是:Entity 是 mutable 但提供有意義的 business method 來改變狀態,像是 `markAsPaid()` 而不是直接 `setStatus()`。DTO 則應該是 immutable,用 `record` 來定義。

---

### Q44. Null Safety (Junior)

**解答**:

1. **@NonNull 的作用**:

   `@NonNull` 是一個標記型註解,通常來自 Spring 的 `org.springframework.lang.NonNull` 或 Lombok。它的作用是告訴開發者和工具「這個參數或回傳值不應該是 null」。

   有些 IDE 像 IntelliJ 會讀取這個註解,如果你傳 null 進去,會給你警告。有些框架也會在 runtime 檢查,如果真的是 null 就拋出例外,讓問題早點暴露。

   雖然它不像 Kotlin 的 null safety 是編譯器強制的,但至少能起到文件和輔助檢查的作用,讓程式碼的意圖更明確。

2. **如何避免 NullPointerException**:

   第一,**不要回傳 null**。如果一個方法可能找不到資料,用 `Optional` 回傳,強迫呼叫者處理「找不到」的情況。

   第二,**用 `@NonNull` 標記參數**。讓呼叫者知道不能傳 null,如果真的傳了,在方法入口就檢查並拋出更清楚的例外。

   第三,**給欄位設預設值**。像是 `List<OrderItem> items = new ArrayList<>()`,不要讓 collection 是 null,用空集合代表「沒有元素」。

   第四,**使用 Objects.requireNonNull()**。在真的需要確保不是 null 的地方,明確地檢查並拋出清楚的訊息。

   第五,**小心 auto-unboxing**。`Integer` 轉 `int` 時,如果是 null 就會炸。要先檢查或用 `Optional`。

3. **Optional vs null**:

   直接回傳 null 的問題是,呼叫者很容易忘記檢查,直接就用了,然後就 NPE。而且光看方法簽名你不知道它會不會回傳 null,要去翻文件或原始碼。

   `Optional` 強制你意識到「這個值可能不存在」。你必須用 `isPresent()`、`orElse()`、`orElseThrow()` 這些方法來處理,編譯器和 IDE 都會提醒你。這讓程式碼更安全,也更清楚表達 API 的語意。

   但要注意,`Optional` 不是萬能的,不要濫用。比如 method 參數就不應該用 Optional,直接標記 `@NonNull` 或 `@Nullable` 就好。`Optional` 適合用在回傳值,表示「可能找不到」。

4. **Java 的 null safety**:

   Java 沒有像 Kotlin 那種語言層級的 null safety,編譯器不會強制你處理 null。但可以透過一些工具和慣例來改善:

   - 用 `Optional` 明確表達可能為 null 的情況
   - 用 `@NonNull`、`@Nullable` 註解標記,讓 IDE 做靜態分析
   - 用 Checker Framework 這類工具做更嚴格的 null 檢查
   - 寫程式時遵循「盡量不用 null」的原則

   但說實話,Java 在這方面還是比 Kotlin 弱很多。Kotlin 從語言設計上就把 nullable 跟 non-nullable 區分開,編譯器會強制你處理,根本不可能寫出忘記檢查 null 的程式碼。

---

### Q45. Magic Number 的處理 (Junior)

**解答**:

1. **為什麼要定義常數**:

   `DEFAULT_EXPERIENCE_REWARD = 100` 這樣定義有幾個好處。第一是**可讀性**,當你看到 `user.addExperience(DEFAULT_EXPERIENCE_REWARD)`,馬上知道這是預設的經驗值獎勵,不用去猜「100 是什麼意思」。

   第二是**維護性**,如果哪天要改預設獎勵,只要改一個地方就好,不用全專案搜尋 `100` 然後不確定哪些該改、哪些不該改。

   第三是**避免錯誤**,如果到處寫死 100,很容易某些地方忘記改,造成不一致。用常數就能確保所有地方都用同一個值。

2. **plusDays(3) 是否該定義常數**:

   **應該定義**。雖然只用了一次,但「訂單過期天數」是一個業務規則,不是任意的數字。如果定義成 `ORDER_EXPIRATION_DAYS = 3`,未來要改成 5 天或 7 天就很容易,而且程式碼也更清楚表達意圖。

   可以定義成 class 的 constant,像是 `private static final int ORDER_EXPIRATION_DAYS = 3`。這樣任何人看到這段程式碼,都知道「喔,原來訂單會在 3 天後過期」,不用去猜。

3. **如何管理設定值**:

   對於那些可能會變動的設定,有幾種做法:

   **application.properties / application.yml**: 適合不同環境(dev/staging/prod)可能有不同值的設定,像是連線設定、timeout、batch size。優點是不用改 code 就能調整,缺點是要重啟應用。

   **Database configuration table**: 適合需要動態調整、不想重啟的設定,像是功能開關、活動參數。可以做個後台讓管理員調整,甚至可以 reload cache。缺點是多了一次資料庫查詢。

   **Enum 或 Constants class**: 適合不太會變動、跟業務邏輯強相關的常數,像是 `DEFAULT_EXPERIENCE_REWARD`、`MAX_USERNAME_LENGTH`。優點是類型安全,缺點是要改就要重新編譯部署。

   選擇哪種要看需求。如果是核心的業務規則,我傾向用 constants class;如果是環境相關的,用 properties;如果需要經常調整,用資料庫。

4. **什麼是 Magic Number**:

   Magic Number 就是那種出現在程式碼裡,但不知道代表什麼意義的數字。像是 `if (status == 3)`,你不知道 3 是什麼意思。或是 `position * 0.95`,為什麼乘以 0.95?這個 0.95 代表什麼?

   避免 Magic Number 的方式就是給它一個有意義的名字。`if (status == OrderStatus.PAID)`、`position * PROGRESS_COMPLETION_THRESHOLD`。這樣程式碼就自我說明了,不用靠註解或去翻文件。

   不過也不要矯枉過正。像是 `list.get(0)`、`count + 1` 這種,0 和 1 的意義很明顯,不用特別定義常數。用常識判斷,如果這個數字有特定的業務意義,或是可能會變動,就該定義成常數。

---

## 第十二部分:部署與維運 (Mid)

### Q46. Profile 的使用 (Junior)

**解答**:

1. **Profile 是什麼**:

   Spring Boot 的 Profile 就是用來管理不同環境的設定檔。開發環境可能需要連本地資料庫、開啟 debug log,但正式環境要連正式資料庫、關掉 debug。Profile 讓你可以為不同環境準備不同的設定,啟動時指定要用哪個就好。

   通常會有幾個 profile:dev(開發)、staging(測試環境)、prod(正式環境)。每個 profile 有自己的 `application-dev.properties`、`application-prod.properties`。

2. **如何設定不同環境**:

   首先建立不同的 properties 檔案,像是 `application-dev.properties` 裡面寫開發環境的資料庫連線、log level 設成 DEBUG。`application-prod.properties` 就寫正式環境的資料庫、log level 設成 INFO 或 WARN。

   然後在 `application.properties` 可以放共用的設定,像是 server port、JWT secret(但這個其實應該用環境變數)。各環境特定的設定就放在各自的 profile 檔案裡。

   Profile 也可以用來控制哪些 Bean 要載入。用 `@Profile("dev")` 標記某個 configuration,那它只有在 dev profile 才會生效。比如開發環境可能要一個 mock 的 email service,正式環境才用真的。

3. **如何指定 profile**:

   有幾種方式。最常見的是用環境變數 `SPRING_PROFILES_ACTIVE=prod` 或是啟動參數 `-Dspring.profiles.active=prod`。

   如果用 Docker,可以在 docker-compose.yml 或 Kubernetes 的 deployment 設定環境變數。如果直接用 jar 跑,就是 `java -jar app.jar --spring.profiles.active=prod`。

   也可以在 application.properties 裡面設 `spring.profiles.active=dev` 當作預設值,但這通常只在開發時用,部署時應該明確指定。

4. **Connection pool 設定放哪**:

   Connection pool 的設定通常會因環境而異。開發環境可能只需要 5-10 個連線,但正式環境可能需要 50-100 個。所以應該放在各自的 profile 檔案裡。

   像是在 `application-prod.properties` 可以寫:

   - `spring.datasource.hikari.maximum-pool-size=50`
   - `spring.datasource.hikari.minimum-idle=10`
   - `spring.datasource.hikari.connection-timeout=30000`

   這樣你就可以針對不同環境的負載,調整最適合的 connection pool 大小。開發環境不用那麼大,正式環境再開大一點。

---

### Q47. Logging 策略 (Mid)

**解答**:

1. **生產環境的 log level**:

   正式環境通常設成 **INFO** 或 **WARN**。DEBUG 太吵了,會記錄一堆細節,log 檔案會爆炸,而且影響效能。ERROR 又太少,很多重要資訊會漏掉。

   INFO 可以記錄重要的 business events,像是「訂單建立成功」、「使用者登入」、「排程任務執行」。WARN 記錄一些異常但不致命的狀況,像是「重試 3 次後成功」、「某個外部 API 回應慢」。ERROR 就是真的發生錯誤,需要立即處理的。

   可以用 profile 來控制,開發環境設 DEBUG,staging 設 INFO,正式環境設 WARN。但要保留調整的彈性,萬一正式環境要 debug,可以動態調整 log level 而不用重啟。

2. **避免 log 檔案無限成長**:

   要設定 **log rotation**,也就是當 log 檔案到達一定大小或時間,就切換到新檔案,舊的壓縮或刪除。

   用 Logback(Spring Boot 預設)的話,可以設定像是「每天產生新檔案」、「單檔最大 100MB」、「最多保留 30 天」。這樣 log 就不會無限成長,又能保留足夠的歷史紀錄供查詢。

   或是用 ELK(Elasticsearch, Logback, Kibana)或類似的集中式 logging 系統,把 log 送到外部儲存,應用程式本身不保留檔案。這樣更好管理,也更容易搜尋和分析。

3. **什麼該記錄,什麼不該**:

   **應該記錄**的:

   - 重要的 business events(訂單建立、付款成功、使用者註冊)
   - 錯誤和例外(包含 stack trace)
   - 效能相關的資訊(API response time、database query time)
   - 安全相關的事件(登入失敗、權限檢查失敗)
   - 外部服務的呼叫和回應

   **不應該記錄**的:

   - 密碼、token、信用卡號這些敏感資料
   - 完整的個人資料(可能違反 GDPR 或隱私法)
   - 太多的 debug 訊息(正式環境)
   - 使用者的完整行為軌跡(除非有明確需求和權限)

   如果真的要記錄某些可能包含敏感資訊的東西,要先做 masking 或 redaction,像是密碼只記前後兩個字元,中間用星號代替。

4. **Distributed Tracing**:

   在微服務或多台機器的環境下,一個請求可能經過好幾個服務。要追蹤整個流程,需要 **Correlation ID** 或 **Trace ID**。

   做法是在請求進來時產生一個唯一的 ID,然後在所有的 log 都加上這個 ID。當你要 debug 某個請求時,就可以用這個 ID 搜尋所有相關的 log,看到完整的請求流程。

   可以用 Spring Cloud Sleuth 自動幫你加上 trace ID 和 span ID,或是手動用 MDC(Mapped Diagnostic Context)來管理。如果要更進階的 tracing,可以用 Zipkin、Jaeger 這類分散式追蹤系統,可以視覺化整個請求的呼叫鏈。

---

### Q48. Health Check 的實作 (Junior)

**解答**:

1. **Health check endpoint 的作用**:

   Health check 是用來監控應用程式的健康狀態。像 Kubernetes、AWS ELB 這些 orchestration 工具或 load balancer,會定期呼叫 health check endpoint,如果回應不正常就把這台機器從服務中移除或重啟。

   這樣可以確保流量只會導到健康的機器,如果某台機器的資料庫連不上或記憶體爆掉,load balancer 就不會再把請求送過去,保護使用者不會遇到錯誤。

2. **應該檢查什麼**:

   基本的要檢查 **資料庫連線**,確保可以正常查詢。如果專案有用到 **Redis**,也要檢查 Redis 是否可連線。如果有呼叫 **重要的外部 API**,也可以檢查(但要小心,如果外部 API 掛了,不代表你的服務也該掛)。

   也可以檢查一些系統資源,像是記憶體使用率、磁碟空間,如果快滿了就回報 unhealthy。但要注意,health check 要夠快,不能執行太複雜的檢查,不然每秒都有人打,會變成額外的負擔。

   我們專案的 HealthController 就是檢查資料庫連線,這是最基本也最重要的。如果資料庫連不上,應用程式基本上就不能用了。

3. **Liveness vs Readiness**:

   **Liveness probe** 是檢查「這個應用程式是不是還活著」。如果失敗,Kubernetes 會**重啟 pod**。這個適合用來偵測 deadlock、記憶體洩漏這種無法自己恢復的狀況。

   **Readiness probe** 是檢查「這個應用程式是不是準備好接收流量」。如果失敗,Kubernetes 會**把它從 service 移除**,但不會重啟。這個適合用來處理暫時性的問題,像是應用程式還在啟動、連不上資料庫、或是正在處理大量請求所以暫時不想接新請求。

   通常 liveness 只檢查最基本的東西,像是應用程式還能回應就好。Readiness 就檢查得仔細一點,包含資料庫、外部依賴這些。

4. **失敗時回傳什麼 status code**:

   Health check 失敗應該回傳 **503 Service Unavailable**。我們的 HealthController 就是這樣做的,status 是 "UP" 就回傳 200 OK,不是的話就回傳 503。

   為什麼是 503 而不是 500?因為 500 代表「伺服器錯誤」,暗示這是個 bug。503 代表「暫時無法服務」,暗示這可能是暫時的,過一下可能就好了。Load balancer 看到 503 會知道「這台機器現在不行,但可能等一下就恢復了」。

---

## 第十三部分:情境題與設計決策 (Mid)

### Q49. 設計一個「限時優惠」功能 (Mid)

**解答**:

1. **Database schema 設計**:

   我會建立一個 `journey_promotions` 表,包含:

   - `journey_id`: 哪個課程
   - `original_price`: 原價
   - `promotion_price`: 優惠價
   - `start_at`: 優惠開始時間
   - `end_at`: 優惠結束時間
   - `created_at`, `updated_at`: audit 欄位

   然後在建立訂單時,會 snapshot 當下的價格到 `order_items` 表。這樣即使優惠結束了,已建立的訂單還是保持當時的價格。訂單裡的 `price` 欄位就是「鎖定」的價格,不會因為課程價格變動而改變。

   也可以考慮在 order_items 加一個 `promotion_id`,記錄是用了哪一個優惠,方便後續追蹤和分析。

2. **價格鎖定的邏輯**:

   關鍵是在 **建立訂單時就 snapshot 價格**。不要只記錄 journey_id 然後每次都去查最新價格,而是在建單的那一刻就把 original_price、discount、final_price 都記到 order_items 裡。

   這樣的設計有個好處:訂單一旦建立,價格就固定了,不管後來課程怎麼改價、優惠有沒有結束,都不影響這張訂單。這也符合商業邏輯,使用者在優惠期間加入購物車,就應該享有那個價格,即使結帳時優惠已結束。

   在 Service 層,建單時會檢查當前時間是否在 `start_at` 和 `end_at` 之間,如果是就用 promotion_price,不是就用 original_price。檢查完後立刻把價格寫進訂單,之後就不再改變。

3. **時區問題**:

   這個很重要但常被忽略。資料庫應該統一用 **UTC** 儲存所有時間,不要用 local time。Java 用 `LocalDateTime` 或更好用 `Instant`(它本身就是 UTC)。

   優惠的 start_at 和 end_at 也都用 UTC。前端顯示時再根據使用者的時區轉換。這樣才不會有「台灣時間 12/1 00:00 開始優惠,但美國使用者看到的是 11/30 就開始了」這種混亂。

   在比較時間時,都轉成 UTC 再比,確保結果一致。像是檢查「現在是否在優惠期間」,就是 `Instant.now().isAfter(startAt) && Instant.now().isBefore(endAt)`。

4. **避免惡意囤積訂單**:

   問題是使用者可能在優惠期間狂建訂單但不付款,優惠結束後慢慢付,或是拿去賣給別人。

   幾個防範方式:

   - **限制未付款訂單數量**: 每個使用者同時只能有 1-2 張未付款訂單。想建新訂單?先把舊的付掉或取消。
   - **縮短過期時間**: 優惠訂單可能只給 1 天或甚至幾小時過期,不是 3 天。這樣減少囤積的誘因。
   - **訂單不可轉讓**: 訂單綁定建立者的帳號,不能改。
   - **優惠名額限制**: 前 100 名才有優惠,賣完就恢復原價。這樣即使有人囤單,也影響有限。

   我會選擇「限制未付款訂單數量」加上「縮短過期時間」,這樣實作簡單,效果也不錯。

---

### Q50. 架構重構決策 (Mid)

**解答**:

1. **如何重構**:

   首先要**識別問題的根源**。Service 太大通常是因為承擔了太多職責,或是沒有適當的抽象。我會先分析這個 Service 做了哪些事情,畫出它的職責邊界。

   然後按照功能領域拆分。比如 OrderService 如果有建立、查詢、付款、取消等功能,可以拆成 OrderCommandService(處理寫入)和 OrderQueryService(處理讀取),或是進一步拆成 OrderCreationService、OrderPaymentService。

   把重複的邏輯抽成 Helper 或 Util 類別,或是獨立的 component。驗證邏輯放 Validator,domain 行為放 Entity,複雜的計算可以抽成獨立的 Calculator 或 Strategy。

   最後,逐步重構,不要一次改太多。先抽一小塊出來,確保測試都過了,再繼續下一塊。

2. **應該引入哪些設計模式**:

   **Strategy Pattern**: 如果有多種付款方式、多種價格計算邏輯,可以用 Strategy 來封裝這些變化。

   **Factory Pattern**: 如果建立物件的邏輯很複雜,用 Factory 來集中管理,Service 就不用管建立的細節。

   **Template Method**: 如果有一些流程很類似但某些步驟不同,可以用 Template Method 定義骨架,子類別實作細節。

   **Domain Events**: 如果操作之間有複雜的連鎖反應,像是訂單付款後要發送 email、更新庫存、記錄 log,可以用 Event 來解耦,不用在 Service 裡寫一長串的 if-then。

   但要注意,**不要為了用設計模式而用**。如果問題不複雜,硬要套設計模式反而會 over-engineering。從實際問題出發,覺得某個模式能解決問題,再引入。

3. **確保不破壞功能**:

   最重要的是**測試**。如果原本的程式碼沒有測試,重構前先補測試。有了完整的測試套件,你才能放心重構,跑測試確認行為沒變。

   採用**小步重構**,每次只改一點點,馬上跑測試,確認沒問題再繼續。不要一口氣重寫整個 Service,風險太高。

   可以用**Strangler Pattern**,就是新舊系統並存一段時間。新功能用新的架構寫,舊功能逐步遷移。這樣可以漸進式重構,不會有「大爆炸式」的風險。

   重構時也要注意 API 的相容性,如果有對外的 API,不要輕易改變 interface。可以內部重構但保持對外的 contract 不變。

4. **如何說服團隊**:

   首先要**量化問題**。不要只說「程式碼很亂」,而是說「平均修一個 bug 要 3 天,測試覆蓋率只有 40%,最近 3 個月有 5 個 production incident」。用數據說話,讓大家看到問題的嚴重性。

   其次,**提出明確的計畫和時程**。不是說「我們要重構」,而是「我們計畫用 4 週的時間,每週花 20% 的時間重構,預計能把 OrderService 從 1000 行減到 500 行,測試覆蓋率提升到 70%」。

   第三,**展示 quick win**。先挑一個小範圍重構,快速見效,讓大家看到改善。比如先把一個 Validator 抽出來,程式碼變乾淨了,bug 也減少了。有了成功案例,就更容易說服團隊繼續。

   最後,**強調長期價值**。技術債不處理會越積越多,維護成本會越來越高,新功能開發會越來越慢。現在投資時間重構,未來會省下更多時間,也能降低風險。

---

## 總結

恭喜你完成 50 道後端面試題的準備！這些題目涵蓋了：

- ✅ Java 語法與現代特性
- ✅ Spring Boot 框架與架構
- ✅ JPA 與資料庫操作
- ✅ 併發控制與交易管理
- ✅ Spring Security 與 JWT
- ✅ 效能優化與最佳實踐
- ✅ 程式碼品質與設計原則
- ✅ 部署維運與監控
- ✅ 架構設計與重構決策

記住,面試不只是背答案,更重要的是:

1. **理解為什麼**這樣設計,trade-offs 是什麼
2. **能舉實例**說明概念,最好是從你的專案經驗
3. **能提出改進**方案,展現你的思考深度
4. **保持謙虛**,遇到不會的誠實說不知道,但展現學習意願

祝你面試順利! 🚀

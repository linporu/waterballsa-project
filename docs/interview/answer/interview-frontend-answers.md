# 前端面試題解答

## 一、React Hooks 與語法細節 (20 題)

### Q1. useEffect 依賴陣列

**解答**:

這個 useEffect 的依賴陣列是空的,因為它只需要在元件 mount 的時候執行一次,用來初始化認證狀態。裡面使用的 `getToken()` 和 `getUserInfo()` 都是從外部模組 import 進來的純函式,它們的參照是穩定的,不會隨著元件 re-render 而改變。

根據 React 的規則,從外部 import 的函式、refs、以及 setState 這類參照穩定的函式,都不需要加入依賴陣列。如果把它們加進去也不會有問題,但是沒有必要,因為它們永遠不會改變。

這裡的重點是:依賴陣列要放的是「會改變的值」,而這些 imported functions 的參照是固定的,所以空陣列就夠了。

---

### Q2. useCallback 的使用時機

**解答**:

1. 這兩個函式需要用 `useCallback` 包裝,主要是因為它們會被放進 Context 的 value 裡面,傳給很多子元件使用。如果不用 `useCallback`,每次 AuthProvider re-render 時,這些函式都會重新建立,導致所有使用 `useAuth()` 的子元件都會跟著 re-render,造成效能問題。

2. 依賴陣列是空的這樣很安全。因為函式內部使用的 `setUser` 是 React 的 setState,它的參照永遠穩定;其他像 `saveToken`、`removeToken`、`authApi.logout()` 這些都是從外部 import 的函式或物件方法,參照也是穩定的。整個函式沒有用到任何會變動的 props 或 state,所以空依賴陣列完全沒問題。

簡單來說,`useCallback` 確保函式參照穩定,空依賴陣列確保函式內容也不會變,兩者搭配起來就能避免不必要的 re-render。

---

### Q3. useRef 保存可變值

**解答**:

1. 這裡同時使用 `isCompleted` state 和 `completedRef` ref,是因為它們有不同的用途。`isCompleted` 是用來觸發 UI 更新的,讓畫面知道影片已經看完了;而 `completedRef` 是用來在 callback 裡面「馬上」檢查是否已經完成,防止重複呼叫。

2. `completedRef` 解決的問題是:YouTube 的 `onEnd` 事件有時候會被觸發多次。如果只用 `isCompleted` state,第一次呼叫時設定 `setIsCompleted(true)`,但 state 更新是非同步的,第二次呼叫進來時可能還看到 `isCompleted` 是 `false`,結果就會重複執行 `onComplete()`,可能造成重複打 API 或重複計算經驗值。

3. Ref 的特性是「同步更新」而且「不觸發 re-render」。當我們設定 `completedRef.current = true` 時,下一行程式碼馬上就能看到新的值,這樣就能即時攔截重複呼叫。State 做不到這點,因為它是 batched update,要等到下一次 render 才會生效。

---

### Q4. 清理 Effect

**解答**:

1. 在 cleanup function 中移除事件監聽器,是為了防止 memory leak。因為 `window` 是全域物件,會一直存在,如果我們在元件 mount 時加了監聽器,但 unmount 時沒有移除,這個監聽器就會一直留在記憶體裡面,而且還會參照到已經被卸載的元件,造成記憶體洩漏。

2. `stopProgressTracking()` 的作用是清除定時器。如果元件卸載了但 interval 還在跑,它會繼續呼叫 `saveCurrentProgress`,而這個函式可能會嘗試更新已經不存在的元件狀態,React 會跳出警告「Can't perform a React state update on an unmounted component」。

3. 忘記寫 cleanup 的後果很嚴重:每次使用者進出這個頁面,就會累積一個新的事件監聽器和一個新的 interval,越用越慢,最後可能導致瀏覽器當機。而且這種問題在開發環境不容易發現,因為影響是累積性的,要用久了才會明顯。

---

### Q5. Custom Hook 的設計

**解答**:

1. 把這四個 handlers 包在一個物件裡回傳,主要有幾個好處。首先是「語意化」,一看就知道這四個函式是一組相關的播放器控制功能。其次是「方便使用」,在使用的地方可以直接寫 `{...playerHandlers}` 把所有 handlers 一次 spread 給 YouTube 元件,不用一個一個傳。還有就是「避免命名衝突」,如果使用者想要重新命名,可以解構成 `{ playerHandlers: myHandlers }`,不會污染命名空間。

2. 這樣設計的優點是讓 API 更清晰、更好維護。如果未來要加入新的 handler,比如 `onError` 或 `onStateChange`,只要加到 `playerHandlers` 物件裡就好,不會改變 hook 的回傳結構。對於使用者來說,這個 hook 的介面很穩定,即使內部實作改變也不會影響到他們的程式碼。而且從使用者的角度看,「播放器的控制方法」是一個概念,把它們群組在一起比散落成四個獨立的函式更符合直覺。

---

### Q6. 複雜的 useEffect 依賴

**解答**:

1. 外面檢查 `if (user && !authLoading)` 是在「決定要不要執行」這個 effect,裡面的 `if (!user)` 是在 async 函式內部的「額外保護」。因為 async 函式是非同步的,在等待期間 `user` 可能變成 null(比如使用者突然登出),所以需要在函式內部再檢查一次,確保執行過程中狀態還是有效的。

2. `updateMissionStatus` 和 `hasPurchased` 雖然是函式,但它們來自 Context 或 custom hook,參照可能會改變。如果不加入依賴陣列,當這些函式更新時,effect 裡面用的還是舊版本的函式,可能會造成邏輯錯誤。React 要求所有在 effect 中使用的「響應式值」都要加入依賴。

3. 如果遺漏 `journeySlug` 或 `missionId`,當使用者切換到不同課程或任務時,effect 不會重新執行,畫面就會顯示錯誤的資料。這是很嚴重的 bug,使用者會看到上一個任務的內容,而不是當前選擇的任務。

---

### Q7. Context Provider 的記憶化

**解答**:

1. 是的,這個 `value` 物件每次 render 都會重新建立新的參照,理論上會導致所有使用 `useAuth()` 的子元件都重新渲染。但實際上影響可能沒那麼嚴重,因為只有當 `user`、`isLoading` 改變,或者 Provider 本身 re-render 時,這個問題才會發生。

2. 如果要優化,可以用 `useMemo` 包裝這個 value:

```tsx
const value = useMemo(
	() => ({
		user,
		isAuthenticated: !!user,
		isLoading,
		login,
		logout,
		updateUser
	}),
	[user, isLoading, login, logout, updateUser]
)
```

但其實這裡已經有 `useCallback` 保護 `login`、`logout`、`updateUser` 了,所以只有 `user` 和 `isLoading` 變化時才會重建,影響已經降到最低。

3. 什麼時候需要 `useMemo`?當 Context value 的重建成本很高,或者子元件非常多而且對 re-render 很敏感時。但如果 Context 本身就是隨著 state 變化而更新的(像這裡的 user),那 useMemo 的意義就不大,因為 state 變化時本來就該通知所有訂閱者。

---

### Q8. State 更新的不可變性

**解答**:

1. 用這麼多層的 `map` 和 spread operator,是因為 React 要求 state 更新必須保持「不可變性」(immutability)。資料結構是 `journey → chapters[] → missions[]`,要更新最裡面的 mission,必須從外層到內層逐層建立新物件,這樣 React 才能偵測到變化並觸發 re-render。

2. 如果直接寫 `prev.chapters[0].missions[0].status = 'COMPLETED'`,這叫做「mutation」,直接修改原本的物件。React 的狀態比較機制是用「參照相等性」(reference equality),如果物件參照沒變,React 會認為沒有更新,就不會 re-render。而且這種寫法還會破壞 React 的時間旅行除錯、undo/redo 等功能,因為舊狀態已經被污染了。

3. 這種深層更新的效能確實不太好,每次都要複製整個 journey 物件和所有 chapters、missions 陣列。更好的做法是用 Immer library,或者考慮用 normalized state(把巢狀資料攤平成 map),或者用 Zustand、Redux Toolkit 這類支援 immutable update 的狀態管理工具。如果資料層級很深,可以考慮把狀態切割得更細,不要全部放在一個大物件裡。

---

### Q9. useCallback 依賴的函式

**解答**:

1. 這種依賴鏈本身不會有問題,只要每個 useCallback 的依賴都正確標註。但如果 `onProgressUpdate` 每次 render 都改變(比如它沒有用 useCallback 包裝),就會導致 `saveCurrentProgress` 也跟著改變,然後 `startProgressTracking` 也改變,形成連鎖反應。這會導致 interval 被頻繁清除重建,影響效能。

2. 如果 `onProgressUpdate` 頻繁改變,最好的做法是在外層把它用 useCallback 穩定化。或者,可以改用 useRef 來存 `onProgressUpdate`,在需要呼叫時從 ref 讀取最新版本,這樣依賴鏈就不會有問題了。

3. `playerRef.current` 不需要加入依賴陣列,因為 ref 本身的參照是穩定的,雖然 `playerRef.current` 的內容可能改變,但改變 ref 的 current 不會觸發 re-render,所以不算是「響應式值」。我們在 callback 裡面讀取的永遠是最新的 current 值,不需要把 ref 加入依賴。

---

### Q10. Custom Hook 的組合

**解答**:

1. 這個 hook 組合了多個其他 hooks,好處是「封裝複雜邏輯」,使用者只要呼叫 `useMission()` 就能拿到所有需要的資料和方法,不用自己組合這些 hooks。缺點是「高度耦合」,如果 `useAuth`、`useJourney`、`useUserPurchase` 的介面改變,這個 hook 也要跟著改。而且測試起來比較困難,需要 mock 好幾個 Context。

2. 如果底層 hooks 的實作改變,只要介面(回傳的屬性和方法)不變,這個 hook 就不受影響。但如果介面改了,比如 `useAuth` 把 `isLoading` 改名成 `loading`,就必須修改這個 hook。所以依賴多個 Context 會增加維護成本,但也是無法避免的,因為這些資料本來就分散在不同的 Context 裡。解決方式可以是 adaptor pattern 或是依賴注入。

3. 把三個 loading 狀態合併成一個,是為了簡化使用者的邏輯。使用這個 hook 的元件只需要看一個 `isLoading` 就知道「是否還在載入中」,不用自己判斷三個狀態。這是一種「抽象化」,隱藏複雜度,讓 API 更簡潔。而且用 OR 邏輯(`||`)很合理,只要有任何一個還在 loading,整體就算 loading。

---

### Q11. useEffect 的執行時機

**解答**:

1. 兩個 effects 的執行順序是按照它們在程式碼中的順序依次執行,而且是在同一個 render 週期中。第一個 effect 處理未登入的重定向,第二個 effect 處理資料載入。它們會在元件 render 完成後,按順序執行。

2. 分成兩個 useEffect 的原因是「關注點分離」(separation of concerns)。第一個專注於「認證狀態檢查」,第二個專注於「資料載入」。這樣每個 effect 的職責很明確,依賴陣列也更容易管理。如果合併成一個,邏輯會變得複雜,而且可能會因為不相關的依賴變化而重複執行。

3. 兩次檢查 user 是必要的,因為第一次檢查(`if (user && !authLoading)`)決定要不要「啟動」資料載入,第二次檢查(`if (!user)`)是在 async 函式「執行過程中」的保護。比如說,假設資料正在載入時使用者突然登出了,`user` 會變成 null,這時候 async 函式還在跑,如果沒有第二次檢查,就可能會用 null 去呼叫 API,造成錯誤。這是 async 操作的防禦性程式設計。

---

### Q12. 條件式資料載入

**解答**:

1. 設計 `showLoading` 參數來區分兩種情境:「初次載入」和「背景更新」。初次載入時(showLoading=true)會顯示 loading 畫面,讓使用者知道正在載入資料;背景更新時(showLoading=false)只設定 `isRefreshing`,不影響現有畫面,使用者可以繼續操作,不會看到突然出現的 loading 狀態。這樣的 UX 比較好,因為背景更新不應該打斷使用者。

2. `finally` block 中同時設定兩個狀態為 false 不會有問題,因為同時只會有一個是 true。如果 `showLoading` 是 true,那 `isLoading` 會被設為 true,`isRefreshing` 保持 false;反之亦然。在 finally 中全部設為 false,可以確保不管哪種情況,結束時都能正確清除 loading 狀態。

3. 註解說「Keep stale data」是因為如果 fetch 失敗,還是保留舊資料比較好。清空 state 會讓畫面突然變空白,使用者體驗很差。保留舊資料至少讓使用者還能看到之前的內容,只是可能不是最新的。這是一種「graceful degradation」,優雅地處理錯誤情況。

---

### Q13. useCallback 與閉包

**解答**:

1. 這個 useCallback 還是有用的!雖然 `purchasedJourneyIds` 是 Set,每次更新都會建立新 instance,但 `hasPurchased` 函式本身的邏輯很簡單,重點是「穩定函式參照」。這個函式會被放進 Context value,傳給很多子元件。用 useCallback 可以確保只有當 Set 改變時才重新建立函式,避免子元件不必要的 re-render。

2. 直接在使用的地方寫 `purchasedJourneyIds.has(journeyId)` 也可以,但有幾個缺點:一是每個使用的地方都要知道內部實作是用 Set,二是如果未來改用其他資料結構(比如 array 或 map),就要改很多地方。把它封裝成函式,提供了一層抽象,隱藏實作細節。

3. 把這個函式放在 Context value 中,好處是「集中邏輯」和「方便使用」。所有需要檢查購買狀態的元件都可以直接呼叫 `hasPurchased(journeyId)`,而不用關心資料是怎麼儲存的。如果未來要加入快取、或者改變查詢邏輯,只要改這一個地方就好,不會影響到使用者。

---

### Q14. Promise.all 的使用

**解答**:

1. 用 `Promise.all` 是為了「平行執行」所有 API 呼叫,大幅提升效能。如果用 `for` loop 依序呼叫,假設每個 API 要 100ms,10 個任務就要 1 秒;用 Promise.all 平行執行,所有請求幾乎同時發出,總時間還是大約 100ms。這在有多個任務時差異非常明顯。

2. 如果其中一個 API call 失敗,Promise.all 會馬上 reject,後面的結果都拿不到。但這段程式碼沒有直接 throw error,而是檢查 `progressResult.success`,所以即使某個 API 失敗,還是會拿到包含 error 的 response,不會中斷整個流程。這是因為用的 API client 回傳 `ApiResponse<T>` 型別,不會 throw,而是用 `success: boolean` 來表示成功或失敗。

3. 如果課程有 100+ 單元,這種做法會有嚴重問題:一次發送 100 個並行請求,可能會被伺服器限流,或是瀏覽器同時連線數上限,導致部分請求失敗。更好的做法是「分批」(chunking),比如每次只發 10 個請求,或者用 throttling 控制並行數。也可以考慮後端提供一個「批次查詢進度」的 API,一次呼叫就能拿到所有進度,而不是逐個查詢。

---

### Q15. Custom Hook 的回傳型別

**解答**:

1. `callApi` 函式中的泛型 `<T>` 代表「這個 API 呼叫會回傳什麼型別的資料」。使用時,TypeScript 可以自動推導,比如 `callApi(() => authApi.login(...))`,TypeScript 知道 login 回傳 `ApiResponse<LoginResponse>`,所以 `T` 就是 `LoginResponse`。這樣呼叫方就能得到正確的型別提示,知道 `response.data` 的結構。

2. 設計成接受一個 `apiCall` 函式作為參數,而不是直接接受 URL 和 data,是為了「保持彈性」。這樣可以包裝任何 API 呼叫,不用關心它的參數是什麼。而且這種設計讓呼叫方保留對 API 的完整控制,你可以傳入任何符合 `() => Promise<ApiResponse<T>>` 簽名的函式,包括有額外邏輯的函式。

3. 自動處理 401 的機制很簡單:在 API 呼叫完成後,檢查 `response.error.status === 401`,如果是就呼叫 `logout()` 清除本地認證狀態,然後用 `router.push('/login')` 重定向到登入頁。這樣所有用這個 hook 的地方都不用自己寫 401 處理邏輯,統一在這裡處理,符合 DRY (Don't Repeat Yourself) 原則。

---

### Q16. React Hook Form 整合

**解答**:

1. `zodResolver(loginSchema)` 的作用是把 Zod schema 轉換成 React Hook Form 能理解的驗證器。React Hook Form 本身不知道怎麼執行 Zod 的驗證規則,需要這個 resolver 當橋樑。當表單送出或欄位失去焦點時,resolver 會呼叫 Zod 驗證,把驗證錯誤轉成 React Hook Form 的格式,這樣表單才能顯示錯誤訊息。

2. 對 `username` 設定空字串 message 是一個小技巧,用來「標記這個欄位有錯誤」但不顯示訊息。這樣可以觸發 UI 的錯誤樣式(比如紅色邊框),但錯誤訊息只顯示在 `password` 欄位下方。因為「使用者名稱或密碼錯誤」這個訊息不需要顯示兩次,只要在其中一個欄位顯示就夠了,但兩個欄位都要有錯誤狀態的視覺回饋。

3. `form.setError` 是「手動設定錯誤」,用於處理 server-side 驗證。Zod schema validation 是「client-side 驗證」,在表單送出前就檢查格式是否正確。兩者的差異是:Zod 檢查「格式」(長度、必填等),setError 處理「商業邏輯錯誤」(帳號密碼不符、帳號已存在等),這類錯誤只有打 API 才知道。

---

### Q17. FormField 與 Controller

**解答**:

1. `render` prop 中的 `field` 參數包含:

   - `value`: 目前的欄位值
   - `onChange`: 更新欄位的函式
   - `onBlur`: 處理失去焦點的函式
   - `name`: 欄位名稱
   - `ref`: 用於註冊欄位的 ref
     這些都是 React Hook Form 的 Controller 提供的,用來連接 uncontrolled component 和表單狀態。

2. `{...field}` spread 到 Input 元件後,會自動把這些屬性傳給 input,所以 Input 就知道:目前的值是什麼(`value`),當使用者輸入時要呼叫什麼(`onChange`),失去焦點時要做什麼(`onBlur`)。React Hook Form 會自動處理表單狀態的同步和驗證。

3. 不需要手動寫 `onChange` 和 `value`,因為 Controller 已經幫我們處理好了。`field` 物件包含的 `onChange` 會自動更新表單狀態,`value` 會自動從表單狀態讀取。這是 React Hook Form 的「controlled component」模式,讓我們可以用宣告式的方式管理表單,不用自己追蹤每個欄位的狀態。

---

### Q18. useFormField 自訂 Hook

**解答**:

1. 用兩個不同的 Context 是因為它們的「生命週期」不同。`FormFieldContext` 是由 `FormField` (Controller) 提供的,包含欄位名稱等資訊;`FormItemContext` 是由 `FormItem` (layout container) 提供的,包含唯一 ID。這種分離讓元件的結構更靈活,`FormItem` 可以包含多個部分(label, control, message),它們都能共用同一個 ID 但各自有不同的用途。

2. `getFieldState` 是「立即讀取」某個欄位的狀態(錯誤、是否被修改等),而 `useFormState` 是一個 hook,會「訂閱」表單狀態的變化,當狀態改變時會觸發 re-render。這裡用 `useFormState({ name: fieldContext.name })` 來訂閱特定欄位的狀態變化,確保當這個欄位有錯誤或被修改時,元件會重新渲染顯示最新狀態。

3. 產生這些 ID 是為了建立「無障礙的表單結構」(accessibility)。`formItemId` 給 input 元素,`formDescriptionId` 給輔助說明文字,`formMessageId` 給錯誤訊息。透過 `aria-describedby` 屬性把這些 ID 連結起來,螢幕閱讀器就能正確地把 label、description、error message 和 input 關聯在一起,讓視障使用者知道這個欄位的完整資訊。

---

### Q19. React 19 的 useFormState

**解答**:

1. React Hook Form 的 `useFormState` 和 React 19 內建的 `useFormState` 是「完全不同的東西」。React Hook Form 的版本是用來訂閱表單狀態變化(errors, isDirty, isValid 等),而 React 19 的 `useFormState` 是配合 Server Actions 使用的,用於處理表單送出後的伺服器回應。

2. 在這個專案中使用的是 React Hook Form 的 `useFormState`,從 import 可以看出:`import { useFormState } from 'react-hook-form'`。這是專門為 client-side 表單狀態管理設計的。

3. React 19 的新 Form API(包括 `useFormState`、`useFormStatus`、Server Actions)主要影響「server-side」的表單處理。但這個專案還是用傳統的 client-side 驗證 + API 呼叫模式,所以 React 19 的 Form API 對這個專案暫時沒有直接影響。未來如果要用 Server Actions,可以考慮重構成 React 19 的新模式,但現有的 React Hook Form 實作也完全沒問題。

---

### Q20. Slot Pattern

**解答**:

1. Radix UI 的 `Slot` 元件是一個「透明容器」,它會把自己的 props 合併到「子元件」上,而不是建立新的 DOM 節點。簡單說,`<Slot id="foo"><input /></Slot>` 最後只會渲染成 `<input id="foo" />`,不會有額外的 wrapper div。這讓我們可以在不增加 DOM 層級的情況下,為子元件加上額外的屬性。

2. 用 `Slot` 而不是 `div` 的原因是「避免不必要的 DOM 嵌套」。如果用 div 包裝,最後會變成 `<div><input /></div>`,多了一層 DOM。而且有些 CSS 選擇器或樣式可能會因為多一層而失效。Slot 讓我們保持 DOM 結構乾淨,只在真正需要的地方加上 props,特別適合建立 UI library,使用者可以自由替換底層元件而不破壞結構。

3. `aria-describedby` 告訴螢幕閱讀器「這個欄位的說明文字和錯誤訊息在哪裡」,`aria-invalid` 表示「這個欄位目前有錯誤」。當視障使用者 focus 到這個 input 時,螢幕閱讀器會自動唸出相關的說明和錯誤訊息,讓他們知道這個欄位的狀態。這些屬性是 Web 無障礙標準(WCAG)的一部分,確保所有人都能順利使用表單,不只是視力正常的使用者。

---

## 二、狀態管理與資料流 (10 題)

### Q21. Context 的分層設計

**解答**:

1. `AuthProvider` 要在最外層,是因為它提供「全站通用」的認證資訊,幾乎所有頁面都需要知道使用者是否登入。`JourneyProvider` 在內層(app layout),是因為它只有在「應用程式內部」才需要,登入頁、註冊頁這些外層路由不需要課程資料,所以沒必要在最外層提供。這種分層設計遵循「按需提供」的原則,避免不必要的 Provider 包裝。

2. 如果把 `UserPurchaseProvider` 和 `AuthProvider` 對調,會出大問題。因為 `UserPurchaseProvider` 內部使用了 `useAuth()` 來取得使用者資訊,如果它在 `AuthProvider` 外層,就會拋出錯誤「useAuth must be used within an AuthProvider」。Context 的依賴關係必須嚴格遵守:被依賴的 Provider 要在外層,依賴它的 Provider 要在內層。

3. Provider 嵌套對效能的影響主要是「re-render 的範圍」。當 `AuthProvider` 的 state 改變時,所有子元件都可能 re-render,包括 `UserPurchaseProvider` 和 `SWRProvider`。但這是無法避免的,因為認證狀態改變時,購買資訊確實需要重新載入。關鍵是要確保每個 Provider 的 value 正確使用 `useMemo` 和 `useCallback`,避免不必要的參照變化,這樣就能把 re-render 控制在最小範圍。

---

### Q22. Context 間的依賴關係

**解答**:

1. `UserPurchaseProvider` 依賴 `useAuth()` 這種設計在這個情境下是合理的,因為購買資訊本來就跟使用者綁定,使用者登出時就應該清空購買資料。但確實要注意依賴關係不要太複雜,如果 Context A 依賴 B,B 依賴 C,C 又依賴 A,就會形成循環依賴,造成問題。目前的設計是單向依賴(UserPurchase → Auth),還算清晰可控。

2. 如果 `AuthContext` 的 `user` 物件經常改變,會導致 `UserPurchaseProvider` 頻繁觸發 useEffect,重複呼叫 `fetchUserPurchaseData`。但實際上 `user` 物件只有在登入、登出、更新個人資料時才會變,不是每次 render 都變,所以影響不大。而且程式碼裡已經做了防護:`if (isAuthenticated && user)`,只有在認證狀態改變時才會重新載入。

3. 更好的方式是「事件驅動」而不是「狀態驅動」。可以在 AuthContext 提供 `onLogin`、`onLogout` 這類事件回調,讓 UserPurchaseProvider 訂閱這些事件,而不是監聽 state 變化。或者用第三方狀態管理工具(Zustand、Jotai)來解耦 Context 之間的依賴。但對於這個規模的專案,目前的做法已經夠好了,不用過度工程化。

---

### Q23. Set 作為 State

**解答**:

1. 用 `Set` 而不是 `Array` 來儲存已購買的課程 ID,主要是為了「查詢效能」。`Set.has(id)` 是 O(1) 時間複雜度,而 `Array.includes(id)` 是 O(n)。雖然課程數量可能不會太多,但在 UI 渲染時可能會頻繁呼叫 `hasPurchased(journeyId)`,用 Set 可以確保每次檢查都是瞬間完成,不會隨著課程數量增加而變慢。

2. 每次都建立新的 Set instance 是為了遵守 React 的「不可變性」原則。如果直接修改現有的 Set(例如 `purchasedJourneyIds.add(newId)`),React 不會偵測到變化,因為 Set 的參照沒變。必須建立新的 Set,React 才能比對參照發現狀態更新,觸發 re-render。這跟更新 object 或 array 的原則一樣,都要建立新 instance。

3. 使用 `Set` 讓 `hasPurchased` 函式的效能非常好。無論有 10 個課程還是 1000 個課程,檢查是否購買都是固定時間。如果用 Array,隨著課程增加,每次 `includes` 都要遍歷整個陣列,越來越慢。這在列表頁面特別重要,可能要同時顯示幾十個課程,每個都要檢查購買狀態,Set 的效能優勢就很明顯了。

---

### Q24. SWR 全域配置

**解答**:

1. `revalidateOnFocus: false` 的作用是關閉「視窗重新獲得焦點時自動重新驗證資料」的功能。預設情況下,當你切換瀏覽器 tab 再切回來,SWR 會重新 fetch 資料確保是最新的。但這個專案的資料更新頻率不高(課程內容、購買資訊),不需要這麼積極地刷新,而且頻繁重新載入會造成不必要的 API 呼叫和流量消耗。所以關閉這個功能,只在真正需要時手動刷新。

2. `dedupingInterval: 2000` 的意思是「2 秒內對同一個 key 的重複請求會被去重」。如果在 2 秒內多個元件同時呼叫 `useSWR('/api/user')`,SWR 只會發送一次實際的網路請求,然後把結果分享給所有元件。這可以避免同一個資料被重複 fetch,特別是在頁面初始載入時,多個元件同時 mount 並請求相同資料的情況。

3. `keepPreviousData: true` 對使用者體驗幫助很大。當資料重新驗證時(revalidate),不會清空舊資料,而是繼續顯示,直到新資料載入完成。這樣使用者不會看到突然的空白畫面或 loading 狀態,而是看到舊資料平滑地更新成新資料。特別是在分頁切換或篩選條件改變時,保留舊資料可以讓畫面更穩定,不會閃爍。

---

### Q25. 跨 Tab 通訊

**解答**:

1. 先 `setItem` 再馬上 `removeItem` 是一個巧妙的技巧,用來「觸發 storage 事件」而不是「真正儲存資料」。因為只有當 localStorage 的值「改變」時,其他 tab 才會收到 `storage` 事件。如果每次都設定相同的值,第二次之後就不會觸發事件了。所以用「設定然後刪除」的方式,每次都能產生一個「新增 → 刪除」的變化,確保其他 tab 都能收到通知。

2. `storage` 事件的機制是:當一個 tab 修改 localStorage 時,瀏覽器會自動通知「其他所有同源的 tab」,讓它們的 `window` 物件觸發 `storage` 事件。這是瀏覽器原生提供的跨 tab 通訊機制,不需要額外的 library 或 polling。在這個專案中,當使用者在 tab A 完成購買後,tab B 和 tab C 會立刻收到通知,自動刷新購買資訊,保持多個 tab 的狀態同步。

3. 在發送通知的 tab(也就是呼叫 `setItem` 的那個 tab)不會觸發自己的 `storage` 事件,這是 `storage` 事件的設計特性。這樣很合理,因為發送者已經知道發生了什麼事(它自己剛完成購買),不需要再收到自己的通知。只有「其他」tab 需要被告知「有東西改變了」。所以在發送者 tab,我們直接呼叫 `fetchUserPurchaseData(true)` 刷新資料;在其他 tab,它們監聽到 `storage` 事件後呼叫 `refreshPurchases()` 更新。

---

### Q26. 樂觀更新 vs 重新載入

**解答**:

1. 這裡不是樂觀更新,而是「先呼叫 API 再更新 UI」的保守做法。樂觀更新是「先更新 UI 讓使用者看到結果,同時在背景呼叫 API,如果失敗再回滾」。但這段程式碼是等 API 回應成功後(`if (result.success)`),才更新 state 和 sidebar。這樣比較安全,但使用者體驗稍微差一點,因為要等 API 完成才看到變化。

2. 如果改成樂觀更新,程式碼要這樣改:

```tsx
const handleDeliverMission = async () => {
  // 1. 先樂觀更新 UI
  setProgress(prev => prev ? { ...prev, status: 'DELIVERED' } : null)
  updateMissionStatus(missionId, 'DELIVERED')

  setIsDelivering(true)
  const result = await missionApi.deliverMission(...)

  if (result.success) {
    toast.success('任務完成！')
  } else {
    // 2. 失敗時回滾
    setProgress(prev => prev ? { ...prev, status: 'COMPLETED' } : null)
    updateMissionStatus(missionId, 'COMPLETED')
    toast.error('完成任務失敗')
  }
  setIsDelivering(false)
}
```

3. 什麼情況適合樂觀更新?「高成功率、低風險、即時性要求高」的操作,比如按讚、收藏、簡單的狀態切換。什麼情況不適合?「可能失敗、有副作用、需要伺服器計算結果」的操作,比如付款、刪除資料、這裡的完成任務(因為要計算經驗值和等級),這些情況最好等 API 確認後再更新,避免給使用者錯誤的資訊。

---

### Q27. Loading State 管理

**解答**:

1. 把三個不同來源的 loading 狀態合併成一個,是為了「簡化使用邏輯」。使用這個 hook 的元件只需要看一個 `isLoading`,就知道「是不是還在等資料」,不用自己判斷「到底是等認證、等課程資料、還是等購買資訊」。這是一種抽象化,把複雜的內部狀態隱藏起來,提供簡潔的 API。而且用 OR 運算(`||`)很合理:只要有任何一個還在 loading,整體就該顯示 loading 狀態。

2. `isDelivering` 分開管理是因為它是「不同性質」的 loading。前面三個 loading 是「載入資料」,而 `isDelivering` 是「執行操作」。載入時整個頁面可能要顯示骨架屏或轉圈圈,但完成任務時只需要讓按鈕變成 loading 狀態,不應該影響整個頁面。分開管理讓 UI 可以做更精細的控制,提供更好的使用者體驗。

3. 避免 loading 狀態的競爭條件,關鍵是「狀態的順序和時機」。比如在 finally block 統一清除 loading,確保不管成功失敗都會清除。還有要避免「快速切換」造成的問題,比如使用者快速切換任務,第一個任務的資料還沒載入完,第二個任務的請求就發出了,可能導致顯示錯誤的資料。解決方法是用「取消機制」(AbortController)或「版本號」,確保只顯示最新請求的結果。

---

### Q28. 防止重複呼叫

**解答**:

1. 影片結束時可能被重複呼叫,是因為 YouTube Player 的 `onEnd` 事件有時候會觸發多次,這可能是 YouTube API 的 bug 或瀏覽器的事件冒泡問題。而且如果使用者快速點擊重播或操作進度條,也可能觸發多次完成事件。如果沒有防護,就會重複呼叫 `onComplete()`,可能造成重複打 API、重複顯示 toast、或者重複計算經驗值。

2. 除了用 `useRef`,還可以用其他方法:
   - 用 debounce/throttle 限制函式呼叫頻率
   - 在 API 層面做冪等性檢查,伺服器拒絕重複的完成請求
   - 檢查 state:`if (isCompleted) return` 但這有時機問題,因為 state 更新是非同步的
   - 用全域的 flag 或 Set 來追蹤已完成的任務 ID

但 `useRef` 是最直接有效的方法,因為它是同步更新的,設定後馬上就能讀到新值。

3. 如果使用者倒轉影片再看一次,`completedRef` **不需要**重置。因為這個 hook 是用來追蹤「本次頁面載入後是否完成過」,而不是「影片播放次數」。一旦使用者看完一次並觸發了 `onComplete()`,就不應該再觸發了,即使他倒轉重看。如果需要支援「重複觀看重複計分」的需求,那就要設計成可重置的,但在這個學習平台的情境下,看完一次就夠了。

---

### Q29. 條件式狀態初始化

**解答**:

1. 在呼叫 `getMissionDetail` API 之前就檢查購買狀態,是為了「避免不必要的 API 呼叫」和「保護付費內容」。如果使用者沒有購買,就不應該拿到任務的詳細內容(可能包含影片 ID、講義等付費資源)。先檢查 mission summary 的 `accessLevel`,如果是 PURCHASED 但使用者沒購買,就直接 early return,不去呼叫詳細 API。這樣既節省伺服器資源,也確保付費內容不會洩漏給未付費使用者。

2. Early return 這種模式的好處是「減少巢狀」和「提早結束」。如果不用 early return,就要寫 `if (hasAccess) { ... 一大段邏輯 }`,程式碼會越來越往右縮排,可讀性很差。用 early return 可以在不符合條件時馬上結束,後面的程式碼不用再檢查這些條件,邏輯更清晰。而且符合「Guard Clause」模式,把異常情況和前置條件檢查放在最前面處理掉。

3. 設定假的 `progress` 物件而不是 `null`,是為了「統一介面」和「避免 null check」。如果設為 null,使用這個資料的地方都要檢查 `if (progress)`,很麻煩而且容易出錯。設定一個預設的 progress 物件(status 為 UNCOMPLETED,進度為 0),UI 就可以直接使用,不用做額外判斷。而且這樣在邏輯上也合理:沒有購買的任務,進度就是「未完成,0% 進度」。

---

### Q30. 狀態同步策略

**解答**:

1. 把 progress 狀態同步到 `JourneyContext`,是因為「同一個資料被用在不同地方」。任務詳細頁面需要知道進度來顯示影片位置和完成按鈕,而側邊欄(sidebar)也需要知道哪些任務已完成,要顯示打勾圖示。如果不同步,側邊欄會顯示舊的狀態,使用者會看到「任務明明完成了,側邊欄還是未完成」,造成困惑。同步到 JourneyContext 讓整個應用的任務狀態保持一致。

2. 如果同一個資料在多個地方存在,確保同步的方法有幾種:
   - **Single Source of Truth**: 只在一個地方存資料,其他地方都從那裡讀取(最理想)
   - **事件驅動更新**: 資料改變時發送事件,所有訂閱者收到通知後更新
   - **定時同步**: 定期輪詢或重新載入確保一致性
   - **樂觀更新**: 本地先更新所有副本,API 成功後確認,失敗則全部回滾

關鍵是要有明確的「資料流向」,避免雙向同步造成循環依賴。

3. 這種做法的潛在問題是「狀態重複」和「同步複雜度」。progress 既存在 `useMission` 的 local state,又存在 `JourneyContext` 的 journey.chapters.missions[].status,如果某一處更新失敗或遺漏,就會不一致。更好的方案是用專門的狀態管理工具(Redux、Zustand),把所有任務進度統一管理;或者用 SWR/React Query 做資料快取,自動同步和重新驗證。但對於這個專案規模,目前的做法已經可以接受,不用過度設計。

---

## 三、與後端互動 (10 題)

### Q31. API Client 架構

**解答**:

1. 設計成 class 而不是單純的函式,主要是為了「狀態管理」和「可擴展性」。Class 可以有 private 的 config 屬性,儲存 baseURL、headers、timeout 等設定,讓多個方法共用這些配置。而且 class 的結構更清晰,可以把相關的邏輯(buildURL、handleResponse、retry 等)組織在一起,私有方法不會暴露給外部。如果只用函式,就要把 config 當參數傳來傳去,或者用閉包,都沒有 class 這麼直觀。

2. `Partial<ApiClientConfig>` 的好處是「允許部分配置」。使用者可以只傳入想要覆蓋的設定,比如只改 `baseURL`,其他用預設值。在 constructor 裡用 `{ ...defaultConfig, ...config }` 合併,沒傳的屬性就用預設值,傳了的就覆蓋。這樣很彈性,不用每次都傳完整的 config 物件,也不用擔心遺漏必要的屬性。TypeScript 的 `Partial<T>` 把所有屬性都變成 optional,非常適合這種「部分覆蓋預設值」的場景。

3. Export 一個 default instance(`apiClient`)是為了「全域共用」和「方便使用」。大部分情況下,整個應用只需要一個 API client,用同樣的 baseURL 和設定。如果每次都要 `new ApiClient()`,很麻煩而且會建立多個實例。提供一個預設實例,大家直接 import 就能用:`import { apiClient } from '@/lib/api'`。但同時也 export `createApiClient` 函式,給特殊情況使用,比如需要打不同的 API server,或者需要特殊的 timeout 設定。

---

### Q32. 統一的錯誤處理

**解答**:

1. 檢查 `content-type` 來決定如何解析 response,是因為伺服器回應的格式不一定都是 JSON。正常的 API 回應通常是 `application/json`,但有些錯誤(像 Nginx 的 502、504)可能回傳 HTML 或純文字。如果直接用 `response.json()` 解析 HTML,會拋出 JSON parse error,使用者看到的錯誤訊息就很奇怪。先檢查 content-type,如果是 JSON 就解析 JSON,否則讀成 text,這樣可以兼容各種回應格式,不會因為格式問題而 crash。

2. `ApiResponse` 的 `success` flag 設計有幾個好處:

   - **型別安全**: TypeScript 可以用 discriminated union,在 `if (result.success)` 後自動知道有 `data`,否則有 `error`
   - **統一處理**: 所有 API 呼叫都用同樣的方式檢查成功或失敗,不用記每個 API 的錯誤格式
   - **避免 null check**: 不用寫 `if (result.data && !result.error)` 這種複雜判斷
   - **明確語意**: `success: true/false` 一看就知道請求是否成功,比單純檢查 status code 或有無 error 更直觀

3. 不直接 throw error 而是回傳包含 error 的物件,是因為「錯誤也是一種正常的回應」。在實際應用中,API 失敗是很常見的(網路問題、驗證失敗、權限不足等),不應該當作 exception 處理。如果用 throw,每個呼叫的地方都要寫 try-catch,很麻煩。而且有些錯誤需要特殊處理(比如 401 要登出,404 要顯示特定訊息),用回傳值可以讓呼叫方更方便地檢查和處理不同的錯誤情況。

---

### Q33. Retry 機制

**解答**:

1. Exponential backoff(`baseDelay * Math.pow(2, attempt)`)是「指數退避」策略。第一次重試等 100ms(2^0),第二次等 200ms(2^1),第三次等 400ms(2^2)。這樣設計的原因是:如果伺服器正在重啟或暫時過載,馬上重試可能還是會失敗,甚至讓問題更嚴重(加重伺服器負擔)。逐漸增加等待時間,給伺服器恢復的機會,也避免大量 client 同時重試造成「驚群效應」。這是業界標準的做法,AWS、Google Cloud 等服務都建議用 exponential backoff。

2. 只有 idempotent methods(GET、PUT、DELETE)才會 retry,是因為「重複執行安全」。GET 是讀取,PUT 和 DELETE 按照 HTTP 規範應該是 idempotent(執行一次和多次結果相同)。但 POST 不是,它通常會「建立新資源」,如果重試可能會建立重複的資料(比如重複下單、重複發文)。所以 POST 不自動 retry,避免副作用。如果真的需要 retry POST,應該在應用層面做冪等性檢查,或者用唯一 ID 防止重複。

3. 如果三次都失敗,最後回傳的是「最後一次」的錯誤,也就是第三次(attempt = 2)的錯誤。程式碼中用 `lastError` 儲存每次失敗的結果,最後 return `lastError ?? ...`。這樣設計合理,因為最後一次的錯誤最有可能反映真實問題。如果前兩次是暫時性網路問題,但第三次是 400 Bad Request,那使用者應該看到 400 的錯誤訊息,而不是 timeout。

---

### Q34. Request Timeout

**解答**:

1. `AbortController` 實現 timeout 的原理是:建立一個 controller,把它的 `signal` 傳給 fetch,然後用 `setTimeout` 在指定時間後呼叫 `controller.abort()`。當 abort 被呼叫時,fetch 會立刻中斷並拋出 `AbortError`。這是標準的瀏覽器 API,不需要額外的 library,而且可以真正取消網路請求,不只是忽略回應。

2. 在 `try` 和 `catch` block 中都要 `clearTimeout`,是為了「確保 timer 被清除」。如果 fetch 正常完成(在 timeout 之前),要清除 timer 避免它在之後觸發 abort;如果 fetch 拋出錯誤(可能是 network error 或其他問題),也要清除 timer 避免 memory leak。不管成功或失敗,timer 都必須被清除,所以兩邊都要寫。其實也可以用 `finally` block,但這裡分開寫也可以,確保每個路徑都有處理到。

3. 如果 timeout 了,`catch` block 會接到 `AbortError`,它的 `error.name === 'AbortError'`。程式碼中有 `isTimeoutError` 方法檢查這個,然後在 `createErrorResponse` 中轉換成友善的錯誤訊息:`{ success: false, error: { message: 'Request timeout' } }`。這樣使用者看到的不是技術性的「AbortError」,而是清楚的「請求逾時」訊息。

---

### Q35. API Interceptors

**解答**:

1. Request interceptor 和 response interceptor 可以用來做很多事情:
   - **Request interceptor**: 自動加入 auth token(`headers.Authorization = 'Bearer ...'`)、加入 trace ID、修改 URL(加入 API version)、記錄請求日誌、加密敏感資料
   - **Response interceptor**: 解密回應、轉換資料格式、統一錯誤處理、記錄回應日誌、更新 token、觸發全域事件(如顯示錯誤提示)

這種設計讓我們可以在不修改每個 API 呼叫的情況下,統一加入橫切關注點(cross-cutting concerns)。

2. Interceptor 是 async 的,因為它們可能需要做非同步操作。比如 request interceptor 可能要先檢查 token 是否過期,過期就先 refresh token 再加到 headers;response interceptor 可能要呼叫另一個 API 來解密資料。如果只支援同步,就限制了 interceptor 的能力。Async 設計讓 interceptor 更強大,可以處理複雜的情境。

3. 如果 interceptor 中拋出錯誤,會被外層的 try-catch 捕捉,最後回傳 error response。但這可能不是理想的行為,因為 interceptor 的錯誤和 API 本身的錯誤性質不同。更好的做法是在 interceptor 中自己處理錯誤,或者記錄日誌後讓請求繼續。實際使用時要注意 interceptor 的錯誤處理,不要讓 interceptor 的 bug 影響到整個 API 呼叫。

---

### Q36. 自動 401 處理

**解答**:

1. 401 處理放在 custom hook 中而不是 API client,是因為它需要「React 特有的功能」:呼叫 `logout()` (來自 Context)和 `router.push()` (Next.js 的路由)。API client 是純粹的資料層,不應該依賴 React 或 Next.js,這樣才能在不同環境中重用(比如測試環境、Node.js server 等)。把 401 處理放在 hook 層,符合「關注點分離」:API client 負責網路請求,hook 負責應用層的邏輯(認證狀態、路由跳轉)。

2. 如果同時有多個 API 呼叫返回 401,確實會觸發多次 `logout()` 和 `router.push()`。但實際上影響不大,因為 `logout()` 是 idempotent 的(執行多次效果一樣),清除本地 token 和 state;`router.push('/login')` 也是,多次跳轉到同一個頁面不會有問題。不過更好的做法是加入「防抖」機制,比如用一個全域 flag 記錄是否已經在處理 401,避免重複執行。

3. 這種設計確實有限制:只有用 `useApi()` hook 包裝的 API 呼叫才會自動處理 401,直接呼叫 API 的地方就需要自己處理。所以專案中的最佳實踐是「所有需要認證的 API 呼叫都用 `callApi()` 包裝」。這是一種約定,需要團隊遵守。或者也可以改成在 API client 的 response interceptor 中檢查 401,但那樣就要想辦法在 API client 中存取 React Context,會變得更複雜。目前的設計在簡潔性和功能性之間取得了平衡。

---

### Q37. 進度追蹤的批次更新

**解答**:

1. 每 10 秒才更新一次進度,而不是即時更新,主要是為了「減少伺服器負擔」和「節省網路流量」。如果使用者每秒都送一次進度更新,一個 10 分鐘的影片就要發送 600 次請求,太浪費了。而且進度資料不需要到秒級的精準度,損失幾秒的進度在使用者體驗上影響不大。10 秒的間隔是權衡後的結果:既不會造成太大的資料損失,又能大幅降低請求頻率。

2. 如果使用者網路不穩定,積累多個進度更新失敗,這個實作會「丟失」那些進度。因為每次更新都是獨立的,失敗了就失敗了,不會重試或累積。但這也不算太大問題,因為:一、使用者關閉頁面前會觸發 `beforeunload` 事件,最後一次儲存;二、即使損失一些進度,使用者下次進來還是會從最後一次成功儲存的位置開始,只是要多看幾秒而已。如果要做得更完善,可以加入 retry 機制或本地 queue,但會增加複雜度。

3. 更新頻率與伺服器負載的平衡需要考慮幾個因素:使用者數量、影片長度、伺服器處理能力、資料重要性。對於進度追蹤這種「非關鍵」資料,10-30 秒的間隔都合理。如果是交易或支付這類「關鍵」資料,就要即時更新並確保送達。也可以做「動態調整」:網路狀況好時更頻繁,網路差時降低頻率;或者用「批次更新」,累積多個操作一次送出。

---

### Q38. 平行 API 呼叫

**解答**:

1. 使用 `Promise.all` 的優點是「平行執行,大幅縮短總等待時間」。如果兩個 API 各需要 500ms,依序執行就要 1 秒,但用 Promise.all 平行執行,總時間還是大約 500ms。這在使用者體驗上差異很大,頁面載入快一倍。而且這兩個請求(已購買課程 & 未付款訂單)沒有依賴關係,完全可以同時發起,沒理由等第一個完成再發第二個。

2. 如果其中一個 API 失敗,Promise.all 會立刻 reject,剩下的請求還是會繼續執行,但它們的結果會被忽略。不過這段程式碼沒有直接用 try-catch 包 Promise.all,而是檢查每個 `result.success`,所以即使其中一個失敗(回傳 `success: false`),也不會中斷整個流程。只是失敗的那個資料會顯示錯誤或用預設值,另一個成功的還是會正常處理。這是因為 API client 設計成「不 throw error」,而是回傳包含 error 的物件。

3. 什麼情況不應該用 Promise.all?當請求之間「有依賴關係」時。比如要先呼叫 API A 拿到 token,再用這個 token 去呼叫 API B,這時必須依序執行。或者當其中一個請求失敗時,後面的請求就不該執行(比如先檢查權限,沒權限就不要載入資料)。還有就是當請求數量很多時(比如 100 個),應該用「分批平行」或「限制並行數」,避免一次發太多請求導致瀏覽器或伺服器過載。

---

### Q39. API Response Type Safety

**解答**:

1. Discriminated union type 的好處是「型別安全的錯誤處理」。TypeScript 透過 `success` 這個「discriminant」(判別屬性),可以在編譯時期就知道:如果 `success: true`,那 `data` 一定存在,而 `error` 不存在;反之,如果 `success: false`,那 `error` 一定存在,而 `data` 不存在。這樣寫程式時 IDE 會有正確的自動完成,也不會出現「以為有 data 結果是 undefined」的 runtime 錯誤。

2. TypeScript 能在 `if (result.success)` 後知道 `result.data` 存在,是因為「control flow analysis」(控制流分析)。當你檢查 `result.success === true` 時,TypeScript 會「窄化」(narrow)type,把 union type 中符合條件的分支挑出來。此時 TypeScript 知道 result 的型別是 `{ success: true; data: T }`,所以可以安全地存取 `result.data`。在 else 分支,TypeScript 會自動推導出 `result` 是 `{ success: false; error: ApiError }`,所以可以存取 `result.error`。這是 TypeScript 最強大的功能之一。

3. 如果不用這種設計,直接讓 `data` 和 `error` 都是 optional:

```tsx
type ApiResponse<T> = {
	data?: T
	error?: ApiError
}
```

會有幾個問題:一、TypeScript 無法推導「有 data 就沒 error,有 error 就沒 data」,你必須手動檢查 `if (result.data && !result.error)`,很麻煩。二、可能出現「既有 data 又有 error」或「兩者都沒有」的無效狀態,TypeScript 無法防止這種錯誤。三、每次存取 `result.data` 都要加 optional chaining (`result.data?.xxx`),程式碼變得很冗長。Discriminated union 完美解決這些問題。

---

### Q40. Optimistic UI Update

**解答**:

1. 這裡是「先呼叫 API 再更新 UI」的保守做法,不是樂觀更新。程式碼明確地等 `await missionApi.deliverMission(...)` 完成,然後檢查 `if (result.success)` 才更新 progress 和 sidebar。樂觀更新是反過來:先假設會成功,馬上更新 UI,讓使用者立刻看到結果,API 在背景執行,失敗再回滾。這裡的做法比較保守,使用者要等 API 回應才看到變化,但更安全。

2. 如果改成樂觀更新,程式碼要這樣改:

```tsx
const handleDeliverMission = async () => {
	if (!user || !mission) return

	// 1. 樂觀更新 UI (先改)
	const previousProgress = progress
	setProgress((prev) => (prev ? { ...prev, status: 'DELIVERED' } : null))
	updateMissionStatus(missionId, 'DELIVERED')

	setIsDelivering(true)
	const result = await missionApi.deliverMission(parseInt(user.id), missionId)

	if (result.success) {
		// 2. 成功:顯示獲得的經驗值
		toast.success('任務完成！', {
			description: `獲得 ${result.data.experienceGained} 經驗值!`
		})
	} else {
		// 3. 失敗:回滾 UI
		setProgress(previousProgress)
		updateMissionStatus(missionId, previousProgress?.status || 'COMPLETED')
		toast.error('完成任務失敗')
	}
	setIsDelivering(false)
}
```

3. 什麼情況適合樂觀更新?「高成功率、可回滾、使用者體驗要求即時回饋」的操作,比如按讚、收藏、簡單的開關切換、發送訊息(可以顯示發送中)。什麼情況不適合?「可能失敗、有重要副作用、需要伺服器計算結果」的操作。像這裡的「完成任務」,因為要計算獲得多少經驗值、是否升級,這些資訊只有伺服器知道,如果樂觀更新,就要猜測或顯示假資料,失敗時使用者體驗會很差。所以這裡選擇保守做法是正確的。

---

## 四、Tailwind CSS 與 UI (5 題)

### Q41. cn() 工具函式

**解答**:

1. `clsx` 和 `twMerge` 各有不同的職責。`clsx` 是用來「組合 class names」,可以接受字串、物件、陣列等各種格式,並自動過濾掉 falsy 的值。例如 `clsx('foo', true && 'bar', { baz: false })` 會回傳 `'foo bar'`。而 `twMerge` 是專門處理 Tailwind CSS 的 class 衝突,當有重複的樣式時,保留最後一個。比如 `twMerge('px-4 px-2')` 會回傳 `'px-2'`,因為後面的覆蓋前面的。

2. 組合這兩個函式是必要的,因為它們解決不同的問題。只用 `clsx` 的話,無法處理 Tailwind class 衝突,`className="px-4 px-2"` 兩個都會保留,可能產生不可預期的結果。只用 `twMerge` 的話,無法方便地處理條件式 class 和物件語法。兩者結合起來,既能靈活組合 class,又能正確處理 Tailwind 的優先級,是最佳實踐。

3. `twMerge` 處理衝突的方式是「辨識 Tailwind 的 class 類型」,然後保留同類型中最後出現的那個。例如 `p-4 p-2`,兩個都是 padding,就保留 `p-2`。又如 `text-red-500 text-blue-500`,兩個都是 text color,保留 `text-blue-500`。但 `px-4 py-2` 不衝突,因為一個是 padding-x,一個是 padding-y,會兩個都保留。這讓我們可以用元件的 props 覆蓋預設樣式,非常方便。

---

### Q42. 條件式樣式

**解答**:

1. `data-error={!!error}` 會在元素上設定一個 data attribute,值是 `"true"` 或 `"false"`。然後 `data-[error=true]:text-destructive` 是 Tailwind 的 data attribute selector,當 `data-error="true"` 時才套用 `text-destructive` class。這兩者配合,實現了「有錯誤時變紅色」的效果。`!!error` 的作用是把 error 物件轉成 boolean,確保 attribute 的值是 true/false 而不是物件。

2. 用 `data-*` attribute 而不是直接寫 `className={error ? 'text-destructive' : ''}` 有幾個好處。首先是「避免 class 字串拼接」,不用擔心 className prop 和條件式 class 的合併問題。其次是「利用 CSS 選擇器」,樣式邏輯寫在 className 字串裡,更清晰易讀。而且 data attribute 可以被外部 CSS 或其他元件查詢,提供了更多彈性。這是現代 React UI library 的常見模式,特別是配合 Tailwind 的 arbitrary variants。

3. 這種寫法對 CSS 優先級的影響是:data attribute selector(`data-[error=true]:text-destructive`)的 specificity 比單純的 class(`.text-destructive`)高一點點,因為它包含了 attribute selector。但因為是寫在同一個 className 字串裡,而且用的是 Tailwind 的 utility class,所以優先級還是很容易掌控。重點是透過 `cn()` 函式,外部傳入的 className 可以覆蓋這個預設樣式,因為 `twMerge` 會處理衝突,保留最後的那個。

---

### Q43. Responsive Video Player

**解答**:

1. `pb-[56.25%]` 是「padding-bottom: 56.25%」,這個數字來自 16:9 的比例計算:9 ÷ 16 = 0.5625 = 56.25%。這是用 padding 實現「響應式等比例容器」的經典技巧。因為 padding 的百分比是相對於「父元素的寬度」計算的,所以當寬度改變時,padding-bottom 也會跟著等比例改變,從而維持 16:9 的比例。

2. Padding trick 實現響應式 16:9 影片的原理是:外層 div 設定 `pb-[56.25%]`,這樣它的高度就會是寬度的 56.25%,也就是 16:9 比例。然後把實際的影片(YouTube iframe)放在一個 `absolute inset-0` 的內層 div,讓它填滿整個容器。這樣不管螢幕寬度如何改變,容器都能保持 16:9 比例,影片也會跟著縮放,不會變形。

3. 需要這麼多層巢狀 div 是有原因的:
   - 最外層 `video-player-wrapper`:控制整體寬度(`w-full`)
   - 第二層 `mx-auto max-w-6xl`:置中並限制最大寬度
   - 第三層 `pb-[56.25%]`:用 padding 撐出 16:9 的空間
   - 第四層 `absolute inset-0`:絕對定位,填滿父元素
   - 最內層 YouTube 元件:實際的 iframe

這種結構看起來複雜,但每一層都有明確的職責,是實現響應式等比例容器的標準做法。如果少一層,就會破壞比例或佈局。

---

### Q44. Radix UI 整合

**解答**:

1. Radix UI 提供了「無樣式的、可訪問的 UI primitives」。它處理了所有複雜的 accessibility 邏輯(ARIA attributes、鍵盤導航、focus 管理等),讓開發者不用自己實作這些細節。直接用原生的 `<label>` 元素雖然簡單,但缺少很多進階功能,比如自動關聯 error message、支援螢幕閱讀器的完整提示等。Radix UI 把這些「應該要有但很麻煩」的功能都做好了,我們只要套用樣式就好。

2. `React.ComponentProps<typeof LabelPrimitive.Root>` 這個型別的意思是「提取 LabelPrimitive.Root 元件的 props 型別」。TypeScript 會自動推導出這個元件接受哪些 props,然後我們的 `FormLabel` 可以繼承這些 props,並加上自己的擴充。這樣的好處是「型別安全且自動更新」:如果 Radix UI 更新了 Label 的 props,我們的程式碼會自動得到新的型別,不用手動維護。

3. Radix UI 和 Tailwind CSS 的搭配非常完美:Radix 負責「功能和行為」(無障礙、狀態管理、互動邏輯),Tailwind 負責「外觀樣式」(顏色、間距、排版)。Radix 的元件是 unstyled 的,給你完全的樣式控制權;Tailwind 的 utility-first 讓你可以快速套用樣式,不用寫 CSS 檔案。兩者結合就是「功能強大又美觀,而且完全客製化」,這也是為什麼 shadcn/ui 這類 UI library 都採用這種組合。

---

### Q45. Accessibility (a11y)

**解答**:

1. `aria-describedby` 的作用是「告訴輔助技術(螢幕閱讀器),哪些元素是這個表單欄位的說明文字」。當視障使用者 focus 到這個 input 時,螢幕閱讀器會自動唸出 `aria-describedby` 指向的內容,包括欄位的 description 和 error message。設定它可以讓視障使用者完整了解這個欄位的用途和當前狀態,不會漏掉重要資訊。

2. `aria-invalid` 告訴螢幕閱讀器「這個欄位目前的值是無效的」。當設為 `true` 時,螢幕閱讀器會特別提示使用者這個欄位有問題,通常會說「invalid」或「錯誤」。這讓視障使用者可以立刻知道哪些欄位需要修正,不用自己猜測或逐個檢查。配合 `aria-describedby` 指向的 error message,使用者就能知道「哪裡錯了」以及「怎麼改正」。

3. 動態設定 `aria-describedby` 根據是否有 error,是為了「只在需要時才關聯 error message」。如果沒有錯誤,`aria-describedby` 只指向 description;有錯誤時,就加上 error message 的 ID。這樣螢幕閱讀器才不會在沒有錯誤的時候唸出空的 error message,造成困擾。而且當錯誤出現時,螢幕閱讀器會自動通知使用者,因為 `aria-describedby` 的內容改變了。這是非常細緻的無障礙設計,確保所有使用者都有良好的體驗。

---

## 五、TypeScript 與型別系統 (5 題)

### Q46. Generic Types 的使用

**解答**:

1. `get` 只有一個泛型 `<T>` 是因為 GET 請求「只有回應,沒有請求 body」。`T` 代表回應的資料型別,使用者只需要指定「會拿回什麼資料」。而 `post` 有兩個泛型 `<T, D>`:T 是回應型別,D 是請求 body 的型別,因為 POST 請求通常要「送出資料」給伺服器。這樣設計讓 TypeScript 可以同時檢查送出的資料格式和回應的資料格式。

2. `D = unknown` 是「泛型的預設值」,意思是如果使用者沒有指定 D 的型別,就預設為 `unknown`。用 `unknown` 而不是 `any`,是因為 `unknown` 更安全:`any` 完全放棄型別檢查,而 `unknown` 強制你在使用前先做型別檢查或斷言。雖然這裡是預設值,很少會真的用到 unknown,但它表達了「我們不確定 body 的型別」這個語意,比 `any` 更正確。

3. 使用時可以明確指定泛型,也可以讓 TypeScript 自動推導。明確指定的範例:

```tsx
const result = await apiClient.post<LoginResponse, LoginRequest>('/auth/login', {
	username: 'test',
	password: '12345678'
})
```

自動推導的範例:

```tsx
const loginData = { username: 'test', password: '12345678' }
const result = await apiClient.post<LoginResponse>('/auth/login', loginData)
// D 會自動推導為 typeof loginData
```

大部分情況下自動推導就夠了,只有在需要明確型別約束時才手動指定。

---

### Q47. Discriminated Unions

**解答**:

1. Discriminated union(判別聯合型別)是一種 TypeScript 模式,用一個「共同的屬性」(discriminant)來區分 union 中的不同分支。這裡的 `success` 就是 discriminant:當 `success: true` 時,型別是 `{ success: true; data: T }`;當 `success: false` 時,型別是 `{ success: false; error: ApiError }`。TypeScript 可以透過檢查 `success` 的值,自動窄化(narrow)型別,知道該分支有哪些屬性可用。

2. TypeScript 利用 `success` 做型別窄化的原理是「control flow analysis」(控制流分析)。當你寫 `if (result.success)` 時,TypeScript 知道在這個 if block 內,`result` 的型別一定是 `{ success: true; data: T }`,因為 `success: false` 的分支已經被排除了。所以可以安全地存取 `result.data`,TypeScript 保證它存在。反之,在 else block 中,`result` 的型別是 `{ success: false; error: ApiError }`,可以存取 `result.error`。這是編譯時期的型別安全,不需要 runtime check。

3. 如果想加入第三種狀態(loading),要設計成這樣:

```tsx
type ApiResponse<T> =
	| { status: 'loading' }
	| { status: 'success'; data: T }
	| { status: 'error'; error: ApiError }
```

這裡把 discriminant 改成 `status`,因為有三個可能的值。使用時:

```tsx
if (result.status === 'loading') {
	// result.status 的型別是 'loading'
} else if (result.status === 'success') {
	// 可以存取 result.data
} else {
	// result.status === 'error', 可以存取 result.error
}
```

這種模式可以擴展到任意多個狀態,是 TypeScript 中處理複雜狀態的最佳實踐。

---

### Q48. React.ComponentProps

**解答**:

1. `React.ComponentProps<'div'>` 會產生「原生 div 元素的所有 props」的型別,包括 `className`、`onClick`、`style`、`aria-*` 等等所有 HTML div 支援的屬性。這是 TypeScript 的 utility type,可以從任何元件或 HTML 元素提取 props 型別。對於原生元素,寫 `'div'`、`'button'`、`'input'` 等字串;對於 React 元件,寫 `typeof MyComponent`。

2. 用 `React.ComponentProps` 而不是自己定義 props interface,有幾個好處:

   - **完整性**:不會遺漏任何原生屬性,使用者可以傳入所有 div 支援的 props
   - **自動更新**:如果 React 或 TypeScript 更新了 HTML 元素的型別定義,你的程式碼會自動跟上
   - **省時省力**:不用手動列舉幾十個屬性,一行搞定
   - **型別一致性**:確保你的元件 props 和原生元素完全相容

3. 這種寫法對元件的可重用性幫助很大。使用者可以把你的元件當成「加強版的 div」來用,所有 div 的 props 都能傳入,不會有意外。例如:

```tsx
<FormItem
	className="custom-class"
	onClick={handleClick}
	data-testid="form-item"
	aria-label="Custom form item"
/>
```

這些 props 都會透過 `{...props}` 傳給內部的 div,元件行為符合直覺。這是建立可組合、可擴展元件的關鍵技巧。

---

### Q49. Type Inference with Zod

**解答**:

1. `z.infer<typeof loginSchema>` 是 Zod 提供的 utility type,可以從 schema 「反向推導」出 TypeScript 型別。它會分析 `loginSchema` 的結構,產生對應的型別:

```tsx
// loginSchema 定義了兩個 string 欄位
// z.infer 推導出:
type LoginFormValues = {
	username: string
	password: string
}
```

這是「從 runtime validation 推導 compile-time type」,讓驗證規則和型別定義保持同步,不用寫兩次。

2. 不直接寫 interface 的原因是「DRY(Don't Repeat Yourself)」和「Single Source of Truth」。如果分開寫:

```tsx
const schema = z.object({ username: z.string(), password: z.string() })
interface LoginFormValues {
	username: string
	password: string
}
```

就要維護兩份資料,很容易不同步。比如你在 schema 加了 `email` 欄位,但忘了改 interface,TypeScript 不會報錯,但 runtime 驗證和 compile-time type 就不一致了。用 `z.infer` 可以確保只有一個定義來源,改 schema 時型別會自動更新。

3. 這種方式確保 runtime validation 和 compile-time type checking 一致性的原理是:schema 既是驗證規則(runtime),又是型別定義的來源(compile-time)。當你修改 schema,比如把 `username` 改成 optional:`z.string().optional()`,`z.infer` 會自動推導出 `username?: string`,TypeScript 會要求所有使用這個型別的地方處理 undefined 的情況。這樣驗證和型別永遠同步,不可能出現「通過型別檢查但驗證失敗」或「驗證通過但型別錯誤」的情況。

---

### Q50. Context Type Safety

**解答**:

1. `createContext` 的泛型是 `AuthContextType | undefined` 而不是直接 `AuthContextType`,是因為「Context 初始值的限制」。React Context 在建立時必須提供一個初始值,但我們無法在建立時就提供完整的 `AuthContextType`(因為 `login`、`logout` 等函式都在 Provider 內部才定義)。設定成 `undefined` 的話,就可以寫 `createContext<AuthContextType | undefined>(undefined)`,用 undefined 作為預設值。

2. `useAuth` 中的 runtime check(`if (context === undefined)`)和 TypeScript 型別檢查的關係是「型別守衛」(type guard)。雖然 context 的型別是 `AuthContextType | undefined`,但檢查後如果不是 undefined,TypeScript 會自動窄化型別為 `AuthContextType`。所以 return 的 context 型別是 `AuthContextType`,不包含 undefined,使用者不用每次都檢查 `context?.user`。這個 runtime check 同時也保護程式邏輯:如果有人在 Provider 外面用 `useAuth()`,會拋出清楚的錯誤訊息,而不是默默傳回 undefined 導致奇怪的 bug。

3. 有一種更好的方式,可以避免每次都檢查 undefined:直接在 `createContext` 時提供一個「假的初始值」,或者用一個 factory function:

```tsx
// 方法 1: 用 as 斷言(不太安全,但方便)
const AuthContext = createContext<AuthContextType>({} as AuthContextType)

// 方法 2: 用 factory pattern
const createAuthContext = () => {
	const context = createContext<AuthContextType | undefined>(undefined)
	const useAuth = () => {
		const ctx = useContext(context)
		if (!ctx) throw new Error('useAuth must be used within AuthProvider')
		return ctx
	}
	return [context.Provider, useAuth] as const
}
```

但實際上目前的做法(undefined + runtime check)是最安全且最清晰的,因為它強制使用者在 Provider 內使用,錯誤訊息也很明確。方法 1 的 `as` 斷言雖然簡單,但失去了型別安全,不建議在正式專案使用。

---

## 結語

恭喜!你已經完成了 50 道前端面試題的所有解答。

這些題目涵蓋了:

- **React Hooks 與語法細節** (20 題):深入理解 useEffect、useCallback、useRef 等核心概念
- **狀態管理與資料流** (10 題):Context 設計、跨 Tab 通訊、樂觀更新等實戰技巧
- **與後端互動** (10 題):API Client 架構、錯誤處理、Retry 機制、型別安全
- **Tailwind CSS 與 UI** (5 題):工具函式、響應式設計、無障礙支援
- **TypeScript 與型別系統** (5 題):泛型、Discriminated Unions、型別推導

**面試準備建議**:

1. 不要死背答案,理解每個設計決策背後的「為什麼」
2. 能夠舉一反三,套用到其他類似的情境
3. 準備好說出「如果改成 XXX 會怎樣?」的答案
4. 可以討論替代方案和權衡取捨

**進階學習方向**:

- 效能優化:React Profiler、虛擬化列表、程式碼分割
- 測試:Jest、React Testing Library、E2E 測試
- 狀態管理:Zustand、Jotai、TanStack Query
- 建置工具:Webpack、Vite、Turbopack

祝你面試順利!💪

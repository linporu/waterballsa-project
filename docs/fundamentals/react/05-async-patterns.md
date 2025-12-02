# 非同步處理模式

> 給熟悉 JavaScript async/await 的開發者

本文聚焦在 React 專案中的**非同步處理最佳實踐**，包括 API 呼叫、錯誤處理、重試機制與 timeout 控制。

📝 **對應面試題**：Q14, Q31-Q40

---

## 目錄

1. [Promise.all 平行呼叫](#1-promiseall-平行呼叫)
2. [AbortController 與 Timeout](#2-abortcontroller-與-timeout)
3. [Retry 重試機制](#3-retry-重試機制)
4. [錯誤處理模式](#4-錯誤處理模式)
5. [Request/Response 攔截器](#5-requestresponse-攔截器)

---

## 1. Promise.all 平行呼叫

### 💡 核心概念

當需要同時呼叫多個獨立的 API 時，使用 `Promise.all()` 可以**平行執行**，大幅提升效能。

### 基本語法

```typescript
// ❌ 序列執行：總時間 = t1 + t2 + t3
const user = await fetchUser() // 等 1 秒
const posts = await fetchPosts() // 再等 1 秒
const comments = await fetchComments() // 再等 1 秒
// 總共 3 秒

// ✅ 平行執行：總時間 = max(t1, t2, t3)
const [user, posts, comments] = await Promise.all([
	fetchUser(), // 同時開始
	fetchPosts(), // 同時開始
	fetchComments() // 同時開始
])
// 總共 1 秒（假設每個都是 1 秒）
```

### 專案實例 1: 載入購買資料

```typescript
// 檔案位置: contexts/user-purchase-context.tsx (第 72-80 行)

const fetchUserPurchaseData = useCallback(
	async (showLoading = true) => {
		if (!user || !isAuthenticated) return

		try {
			const userId = parseInt(user.id)

			// ✅ 平行呼叫兩個 API
			const [purchasesResult, ordersResult] = await Promise.all([
				userPurchaseApi.getUserPurchasedJourneys(userId),
				userPurchaseApi.getUserOrders(userId, {
					page: 1,
					limit: 50,
					status: 'UNPAID'
				})
			])

			// 處理結果
			if (purchasesResult.success) {
				const journeyIds = new Set(purchasesResult.data.map((p) => p.journeyId))
				setPurchasedJourneyIds(journeyIds)
			}

			if (ordersResult.success) {
				setUnpaidOrders(ordersResult.data.items)
			}
		} catch (error) {
			// 錯誤處理
		}
	},
	[user, isAuthenticated]
)
```

### 專案實例 2: 批次載入任務進度

```typescript
// 檔案位置: contexts/journey-context.tsx (第 46-63 行)

const fetchJourney = useCallback(async (slug: string, userId?: string) => {
	const result = await journeyApi.getJourneyBySlug(slug)

	if (result.success && userId) {
		const journeyData = result.data

		// 收集所有任務 ID
		const allMissionIds = journeyData.chapters.flatMap((chapter) =>
			chapter.missions.map((mission) => mission.id)
		)

		// ✅ 平行載入所有任務的進度（可能有數十個）
		const progressResults = await Promise.all(
			allMissionIds.map((missionId) =>
				missionApi.getUserMissionProgress(parseInt(userId), missionId)
			)
		)

		// 建立 missionId -> status 的對照表
		const statusMap = new Map<number, MissionStatus>()
		progressResults.forEach((progressResult, index) => {
			if (progressResult.success && progressResult.data.status) {
				statusMap.set(allMissionIds[index], progressResult.data.status)
			}
		})

		// 更新 journey 資料...
	}
}, [])
```

### ⚠️ 常見陷阱

1. **Promise.all 遇到錯誤會立即中斷**

   ```typescript
   // ❌ 任何一個失敗就全部中斷
   const [a, b, c] = await Promise.all([
   	fetch('/api/a'), // 成功
   	fetch('/api/b'), // 失敗 ❌
   	fetch('/api/c') // 不會執行
   ])

   // ✅ 使用 Promise.allSettled 繼續執行
   const results = await Promise.allSettled([fetch('/api/a'), fetch('/api/b'), fetch('/api/c')])

   results.forEach((result, index) => {
   	if (result.status === 'fulfilled') {
   		console.log(`API ${index} 成功:`, result.value)
   	} else {
   		console.log(`API ${index} 失敗:`, result.reason)
   	}
   })
   ```

2. **不要在迴圈中 await**

   ```typescript
   // ❌ 序列執行（慢）
   for (const id of missionIds) {
   	await fetchMission(id) // 一個一個等
   }

   // ✅ 平行執行（快）
   await Promise.all(missionIds.map((id) => fetchMission(id)))
   ```

### 📝 相關面試題

- [Q31: Promise.all 的用途](../interview/interview-frontend.md#q31)
- [Q32: Promise.all vs Promise.allSettled](../interview/interview-frontend.md#q32)

---

## 2. AbortController 與 Timeout

### 💡 核心概念

**AbortController** 可以：

1. **取消正在進行的 fetch 請求**
2. **實作 timeout 超時控制**

### 基本語法

```typescript
// 建立 AbortController
const controller = new AbortController()

// 發送請求時傳入 signal
fetch('/api/data', {
	signal: controller.signal
})

// 取消請求
controller.abort()
```

### 專案實例：Timeout 控制

```typescript
// 檔案位置: lib/api/core/client.ts (第 135-164 行)

private async executeRequest<T>(
  requestConfig: RequestConfig,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  // ✅ 建立 AbortController
  const controller = new AbortController()

  // ✅ 設定 timeout：時間到就取消請求
  const timeoutId = setTimeout(
    () => controller.abort(),
    requestConfig.timeout  // 預設 30 秒
  )

  try {
    // 發送請求，傳入 signal
    const response = await fetch(requestConfig.url, {
      method: requestConfig.method,
      headers: requestConfig.headers,
      body: requestConfig.body ? JSON.stringify(requestConfig.body) : undefined,
      signal: controller.signal,  // ✅ 連接 AbortController
      ...options,
    })

    clearTimeout(timeoutId)  // ✅ 請求完成，清除 timeout
    return await this.handleResponse<T>(response)
  } catch (error) {
    clearTimeout(timeoutId)

    // ✅ 檢查是否為 timeout 錯誤
    if (error instanceof Error && error.name === 'AbortError') {
      return {
        success: false,
        error: { message: 'Request timeout' },
      }
    }

    throw error
  }
}
```

### 實際範例：取消請求（React 18 useEffect）

```typescript
function SearchComponent() {
	const [query, setQuery] = useState('')
	const [results, setResults] = useState([])

	useEffect(() => {
		// ✅ 建立 AbortController
		const controller = new AbortController()

		async function search() {
			try {
				const response = await fetch(`/api/search?q=${query}`, {
					signal: controller.signal
				})
				const data = await response.json()
				setResults(data)
			} catch (error) {
				if (error.name === 'AbortError') {
					console.log('請求被取消') // 正常情況，不需要處理
				} else {
					console.error('搜尋失敗:', error)
				}
			}
		}

		if (query) {
			search()
		}

		// ✅ Cleanup: 取消前一次的請求
		return () => {
			controller.abort() // 當 query 改變或元件卸載時取消
		}
	}, [query])

	return <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="搜尋..." />
}
```

### 💡 為什麼需要 AbortController？

```typescript
// 問題場景：使用者快速輸入 "react"
// 輸入 "r" → 發送請求 A
// 輸入 "re" → 發送請求 B
// 輸入 "rea" → 發送請求 C
// 輸入 "reac" → 發送請求 D
// 輸入 "react" → 發送請求 E

// ❌ 沒有 AbortController：
// - 5 個請求都會執行
// - 如果請求 A 最慢，最後會顯示 "r" 的結果（錯誤！）

// ✅ 有 AbortController：
// - 新請求發送時，舊請求被取消
// - 只有最後一個請求（"react"）會完成
```

### 📝 相關面試題

- [Q33: AbortController 的應用](../interview/interview-frontend.md#q33)
- [Q34: Fetch timeout 實作](../interview/interview-frontend.md#q34)

---

## 3. Retry 重試機制

### 💡 核心概念

網路請求可能因為**暫時性錯誤**（網路抖動、伺服器暫時過載）失敗，此時應該自動重試。

### 專案實例：Exponential Backoff 指數退避

```typescript
// 檔案位置: lib/api/core/client.ts (第 169-244 行)

private async request<T>(
  method: HttpMethod,
  path: string,
  data?: unknown,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  // 重試設定
  const maxRetries = this.config.retry!.maxRetries!  // 預設 3 次
  const baseDelay = this.config.retry!.retryDelay!   // 預設 100ms

  let lastError: ApiResponse<T> | null = null

  // ✅ 重試迴圈：最多嘗試 maxRetries + 1 次（原始請求 + 重試）
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      // 執行請求
      let result = await this.executeRequest<T>(requestConfig, options)

      // ✅ 如果失敗且應該重試
      if (
        !result.success &&
        attempt < maxRetries &&
        this.shouldRetry(method, result.error)
      ) {
        lastError = result

        // ✅ Exponential backoff: 100ms, 200ms, 400ms, 800ms...
        await this.sleep(baseDelay * Math.pow(2, attempt))
        continue  // 進入下一次重試
      }

      // 成功或不應重試，返回結果
      return result
    } catch (error) {
      const errorResponse = this.createErrorResponse(error)

      // 檢查是否應該重試
      if (
        attempt < maxRetries &&
        !errorResponse.success &&
        this.shouldRetry(method, errorResponse.error)
      ) {
        lastError = errorResponse
        await this.sleep(baseDelay * Math.pow(2, attempt))
        continue
      }

      return errorResponse
    }
  }

  return lastError ?? this.createErrorResponse(new Error('Unknown error'))
}
```

### 重試判斷邏輯

```typescript
// 檔案位置: lib/api/core/client.ts (第 97-123 行)

// ✅ 檢查 HTTP method 是否為冪等 (idempotent)
private isIdempotentMethod(method: HttpMethod): boolean {
  return ['GET', 'PUT', 'DELETE'].includes(method)
  // GET: 讀取資料，可以重試
  // PUT: 更新資料（整體取代），重複執行結果相同
  // DELETE: 刪除資料，重複執行結果相同
  // POST: ❌ 不可重試（會重複建立資料）
}

// ✅ 判斷是否應該重試
private shouldRetry(
  method: HttpMethod,
  error: { message: string; status?: number }
): boolean {
  // 1. 只重試冪等方法
  if (!this.isIdempotentMethod(method)) {
    return false
  }

  // 2. 只重試網路錯誤（沒有 HTTP status code）
  // 不重試 4xx, 5xx 錯誤（這些是業務邏輯錯誤，重試無意義）
  return !error.status
}
```

### 💡 Exponential Backoff 指數退避

```typescript
// 為什麼要用指數退避？

// ❌ 固定延遲（如 100ms）：
// - 伺服器過載時，大量請求同時重試，雪上加霜

// ✅ 指數退避：
// - 第 1 次重試：等 100ms
// - 第 2 次重試：等 200ms
// - 第 3 次重試：等 400ms
// - 第 4 次重試：等 800ms

// 公式：baseDelay * 2^attempt

for (let attempt = 0; attempt < maxRetries; attempt++) {
	const delay = baseDelay * Math.pow(2, attempt)
	await sleep(delay)
	// retry...
}
```

### 實際範例：自訂重試條件

```typescript
const client = new ApiClient({
	retry: {
		maxRetries: 3,
		retryDelay: 1000, // 1 秒

		// ✅ 自訂重試條件
		retryOn: (error) => {
			// 重試 503 Service Unavailable 和網路錯誤
			return !error.status || error.status === 503
		}
	}
})
```

### 📝 相關面試題

- [Q35: Retry 機制的實作](../interview/interview-frontend.md#q35)
- [Q36: Exponential Backoff 原理](../interview/interview-frontend.md#q36)
- [Q37: 冪等性與重試](../interview/interview-frontend.md#q37)

---

## 4. 錯誤處理模式

### 💡 Discriminated Union 錯誤處理

專案使用 **Discriminated Union** 型別，強制檢查錯誤：

```typescript
// 所有 API 回應都是這個型別
type ApiResponse<T> = { success: true; data: T } | { success: false; error: ApiError }

// ✅ 使用時必須檢查 success
async function fetchUser() {
	const result = await userApi.getUserInfo()

	if (result.success) {
		// TypeScript 確保這裡有 data
		console.log(result.data.email)
	} else {
		// TypeScript 確保這裡有 error
		console.error(result.error.message)
	}
}
```

### 專案實例：錯誤回應處理

```typescript
// 檔案位置: lib/api/core/client.ts (第 32-68 行)

private async handleResponse<T>(response: Response): Promise<ApiResponse<T>> {
  try {
    const contentType = response.headers.get('content-type')
    const isJSON = contentType?.includes('application/json')

    // ✅ HTTP 錯誤 (4xx, 5xx)
    if (!response.ok) {
      const errorData = isJSON
        ? await response.json()
        : { error: await response.text() }

      return {
        success: false,
        error: {
          message: errorData.error || `HTTP error ${response.status}`,
          status: response.status,
          code: errorData.code,
        },
      }
    }

    // ✅ 成功回應
    const data = isJSON ? await response.json() : await response.text()
    return {
      success: true,
      data: data as T,
    }
  } catch (error) {
    // ✅ JSON 解析錯誤
    return {
      success: false,
      error: {
        message: error instanceof Error ? error.message : 'Failed to parse response',
      },
    }
  }
}
```

### 錯誤分類

| 錯誤類型           | HTTP Status | 是否重試 | 範例                      |
| ------------------ | ----------- | -------- | ------------------------- |
| **網路錯誤**       | 無          | ✅ 是    | 斷網、DNS 失敗            |
| **Timeout**        | 無          | ✅ 是    | 請求超時                  |
| **4xx 客戶端錯誤** | 400-499     | ❌ 否    | 401 未授權、404 Not Found |
| **5xx 伺服器錯誤** | 500-599     | ❌ 否\*  | 500 Internal Error        |

\* 除非是暫時性錯誤（如 503），否則不重試

### 實際範例：元件中的錯誤處理

```typescript
function DataComponent() {
	const [data, setData] = useState(null)
	const [error, setError] = useState(null)
	const [isLoading, setIsLoading] = useState(false)

	useEffect(() => {
		async function fetchData() {
			setIsLoading(true)
			setError(null)

			const result = await api.getData()

			if (result.success) {
				setData(result.data)
			} else {
				// ✅ 根據錯誤類型處理
				if (result.error.status === 401) {
					// 未授權：導向登入頁
					router.push('/login')
				} else if (result.error.status === 404) {
					// 資料不存在
					setError('資料不存在')
				} else {
					// 其他錯誤
					setError(result.error.message)
					toast.error('載入失敗', {
						description: result.error.message
					})
				}
			}

			setIsLoading(false)
		}

		fetchData()
	}, [])

	if (isLoading) return <Loading />
	if (error) return <Error message={error} />
	if (!data) return null

	return <DataDisplay data={data} />
}
```

### 📝 相關面試題

- [Q38: API 錯誤處理策略](../interview/interview-frontend.md#q38)
- [Q39: Discriminated Union 的優勢](../interview/interview-frontend.md#q39)

---

## 5. Request/Response 攔截器

### 💡 核心概念

**攔截器 (Interceptors)** 允許在請求發送前或回應返回後統一處理邏輯，例如：

- 請求攔截器：加入 Authentication token
- 回應攔截器：統一處理 401 錯誤

### 基本結構

```typescript
// 檔案位置: lib/api/api-schema.ts
export interface ApiClientConfig {
	baseURL: string
	headers?: Record<string, string>
	timeout?: number
	retry?: RetryConfig
	interceptors?: {
		request?: (config: RequestConfig) => Promise<RequestConfig>
		response?: <T>(response: ApiResponse<T>) => Promise<ApiResponse<T>>
	}
}
```

### 實際範例：Request 攔截器（加入 Token）

```typescript
const client = new ApiClient({
	baseURL: 'https://api.example.com',

	// ✅ Request 攔截器
	interceptors: {
		request: async (config) => {
			// 從 cookie 或 localStorage 取得 token
			const token = getToken()

			if (token) {
				// 加入 Authorization header
				config.headers = {
					...config.headers,
					Authorization: `Bearer ${token}`
				}
			}

			return config
		}
	}
})
```

### 實際範例：Response 攔截器（處理 401）

```typescript
const client = new ApiClient({
	baseURL: 'https://api.example.com',

	// ✅ Response 攔截器
	interceptors: {
		response: async (response) => {
			// 統一處理 401 未授權
			if (!response.success && response.error.status === 401) {
				// 清除 token
				removeToken()

				// 導向登入頁
				window.location.href = '/login'

				// 或拋出特殊錯誤
				return {
					success: false,
					error: {
						message: '登入已過期，請重新登入',
						status: 401
					}
				}
			}

			return response
		}
	}
})
```

### 專案實例：攔截器的應用

```typescript
// 檔案位置: lib/api/core/client.ts (第 188-219 行)

private async request<T>(
  method: HttpMethod,
  path: string,
  data?: unknown,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  let requestConfig: RequestConfig = {
    method,
    url: this.buildURL(path),
    headers: { ...this.config.headers, ...options?.headers },
    body: data,
    timeout: options?.timeout ?? this.config.timeout!,
  }

  // ✅ 應用 request 攔截器
  if (this.config.interceptors?.request) {
    requestConfig = await this.config.interceptors.request(requestConfig)
  }

  // 執行請求...
  let result = await this.executeRequest<T>(requestConfig, options)

  // ✅ 應用 response 攔截器
  if (this.config.interceptors?.response) {
    result = await this.config.interceptors.response(result)
  }

  return result
}
```

### 進階：鏈式攔截器

```typescript
// 如果需要多個攔截器，可以組合它們
function composeInterceptors(...interceptors) {
	return async (config) => {
		let result = config
		for (const interceptor of interceptors) {
			result = await interceptor(result)
		}
		return result
	}
}

const client = new ApiClient({
	interceptors: {
		request: composeInterceptors(addAuthToken, addTimestamp, addRequestId)
	}
})
```

### 📝 相關面試題

- [Q40: Interceptor 的應用場景](../interview/interview-frontend.md#q40)
- [Q14: Request/Response 攔截器設計](../interview/interview-frontend.md#q14)

---

## 🎯 總結

### 非同步處理核心模式

| 模式                    | 用途                 | 關鍵 API                   |
| ----------------------- | -------------------- | -------------------------- |
| **Promise.all**         | 平行呼叫多個 API     | `await Promise.all([...])` |
| **AbortController**     | 取消請求、Timeout    | `controller.abort()`       |
| **Retry**               | 處理暫時性錯誤       | Exponential backoff        |
| **Discriminated Union** | 型別安全的錯誤處理   | `if (result.success)`      |
| **Interceptor**         | 統一處理 token、錯誤 | Request/Response 攔截      |

### 最佳實踐

1. **能平行就平行**：使用 `Promise.all()` 而非序列 await
2. **設定 timeout**：避免請求永久掛起
3. **只重試冪等操作**：GET、PUT、DELETE 可重試，POST 不行
4. **Exponential backoff**：避免雪崩效應
5. **型別安全的錯誤處理**：使用 Discriminated Union
6. **用攔截器統一處理**：Token、401 錯誤等

### 常見錯誤

❌ **迴圈中 await**（序列執行，慢）

```typescript
for (const id of ids) {
	await fetch(id) // 一個一個等
}
```

❌ **POST 請求自動重試**（可能重複建立資料）

```typescript
// 危險！
await retryRequest('POST', '/api/orders', data)
```

❌ **忘記清理 timeout**（記憶體洩漏）

```typescript
const timer = setTimeout(...)
// 忘記 clearTimeout(timer)
```

---

## 🚀 下一步

- 學習 [Tailwind CSS & 無障礙](./06-tailwind-a11y.md)
- 回顧 [React Hooks 快速指南](./01-react-hooks.md)

---

## 📖 延伸閱讀

- [MDN: Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [HTTP Idempotency](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)

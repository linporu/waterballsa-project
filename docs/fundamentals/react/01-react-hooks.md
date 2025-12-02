# React Hooks 快速指南

> 專為熟悉 Vue 3 Composition API 的開發者設計

React Hooks 是 React 16.8 引入的功能，讓你在不寫 class 的情況下使用 state 和其他 React 特性。如果你熟悉 Vue 3 的 Composition API，會發現很多相似的概念，但也有關鍵差異。

📝 **對應面試題**：Q1-Q15, Q21-Q30

---

## 目錄

1. [useState - 狀態管理](#1-usestate---狀態管理)
2. [useEffect - 副作用處理](#2-useeffect---副作用處理)
3. [useCallback - 函式記憶化](#3-usecallback---函式記憶化)
4. [useMemo - 值記憶化](#4-usememo---值記憶化)
5. [useRef - 可變引用](#5-useref---可變引用)
6. [Custom Hooks - 邏輯復用](#6-custom-hooks---邏輯復用)

---

## 1. useState - 狀態管理

### 基本語法

```typescript
const [state, setState] = useState(initialValue)
```

### 🆚 Vue vs React

| Vue 3                  | React                                   |
| ---------------------- | --------------------------------------- |
| `const count = ref(0)` | `const [count, setCount] = useState(0)` |
| `count.value++`        | `setCount(count + 1)`                   |
| 直接修改 `.value`      | 呼叫 setter 函式                        |

**關鍵差異**：React 的 state 是**不可變 (immutable)** 的，不能直接修改，必須透過 setter 函式。

### 專案實例

```typescript
// 檔案位置: contexts/auth-context.tsx (第 38-39 行)
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)
	const [isLoading, setIsLoading] = useState(true)

	// ✅ 正確：使用 setter 更新狀態
	const login = (token: string, userData: UserInfo) => {
		setUser(userData) // 更新使用者資訊
	}

	// ❌ 錯誤：不能直接修改
	// user.name = 'New Name'  // 這樣不會觸發重新渲染！
}
```

```typescript
// 檔案位置: hooks/use-video-progress.ts (第 31 行)
export function useVideoProgress(options: UseVideoProgressOptions) {
	const [isCompleted, setIsCompleted] = useState(false)

	const handleEnd = () => {
		setIsCompleted(true) // 標記影片已完成
	}
}
```

### ⚠️ 常見陷阱

1. **直接修改 state**

   ```typescript
   // ❌ 錯誤
   const [user, setUser] = useState({ name: 'Alice' })
   user.name = 'Bob' // 不會觸發重新渲染！

   // ✅ 正確：創建新物件
   setUser({ ...user, name: 'Bob' })
   ```

2. **基於舊值更新**

   ```typescript
   // ❌ 可能有問題（非同步更新）
   setCount(count + 1)
   setCount(count + 1) // 不會 +2，只會 +1

   // ✅ 正確：使用函式形式
   setCount((prev) => prev + 1)
   setCount((prev) => prev + 1) // 會正確 +2
   ```

### 📝 相關面試題

- [Q1: useEffect 依賴陣列](../interview/interview-frontend.md#q1)
- [Q21: Context 搭配 useState](../interview/interview-frontend.md#q21)

---

## 2. useEffect - 副作用處理

### 基本語法

```typescript
useEffect(() => {
	// 副作用邏輯

	return () => {
		// cleanup 清理函式（可選）
	}
}, [dependencies]) // 依賴陣列
```

### 🆚 Vue vs React

| Vue 3                      | React                                     |
| -------------------------- | ----------------------------------------- |
| `onMounted(() => {...})`   | `useEffect(() => {...}, [])`              |
| `watch(source, callback)`  | `useEffect(() => {...}, [source])`        |
| `watchEffect(() => {...})` | 自動追蹤依賴 ❌ React 需手動聲明          |
| `onUnmounted(() => {...})` | `useEffect(() => { return cleanup }, [])` |

**關鍵差異**：Vue 自動追蹤依賴，React 需要**手動在依賴陣列中聲明**所有使用的 props 和 state。

### 專案實例

#### 範例 1: 元件掛載時執行（類似 onMounted）

```typescript
// 檔案位置: contexts/auth-context.tsx (第 42-55 行)
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)
	const [isLoading, setIsLoading] = useState(true)

	// 初始化時檢查 token（只執行一次）
	useEffect(() => {
		const token = getToken()
		if (token) {
			const cachedUser = getUserInfo()
			if (cachedUser) {
				setUser(cachedUser)
			} else {
				removeToken()
			}
		}
		setIsLoading(false)
	}, []) // ⚠️ 空陣列 = 只在元件掛載時執行一次
}
```

#### 範例 2: 監聽變數變化（類似 watch）

```typescript
// 檔案位置: hooks/use-mission.ts (第 42-46 行)
export function useMission(): UseMissionReturn {
	const { isAuthenticated, isLoading: authLoading } = useAuth()
	const router = useRouter()

	// 當驗證狀態改變時，檢查是否需要重新導向
	useEffect(() => {
		if (!authLoading && !isAuthenticated) {
			router.push('/login') // 未登入則導向登入頁
		}
	}, [authLoading, isAuthenticated, router]) // ✅ 依賴所有使用的變數
}
```

#### 範例 3: 清理副作用（cleanup function）

```typescript
// 檔案位置: hooks/use-video-progress.ts (第 65-71 行)
export function useVideoProgress(options: UseVideoProgressOptions) {
	const saveCurrentProgress = useCallback(() => {
		const currentTime = playerRef.current?.getCurrentTime()
		if (currentTime) onProgressUpdate(Math.floor(currentTime))
	}, [onProgressUpdate])

	// 監聽頁面離開事件，儲存進度
	useEffect(() => {
		window.addEventListener('beforeunload', saveCurrentProgress)

		// ✅ cleanup: 元件卸載時移除監聽器
		return () => {
			window.removeEventListener('beforeunload', saveCurrentProgress)
			stopProgressTracking() // 同時停止計時器
		}
	}, [saveCurrentProgress, stopProgressTracking])
	// ⚠️ 依賴陣列包含所有使用的函式
}
```

### ⚠️ 常見陷阱

1. **忘記加依賴陣列**

   ```typescript
   // ❌ 錯誤：每次渲染都執行
   useEffect(() => {
   	fetchData()
   })

   // ✅ 正確：只在掛載時執行
   useEffect(() => {
   	fetchData()
   }, [])
   ```

2. **遺漏依賴項目**

   ```typescript
   // ❌ 錯誤：ESLint 會警告
   useEffect(() => {
   	console.log(user.name) // 使用了 user，但沒加到依賴
   }, [])

   // ✅ 正確
   useEffect(() => {
   	console.log(user.name)
   }, [user]) // 或 [user.name]
   ```

3. **忘記清理副作用**

   ```typescript
   // ❌ 可能造成 memory leak
   useEffect(() => {
   	const timer = setInterval(() => console.log('tick'), 1000)
   	// 忘記 return cleanup
   }, [])

   // ✅ 正確
   useEffect(() => {
   	const timer = setInterval(() => console.log('tick'), 1000)
   	return () => clearInterval(timer) // 清理計時器
   }, [])
   ```

### 💡 依賴陣列規則

| 依賴陣列 | 執行時機                     |
| -------- | ---------------------------- |
| 不寫     | 每次渲染都執行（通常是錯誤） |
| `[]`     | 只在掛載時執行一次           |
| `[a, b]` | 掛載時 + a 或 b 改變時執行   |

### 📝 相關面試題

- [Q1: useEffect 依賴陣列的用途](../interview/interview-frontend.md#q1)
- [Q2: useEffect cleanup function](../interview/interview-frontend.md#q2)
- [Q3: useEffect 執行兩次的原因](../interview/interview-frontend.md#q3)

---

## 3. useCallback - 函式記憶化

### 基本語法

```typescript
const memoizedCallback = useCallback(
	() => {
		// 函式邏輯
	},
	[dependencies] // 依賴陣列
)
```

### 💡 用途

**防止函式在每次渲染時重新建立**，特別用於：

1. 傳遞給子元件的 callback（避免不必要的重新渲染）
2. 作為其他 Hook 的依賴項目（如 useEffect）

### 🆚 Vue vs React

Vue 沒有直接對應的概念，因為 Vue 的響應式系統會自動優化。React 需要手動優化。

### 專案實例

```typescript
// 檔案位置: contexts/auth-context.tsx (第 57-61 行)
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)

	// ✅ 使用 useCallback 記憶化函式
	const login = useCallback((token: string, userData: UserInfo) => {
		saveToken(token)
		setUserInfo(userData)
		setUser(userData)
	}, []) // 空依賴：函式內不依賴任何 props 或 state

	const logout = useCallback(async () => {
		try {
			await authApi.logout()
		} finally {
			removeToken()
			removeUserInfo()
			setUser(null)
		}
	}, []) // 空依賴：所有函式都是穩定的

	// 這些函式可以安全地傳遞給子元件或用於 useEffect 依賴
}
```

```typescript
// 檔案位置: hooks/use-video-progress.ts (第 37-42 行)
export function useVideoProgress(options: UseVideoProgressOptions) {
	const { onProgressUpdate } = options

	const saveCurrentProgress = useCallback(() => {
		const currentTime = playerRef.current?.getCurrentTime()
		if (!currentTime || currentTime <= 0) return

		onProgressUpdate(Math.floor(currentTime))
	}, [onProgressUpdate]) // ✅ 依賴 onProgressUpdate

	// saveCurrentProgress 可以安全地用在其他 useEffect 中
	useEffect(() => {
		window.addEventListener('beforeunload', saveCurrentProgress)
		return () => window.removeEventListener('beforeunload', saveCurrentProgress)
	}, [saveCurrentProgress]) // 不會造成無限迴圈
}
```

### ⚠️ 常見陷阱

1. **忘記加依賴，導致閉包問題**

   ```typescript
   // ❌ 錯誤：永遠使用舊的 count 值
   const [count, setCount] = useState(0)
   const increment = useCallback(() => {
   	setCount(count + 1) // 閉包，count 永遠是 0
   }, []) // 沒加 count 到依賴

   // ✅ 正確：使用函式形式更新
   const increment = useCallback(() => {
   	setCount((prev) => prev + 1) // 不依賴外部 count
   }, [])
   ```

2. **過度使用 useCallback**

   ```typescript
   // ❌ 不必要：簡單的內部函式不需要 useCallback
   const handleClick = useCallback(() => {
   	console.log('clicked')
   }, [])

   // ✅ 只在這些情況才需要：
   // 1. 傳遞給經過 memo() 優化的子元件
   // 2. 作為其他 Hook 的依賴項目
   ```

### 📝 相關面試題

- [Q4: useCallback 的用途](../interview/interview-frontend.md#q4)
- [Q5: useCallback vs useMemo](../interview/interview-frontend.md#q5)

---

## 4. useMemo - 值記憶化

### 基本語法

```typescript
const memoizedValue = useMemo(
	() => computeExpensiveValue(a, b),
	[a, b] // 依賴陣列
)
```

### 💡 用途

**記憶化計算結果**，避免在每次渲染時重新計算昂貴的運算。

### 🆚 Vue vs React

| Vue 3                                            | React                                              |
| ------------------------------------------------ | -------------------------------------------------- |
| `const double = computed(() => count.value * 2)` | `const double = useMemo(() => count * 2, [count])` |
| 自動追蹤依賴                                     | 手動聲明依賴                                       |

### 專案實例

```typescript
// 檔案位置: contexts/auth-context.tsx (第 86-93 行)
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)
	const [isLoading, setIsLoading] = useState(true)

	// ✅ 使用 useMemo 避免每次渲染都建立新物件
	const value: AuthContextType = useMemo(
		() => ({
			user,
			isAuthenticated: !!user, // 計算屬性
			isLoading,
			login,
			logout,
			updateUser
		}),
		[user, isLoading, login, logout, updateUser]
	)
	// 只有當依賴改變時才重新建立 value 物件

	return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}
```

### 實際範例：避免不必要的過濾計算

```typescript
function MissionList({ missions, filter }) {
	// ✅ 只在 missions 或 filter 改變時重新過濾
	const filteredMissions = useMemo(() => {
		console.log('重新計算過濾結果')
		return missions.filter((m) => m.type === filter)
	}, [missions, filter])

	return (
		<div>
			{filteredMissions.map((m) => (
				<Mission key={m.id} {...m} />
			))}
		</div>
	)
}
```

### ⚠️ 常見陷阱

1. **過度使用 useMemo**

   ```typescript
   // ❌ 不必要：簡單運算不需要 memo
   const doubled = useMemo(() => count * 2, [count])

   // ✅ 直接計算即可
   const doubled = count * 2
   ```

2. **用 useMemo 取代 useCallback**

   ```typescript
   // ❌ 可以，但不語義化
   const onClick = useMemo(() => () => console.log('click'), [])

   // ✅ 更清楚：使用 useCallback
   const onClick = useCallback(() => console.log('click'), [])
   ```

### 💡 何時使用 useMemo？

- ✅ 昂貴的運算（複雜的陣列操作、排序、過濾）
- ✅ 建立物件/陣列並傳遞給 Context Provider
- ✅ 避免子元件不必要的重新渲染
- ❌ 簡單的計算（加減乘除、字串拼接）

### 📝 相關面試題

- [Q5: useCallback vs useMemo](../interview/interview-frontend.md#q5)
- [Q6: useMemo 優化效能](../interview/interview-frontend.md#q6)

---

## 5. useRef - 可變引用

### 基本語法

```typescript
const ref = useRef(initialValue)
// ref.current 可以直接修改，不會觸發重新渲染
```

### 💡 用途

1. **存取 DOM 元素**（類似 Vue 的 template ref）
2. **儲存可變值**（不需要觸發渲染的變數）
3. **保存前一次的值**

### 🆚 Vue vs React

| Vue 3                                      | React                                             |
| ------------------------------------------ | ------------------------------------------------- |
| `const inputRef = ref<HTMLInputElement>()` | `const inputRef = useRef<HTMLInputElement>(null)` |
| `<input ref="inputRef">`                   | `<input ref={inputRef}>`                          |
| `inputRef.value?.focus()`                  | `inputRef.current?.focus()`                       |

**關鍵差異**：Vue 的 `ref()` 是響應式的，React 的 `useRef()` 修改不會觸發重新渲染。

### 專案實例

#### 範例 1: 存取 DOM 元素

```typescript
// 檔案位置: hooks/use-video-progress.ts (第 32-34 行)
export function useVideoProgress(options: UseVideoProgressOptions) {
	const [isCompleted, setIsCompleted] = useState(false)
	const completedRef = useRef(false) // ✅ 用來防止重複呼叫
	const playerRef = useRef<YouTubeEvent['target'] | null>(null) // ✅ 儲存播放器實例
	const progressIntervalRef = useRef<NodeJS.Timeout | null>(null) // ✅ 儲存計時器 ID

	// 取得播放器當前時間
	const saveCurrentProgress = useCallback(() => {
		const currentTime = playerRef.current?.getCurrentTime() // 存取 ref
		if (!currentTime || currentTime <= 0) return
		onProgressUpdate(Math.floor(currentTime))
	}, [onProgressUpdate])
}
```

#### 範例 2: 防止重複執行

```typescript
// 檔案位置: hooks/use-video-progress.ts (第 90-103 行)
const handleEnd = useCallback(() => {
	stopProgressTracking()

	// ✅ 使用 ref 防止重複觸發完成事件
	if (completedRef.current) return // 已經完成，不重複執行

	completedRef.current = true // 標記為已完成（不觸發渲染）
	setIsCompleted(true) // 更新 UI 狀態（觸發渲染）

	onProgressUpdate(durationSeconds)
	onComplete()
}, [stopProgressTracking, durationSeconds, onProgressUpdate, onComplete])
```

### 實際範例：儲存前一次的值

```typescript
function Counter() {
	const [count, setCount] = useState(0)
	const prevCountRef = useRef<number>()

	useEffect(() => {
		prevCountRef.current = count // 儲存當前值
	})

	const prevCount = prevCountRef.current // 讀取前一次的值
	return (
		<div>
			現在: {count}, 之前: {prevCount}
		</div>
	)
}
```

### ⚠️ 常見陷阱

1. **把 useRef 當作 useState 用**

   ```typescript
   // ❌ 錯誤：修改 ref 不會觸發重新渲染
   const countRef = useRef(0)
   const increment = () => {
   	countRef.current++ // 改了但 UI 不會更新！
   }

   // ✅ 需要觸發渲染就用 useState
   const [count, setCount] = useState(0)
   ```

2. **在渲染階段修改 ref**

   ```typescript
   // ❌ 錯誤：在渲染時修改 ref
   function Component() {
   	const ref = useRef(0)
   	ref.current++ // 不應該在這裡修改
   	return <div>{ref.current}</div>
   }

   // ✅ 正確：在 useEffect 或事件處理中修改
   useEffect(() => {
   	ref.current++
   })
   ```

### 💡 useRef vs useState

|            | useRef               | useState           |
| ---------- | -------------------- | ------------------ |
| **修改時** | 不觸發渲染           | 觸發渲染           |
| **用途**   | 儲存可變值、DOM 引用 | UI 狀態            |
| **何時用** | 不需要顯示在 UI 的值 | 需要顯示在 UI 的值 |

### 📝 相關面試題

- [Q8: useRef 的使用情境](../interview/interview-frontend.md#q8)
- [Q9: useRef vs useState](../interview/interview-frontend.md#q9)

---

## 6. Custom Hooks - 邏輯復用

### 💡 概念

Custom Hook 就是一個**以 `use` 開頭的函式**，內部可以使用其他 Hooks。用來抽取和復用元件邏輯。

### 🆚 Vue vs React

| Vue 3                          | React                          |
| ------------------------------ | ------------------------------ |
| Composable functions           | Custom Hooks                   |
| `export function useCounter()` | `export function useCounter()` |
| 可以回傳任何值                 | 可以回傳任何值                 |

概念非常相似！

### 專案實例

#### 範例 1: 影片進度追蹤 Hook

```typescript
// 檔案位置: hooks/use-video-progress.ts (完整檔案)
export function useVideoProgress({
	initialProgress,
	durationSeconds,
	onProgressUpdate,
	onComplete
}: UseVideoProgressOptions): UseVideoProgressReturn {
	// ✅ 內部使用多個 Hooks
	const [isCompleted, setIsCompleted] = useState(false)
	const completedRef = useRef(false)
	const playerRef = useRef<YouTubeEvent['target'] | null>(null)
	const progressIntervalRef = useRef<NodeJS.Timeout | null>(null)

	// 封裝複雜的邏輯
	const saveCurrentProgress = useCallback(() => {
		const currentTime = playerRef.current?.getCurrentTime()
		if (!currentTime || currentTime <= 0) return
		onProgressUpdate(Math.floor(currentTime))
	}, [onProgressUpdate])

	const startProgressTracking = useCallback(() => {
		if (progressIntervalRef.current) {
			clearInterval(progressIntervalRef.current)
		}
		progressIntervalRef.current = setInterval(saveCurrentProgress, PROGRESS_UPDATE_INTERVAL_MS)
	}, [saveCurrentProgress])

	// ... 更多邏輯

	// ✅ 回傳狀態和處理函式
	return {
		isCompleted,
		playerHandlers: {
			onReady: handleReady,
			onPlay: handlePlay,
			onPause: handlePause,
			onEnd: handleEnd
		}
	}
}
```

#### 範例 2: 使用 Custom Hook

```typescript
// 在元件中使用
function MissionVideo({ missionId }) {
	// ✅ 呼叫 custom hook，取得封裝好的邏輯
	const { isCompleted, playerHandlers } = useVideoProgress({
		initialProgress: 0,
		durationSeconds: 600,
		onProgressUpdate: (time) => console.log('進度:', time),
		onComplete: () => console.log('完成！')
	})

	return (
		<div>
			<YouTube {...playerHandlers} />
			{isCompleted && <div>✅ 已完成</div>}
		</div>
	)
}
```

#### 範例 3: 整合多個 Contexts 的 Hook

```typescript
// 檔案位置: hooks/use-mission.ts (第 24-40 行)
export function useMission(): UseMissionReturn {
	const params = useParams()
	const router = useRouter()

	// ✅ 整合多個 contexts
	const { user, isAuthenticated, isLoading: authLoading } = useAuth()
	const { updateMissionStatus } = useJourney()
	const { hasPurchased, isLoading: purchaseLoading } = useUserPurchase()

	// 自己的 state
	const [mission, setMission] = useState<MissionDetail | null>(null)
	const [progress, setProgress] = useState<UserMissionProgress | null>(null)
	const [isLoading, setIsLoading] = useState(true)

	// 複雜的副作用邏輯
	useEffect(() => {
		if (!authLoading && !isAuthenticated) {
			router.push('/login')
		}
	}, [authLoading, isAuthenticated, router])

	// ... 更多邏輯

	return {
		mission,
		progress,
		isLoading,
		handleProgressUpdate,
		handleVideoComplete,
		handleDeliverMission
	}
}
```

### 💡 Custom Hook 設計原則

1. **以 `use` 開頭命名**（必須！）

   ```typescript
   // ✅ 正確
   function useFormData() { ... }

   // ❌ 錯誤：ESLint 會報錯
   function getFormData() {
     const [data, setData] = useState()  // Hook 只能在 use 開頭的函式中
   }
   ```

2. **回傳需要的狀態和函式**

   ```typescript
   // ✅ 好的設計：清楚的回傳值
   function useCounter(initial: number) {
   	const [count, setCount] = useState(initial)
   	const increment = () => setCount((c) => c + 1)
   	const decrement = () => setCount((c) => c - 1)

   	return { count, increment, decrement }
   }
   ```

3. **封裝複雜邏輯**
   - 整合多個 API 呼叫
   - 處理複雜的狀態邏輯
   - 整合多個 contexts

### ⚠️ 常見陷阱

1. **在非 Hook 函式中呼叫 Hook**

   ```typescript
   // ❌ 錯誤
   function fetchData() {
   	const [data, setData] = useState() // 違反 Hook 規則
   }

   // ✅ 正確：改名為 use 開頭
   function useFetchData() {
   	const [data, setData] = useState()
   	// ...
   }
   ```

2. **條件式呼叫 Hook**

   ```typescript
   // ❌ 錯誤：Hook 不能在條件式中呼叫
   if (shouldFetch) {
   	const data = useFetchData()
   }

   // ✅ 正確：把條件放在 Hook 內部
   const data = useFetchData(shouldFetch)
   ```

### 📝 相關面試題

- [Q10: Custom Hook 的設計原則](../interview/interview-frontend.md#q10)
- [Q11: Custom Hook vs 一般函式](../interview/interview-frontend.md#q11)
- [Q12: Hook 規則](../interview/interview-frontend.md#q12)

---

## 🎯 總結

### React Hooks 核心規則

1. **只能在 React 函式元件或 Custom Hook 中呼叫**
2. **只能在最頂層呼叫**（不能在迴圈、條件式、巢狀函式中）
3. **Custom Hook 必須以 `use` 開頭命名**

### 快速對照表

| Hook          | 用途          | Vue 對應                 |
| ------------- | ------------- | ------------------------ |
| `useState`    | 管理狀態      | `ref()`, `reactive()`    |
| `useEffect`   | 處理副作用    | `watch()`, `onMounted()` |
| `useCallback` | 記憶化函式    | 無（不需要）             |
| `useMemo`     | 記憶化值      | `computed()`             |
| `useRef`      | 可變引用、DOM | `ref()`（DOM）           |
| Custom Hook   | 邏輯復用      | Composable               |

### 給 Vue 開發者的重點提醒

1. **依賴陣列不會自動追蹤**：必須手動列出所有依賴
2. **狀態不可變**：不能直接修改 state，必須呼叫 setter
3. **useCallback/useMemo 是必要的**：React 沒有自動優化
4. **useRef 修改不觸發渲染**：和 Vue 的 `ref()` 不同

---

## 🚀 下一步

- 學習 [TypeScript + React 型別技巧](./02-typescript-react.md)
- 深入 [Context API 與狀態管理](./03-context-api.md)

---

## 📖 延伸閱讀

- [React Hooks 官方文件（中文）](https://zh-hant.react.dev/reference/react)
- [Hook 規則](https://zh-hant.react.dev/warnings/invalid-hook-call-warning)
- [自訂 Hook](https://zh-hant.react.dev/learn/reusing-logic-with-custom-hooks)

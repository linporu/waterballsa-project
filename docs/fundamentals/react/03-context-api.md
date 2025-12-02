# Context API 與狀態管理

> 專為熟悉 Vue provide/inject 的開發者設計

React Context API 是 React 的內建狀態管理方案，類似 Vue 的 `provide/inject`，用於跨元件共享狀態而不需要逐層傳遞 props。

📝 **對應面試題**：Q7, Q21-Q30

---

## 目錄

1. [Context 基本模式](#1-context-基本模式)
2. [Provider 組織架構](#2-provider-組織架構)
3. [Context 間的依賴關係](#3-context-間的依賴關係)
4. [效能優化](#4-效能優化)
5. [最佳實踐](#5-最佳實踐)

---

## 1. Context 基本模式

### 🆚 Vue vs React

| Vue 3                         | React                               |
| ----------------------------- | ----------------------------------- |
| `provide('key', value)`       | `<Context.Provider value={value}>`  |
| `const value = inject('key')` | `const value = useContext(Context)` |
| 自動型別推導                  | 需手動定義型別                      |

### 基本結構

React Context 通常包含四個部分：

```typescript
// 1️⃣ 定義 Context 型別
interface MyContextType {
	value: string
	setValue: (v: string) => void
}

// 2️⃣ 建立 Context
const MyContext = createContext<MyContextType | undefined>(undefined)

// 3️⃣ 建立 Provider 元件
export function MyProvider({ children }: { children: ReactNode }) {
	const [value, setValue] = useState('')

	return <MyContext.Provider value={{ value, setValue }}>{children}</MyContext.Provider>
}

// 4️⃣ 建立 custom hook（型別安全）
export function useMyContext() {
	const context = useContext(MyContext)
	if (context === undefined) {
		throw new Error('useMyContext must be used within MyProvider')
	}
	return context
}
```

### 專案實例：Auth Context

```typescript
// 檔案位置: contexts/auth-context.tsx (第 22-104 行)

// 1️⃣ 定義 Context 型別
interface AuthContextType {
	user: UserInfo | null
	isAuthenticated: boolean
	isLoading: boolean
	login: (token: string, user: UserInfo) => void
	logout: () => Promise<void>
	updateUser: (user: UserInfo) => void
}

// 2️⃣ 建立 Context（初始值 undefined）
const AuthContext = createContext<AuthContextType | undefined>(undefined)

// 3️⃣ Provider 元件
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)
	const [isLoading, setIsLoading] = useState(true)

	// 初始化：檢查是否有 token
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
	}, [])

	// ✅ 使用 useCallback 記憶化函式
	const login = useCallback((token: string, userData: UserInfo) => {
		saveToken(token)
		setUserInfo(userData)
		setUser(userData)
	}, [])

	const logout = useCallback(async () => {
		try {
			await authApi.logout()
		} finally {
			removeToken()
			removeUserInfo()
			setUser(null)
		}
	}, [])

	const updateUser = useCallback((userData: UserInfo) => {
		setUserInfo(userData)
		setUser(userData)
	}, [])

	// ✅ 封裝成物件傳入 Provider
	const value: AuthContextType = {
		user,
		isAuthenticated: !!user,
		isLoading,
		login,
		logout,
		updateUser
	}

	return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}

// 4️⃣ Custom hook（型別安全 + 執行時檢查）
export function useAuth() {
	const context = useContext(AuthContext)
	if (context === undefined) {
		throw new Error('useAuth must be used within an AuthProvider')
	}
	return context
}
```

### 使用範例

```typescript
// 在元件中使用
function ProfilePage() {
	// ✅ 一行解構取得所需的值和函式
	const { user, isAuthenticated, logout } = useAuth()

	if (!isAuthenticated) {
		return <LoginForm />
	}

	return (
		<div>
			<h1>Welcome, {user?.name}</h1>
			<button onClick={logout}>Logout</button>
		</div>
	)
}
```

### 📝 相關面試題

- [Q21: Context 的建立與使用](../interview/interview-frontend.md#q21)
- [Q22: useContext 型別安全](../interview/interview-frontend.md#q22)

---

## 2. Provider 組織架構

### 💡 多個 Provider 的組合

在實際專案中，通常會有多個 Context Providers。有兩種組織方式：

#### 方式 1: 巢狀結構

```typescript
// 檔案位置: app/layout.tsx (常見模式)
function RootLayout({ children }: { children: ReactNode }) {
	return (
		<html>
			<body>
				<AuthProvider>
					<UserPurchaseProvider>
						<JourneyProvider>{children}</JourneyProvider>
					</UserPurchaseProvider>
				</AuthProvider>
			</body>
		</html>
	)
}
```

#### 方式 2: 組合 Provider（推薦）

```typescript
// 建立組合 Provider
function AppProviders({ children }: { children: ReactNode }) {
	return (
		<AuthProvider>
			<UserPurchaseProvider>
				<JourneyProvider>{children}</JourneyProvider>
			</UserPurchaseProvider>
		</AuthProvider>
	)
}

// 使用時更簡潔
function RootLayout({ children }: { children: ReactNode }) {
	return (
		<html>
			<body>
				<AppProviders>{children}</AppProviders>
			</body>
		</html>
	)
}
```

### 💡 Provider 的順序很重要！

如果 Provider B 依賴 Provider A，則 A 必須在外層：

```typescript
// ✅ 正確：UserPurchaseProvider 依賴 AuthProvider
<AuthProvider>
  <UserPurchaseProvider>  {/* 內部使用 useAuth() */}
    {children}
  </UserPurchaseProvider>
</AuthProvider>

// ❌ 錯誤：會報錯
<UserPurchaseProvider>
  <AuthProvider>
    {children}  {/* useAuth() 找不到 Provider！ */}
  </AuthProvider>
</UserPurchaseProvider>
```

### 📝 相關面試題

- [Q23: 多個 Provider 的組合](../interview/interview-frontend.md#q23)

---

## 3. Context 間的依賴關係

### 專案實例：UserPurchaseProvider 依賴 AuthProvider

```typescript
// 檔案位置: contexts/user-purchase-context.tsx (第 40-80 行)

export function UserPurchaseProvider({ children }: UserPurchaseProviderProps) {
	// ✅ 依賴 AuthProvider 提供的資料
	const { user, isAuthenticated } = useAuth()

	const [purchasedJourneyIds, setPurchasedJourneyIds] = useState<Set<number>>(new Set())
	const [unpaidOrders, setUnpaidOrders] = useState<OrderSummary[]>([])
	const [isLoading, setIsLoading] = useState(false)
	const [isRefreshing, setIsRefreshing] = useState(false)

	// 當使用者登入/登出時，重新載入購買資料
	const fetchUserPurchaseData = useCallback(
		async (showLoading = true) => {
			if (!user || !isAuthenticated) {
				// ✅ 未登入時清除資料
				setPurchasedJourneyIds(new Set())
				setUnpaidOrders([])
				return
			}

			try {
				if (showLoading) {
					setIsLoading(true)
				} else {
					setIsRefreshing(true)
				}

				const userId = parseInt(user.id)

				// ✅ 平行呼叫兩個 API（效能優化）
				const [purchasesResult, ordersResult] = await Promise.all([
					userPurchaseApi.getUserPurchasedJourneys(userId),
					userPurchaseApi.getUserOrders(userId, {
						page: 1,
						limit: 50,
						status: 'UNPAID'
					})
				])

				// 處理結果...
				if (purchasesResult.success) {
					const journeyIds = new Set(purchasesResult.data.map((p) => p.journeyId))
					setPurchasedJourneyIds(journeyIds)
				}

				if (ordersResult.success) {
					setUnpaidOrders(ordersResult.data.items)
				}
			} finally {
				setIsLoading(false)
				setIsRefreshing(false)
			}
		},
		[user, isAuthenticated] // ✅ 依賴 user 和 isAuthenticated
	)

	// ✅ 當 user 或 isAuthenticated 改變時，重新載入資料
	useEffect(() => {
		fetchUserPurchaseData()
	}, [fetchUserPurchaseData])

	// ...
}
```

### 專案實例：JourneyProvider 的獨立設計

```typescript
// 檔案位置: contexts/journey-context.tsx (第 28-80 行)

export function JourneyProvider({ children }: JourneyProviderProps) {
	const [journey, setJourney] = useState<JourneyDetail | null>(null)
	const [isLoading, setIsLoading] = useState(false)
	const [error, setError] = useState<string | null>(null)

	// ✅ 透過參數接收 userId，而非直接依賴 AuthContext
	const fetchJourney = useCallback(async (slug: string, userId?: string) => {
		if (!slug) return

		try {
			setIsLoading(true)
			setError(null)

			const result = await journeyApi.getJourneyBySlug(slug)
			if (result.success) {
				let journeyData = result.data

				// 如果有提供 userId，載入所有任務的進度
				if (userId) {
					const allMissionIds = journeyData.chapters.flatMap((chapter) =>
						chapter.missions.map((mission) => mission.id)
					)

					// ✅ 平行載入所有任務進度
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

					// ✅ 不可變更新：建立新物件而非修改原物件
					journeyData = {
						...journeyData,
						chapters: journeyData.chapters.map((chapter) => ({
							...chapter,
							missions: chapter.missions.map((mission) => ({
								...mission,
								status: statusMap.get(mission.id) ?? mission.status
							}))
						}))
					}
				}

				setJourney(journeyData)
			} else {
				setError(result.error?.message || 'Failed to load journey')
			}
		} catch (err) {
			setError(err instanceof Error ? err.message : 'An error occurred')
		} finally {
			setIsLoading(false)
		}
	}, [])

	// ...
}
```

### 💡 Context 依賴的設計策略

| 策略         | 優點           | 缺點             | 適用情境                           |
| ------------ | -------------- | ---------------- | ---------------------------------- |
| **直接依賴** | 簡單、自動同步 | 耦合度高、難測試 | 強相依的資料（如購買資料依賴登入） |
| **參數注入** | 解耦、易測試   | 需手動傳遞       | 弱相依或可選的資料                 |

### 📝 相關面試題

- [Q24: Context 間的依賴處理](../interview/interview-frontend.md#q24)
- [Q25: Promise.all 平行呼叫](../interview/interview-frontend.md#q25)

---

## 4. 效能優化

### 問題：Context 更新導致所有子元件重新渲染

```typescript
// ❌ 問題範例
function MyProvider({ children }: { children: ReactNode }) {
	const [count, setCount] = useState(0)
	const [name, setName] = useState('')

	// 每次 count 或 name 改變，value 都會是新物件
	const value = {
		count,
		name,
		setCount,
		setName
	}

	return <MyContext.Provider value={value}>{children}</MyContext.Provider>
}

// 使用 MyContext 的所有子元件都會重新渲染，即使只用到 name
```

### 解法 1: useMemo 記憶化 value

```typescript
// ✅ 使用 useMemo
function MyProvider({ children }: { children: ReactNode }) {
	const [count, setCount] = useState(0)
	const [name, setName] = useState('')

	// ✅ 只有依賴改變時才建立新物件
	const value = useMemo(
		() => ({
			count,
			name,
			setCount,
			setName
		}),
		[count, name] // setCount 和 setName 是穩定的，可以不加
	)

	return <MyContext.Provider value={value}>{children}</MyContext.Provider>
}
```

### 解法 2: useCallback 記憶化函式

```typescript
// ✅ 專案實例：contexts/auth-context.tsx
export function AuthProvider({ children }: AuthProviderProps) {
	const [user, setUser] = useState<UserInfo | null>(null)

	// ✅ useCallback 確保函式不會在每次渲染時重新建立
	const login = useCallback((token: string, userData: UserInfo) => {
		saveToken(token)
		setUserInfo(userData)
		setUser(userData)
	}, []) // 空依賴：函式不依賴任何外部變數

	const logout = useCallback(async () => {
		try {
			await authApi.logout()
		} finally {
			removeToken()
			removeUserInfo()
			setUser(null)
		}
	}, [])

	const value: AuthContextType = {
		user,
		isAuthenticated: !!user,
		isLoading,
		login, // ✅ 穩定的函式引用
		logout, // ✅ 穩定的函式引用
		updateUser
	}

	return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}
```

### 解法 3: 分割 Context（進階）

如果 Context 包含多個不相關的資料，考慮分割：

```typescript
// ❌ 單一 Context 包含不相關資料
interface AppContextType {
	user: User | null
	theme: 'light' | 'dark'
	locale: string
}
// 問題：theme 改變會導致使用 user 的元件也重新渲染

// ✅ 分割成獨立的 Contexts
;<UserProvider>
	<ThemeProvider>
		<LocaleProvider>{children}</LocaleProvider>
	</ThemeProvider>
</UserProvider>
```

### 📝 相關面試題

- [Q26: Context 效能優化](../interview/interview-frontend.md#q26)
- [Q27: useMemo 在 Context 中的應用](../interview/interview-frontend.md#q27)
- [Q28: useCallback 避免重新渲染](../interview/interview-frontend.md#q28)

---

## 5. 最佳實踐

### ✅ DO：必須做的

1. **永遠建立 custom hook**

   ```typescript
   // ✅ 好
   export function useAuth() {
   	const context = useContext(AuthContext)
   	if (context === undefined) {
   		throw new Error('useAuth must be used within an AuthProvider')
   	}
   	return context
   }

   // ❌ 差
   export { AuthContext } // 讓使用者直接用 useContext(AuthContext)
   ```

2. **用 useCallback 包裝 Context 中的函式**

   ```typescript
   // ✅ 好
   const login = useCallback((token: string, user: UserInfo) => {
   	saveToken(token)
   	setUser(user)
   }, [])

   // ❌ 差：每次渲染都是新函式
   const login = (token: string, user: UserInfo) => {
   	saveToken(token)
   	setUser(user)
   }
   ```

3. **用 useMemo 包裝 Context value（如果包含計算值）**

   ```typescript
   // ✅ 好
   const value = useMemo(
   	() => ({
   		user,
   		isAuthenticated: !!user,
   		login,
   		logout
   	}),
   	[user, login, logout]
   )

   // ❌ 可接受但不理想：每次渲染都建立新物件
   const value = {
   	user,
   	isAuthenticated: !!user,
   	login,
   	logout
   }
   ```

### ❌ DON'T：避免的做法

1. **不要在 Context 中存放頻繁變化的值**

   ```typescript
   // ❌ 差：滑鼠位置每幀都改變
   function MouseProvider({ children }) {
   	const [mousePos, setMousePos] = useState({ x: 0, y: 0 })

   	useEffect(() => {
   		const handleMouseMove = (e) => setMousePos({ x: e.clientX, y: e.clientY })
   		window.addEventListener('mousemove', handleMouseMove)
   		return () => window.removeEventListener('mousemove', handleMouseMove)
   	}, [])

   	return <MouseContext.Provider value={mousePos}>{children}</MouseContext.Provider>
   }
   // 所有子元件會瘋狂重新渲染！

   // ✅ 好：用 ref 或事件訂閱模式
   ```

2. **不要過度使用 Context**

   ```typescript
   // ❌ 差：只有父子關係，用 props 就好
   <ButtonGroupContext.Provider value={{ variant }}>
     <Button />
   </ButtonGroupContext.Provider>

   // ✅ 好：直接傳 props
   <Button variant={variant} />
   ```

3. **不要讓 Context 包含太多不相關的資料**

   ```typescript
   // ❌ 差：大雜燴 Context
   interface AppContextType {
   	user: User
   	theme: Theme
   	todos: Todo[]
   	settings: Settings
   	// ... 更多不相關的資料
   }

   // ✅ 好：按功能分割
   ;<UserProvider>
   	<ThemeProvider>
   		<TodoProvider>{children}</TodoProvider>
   	</ThemeProvider>
   </UserProvider>
   ```

### 💡 何時使用 Context？

✅ **應該用 Context**：

- 使用者驗證狀態（user, login, logout）
- 主題設定（theme, toggleTheme）
- 語言/國際化（locale, t）
- 全域 toast/notification
- 購物車狀態

❌ **不該用 Context**：

- 父子元件間的簡單資料傳遞（用 props）
- 伺服器端資料快取（用 TanStack Query / SWR）
- 頻繁變化的資料（如動畫、滑鼠位置）
- 表單狀態（用 React Hook Form）

### 📝 相關面試題

- [Q29: Context 的使用時機](../interview/interview-frontend.md#q29)
- [Q30: Context vs Props vs State Management Library](../interview/interview-frontend.md#q30)

---

## 🎯 總結

### Context API 核心要點

| 概念              | 說明                    | Vue 對應           |
| ----------------- | ----------------------- | ------------------ |
| **createContext** | 建立 Context            | 無直接對應         |
| **Provider**      | 提供資料給子元件        | `provide()`        |
| **useContext**    | 消費 Context 資料       | `inject()`         |
| **Custom Hook**   | 型別安全的 Context hook | 無需要（自動推導） |

### React Context vs Vue Provide/Inject

| 特性              | React Context                               | Vue Provide/Inject               |
| ----------------- | ------------------------------------------- | -------------------------------- |
| **型別安全**      | 需手動設定（custom hook + undefined check） | 自動推導                         |
| **效能優化**      | 需手動 useMemo/useCallback                  | 自動優化                         |
| **巢狀 Provider** | 內層會覆蓋外層                              | 內層會覆蓋外層                   |
| **除錯**          | React DevTools 顯示 Provider 樹             | Vue DevTools 顯示 provide/inject |

### 記憶口訣

1. **Context 四步驟**：定義型別 → createContext → Provider 元件 → custom hook
2. **依賴順序**：被依賴的 Provider 在外層
3. **效能優化**：useMemo value + useCallback 函式
4. **型別安全**：undefined + throw error 模式

---

## 🚀 下一步

- 學習 [表單處理](./04-form-handling.md)
- 深入 [非同步處理模式](./05-async-patterns.md)

---

## 📖 延伸閱讀

- [React Context 官方文件（中文）](https://zh-hant.react.dev/reference/react/createContext)
- [useContext Hook](https://zh-hant.react.dev/reference/react/useContext)
- [Context 效能優化](https://zh-hant.react.dev/reference/react/memo)

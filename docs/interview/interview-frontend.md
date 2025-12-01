# 前端面試題 - 技術細節考核

## 說明

本文件包含 50 道前端技術問題，基於 Waterball Software Academy 課程平台重製專案的實際程式碼。

**面試目標**：考核對專案實作的理解，特別是 React Hooks、狀態管理、API 整合、TypeScript 的細節掌握。

**難度分布**：Junior (1/3) → Mid-level (2/3)

---

## 一、React Hooks 與語法細節 (20 題)

### Q1. useEffect 依賴陣列

在 `contexts/auth-context.tsx` 中的初始化 useEffect：

```tsx
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
```

**問題**：為什麼這個 useEffect 的依賴陣列是空的？如果我們在裡面使用了 `getToken()` 和 `getUserInfo()`，為什麼不需要把它們加到依賴陣列中？

---

### Q2. useCallback 的使用時機

在 `contexts/auth-context.tsx` 中：

```tsx
const login = useCallback((token: string, userData: UserInfo) => {
	saveToken(token)
	setUserInfo(userData)
	setUser(userData)
}, [])

const logout = useCallback(async () => {
	try {
		await authApi.logout()
	} catch (error) {
		console.error('Logout API call failed:', error)
	} finally {
		removeToken()
		removeUserInfo()
		setUser(null)
	}
}, [])
```

**問題**：

1. 為什麼 `login` 和 `logout` 需要用 `useCallback` 包裝？
2. 這兩個函式的依賴陣列都是空的，這樣安全嗎？為什麼不需要把 `setUser` 加入依賴？

---

### Q3. useRef 保存可變值

在 `hooks/use-video-progress.ts` 中：

```tsx
const [isCompleted, setIsCompleted] = useState(false)
const completedRef = useRef(false)
const playerRef = useRef<YouTubeEvent['target'] | null>(null)
const progressIntervalRef = useRef<NodeJS.Timeout | null>(null)

const handleEnd = useCallback(() => {
	stopProgressTracking()

	// Prevent duplicate completion calls
	if (completedRef.current) return

	completedRef.current = true
	setIsCompleted(true)

	onProgressUpdate(durationSeconds)
	onComplete()
}, [stopProgressTracking, durationSeconds, onProgressUpdate, onComplete])
```

**問題**：

1. 為什麼需要同時使用 `isCompleted` state 和 `completedRef` ref？
2. `completedRef` 解決了什麼問題？
3. 如果只用 `isCompleted` state 會發生什麼事？

---

### Q4. 清理 Effect

在 `hooks/use-video-progress.ts` 中：

```tsx
useEffect(() => {
	window.addEventListener('beforeunload', saveCurrentProgress)
	return () => {
		window.removeEventListener('beforeunload', saveCurrentProgress)
		stopProgressTracking()
	}
}, [saveCurrentProgress, stopProgressTracking])
```

**問題**：

1. 為什麼需要在 return 的 cleanup function 中移除事件監聽器？
2. `stopProgressTracking()` 在這裡的作用是什麼？
3. 如果忘記寫 cleanup function 會導致什麼問題？

---

### Q5. Custom Hook 的設計

看 `hooks/use-video-progress.ts` 的整體結構：

```tsx
export function useVideoProgress({
	initialProgress,
	durationSeconds,
	onProgressUpdate,
	onComplete
}: UseVideoProgressOptions): UseVideoProgressReturn {
	// ... internal states and logic

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

**問題**：

1. 為什麼要把 player handlers 包在一個物件裡回傳，而不是直接回傳四個獨立的函式？
2. 這樣的設計有什麼優點？

---

### Q6. 複雜的 useEffect 依賴

在 `hooks/use-mission.ts` 中：

```tsx
useEffect(() => {
	async function fetchMissionData() {
		if (!user) return
		// ... fetch logic
	}

	if (user && !authLoading) {
		fetchMissionData()
	}
}, [user, authLoading, journeySlug, missionId, updateMissionStatus, hasPurchased])
```

**問題**：

1. 為什麼要在 useEffect 外面檢查 `if (user && !authLoading)`，又在函式裡面檢查 `if (!user)`？
2. `updateMissionStatus` 和 `hasPurchased` 是函式，為什麼要加入依賴陣列？
3. 如果遺漏了 `journeySlug` 或 `missionId` 會發生什麼事？

---

### Q7. Context Provider 的記憶化

在 `contexts/auth-context.tsx` 中：

```tsx
const value: AuthContextType = {
	user,
	isAuthenticated: !!user,
	isLoading,
	login,
	logout,
	updateUser
}

return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
```

**問題**：

1. 這裡的 `value` 物件每次 render 都會重新建立，這會導致所有使用 `useAuth()` 的元件都重新渲染嗎？
2. 如果會，我們應該怎麼優化？如果不會，為什麼？
3. 什麼時候需要用 `useMemo` 包裝 Context value？

---

### Q8. State 更新的不可變性

在 `contexts/journey-context.tsx` 中：

```tsx
const updateMissionStatus = useCallback((missionId: number, status: MissionStatus) => {
	setJourney((prev) => {
		if (!prev) return null

		return {
			...prev,
			chapters: prev.chapters.map((chapter) => ({
				...chapter,
				missions: chapter.missions.map((mission) =>
					mission.id === missionId ? { ...mission, status } : mission
				)
			}))
		}
	})
}, [])
```

**問題**：

1. 為什麼要用這麼多層的 `map` 和 spread operator？
2. 如果我直接寫 `prev.chapters[0].missions[0].status = 'COMPLETED'` 會有什麼問題？
3. 這種深層更新的效能如何？有沒有更好的做法？

---

### Q9. useCallback 依賴的函式

在 `hooks/use-video-progress.ts` 中：

```tsx
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
```

**問題**：

1. `saveCurrentProgress` 依賴 `onProgressUpdate`，`startProgressTracking` 依賴 `saveCurrentProgress`，這種依賴鏈是否會導致問題？
2. 如果 `onProgressUpdate` 每次 render 都改變，會發生什麼？
3. 為什麼 `playerRef.current` 不需要加入依賴陣列？

---

### Q10. Custom Hook 的組合

在 `hooks/use-mission.ts` 中：

```tsx
export function useMission(): UseMissionReturn {
	const params = useParams()
	const router = useRouter()
	const { user, isAuthenticated, isLoading: authLoading } = useAuth()
	const { updateMissionStatus } = useJourney()
	const { hasPurchased, isLoading: purchaseLoading } = useUserPurchase()

	// ... more logic

	return {
		mission,
		progress,
		isLoading: authLoading || isLoading || purchaseLoading
		// ...
	}
}
```

**問題**：

1. 這個 hook 組合了多個其他 hooks，這樣的設計有什麼優缺點？
2. 如果 `useAuth` 或 `useJourney` 的實作改變，這個 hook 會受到什麼影響？
3. 為什麼要把三個 loading 狀態合併成一個回傳？

---

### Q11. useEffect 的執行時機

在 `hooks/use-mission.ts` 中有兩個 useEffect：

```tsx
// Effect 1: 重定向
useEffect(() => {
	if (!authLoading && !isAuthenticated) {
		router.push('/login')
	}
}, [authLoading, isAuthenticated, router])

// Effect 2: 載入資料
useEffect(() => {
	async function fetchMissionData() {
		if (!user) return
		// ... fetch logic
	}

	if (user && !authLoading) {
		fetchMissionData()
	}
}, [user, authLoading, journeySlug, missionId /* ... */])
```

**問題**：

1. 這兩個 effects 的執行順序是什麼？
2. 為什麼要分成兩個 useEffect 而不是合併成一個？
3. 第二個 effect 中的 `if (!user) return` 和外面的 `if (user && !authLoading)` 都是檢查 user，為什麼需要兩次檢查？

---

### Q12. 條件式資料載入

在 `contexts/user-purchase-context.tsx` 中：

```tsx
const fetchUserPurchaseData = useCallback(
	async (showLoading = true) => {
		if (!user || !isAuthenticated) {
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
			// ... fetch logic
		} finally {
			setIsLoading(false)
			setIsRefreshing(false)
		}
	},
	[user, isAuthenticated]
)
```

**問題**：

1. 為什麼要設計 `showLoading` 參數來區分 loading 和 refreshing 狀態？
2. `finally` block 中同時設定兩個狀態為 false，這會導致問題嗎？
3. 如果 fetch 失敗，為什麼註解說「Keep stale data - don't clear existing state」？

---

### Q13. useCallback 與閉包

在 `contexts/user-purchase-context.tsx` 中：

```tsx
const hasPurchased = useCallback(
	(journeyId: number): boolean => {
		return purchasedJourneyIds.has(journeyId)
	},
	[purchasedJourneyIds]
)
```

**問題**：

1. 如果 `purchasedJourneyIds` 是一個 Set，每次更新時都是新的 Set instance，這個 useCallback 還有用嗎？
2. 為什麼不直接在使用的地方寫 `purchasedJourneyIds.has(journeyId)`？
3. 把這個函式放在 Context value 中有什麼好處？

---

### Q14. Promise.all 的使用

在 `contexts/journey-context.tsx` 中：

```tsx
if (userId) {
	const allMissionIds = journeyData.chapters.flatMap((chapter) =>
		chapter.missions.map((mission) => mission.id)
	)

	const progressResults = await Promise.all(
		allMissionIds.map((missionId) => missionApi.getUserMissionProgress(parseInt(userId), missionId))
	)

	// Build missionId -> status map
	const statusMap = new Map<number, MissionStatus>()
	progressResults.forEach((progressResult, index) => {
		if (progressResult.success && progressResult.data.status) {
			statusMap.set(allMissionIds[index], progressResult.data.status)
		}
	})
}
```

**問題**：

1. 為什麼要用 `Promise.all` 而不是用 `for` loop 依序呼叫 API？
2. 如果其中一個 API call 失敗了，會發生什麼事？
3. 這種做法在課程有 100+ 單元時會有什麼問題？

---

### Q15. Custom Hook 的回傳型別

看 `hooks/use-api.ts`：

```tsx
export function useApi() {
	const { logout } = useAuth()
	const router = useRouter()

	const callApi = useCallback(
		async <T,>(apiCall: () => Promise<ApiResponse<T>>): Promise<ApiResponse<T>> => {
			const response = await apiCall()

			if (!response.success && response.error.status === 401) {
				logout()
				router.push('/login')
			}

			return response
		},
		[logout, router]
	)

	return { callApi }
}
```

**問題**：

1. `callApi` 函式中的泛型 `<T>` 是什麼意思？
2. 為什麼要設計成接受一個 `apiCall` 函式作為參數，而不是直接接受 URL 和 data？
3. 這個 hook 如何實現「自動處理 401」的功能？

---

### Q16. React Hook Form 整合

在 `components/auth/login-form.tsx` 中：

```tsx
const form = useForm<LoginFormValues>({
	resolver: zodResolver(loginSchema),
	defaultValues: {
		username: '',
		password: ''
	}
})

async function onSubmit(data: LoginFormValues) {
	setIsLoading(true)
	try {
		const result = await authApi.login(data)
		if (result.success) {
			login(result.data.accessToken, result.data.user)
			toast.success('登入成功！')
			router.push('/')
			return
		}
		toast.error(result.error.message)

		if (result.error.status === 401) {
			form.setError('username', { type: 'manual', message: ' ' })
			form.setError('password', {
				type: 'manual',
				message: '使用者名稱或密碼錯誤'
			})
		}
	} finally {
		setIsLoading(false)
	}
}
```

**問題**：

1. `zodResolver(loginSchema)` 的作用是什麼？
2. 為什麼在 401 錯誤時要對 `username` 設定空字串 message？
3. `form.setError` 和 Zod schema validation 有什麼不同？

---

### Q17. FormField 與 Controller

在 `components/auth/login-form.tsx` 中：

```tsx
<FormField
	control={form.control}
	name="username"
	render={({ field }) => (
		<FormItem>
			<FormLabel>使用者名稱</FormLabel>
			<FormControl>
				<Input
					placeholder="請輸入使用者名稱"
					autoComplete="username"
					{...field}
					disabled={isLoading}
				/>
			</FormControl>
			<FormMessage />
		</FormItem>
	)}
/>
```

**問題**：

1. `render` prop 中的 `field` 參數包含哪些屬性？
2. `{...field}` spread 到 Input 元件後，會自動處理哪些事情？
3. 為什麼不需要手動寫 `onChange` 和 `value`？

---

### Q18. useFormField 自訂 Hook

在 `components/ui/form.tsx` 中：

```tsx
const useFormField = () => {
	const fieldContext = React.useContext(FormFieldContext)
	const itemContext = React.useContext(FormItemContext)
	const { getFieldState } = useFormContext()
	const formState = useFormState({ name: fieldContext.name })
	const fieldState = getFieldState(fieldContext.name, formState)

	if (!fieldContext) {
		throw new Error('useFormField should be used within <FormField>')
	}

	const { id } = itemContext

	return {
		id,
		name: fieldContext.name,
		formItemId: `${id}-form-item`,
		formDescriptionId: `${id}-form-item-description`,
		formMessageId: `${id}-form-item-message`,
		...fieldState
	}
}
```

**問題**：

1. 這個 hook 為什麼要用兩個不同的 Context（FormFieldContext 和 FormItemContext）？
2. `getFieldState` 和 `useFormState` 的差異是什麼？
3. 為什麼要產生 `formItemId`、`formDescriptionId` 等 ID？這些 ID 的作用是什麼？

---

### Q19. React 19 的 useFormState

在專案中使用了 React 19，看 `components/ui/form.tsx`：

```tsx
import {
	Controller,
	FormProvider,
	useFormContext,
	useFormState,
	type ControllerProps,
	type FieldPath,
	type FieldValues
} from 'react-hook-form'
```

**問題**：

1. React Hook Form 的 `useFormState` 和 React 19 內建的 `useFormState` 有什麼不同？
2. 在這個專案中使用的是哪一個？
3. React 19 的新 Form API 對這個專案有什麼影響？

---

### Q20. Slot Pattern

在 `components/ui/form.tsx` 中：

```tsx
function FormControl({ ...props }: React.ComponentProps<typeof Slot>) {
	const { error, formItemId, formDescriptionId, formMessageId } = useFormField()

	return (
		<Slot
			data-slot="form-control"
			id={formItemId}
			aria-describedby={!error ? `${formDescriptionId}` : `${formDescriptionId} ${formMessageId}`}
			aria-invalid={!!error}
			{...props}
		/>
	)
}
```

**問題**：

1. Radix UI 的 `Slot` 元件是什麼？它如何運作？
2. 為什麼要用 `Slot` 而不是直接用 `div` 或其他 HTML 元素？
3. `aria-describedby` 和 `aria-invalid` 的作用是什麼？為什麼要設定這些屬性？

---

## 二、狀態管理與資料流 (10 題)

### Q21. Context 的分層設計

專案中有多個 Context：

```tsx
// app/layout.tsx
<AuthProvider>
  <UserPurchaseProvider>
    <SWRProvider>
      {children}
    </SWRProvider>
  </UserPurchaseProvider>
</AuthProvider>

// app/(app)/layout.tsx
<JourneyProvider>
  <SidebarProvider>
    {/* ... */}
  </SidebarProvider>
</JourneyProvider>
```

**問題**：

1. 為什麼 `AuthProvider` 要在最外層，而 `JourneyProvider` 在內層？
2. 如果把 `UserPurchaseProvider` 和 `AuthProvider` 的順序對調會有什麼問題？
3. 這種 Provider 嵌套結構對效能有什麼影響？

---

### Q22. Context 間的依賴關係

在 `contexts/user-purchase-context.tsx` 中：

```tsx
export function UserPurchaseProvider({ children }: UserPurchaseProviderProps) {
	const { user, isAuthenticated } = useAuth()

	useEffect(() => {
		if (isAuthenticated && user) {
			fetchUserPurchaseData(true)
		} else {
			setPurchasedJourneyIds(new Set())
			setUnpaidOrders([])
		}
	}, [isAuthenticated, user, fetchUserPurchaseData])
}
```

**問題**：

1. `UserPurchaseProvider` 依賴 `useAuth()`，這種 Context 間的依賴關係好嗎？
2. 如果 `AuthContext` 的 `user` 物件經常改變，會導致什麼問題？
3. 有沒有更好的方式來處理這種依賴關係？

---

### Q23. Set 作為 State

在 `contexts/user-purchase-context.tsx` 中：

```tsx
const [purchasedJourneyIds, setPurchasedJourneyIds] = useState<Set<number>>(new Set())

// 更新時
if (purchasesResult.success) {
	const journeyIdsSet = new Set(purchasesResult.data.journeys.map((j) => j.journeyId))
	setPurchasedJourneyIds(journeyIdsSet)
}
```

**問題**：

1. 為什麼選擇用 `Set` 而不是 `Array` 來儲存已購買的課程 ID？
2. 每次都建立新的 Set instance 而不是修改現有的 Set，這是為什麼？
3. 使用 `Set` 對 `hasPurchased` 函式的效能有什麼幫助？

---

### Q24. SWR 全域配置

在 `providers/swr-provider.tsx` 中：

```tsx
<SWRConfig
	value={{
		revalidateOnFocus: false,
		shouldRetryOnError: true,
		errorRetryCount: 3,
		errorRetryInterval: 1000,
		dedupingInterval: 2000,
		revalidateIfStale: false,
		keepPreviousData: true
	}}
>
	{children}
</SWRConfig>
```

**問題**：

1. `revalidateOnFocus: false` 的作用是什麼？為什麼要關閉它？
2. `dedupingInterval: 2000` 是什麼意思？
3. `keepPreviousData: true` 對使用者體驗有什麼幫助？

---

### Q25. 跨 Tab 通訊

在 `contexts/user-purchase-context.tsx` 中：

```tsx
const invalidateAndRefresh = useCallback(async () => {
	await fetchUserPurchaseData(true)

	// Notify other tabs
	localStorage.setItem('purchase_updated', Date.now().toString())
	localStorage.removeItem('purchase_updated')
}, [fetchUserPurchaseData])

useEffect(() => {
	const handleStorageChange = (e: StorageEvent) => {
		if (e.key === 'purchase_updated' && e.newValue) {
			refreshPurchases()
		}
	}

	window.addEventListener('storage', handleStorageChange)
	return () => window.removeEventListener('storage', handleStorageChange)
}, [refreshPurchases])
```

**問題**：

1. 為什麼先 `setItem` 再馬上 `removeItem`？這樣做的目的是什麼？
2. `storage` 事件如何實現跨 Tab 通訊？
3. 為什麼在發送通知的 tab 中不會觸發 `storage` 事件？

---

### Q26. 樂觀更新 vs 重新載入

在 `contexts/journey-context.tsx` 中：

```tsx
const updateMissionStatus = useCallback((missionId: number, status: MissionStatus) => {
	setJourney((prev) => {
		if (!prev) return null
		return {
			...prev,
			chapters: prev.chapters.map((chapter) => ({
				...chapter,
				missions: chapter.missions.map((mission) =>
					mission.id === missionId ? { ...mission, status } : mission
				)
			}))
		}
	})
}, [])
```

**問題**：

1. 這是樂觀更新（optimistic update）嗎？為什麼？
2. 如果 API 呼叫失敗，這個狀態更新會回滾嗎？
3. 什麼情況下應該使用樂觀更新？什麼情況下應該等 API 回應再更新？

---

### Q27. Loading State 管理

在 `hooks/use-mission.ts` 中：

```tsx
const { user, isAuthenticated, isLoading: authLoading } = useAuth()
const { hasPurchased, isLoading: purchaseLoading } = useUserPurchase()

const [isLoading, setIsLoading] = useState(true)
const [isDelivering, setIsDelivering] = useState(false)

return {
	// ...
	isLoading: authLoading || isLoading || purchaseLoading,
	isDelivering
	// ...
}
```

**問題**：

1. 為什麼要把三個不同來源的 loading 狀態合併成一個？
2. `isDelivering` 為什麼要分開管理？
3. 如何避免 loading 狀態之間的競爭條件（race condition）？

---

### Q28. 防止重複呼叫

在 `hooks/use-video-progress.ts` 中：

```tsx
const completedRef = useRef(false)

const handleEnd = useCallback(() => {
	stopProgressTracking()

	if (completedRef.current) return

	completedRef.current = true
	setIsCompleted(true)

	onProgressUpdate(durationSeconds)
	onComplete()
}, [stopProgressTracking, durationSeconds, onProgressUpdate, onComplete])
```

**問題**：

1. 為什麼影片結束時可能會被重複呼叫？
2. 除了用 `useRef`，還有其他方法可以防止重複呼叫嗎？
3. 如果使用者倒轉影片再看一次，`completedRef` 需要重置嗎？

---

### Q29. 條件式狀態初始化

在 `hooks/use-mission.ts` 中：

```tsx
if (
	missionSummary &&
	missionSummary.accessLevel === 'PURCHASED' &&
	!hasPurchased(fetchedJourneyId)
) {
	setIsPurchaseRequired(true)
	setMission(null)
	setProgress({
		missionId,
		status: 'UNCOMPLETED',
		watchPositionSeconds: 0
	})
	return // Exit early
}

// Only fetch mission details if access is granted
const missionResult = await missionApi.getMissionDetail(fetchedJourneyId, missionId)
```

**問題**：

1. 為什麼要在呼叫 `getMissionDetail` API 之前就檢查購買狀態？
2. Early return 這種模式的好處是什麼？
3. 為什麼要設定假的 `progress` 物件而不是設為 `null`？

---

### Q30. 狀態同步策略

在 `hooks/use-mission.ts` 中：

```tsx
const progressResult = await missionApi.getUserMissionProgress(parseInt(user.id), missionId)
if (progressResult.success) {
	setProgress(progressResult.data)
	// Sync status to JourneyContext so sidebar shows correct state
	if (progressResult.data.status) {
		updateMissionStatus(missionId, progressResult.data.status)
	}
}
```

**問題**：

1. 為什麼要把 progress 狀態同步到 `JourneyContext`？
2. 如果同一個資料在多個地方存在，如何確保它們保持同步？
3. 這種做法有什麼潛在問題？有沒有更好的狀態管理方案？

---

## 三、與後端互動 (10 題)

### Q31. API Client 架構

看 `lib/api/core/client.ts` 的 class 設計：

```tsx
export class ApiClient {
	private config: ApiClientConfig

	constructor(config?: Partial<ApiClientConfig>) {
		this.config = { ...defaultConfig, ...config }
	}

	async get<T>(path: string, options?: RequestOptions): Promise<ApiResponse<T>>
	async post<T, D = unknown>(
		path: string,
		data?: D,
		options?: RequestOptions
	): Promise<ApiResponse<T>>
	// ...
}

export const apiClient = new ApiClient()
```

**問題**：

1. 為什麼要設計成 class 而不是單純的函式？
2. `constructor` 接受 `Partial<ApiClientConfig>` 的好處是什麼？
3. 為什麼要 export 一個 default instance（`apiClient`）？

---

### Q32. 統一的錯誤處理

在 `lib/api/core/client.ts` 中：

```tsx
private async handleResponse<T>(response: Response): Promise<ApiResponse<T>> {
  try {
    const contentType = response.headers.get('content-type')
    const isJSON = contentType?.includes('application/json')

    if (!response.ok) {
      const errorData = isJSON
        ? await response.json()
        : { error: await response.text() }
      return {
        success: false,
        error: {
          message: errorData.error || errorData.message || `HTTP error ${response.status}`,
          status: response.status,
          code: errorData.code,
        },
      }
    }

    const data = isJSON ? await response.json() : await response.text()
    return {
      success: true,
      data: data as T,
    }
  } catch (error) {
    return {
      success: false,
      error: {
        message: error instanceof Error ? error.message : 'Failed to parse response',
      },
    }
  }
}
```

**問題**：

1. 為什麼要檢查 `content-type` 來決定如何解析 response？
2. `ApiResponse` 的 `success` flag 設計有什麼好處？
3. 為什麼不直接 throw error 而是回傳包含 error 的物件？

---

### Q33. Retry 機制

在 `lib/api/core/client.ts` 中：

```tsx
private async request<T>(
  method: HttpMethod,
  path: string,
  data?: unknown,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  // ...
  const maxRetries = this.config.retry!.maxRetries!
  const baseDelay = this.config.retry!.retryDelay!

  let lastError: ApiResponse<T> | null = null

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      let result = await this.executeRequest<T>(requestConfig, options)

      if (
        !result.success &&
        attempt < maxRetries &&
        this.shouldRetry(method, result.error)
      ) {
        lastError = result
        await this.sleep(baseDelay * Math.pow(2, attempt))
        continue
      }

      return result
    } catch (error) {
      // ...
    }
  }
}
```

**問題**：

1. Exponential backoff（`baseDelay * Math.pow(2, attempt)`）是什麼？為什麼要這樣設計？
2. 為什麼只有 idempotent methods（GET, PUT, DELETE）才會 retry？
3. 如果三次都失敗了，最後回傳的是哪一次的錯誤？

---

### Q34. Request Timeout

在 `lib/api/core/client.ts` 中：

```tsx
private async executeRequest<T>(
  requestConfig: RequestConfig,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  const controller = new AbortController()
  const timeoutId = setTimeout(
    () => controller.abort(),
    requestConfig.timeout
  )

  try {
    const response = await fetch(requestConfig.url, {
      method: requestConfig.method,
      headers: requestConfig.headers,
      body: requestConfig.body ? JSON.stringify(requestConfig.body) : undefined,
      signal: controller.signal,
      ...options,
    })

    clearTimeout(timeoutId)
    return await this.handleResponse<T>(response)
  } catch (error) {
    clearTimeout(timeoutId)
    throw error
  }
}
```

**問題**：

1. `AbortController` 是如何實現 timeout 的？
2. 為什麼在 `try` 和 `catch` block 中都要 `clearTimeout`？
3. 如果 timeout 了，`catch` block 會接到什麼樣的 error？

---

### Q35. API Interceptors

在 `lib/api/core/client.ts` 中：

```tsx
private async request<T>(/* ... */): Promise<ApiResponse<T>> {
  let requestConfig: RequestConfig = {
    method,
    url: this.buildURL(path),
    headers: {
      ...this.config.headers,
      ...options?.headers,
    },
    body: data,
    timeout: options?.timeout ?? this.config.timeout!,
  }

  // Apply request interceptor if configured
  if (this.config.interceptors?.request) {
    requestConfig = await this.config.interceptors.request(requestConfig)
  }

  // ... execute request

  // Apply response interceptor if configured
  if (this.config.interceptors?.response) {
    result = await this.config.interceptors.response(result)
  }

  return result
}
```

**問題**：

1. Request interceptor 和 response interceptor 可以用來做什麼？請舉例。
2. 為什麼 interceptor 是 async 的？
3. 如果 interceptor 中拋出錯誤會怎樣？

---

### Q36. 自動 401 處理

在 `hooks/use-api.ts` 中：

```tsx
export function useApi() {
	const { logout } = useAuth()
	const router = useRouter()

	const callApi = useCallback(
		async <T,>(apiCall: () => Promise<ApiResponse<T>>): Promise<ApiResponse<T>> => {
			const response = await apiCall()

			if (!response.success && response.error.status === 401) {
				logout()
				router.push('/login')
			}

			return response
		},
		[logout, router]
	)

	return { callApi }
}
```

**問題**：

1. 為什麼 401 處理要在 custom hook 中而不是在 API client 中？
2. 如果同時有多個 API 呼叫返回 401，會觸發多次 logout 嗎？
3. 這種設計在使用時有什麼限制？所有 API 呼叫都必須用這個 hook 嗎？

---

### Q37. 進度追蹤的批次更新

在 `hooks/use-video-progress.ts` 中：

```tsx
const PROGRESS_UPDATE_INTERVAL_MS = 10000 // Update progress every 10 seconds

const startProgressTracking = useCallback(() => {
	if (progressIntervalRef.current) {
		clearInterval(progressIntervalRef.current)
	}

	progressIntervalRef.current = setInterval(saveCurrentProgress, PROGRESS_UPDATE_INTERVAL_MS)
}, [saveCurrentProgress])
```

**問題**：

1. 為什麼要每 10 秒才更新一次進度，而不是即時更新？
2. 如果使用者網路不穩定，積累多個進度更新失敗怎麼辦？
3. 如何平衡更新頻率與伺服器負載？

---

### Q38. 平行 API 呼叫

在 `contexts/user-purchase-context.tsx` 中：

```tsx
const [purchasesResult, ordersResult] = await Promise.all([
	userPurchaseApi.getUserPurchasedJourneys(userId),
	userPurchaseApi.getUserOrders(userId, {
		page: 1,
		limit: 50,
		status: 'UNPAID'
	})
])
```

**問題**：

1. 使用 `Promise.all` 的優點是什麼？
2. 如果其中一個 API 失敗，另一個會繼續執行嗎？
3. 什麼情況下不應該用 `Promise.all`？

---

### Q39. API Response Type Safety

看 API response 的型別定義：

```tsx
type ApiResponse<T> = { success: true; data: T } | { success: false; error: ApiError }

const result = await authApi.login(data)

if (result.success) {
	// TypeScript knows result.data exists here
	login(result.data.accessToken, result.data.user)
} else {
	// TypeScript knows result.error exists here
	toast.error(result.error.message)
}
```

**問題**：

1. 這種 discriminated union type 有什麼好處？
2. 為什麼 TypeScript 能在 `if (result.success)` 後知道 `result.data` 存在？
3. 如果不用這種設計，直接讓 `data` 和 `error` 都是 optional 會有什麼問題？

---

### Q40. Optimistic UI Update

在實際使用中，例如 `hooks/use-mission.ts` 的 `handleDeliverMission`：

```tsx
const handleDeliverMission = async () => {
	if (!user || !mission) return

	setIsDelivering(true)
	const result = await missionApi.deliverMission(parseInt(user.id), missionId)

	if (result.success) {
		setProgress((prev) => (prev ? { ...prev, status: 'DELIVERED' } : null))
		updateMissionStatus(missionId, 'DELIVERED')

		toast.success('任務完成！', {
			description: `獲得 ${result.data.experienceGained} 經驗值！`
		})
	} else {
		toast.error('完成任務失敗')
	}
	setIsDelivering(false)
}
```

**問題**：

1. 這裡是先呼叫 API 再更新 UI，還是樂觀更新（先更新 UI 再呼叫 API）？
2. 如果改成樂觀更新，程式碼要怎麼改？
3. 什麼情況下適合樂觀更新？什麼情況下不適合？

---

## 四、Tailwind CSS 與 UI (5 題)

### Q41. cn() 工具函式

在 `lib/utils.ts` 中：

```tsx
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
	return twMerge(clsx(inputs))
}
```

使用範例：

```tsx
<div className={cn('grid gap-2', error && 'text-destructive', className)} />
```

**問題**：

1. `clsx` 和 `twMerge` 各自的作用是什麼？
2. 為什麼要組合這兩個函式？只用其中一個不行嗎？
3. `twMerge` 如何處理衝突的 Tailwind classes（例如 `p-4 p-2`）？

---

### Q42. 條件式樣式

在 `components/ui/form.tsx` 中：

```tsx
<Label
	data-slot="form-label"
	data-error={!!error}
	className={cn('data-[error=true]:text-destructive', className)}
	htmlFor={formItemId}
	{...props}
/>
```

**問題**：

1. `data-error={!!error}` 和 `data-[error=true]:text-destructive` 是如何配合運作的？
2. 為什麼要用 `data-*` attribute 而不是直接用 `className={error ? 'text-destructive' : ''}`？
3. 這種寫法對 CSS 優先級（specificity）有什麼影響？

---

### Q43. Responsive Video Player

在 `components/mission/video-player.tsx` 中：

```tsx
<div className="video-player-wrapper w-full">
	<div className="relative mx-auto w-full max-w-6xl overflow-hidden rounded-lg bg-black">
		<div className="relative pb-[56.25%]">
			<div className="absolute inset-0">
				<YouTube
					videoId={videoId}
					opts={opts}
					className="absolute inset-0 h-full w-full"
					iframeClassName="w-full h-full"
				/>
			</div>
		</div>
	</div>
</div>
```

**問題**：

1. `pb-[56.25%]` 是什麼？為什麼用這個數字？
2. 這種 padding trick 如何實現響應式的 16:9 影片？
3. 為什麼需要這麼多層的巢狀 div？

---

### Q44. Radix UI 整合

看 `components/ui/form.tsx` 使用 Radix UI：

```tsx
import * as LabelPrimitive from '@radix-ui/react-label'
import { Slot } from '@radix-ui/react-slot'

function FormLabel({ className, ...props }: React.ComponentProps<typeof LabelPrimitive.Root>) {
	const { error, formItemId } = useFormField()

	return (
		<Label
			data-slot="form-label"
			data-error={!!error}
			className={cn('data-[error=true]:text-destructive', className)}
			htmlFor={formItemId}
			{...props}
		/>
	)
}
```

**問題**：

1. Radix UI 提供什麼功能？為什麼不直接用原生的 `<label>` 元素？
2. `React.ComponentProps<typeof LabelPrimitive.Root>` 這個型別是什麼意思？
3. Radix UI 和 Tailwind CSS 如何搭配使用？

---

### Q45. Accessibility (a11y)

在 `components/ui/form.tsx` 中：

```tsx
<Slot
	data-slot="form-control"
	id={formItemId}
	aria-describedby={!error ? `${formDescriptionId}` : `${formDescriptionId} ${formMessageId}`}
	aria-invalid={!!error}
	{...props}
/>
```

**問題**：

1. `aria-describedby` 的作用是什麼？為什麼要設定它？
2. `aria-invalid` 對螢幕閱讀器（screen reader）有什麼幫助？
3. 為什麼要動態設定 `aria-describedby` 根據是否有 error？

---

## 五、TypeScript 與型別系統 (5 題)

### Q46. Generic Types 的使用

在 `lib/api/core/client.ts` 中：

```tsx
async get<T>(
  path: string,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  return this.request<T>('GET', path, undefined, options)
}

async post<T, D = unknown>(
  path: string,
  data?: D,
  options?: RequestOptions
): Promise<ApiResponse<T>> {
  return this.request<T>('POST', path, data, options)
}
```

**問題**：

1. 為什麼 `get` 只有一個泛型 `<T>`，而 `post` 有兩個 `<T, D>`？
2. `D = unknown` 是什麼意思？為什麼用 `unknown` 而不是 `any`？
3. 使用時如何指定這些泛型的型別？

---

### Q47. Discriminated Unions

在 API response 型別中：

```tsx
export type ApiResponse<T> = { success: true; data: T } | { success: false; error: ApiError }

export interface ApiError {
	message: string
	status?: number
	code?: string
}
```

**問題**：

1. 什麼是 discriminated union？`success` 欄位在這裡的作用是什麼？
2. TypeScript 如何利用 `success` 來做型別窄化（type narrowing）？
3. 如果我想加入第三種狀態（例如 loading），應該怎麼設計型別？

---

### Q48. React.ComponentProps

在 `components/ui/form.tsx` 中：

```tsx
function FormItem({ className, ...props }: React.ComponentProps<'div'>) {
	const id = React.useId()

	return (
		<FormItemContext.Provider value={{ id }}>
			<div data-slot="form-item" className={cn('grid gap-2', className)} {...props} />
		</FormItemContext.Provider>
	)
}
```

**問題**：

1. `React.ComponentProps<'div'>` 會產生什麼型別？
2. 為什麼要用 `React.ComponentProps` 而不是自己定義 props interface？
3. 這種寫法對元件的可重用性有什麼好處？

---

### Q49. Type Inference with Zod

在 `components/auth/login-form.tsx` 中：

```tsx
const loginSchema = z.object({
	username: z.string().min(1, '請輸入使用者名稱').min(3, '使用者名稱至少需要 3 個字元'),
	password: z.string().min(1, '請輸入密碼').min(8, '密碼至少需要 8 個字元')
})

type LoginFormValues = z.infer<typeof loginSchema>

const form = useForm<LoginFormValues>({
	resolver: zodResolver(loginSchema)
	// ...
})
```

**問題**：

1. `z.infer<typeof loginSchema>` 如何從 Zod schema 推導出 TypeScript 型別？
2. 為什麼不直接寫 `interface LoginFormValues { username: string; password: string }`？
3. 這種方式如何確保 runtime validation 和 compile-time type checking 的一致性？

---

### Q50. Context Type Safety

在 `contexts/auth-context.tsx` 中：

```tsx
interface AuthContextType {
	user: UserInfo | null
	isAuthenticated: boolean
	isLoading: boolean
	login: (token: string, user: UserInfo) => void
	logout: () => Promise<void>
	updateUser: (user: UserInfo) => void
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function useAuth() {
	const context = useContext(AuthContext)
	if (context === undefined) {
		throw new Error('useAuth must be used within an AuthProvider')
	}
	return context
}
```

**問題**：

1. 為什麼 `createContext` 的泛型是 `AuthContextType | undefined` 而不是直接 `AuthContextType`？
2. `useAuth` 中的 runtime check（`if (context === undefined)`）和 TypeScript 型別檢查有什麼關係？
3. 有沒有更好的方式來確保 Context 的型別安全，讓我們不需要每次都檢查 `undefined`？

---

## 結語

這 50 道題目涵蓋了專案中的核心前端技術實作細節。面試時，面試官可以：

1. **指著程式碼問**：「這裡為什麼這樣寫？」
2. **追問細節**：「如果改成 XXX 會怎樣？」
3. **考驗理解**：「這個設計解決了什麼問題？」
4. **探討替代方案**：「有沒有更好的做法？」

祝面試順利！🚀

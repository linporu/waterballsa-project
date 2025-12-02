# TypeScript + React 型別技巧

> 給熟悉 TypeScript 的開發者，聚焦 React 特有的型別應用

你已經熟悉 TypeScript，這篇文件只專注在 React 生態系中的**特殊型別技巧**。

📝 **對應面試題**：Q46-Q50

---

## 目錄

1. [泛型在 React 中的應用](#1-泛型在-react-中的應用)
2. [React.ComponentProps 擷取元件 Props](#2-reactcomponentprops-擷取元件-props)
3. [Discriminated Unions 處理 API 回應](#3-discriminated-unions-處理-api-回應)
4. [Zod 型別推導](#4-zod-型別推導)
5. [Context 型別安全](#5-context-型別安全)

---

## 1. 泛型在 React 中的應用

### 💡 核心概念

React 中的泛型 (Generics) 最常用於：

- **可復用的 API client**
- **表單欄位元件**
- **Context Providers**

### 專案實例：API Client 泛型方法

```typescript
// 檔案位置: lib/api/core/client.ts (第 32-68 行)
export class ApiClient {
	/**
	 * 泛型方法：T 代表回應的資料型別
	 */
	private async handleResponse<T>(response: Response): Promise<ApiResponse<T>> {
		try {
			const contentType = response.headers.get('content-type')
			const isJSON = contentType?.includes('application/json')

			if (!response.ok) {
				const errorData = isJSON ? await response.json() : { error: await response.text() }
				return {
					success: false,
					error: {
						message: errorData.error || `HTTP error ${response.status}`,
						status: response.status,
						code: errorData.code
					}
				}
			}

			const data = isJSON ? await response.json() : await response.text()
			return {
				success: true,
				data: data as T // ✅ 型別斷言為泛型 T
			}
		} catch (error) {
			return {
				success: false,
				error: {
					message: error instanceof Error ? error.message : 'Failed to parse response'
				}
			}
		}
	}
}
```

### 使用範例

```typescript
// 呼叫時指定回應型別
interface UserInfo {
	id: number
	email: string
	name: string
}

// TypeScript 會自動推導 result 的型別為 ApiResponse<UserInfo>
const result = await client.get<UserInfo>('/api/users/me')

if (result.success) {
	console.log(result.data.email) // ✅ TypeScript 知道 data 有 email
} else {
	console.log(result.error.message) // ✅ TypeScript 知道 error 有 message
}
```

### 實例：泛型表單欄位元件

```typescript
// 檔案位置: components/ui/form.tsx (第 21-43 行)
type FormFieldContextValue<
	TFieldValues extends FieldValues = FieldValues,
	TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>
> = {
	name: TName
}

const FormField = <
	TFieldValues extends FieldValues = FieldValues,
	TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>
>({
	...props
}: ControllerProps<TFieldValues, TName>) => {
	return (
		<FormFieldContext.Provider value={{ name: props.name }}>
			<Controller {...props} />
		</FormFieldContext.Provider>
	)
}
```

### 💡 為什麼要用泛型？

```typescript
// ❌ 沒有泛型：每次都要寫型別轉換
const data = await client.get('/api/users/me')
const user = data as UserInfo // 不安全！

// ✅ 使用泛型：一次指定，處處型別安全
const result = await client.get<UserInfo>('/api/users/me')
if (result.success) {
	// TypeScript 自動知道 result.data 是 UserInfo
}
```

### 📝 相關面試題

- [Q46: 泛型在 React 中的應用](../interview/interview-frontend.md#q46)
- [Q47: 型別推導與泛型約束](../interview/interview-frontend.md#q47)

---

## 2. React.ComponentProps 擷取元件 Props

### 💡 核心概念

`React.ComponentProps<T>` 可以**擷取任何元件或 HTML 元素的 props 型別**，常用於擴展現有元件。

### 基本語法

```typescript
// 擷取 HTML 元素的 props
type ButtonProps = React.ComponentProps<'button'>

// 擷取自訂元件的 props
type MyButtonProps = React.ComponentProps<typeof MyButton>
```

### 專案實例

```typescript
// 檔案位置: components/ui/form.tsx (第 76-88 行)
function FormItem({ className, ...props }: React.ComponentProps<'div'>) {
	const id = React.useId()

	return (
		<FormItemContext.Provider value={{ id }}>
			<div
				data-slot="form-item"
				className={cn('grid gap-2', className)}
				{...props} // ✅ 所有 div 的原生 props 都能傳入
			/>
		</FormItemContext.Provider>
	)
}
```

```typescript
// 檔案位置: components/ui/form.tsx (第 90-104 行)
function FormLabel({ className, ...props }: React.ComponentProps<typeof LabelPrimitive.Root>) {
	// ✅ 擷取 Radix UI Label 元件的所有 props
	const { error, formItemId } = useFormField()

	return (
		<Label
			data-slot="form-label"
			data-error={!!error}
			className={cn('data-[error=true]:text-destructive', className)}
			htmlFor={formItemId}
			{...props} // 傳遞所有原本的 props
		/>
	)
}
```

### 實用範例：擴展按鈕元件

```typescript
// 擴展原生 button，加上 loading 狀態
interface ButtonProps extends React.ComponentProps<'button'> {
	isLoading?: boolean
	variant?: 'primary' | 'secondary'
}

function Button({ isLoading, variant, children, ...props }: ButtonProps) {
	return (
		<button
			disabled={isLoading || props.disabled}
			className={cn('btn', `btn-${variant}`)}
			{...props} // ✅ 保留所有原生 button props (onClick, type, etc.)
		>
			{isLoading ? 'Loading...' : children}
		</button>
	)
}

// 使用時享有完整的型別提示
;<Button
	variant="primary"
	isLoading={false}
	onClick={(e) => console.log(e)} // ✅ TypeScript 知道這是滑鼠事件
	type="submit" // ✅ 原生 button props 也有提示
>
	Submit
</Button>
```

### 💡 為什麼用 ComponentProps？

```typescript
// ❌ 手動列舉：容易遺漏、難維護
interface ButtonProps {
	onClick?: (e: React.MouseEvent) => void
	disabled?: boolean
	type?: 'button' | 'submit' | 'reset'
	// ... 還有很多原生 props
}

// ✅ 使用 ComponentProps：自動包含所有原生 props
interface ButtonProps extends React.ComponentProps<'button'> {
	isLoading?: boolean // 只需加入自訂 props
}
```

### 📝 相關面試題

- [Q48: React.ComponentProps 的應用](../interview/interview-frontend.md#q48)

---

## 3. Discriminated Unions 處理 API 回應

### 💡 核心概念

**Discriminated Union**（辨識聯合型別）使用一個共同的「標記」欄位（如 `success`）來區分不同的型別變體，讓 TypeScript 能**自動窄化型別**。

### 專案實例：API 回應型別

```typescript
// 檔案位置: lib/api/api-schema.ts
/**
 * ✅ 使用 Discriminated Union
 * - success: true → 一定有 data
 * - success: false → 一定有 error
 */
export type ApiResponse<T> =
	| { success: true; data: T } // 成功分支
	| { success: false; error: ApiError } // 失敗分支

interface ApiError {
	message: string
	status?: number
	code?: string
}
```

### 使用範例：型別自動窄化

```typescript
async function fetchUser() {
	const result = await userApi.getUserInfo()

	// ✅ TypeScript 根據 success 自動窄化型別
	if (result.success) {
		// 在這裡，TypeScript 知道一定有 result.data
		console.log(result.data.email) // ✅ 正確
		console.log(result.error) // ❌ TypeScript 會報錯：success=true 時沒有 error
	} else {
		// 在這裡，TypeScript 知道一定有 result.error
		console.log(result.error.message) // ✅ 正確
		console.log(result.data) // ❌ TypeScript 會報錯：success=false 時沒有 data
	}
}
```

### 專案中的實際應用

```typescript
// 檔案位置: hooks/use-mission.ts (第 49-65 行)
useEffect(() => {
	async function fetchMissionData() {
		if (!user) return

		try {
			// 呼叫 API
			const journeyResult = await journeyApi.getJourneyBySlug(journeySlug)

			// ✅ TypeScript 強制檢查 success
			if (!journeyResult.success) {
				throw new Error(journeyResult.error?.message || 'Failed to fetch journey')
			}

			// ✅ 這裡 TypeScript 知道 journeyResult.data 一定存在
			const fetchedJourneyId = journeyResult.data.id
			const chapters = journeyResult.data.chapters
			// ...
		} catch (error) {
			setError(error.message)
		}
	}
	fetchMissionData()
}, [user, journeySlug])
```

### 💡 對比：沒有 Discriminated Union

```typescript
// ❌ 差勁的設計：data 和 error 同時存在
type BadApiResponse<T> = {
	data?: T
	error?: ApiError
}

// 使用時需要處處判斷
const result = await api.get<User>('/users/me')
if (result.data) {
	// ⚠️ TypeScript 不確定是否真的有 data
	console.log(result.data.email) // data 可能是 undefined
}

// ✅ 好的設計：使用 Discriminated Union
type GoodApiResponse<T> = { success: true; data: T } | { success: false; error: ApiError }

const result = await api.get<User>('/users/me')
if (result.success) {
	// ✅ TypeScript 確定有 data，且不可能是 undefined
	console.log(result.data.email)
}
```

### 📝 相關面試題

- [Q49: Discriminated Unions 的應用](../interview/interview-frontend.md#q49)
- [Q50: 型別窄化 (Type Narrowing)](../interview/interview-frontend.md#q50)

---

## 4. Zod 型別推導

### 💡 核心概念

Zod 是一個 TypeScript-first 的 schema 驗證庫，可以**從 schema 自動推導出 TypeScript 型別**，實現「單一來源的真實」。

### 基本語法

```typescript
import { z } from 'zod'

// 1. 定義 Zod schema
const UserSchema = z.object({
	email: z.string().email(),
	password: z.string().min(8),
	age: z.number().optional()
})

// 2. 從 schema 推導型別
type User = z.infer<typeof UserSchema>
// 等同於：
// type User = {
//   email: string
//   password: string
//   age?: number | undefined
// }
```

### 專案實例：登入表單 Schema

```typescript
// 檔案位置: components/auth/login-form.tsx
import { z } from 'zod'

// ✅ 定義 schema（同時用於驗證與型別）
const loginFormSchema = z.object({
	email: z
		.string()
		.min(1, { message: '請輸入電子郵件' })
		.email({ message: '請輸入有效的電子郵件格式' }),
	password: z.string().min(8, { message: '密碼至少需要 8 個字元' })
})

// ✅ 從 schema 自動推導型別
type LoginFormData = z.infer<typeof loginFormSchema>
// 推導結果：
// type LoginFormData = {
//   email: string
//   password: string
// }
```

### 搭配 React Hook Form 使用

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'

function LoginForm() {
	// ✅ zodResolver 連接 Zod schema 與表單驗證
	const form = useForm<LoginFormData>({
		resolver: zodResolver(loginFormSchema),
		defaultValues: {
			email: '',
			password: ''
		}
	})

	const onSubmit = (data: LoginFormData) => {
		// ✅ data 已經通過 Zod 驗證，型別安全
		console.log(data.email, data.password)
	}

	return <form onSubmit={form.handleSubmit(onSubmit)}>{/* ... */}</form>
}
```

### 💡 為什麼用 Zod？

**單一來源的真實 (Single Source of Truth)**：

```typescript
// ❌ 壞做法：分開定義型別和驗證
type User = {
	email: string
	age?: number
}

function validate(user: User) {
	if (!user.email.includes('@')) return false
	if (user.age && user.age < 0) return false
	return true
}
// 問題：型別和驗證邏輯分離，容易不同步

// ✅ 好做法：Zod schema 同時提供型別和驗證
const UserSchema = z.object({
	email: z.string().email(),
	age: z.number().min(0).optional()
})
type User = z.infer<typeof UserSchema>
// 型別和驗證邏輯永遠同步！
```

### 進階：API Schema 自動型別推導

```typescript
// API 回應 schema
const UserInfoSchema = z.object({
	id: z.number(),
	email: z.string().email(),
	name: z.string(),
	roles: z.array(z.string())
})

// ✅ 自動推導型別
export type UserInfo = z.infer<typeof UserInfoSchema>

// 在 API client 中使用
async function getUserInfo(): Promise<ApiResponse<UserInfo>> {
	const response = await fetch('/api/users/me')
	const data = await response.json()

	// ✅ 執行時驗證 + 型別安全
	const parsed = UserInfoSchema.safeParse(data)
	if (!parsed.success) {
		return { success: false, error: { message: 'Invalid response' } }
	}

	return { success: true, data: parsed.data } // data 是 UserInfo 型別
}
```

### 📝 相關面試題

- [Q18: Zod schema 驗證](../interview/interview-frontend.md#q18)
- [Q19: zodResolver 與 React Hook Form](../interview/interview-frontend.md#q19)

---

## 5. Context 型別安全

### 💡 核心概念

React Context 需要兩層型別安全：

1. **Context value 的型別**
2. **檢查是否在 Provider 內使用**

### 專案實例：Auth Context

```typescript
// 檔案位置: contexts/auth-context.tsx (第 22-31 行)

// 1️⃣ 定義 Context 的值型別
interface AuthContextType {
	user: UserInfo | null
	isAuthenticated: boolean
	isLoading: boolean
	login: (token: string, user: UserInfo) => void
	logout: () => Promise<void>
	updateUser: (user: UserInfo) => void
}

// 2️⃣ 建立 Context，初始值為 undefined
const AuthContext = createContext<AuthContextType | undefined>(undefined)
//                                                   ^^^^^^^^^ 允許 undefined
```

```typescript
// 3️⃣ 建立型別安全的 custom hook
export function useAuth() {
	const context = useContext(AuthContext)

	// ✅ 執行時檢查：確保在 Provider 內使用
	if (context === undefined) {
		throw new Error('useAuth must be used within an AuthProvider')
	}

	// ✅ 回傳的型別是 AuthContextType（不含 undefined）
	return context
}
```

### 使用範例

```typescript
function ProfilePage() {
	// ✅ TypeScript 知道這些欄位一定存在
	const { user, isAuthenticated, logout } = useAuth()

	if (!isAuthenticated) {
		return <div>Please login</div>
	}

	// ✅ user 型別為 UserInfo | null
	return (
		<div>
			<h1>{user?.name}</h1> {/* 需要使用 optional chaining */}
			<button onClick={logout}>Logout</button>
		</div>
	)
}
```

### 💡 為什麼要這樣設計？

```typescript
// ❌ 壞做法：允許 null 作為預設值
const AuthContext = createContext<AuthContextType | null>(null)

export function useAuth() {
  return useContext(AuthContext)  // 回傳 AuthContextType | null
}

// 使用時每次都要判斷 null
function Component() {
  const auth = useAuth()
  if (!auth) return null  // 到處都要寫這個！
  auth.login(...)
}

// ✅ 好做法：使用 undefined + 執行時檢查
const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function useAuth() {
  const context = useContext(AuthContext)
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider')
  }
  return context  // 回傳 AuthContextType（沒有 undefined）
}

// 使用時不需要判斷，直接使用
function Component() {
  const { login } = useAuth()  // ✅ 一定有值
  login(...)
}
```

### 進階：多個 Context 的型別整合

```typescript
// 檔案位置: hooks/use-mission.ts (第 24-29 行)
export function useMission(): UseMissionReturn {
	// ✅ 整合多個型別安全的 contexts
	const { user, isAuthenticated, isLoading: authLoading } = useAuth()
	const { updateMissionStatus } = useJourney()
	const { hasPurchased, isLoading: purchaseLoading } = useUserPurchase()

	// TypeScript 自動檢查所有欄位的型別
	// ...
}
```

### 📝 相關面試題

- [Q21: Context 型別定義](../interview/interview-frontend.md#q21)
- [Q22: useContext 型別安全](../interview/interview-frontend.md#q22)

---

## 🎯 總結

### TypeScript + React 核心技巧

| 技巧                     | 用途                   | 關鍵點                          |
| ------------------------ | ---------------------- | ------------------------------- |
| **Generics**             | API client、可復用元件 | `<T>` 讓函式/元件支援多種型別   |
| **ComponentProps**       | 擴展元件 props         | 自動繼承所有原生 props          |
| **Discriminated Unions** | API 回應、狀態管理     | 用 `success` 等標記欄位區分型別 |
| **Zod + infer**          | 表單驗證               | Schema 同時提供驗證和型別       |
| **Context + undefined**  | 全域狀態               | undefined + throw error 模式    |

### 最佳實踐

1. **優先使用 Discriminated Unions** 處理多種可能的狀態
2. **用 React.ComponentProps** 擴展元件，避免手動列舉 props
3. **Zod schema 作為單一來源**，型別從 schema 推導
4. **Context 用 undefined + 執行時檢查**，避免到處判斷 null
5. **泛型讓 API client 保持型別安全**，不需要手動轉型

---

## 🚀 下一步

- 學習 [Context API 與狀態管理](./03-context-api.md)
- 深入 [表單處理](./04-form-handling.md)

---

## 📖 延伸閱讀

- [TypeScript 官方文件](https://www.typescriptlang.org/docs/)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [Zod 官方文件](https://zod.dev/)

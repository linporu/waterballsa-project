# 表單處理 (React Hook Form + Zod)

> 給熟悉 Vue 表單處理的開發者

React 的表單處理與 Vue 的 `v-model` 有所不同。本專案使用 **React Hook Form** 處理表單狀態，搭配 **Zod** 進行驗證。

📝 **對應面試題**：Q16-Q20

---

## 目錄

1. [React Hook Form 基礎](#1-react-hook-form-基礎)
2. [Zod Schema 驗證](#2-zod-schema-驗證)
3. [FormField 與 Controller](#3-formfield-與-controller)
4. [錯誤處理](#4-錯誤處理)
5. [實戰技巧](#5-實戰技巧)

---

## 1. React Hook Form 基礎

### 🆚 Vue vs React

| Vue 3             | React (Controlled Component) | React Hook Form          |
| ----------------- | ---------------------------- | ------------------------ |
| `v-model="email"` | `value={email}` + `onChange` | `{...register('email')}` |
| 自動雙向綁定      | 手動處理狀態                 | 自動處理（uncontrolled） |
| 模板驅動          | 狀態驅動                     | Hook 驅動                |

### 基本用法

```typescript
import { useForm } from 'react-hook-form'

function SimpleForm() {
	// ✅ 初始化表單
	const form = useForm({
		defaultValues: {
			email: '',
			password: ''
		}
	})

	// ✅ 提交處理
	const onSubmit = (data) => {
		console.log(data) // { email: '...', password: '...' }
	}

	return (
		<form onSubmit={form.handleSubmit(onSubmit)}>
			<input {...form.register('email')} />
			<input {...form.register('password')} type="password" />
			<button type="submit">Submit</button>
		</form>
	)
}
```

### 💡 核心 API

| API              | 用途                                    |
| ---------------- | --------------------------------------- |
| `useForm()`      | 初始化表單                              |
| `register()`     | 註冊欄位                                |
| `handleSubmit()` | 處理提交（自動驗證）                    |
| `formState`      | 取得表單狀態（errors, isSubmitting 等） |
| `setError()`     | 手動設定錯誤                            |
| `reset()`        | 重設表單                                |

### 專案實例：登入表單

```typescript
// 檔案位置: components/auth/login-form.tsx (第 34-45 行)

export function LoginForm() {
	const [isLoading, setIsLoading] = useState(false)
	const router = useRouter()
	const { login } = useAuth()

	// ✅ 初始化表單，並連接 Zod validator
	const form = useForm<LoginFormValues>({
		resolver: zodResolver(loginSchema), // 連接 Zod schema
		defaultValues: {
			username: '',
			password: ''
		}
	})

	// ✅ 提交處理函式
	async function onSubmit(data: LoginFormValues) {
		setIsLoading(true)
		try {
			const result = await authApi.login(data)
			if (result.success) {
				login(result.data.accessToken, result.data.user)
				toast.success('登入成功！')
				router.push('/')
			} else {
				toast.error(result.error.message)
			}
		} finally {
			setIsLoading(false)
		}
	}

	return (
		<Form {...form}>
			<form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
				{/* FormField 元件... */}
			</form>
		</Form>
	)
}
```

### 📝 相關面試題

- [Q16: React Hook Form 基本用法](../interview/interview-frontend.md#q16)
- [Q17: useForm 的設定選項](../interview/interview-frontend.md#q17)

---

## 2. Zod Schema 驗證

### 💡 核心概念

Zod 是 TypeScript-first 的 schema 驗證庫，可以：

1. **定義驗證規則**
2. **自動推導 TypeScript 型別**
3. **提供清楚的錯誤訊息**

### 基本語法

```typescript
import { z } from 'zod'

// 定義 schema
const schema = z.object({
	email: z.string().email('無效的 email 格式'),
	age: z.number().min(18, '必須年滿 18 歲'),
	password: z.string().min(8, '密碼至少 8 個字元')
})

// 推導型別
type FormData = z.infer<typeof schema>
// 等同於：
// type FormData = {
//   email: string
//   age: number
//   password: string
// }
```

### 專案實例：登入 Schema

```typescript
// 檔案位置: components/auth/login-form.tsx (第 24-32 行)

// ✅ 定義驗證 schema
const loginSchema = z.object({
	username: z
		.string()
		.min(1, '請輸入使用者名稱') // 第一層：不能為空
		.min(3, '使用者名稱至少需要 3 個字元'), // 第二層：長度檢查
	password: z.string().min(1, '請輸入密碼').min(8, '密碼至少需要 8 個字元')
})

// ✅ 自動推導型別
type LoginFormValues = z.infer<typeof loginSchema>
// 結果：
// type LoginFormValues = {
//   username: string
//   password: string
// }
```

### 連接 React Hook Form

```typescript
import { zodResolver } from '@hookform/resolvers/zod'

const form = useForm<LoginFormValues>({
	resolver: zodResolver(loginSchema), // ✅ 連接 Zod
	defaultValues: {
		username: '',
		password: ''
	}
})

// 當使用者提交表單時：
// 1. zodResolver 會用 loginSchema 驗證資料
// 2. 如果驗證失敗，錯誤會自動顯示在表單中
// 3. 如果驗證成功，onSubmit 才會被呼叫
```

### 常見 Zod 驗證規則

```typescript
const schema = z
	.object({
		// 字串驗證
		email: z.string().email('無效的 email'),
		username: z.string().min(3).max(20),
		url: z.string().url('無效的 URL'),

		// 數字驗證
		age: z.number().min(0).max(150),
		price: z.number().positive('價格必須大於 0'),

		// 可選欄位
		nickname: z.string().optional(),
		phone: z.string().nullable(),

		// 列舉
		role: z.enum(['admin', 'user', 'guest']),

		// 自訂驗證
		password: z
			.string()
			.refine((val) => /^(?=.*[A-Z])(?=.*[0-9])/.test(val), {
				message: '密碼必須包含大寫字母和數字'
			})

		// 確認密碼
	})
	.refine((data) => data.password === data.confirmPassword, {
		message: '密碼不一致',
		path: ['confirmPassword'] // 錯誤顯示在哪個欄位
	})
```

### 📝 相關面試題

- [Q18: Zod schema 驗證規則](../interview/interview-frontend.md#q18)
- [Q19: zodResolver 的作用](../interview/interview-frontend.md#q19)

---

## 3. FormField 與 Controller

### 💡 核心概念

**FormField** 是專案中的自訂元件（基於 shadcn/ui），整合了：

- React Hook Form 的 `Controller`
- 欄位標籤 (`FormLabel`)
- 欄位控制 (`FormControl`)
- 錯誤訊息 (`FormMessage`)

### 專案實例：使用 FormField

```typescript
// 檔案位置: components/auth/login-form.tsx (第 92-108 行)

<FormField
	control={form.control} // ✅ 傳入表單控制權
	name="username" // ✅ 欄位名稱（對應 schema）
	render={(
		{ field } // ✅ render prop 模式
	) => (
		<FormItem>
			<FormLabel>使用者名稱</FormLabel>
			<FormControl>
				<Input
					placeholder="請輸入使用者名稱"
					autoComplete="username"
					{...field} // ✅ 展開 field props (value, onChange, onBlur, ref)
					disabled={isLoading}
				/>
			</FormControl>
			<FormMessage /> {/* ✅ 自動顯示驗證錯誤 */}
		</FormItem>
	)}
/>
```

### 💡 `{...field}` 展開了什麼？

```typescript
// field 物件包含：
{
  value: string,           // 欄位當前值
  onChange: (e) => void,   // 更新值的函式
  onBlur: () => void,      // 失焦時觸發驗證
  ref: RefCallback,        // DOM 引用
  name: string,            // 欄位名稱
}

// {...field} 等同於：
<Input
  value={field.value}
  onChange={field.onChange}
  onBlur={field.onBlur}
  ref={field.ref}
  name={field.name}
/>
```

### FormField 內部實作

```typescript
// 檔案位置: components/ui/form.tsx (第 32-43 行)

const FormField = <
	TFieldValues extends FieldValues = FieldValues,
	TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>
>({
	...props
}: ControllerProps<TFieldValues, TName>) => {
	return (
		<FormFieldContext.Provider value={{ name: props.name }}>
			<Controller {...props} /> {/* ✅ React Hook Form 的 Controller */}
		</FormFieldContext.Provider>
	)
}
```

### 為什麼要用 FormField？

```typescript
// ❌ 不用 FormField：手動處理所有東西
<div>
  <label htmlFor="email">Email</label>
  <input
    id="email"
    {...register('email')}
  />
  {errors.email && <span>{errors.email.message}</span>}
</div>

// ✅ 用 FormField：一致的樣式和行為
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl>
        <Input {...field} />
      </FormControl>
      <FormMessage />  {/* 自動處理錯誤顯示 */}
    </FormItem>
  )}
/>
```

### 📝 相關面試題

- [Q20: FormField 的設計模式](../interview/interview-frontend.md#q20)

---

## 4. 錯誤處理

### 自動錯誤（來自 Zod 驗證）

```typescript
// Schema 定義的錯誤會自動顯示
const schema = z.object({
	username: z.string().min(3, '使用者名稱至少需要 3 個字元')
})

// 使用者輸入 "ab" 時，FormMessage 會自動顯示：
// "使用者名稱至少需要 3 個字元"
```

### 手動設定錯誤

```typescript
// 檔案位置: components/auth/login-form.tsx (第 70-79 行)

async function onSubmit(data: LoginFormValues) {
	const result = await authApi.login(data)

	if (!result.success) {
		// ✅ 後端返回 401：手動設定欄位錯誤
		if (result.error.status === 401) {
			form.setError('username', {
				type: 'manual',
				message: ' ' // 空訊息（不顯示在 username 欄位）
			})
			form.setError('password', {
				type: 'manual',
				message: '使用者名稱或密碼錯誤' // 顯示在 password 欄位
			})
		}
	}
}
```

### 全域錯誤（Toast）

```typescript
// ✅ 使用 toast 顯示全域錯誤
if (result.success) {
	toast.success('登入成功！', {
		description: `歡迎回來，${result.data.user.username}`
	})
} else {
	toast.error(result.error.message) // 顯示後端錯誤訊息
}
```

### 錯誤顯示策略

| 情境               | 顯示方式                | 範例                   |
| ------------------ | ----------------------- | ---------------------- |
| **欄位驗證失敗**   | FormMessage（欄位下方） | "密碼至少 8 個字元"    |
| **伺服器欄位錯誤** | setError + FormMessage  | "使用者名稱已被使用"   |
| **全域錯誤**       | Toast                   | "網路錯誤，請稍後再試" |
| **操作成功**       | Toast                   | "登入成功！"           |

### 📝 相關面試題

- [Q16: 表單錯誤處理策略](../interview/interview-frontend.md#q16)
- [Q17: setError 的使用時機](../interview/interview-frontend.md#q17)

---

## 5. 實戰技巧

### 技巧 1: 提交中的 Loading 狀態

```typescript
// ✅ 使用獨立的 loading state
const [isLoading, setIsLoading] = useState(false)

async function onSubmit(data: FormValues) {
	setIsLoading(true)
	try {
		await api.submit(data)
	} finally {
		setIsLoading(false) // 無論成功或失敗都要重設
	}
}

return (
	<>
		<Input {...field} disabled={isLoading} />
		<Button type="submit" disabled={isLoading}>
			{isLoading ? '提交中...' : '提交'}
		</Button>
	</>
)
```

### 技巧 2: 重設表單

```typescript
// 提交成功後重設表單
async function onSubmit(data: FormValues) {
	const result = await api.submit(data)
	if (result.success) {
		form.reset() // ✅ 重設為 defaultValues
		toast.success('提交成功')
	}
}

// 或重設為新的值
form.reset({
	email: 'new@example.com',
	name: ''
})
```

### 技巧 3: 監聽欄位變化

```typescript
import { useWatch } from 'react-hook-form'

function MyForm() {
	const form = useForm()

	// ✅ 監聽特定欄位
	const email = useWatch({
		control: form.control,
		name: 'email'
	})

	// 當 email 改變時觸發副作用
	useEffect(() => {
		console.log('Email changed:', email)
	}, [email])
}
```

### 技巧 4: 動態欄位

```typescript
import { useFieldArray } from 'react-hook-form'

function TodoListForm() {
	const form = useForm({
		defaultValues: {
			todos: [{ text: '' }]
		}
	})

	const { fields, append, remove } = useFieldArray({
		control: form.control,
		name: 'todos'
	})

	return (
		<form>
			{fields.map((field, index) => (
				<div key={field.id}>
					<Input {...form.register(`todos.${index}.text`)} />
					<button type="button" onClick={() => remove(index)}>
						刪除
					</button>
				</div>
			))}
			<button type="button" onClick={() => append({ text: '' })}>
				新增待辦事項
			</button>
		</form>
	)
}
```

### 技巧 5: 條件式驗證

```typescript
const schema = z
	.object({
		needsAddress: z.boolean(),
		address: z.string().optional()
	})
	.refine(
		(data) => {
			// ✅ 如果勾選需要地址，則地址必填
			if (data.needsAddress) {
				return data.address && data.address.length > 0
			}
			return true
		},
		{
			message: '請填寫地址',
			path: ['address']
		}
	)
```

### 技巧 6: 整合無障礙 (A11y)

```typescript
// 檔案位置: components/ui/form.tsx (第 94-104 行)

function FormLabel({ className, ...props }: ComponentProps<typeof LabelPrimitive.Root>) {
	const { error, formItemId } = useFormField()

	return (
		<Label
			data-slot="form-label"
			data-error={!!error} // ✅ 用 data attribute 標記錯誤狀態
			className={cn('data-[error=true]:text-destructive', className)}
			htmlFor={formItemId} // ✅ 連接 label 與 input
			{...props}
		/>
	)
}
```

```typescript
// FormControl 會自動加上無障礙屬性
<Input
	id="email-input"
	aria-describedby="email-description" // 描述文字
	aria-invalid={!!error} // 錯誤狀態
	aria-errormessage="email-error" // 錯誤訊息 ID
	{...field}
/>
```

---

## 🎯 總結

### React Hook Form + Zod 核心流程

```
1. 定義 Zod schema
   ↓
2. 用 z.infer 推導型別
   ↓
3. useForm + zodResolver
   ↓
4. FormField 包裝欄位
   ↓
5. handleSubmit 自動驗證
   ↓
6. onSubmit 處理資料
```

### 與 Vue 對比

| 功能         | Vue               | React Hook Form                   |
| ------------ | ----------------- | --------------------------------- |
| **雙向綁定** | `v-model`         | `{...register()}` 或 `{...field}` |
| **驗證庫**   | VeeValidate, Yup  | Zod + zodResolver                 |
| **錯誤顯示** | `<ErrorMessage>`  | `<FormMessage>`                   |
| **提交處理** | `@submit.prevent` | `handleSubmit(onSubmit)`          |
| **重設表單** | `resetForm()`     | `form.reset()`                    |

### 記憶口訣

1. **Schema 先行**：先定義 Zod schema，型別自動推導
2. **zodResolver 連接**：useForm 的 resolver 參數
3. **FormField 包裝**：統一樣式和錯誤處理
4. **`{...field}` 展開**：自動處理 value, onChange, onBlur
5. **手動錯誤用 setError**：伺服器驗證失敗時

---

## 🚀 下一步

- 學習 [非同步處理模式](./05-async-patterns.md)
- 深入 [Tailwind CSS & 無障礙](./06-tailwind-a11y.md)

---

## 📖 延伸閱讀

- [React Hook Form 官方文件](https://react-hook-form.com/)
- [Zod 官方文件](https://zod.dev/)
- [shadcn/ui Form 元件](https://ui.shadcn.com/docs/components/form)

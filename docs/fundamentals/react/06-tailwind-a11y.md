# Tailwind CSS & 無障礙 (Accessibility)

> 快速掌握專案中的樣式工具與無障礙設計

本文聚焦在專案中使用的 **Tailwind CSS 實用技巧**與**無障礙 (A11y) 基礎**，幫助你快速上手。

📝 **對應面試題**：Q20, Q41-Q45

---

## 目錄

1. [cn() 工具函式](#1-cn-工具函式)
2. [條件式樣式](#2-條件式樣式)
3. [Data Attributes 樣式](#3-data-attributes-樣式)
4. [ARIA 屬性](#4-aria-屬性)
5. [Radix UI Slot Pattern](#5-radix-ui-slot-pattern)

---

## 1. cn() 工具函式

### 💡 核心概念

`cn()` 是專案中最常用的工具函式，用於**合併和覆蓋 Tailwind CSS 類別**。

### 實作原理

```typescript
// 檔案位置: lib/utils.ts (完整檔案)
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
	return twMerge(clsx(inputs))
}
```

### 💡 為什麼需要 cn()？

```typescript
// ❌ 問題：單純串接 className 會有衝突
<div className={`bg-red-500 ${className}`} />
// 如果 className = "bg-blue-500"
// 結果：class="bg-red-500 bg-blue-500"
// 實際顯示：取決於 CSS 載入順序（不可預測）

// ✅ 解法：使用 cn() 自動處理衝突
<div className={cn('bg-red-500', className)} />
// 如果 className = "bg-blue-500"
// 結果：class="bg-blue-500"  （後者覆蓋前者）
```

### 基本用法

```typescript
import { cn } from '@/lib/utils'

// 1️⃣ 串接多個類別
cn('text-red-500', 'font-bold')
// 結果: "text-red-500 font-bold"

// 2️⃣ 條件式類別
cn('text-red-500', isActive && 'font-bold')
// isActive = true:  "text-red-500 font-bold"
// isActive = false: "text-red-500"

// 3️⃣ 覆蓋預設樣式
cn('bg-red-500 text-white', 'bg-blue-500')
// 結果: "text-white bg-blue-500"  ✅ bg-blue-500 覆蓋 bg-red-500

// 4️⃣ 物件語法（來自 clsx）
cn({
	'text-red-500': hasError,
	'text-green-500': !hasError
})
// hasError = true:  "text-red-500"
// hasError = false: "text-green-500"

// 5️⃣ 陣列語法
cn(['text-red-500', 'font-bold'], className)
```

### 專案實例

```typescript
// 檔案位置: components/ui/form.tsx (第 76-87 行)

function FormItem({ className, ...props }: React.ComponentProps<'div'>) {
	const id = React.useId()

	return (
		<FormItemContext.Provider value={{ id }}>
			<div
				data-slot="form-item"
				className={cn('grid gap-2', className)} // ✅ 允許外部覆蓋預設樣式
				{...props}
			/>
		</FormItemContext.Provider>
	)
}

// 使用時：
;<FormItem className="gap-4">
	{' '}
	{/* gap-4 覆蓋預設的 gap-2 */}
	{/* ... */}
</FormItem>
```

```typescript
// 檔案位置: components/ui/form.tsx (第 90-104 行)

function FormLabel({ className, ...props }: ComponentProps<typeof LabelPrimitive.Root>) {
	const { error, formItemId } = useFormField()

	return (
		<Label
			data-slot="form-label"
			data-error={!!error}
			// ✅ 條件式樣式 + 外部自訂樣式
			className={cn('data-[error=true]:text-destructive', className)}
			htmlFor={formItemId}
			{...props}
		/>
	)
}
```

### 💡 cn() 的三個功能

| 功能                  | 來源             | 說明                                              |
| --------------------- | ---------------- | ------------------------------------------------- |
| **條件式類別**        | `clsx`           | `cn('text-red', isActive && 'font-bold')`         |
| **物件/陣列語法**     | `clsx`           | `cn({ 'text-red': hasError })`                    |
| **智慧合併 Tailwind** | `tailwind-merge` | `cn('bg-red-500', 'bg-blue-500')` → `bg-blue-500` |

### 📝 相關面試題

- [Q41: cn() 工具函式的作用](../interview/interview-frontend.md#q41)

---

## 2. 條件式樣式

### 基本模式

```typescript
// 1️⃣ 簡單條件
<div className={cn('text-base', isLarge && 'text-lg')} />

// 2️⃣ 三元運算子
<div className={cn(isError ? 'text-red-500' : 'text-green-500')} />

// 3️⃣ 多重條件
<Button
  className={cn(
    'px-4 py-2',
    variant === 'primary' && 'bg-blue-500',
    variant === 'secondary' && 'bg-gray-500',
    disabled && 'opacity-50 cursor-not-allowed'
  )}
/>

// 4️⃣ 物件語法（更清晰）
<div
  className={cn({
    'text-red-500': hasError,
    'text-yellow-500': hasWarning,
    'text-green-500': !hasError && !hasWarning,
  })}
/>
```

### 專案實例：按鈕變體

```typescript
interface ButtonProps {
	variant?: 'primary' | 'secondary' | 'outline'
	size?: 'sm' | 'md' | 'lg'
	isLoading?: boolean
}

function Button({ variant = 'primary', size = 'md', isLoading, className, ...props }: ButtonProps) {
	return (
		<button
			className={cn(
				// 基礎樣式
				'rounded font-medium transition-colors',

				// 變體樣式
				{
					'bg-blue-500 text-white hover:bg-blue-600': variant === 'primary',
					'bg-gray-500 text-white hover:bg-gray-600': variant === 'secondary',
					'border-2 border-blue-500 text-blue-500 hover:bg-blue-50': variant === 'outline'
				},

				// 尺寸樣式
				{
					'px-2 py-1 text-sm': size === 'sm',
					'px-4 py-2 text-base': size === 'md',
					'px-6 py-3 text-lg': size === 'lg'
				},

				// 狀態樣式
				isLoading && 'opacity-50 cursor-wait',

				// 外部自訂樣式
				className
			)}
			disabled={isLoading}
			{...props}
		/>
	)
}

// 使用範例
;<Button variant="primary" size="lg" isLoading={isSubmitting}>
	提交
</Button>
```

### 📝 相關面試題

- [Q42: 條件式 Tailwind 類別](../interview/interview-frontend.md#q42)

---

## 3. Data Attributes 樣式

### 💡 核心概念

使用 **data-\*** 屬性搭配 Tailwind 的 `data-[*]` 選擇器，實現基於狀態的樣式。

### 基本語法

```typescript
// 設定 data attribute
<div data-state="active" />
<div data-error={true} />
<div data-size="large" />

// 對應的 Tailwind 類別
className="data-[state=active]:bg-blue-500"
className="data-[error=true]:text-red-500"
className="data-[size=large]:text-2xl"
```

### 專案實例

```typescript
// 檔案位置: components/ui/form.tsx (第 90-104 行)

function FormLabel({ className, ...props }: ComponentProps<typeof LabelPrimitive.Root>) {
	const { error, formItemId } = useFormField()

	return (
		<Label
			data-slot="form-label"
			data-error={!!error} // ✅ 設定 data attribute
			// ✅ 根據 data-error 改變樣式
			className={cn('data-[error=true]:text-destructive', className)}
			htmlFor={formItemId}
			{...props}
		/>
	)
}

// 渲染結果：
// 無錯誤: <label data-error="false" class="">...</label>
// 有錯誤: <label data-error="true" class="text-destructive">...</label>
```

```typescript
// 檔案位置: components/ui/form.tsx (第 76-87 行)

function FormItem({ className, ...props }: React.ComponentProps<'div'>) {
	return (
		<div
			data-slot="form-item" // ✅ 用於識別元件類型
			className={cn('grid gap-2', className)}
			{...props}
		/>
	)
}
```

### 實際範例：互動狀態

```typescript
function Accordion() {
	const [isOpen, setIsOpen] = useState(false)

	return (
		<div>
			<button
				data-state={isOpen ? 'open' : 'closed'}
				className={cn(
					'px-4 py-2',
					// ✅ 根據 data-state 切換圖示旋轉
					'data-[state=open]:rotate-180',
					'transition-transform'
				)}
				onClick={() => setIsOpen(!isOpen)}
			>
				<ChevronDown />
			</button>

			<div
				data-state={isOpen ? 'open' : 'closed'}
				className={cn(
					// ✅ 根據 data-state 控制顯示/隱藏
					'data-[state=closed]:hidden',
					'data-[state=open]:block'
				)}
			>
				Content
			</div>
		</div>
	)
}
```

### 💡 為什麼用 data-\* 而非條件判斷？

```typescript
// ❌ 傳統方式：邏輯分散
<div className={isActive ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-700'}>

// ✅ data-* 方式：樣式集中
<div
  data-active={isActive}
  className={cn(
    'bg-gray-200 text-gray-700',  // 預設樣式
    'data-[active=true]:bg-blue-500',  // 啟用時的背景
    'data-[active=true]:text-white',   // 啟用時的文字
  )}
>

// 優點：
// 1. 樣式邏輯集中在 className
// 2. 容易覆蓋（外部可以傳入 className）
// 3. 與 Radix UI 等函式庫整合良好
```

### 📝 相關面試題

- [Q43: data-\* 屬性的樣式應用](../interview/interview-frontend.md#q43)

---

## 4. ARIA 屬性

### 💡 核心概念

**ARIA (Accessible Rich Internet Applications)** 屬性幫助螢幕閱讀器理解網頁結構與狀態，是無障礙設計的核心。

### 常用 ARIA 屬性

| 屬性               | 用途                         | 範例                                         |
| ------------------ | ---------------------------- | -------------------------------------------- |
| `aria-label`       | 提供元素的替代文字           | `<button aria-label="關閉">&times;</button>` |
| `aria-labelledby`  | 指向提供標籤的元素 ID        | `<div aria-labelledby="title-id">`           |
| `aria-describedby` | 指向提供描述的元素 ID        | `<input aria-describedby="error-msg-id">`    |
| `aria-invalid`     | 標記欄位驗證狀態             | `<input aria-invalid="true">`                |
| `aria-required`    | 標記必填欄位                 | `<input aria-required="true">`               |
| `aria-hidden`      | 隱藏元素（螢幕閱讀器也忽略） | `<div aria-hidden="true">`                   |
| `aria-expanded`    | 標記展開/收合狀態            | `<button aria-expanded="false">`             |
| `aria-live`        | 宣告動態內容更新             | `<div aria-live="polite">`                   |

### 專案實例：表單無障礙

```typescript
// 檔案位置: components/ui/form.tsx (第 107-123 行)

function FormControl({ ...props }: React.ComponentProps<typeof Slot>) {
	const { error, formItemId, formDescriptionId, formMessageId } = useFormField()

	return (
		<Slot
			data-slot="form-control"
			id={formItemId} // ✅ 連接到 label 的 htmlFor
			// ✅ aria-describedby: 指向描述文字
			aria-describedby={
				!error
					? `${formDescriptionId}` // 無錯誤：只連接描述
					: `${formDescriptionId} ${formMessageId}` // 有錯誤：連接描述 + 錯誤訊息
			}
			// ✅ aria-invalid: 標記驗證狀態
			aria-invalid={!!error}
			{...props}
		/>
	)
}
```

完整流程範例：

```typescript
// 渲染結果（無錯誤）
<label htmlFor="email-form-item" id="email-label">
  Email
</label>
<input
  id="email-form-item"
  aria-describedby="email-form-item-description"
  aria-invalid="false"
/>
<p id="email-form-item-description">
  請輸入您的電子郵件地址
</p>

// 渲染結果（有錯誤）
<label htmlFor="email-form-item" id="email-label" data-error="true" class="text-destructive">
  Email
</label>
<input
  id="email-form-item"
  aria-describedby="email-form-item-description email-form-item-message"
  aria-invalid="true"
/>
<p id="email-form-item-description">
  請輸入您的電子郵件地址
</p>
<p id="email-form-item-message" class="text-destructive">
  請輸入有效的 email 格式
</p>
```

### 實際範例：Modal 對話框

```typescript
function Modal({ isOpen, onClose, title, children }) {
	return (
		<div
			role="dialog" // ✅ 告訴螢幕閱讀器這是對話框
			aria-modal="true" // ✅ 標記為 modal（背景不可互動）
			aria-labelledby="modal-title" // ✅ 連接標題
			aria-hidden={!isOpen} // ✅ 關閉時隱藏
			className={cn('fixed inset-0 z-50', !isOpen && 'hidden')}
		>
			<div className="bg-black/50" onClick={onClose} />

			<div className="bg-white p-6 rounded-lg">
				<h2 id="modal-title">{title}</h2> {/* ✅ 提供標題 */}
				<button
					onClick={onClose}
					aria-label="關閉對話框" // ✅ 提供替代文字
				>
					&times;
				</button>
				{children}
			</div>
		</div>
	)
}
```

### 實際範例：Loading 狀態

```typescript
function LoadingButton({ isLoading, children, ...props }) {
	return (
		<button
			disabled={isLoading}
			aria-busy={isLoading} // ✅ 告訴螢幕閱讀器正在處理
			aria-live="polite" // ✅ 狀態改變時通知
			{...props}
		>
			{isLoading ? (
				<>
					<Spinner aria-hidden="true" /> {/* ✅ 裝飾性圖示，隱藏 */}
					<span>載入中...</span>
				</>
			) : (
				children
			)}
		</button>
	)
}
```

### 📝 相關面試題

- [Q44: ARIA 屬性的應用](../interview/interview-frontend.md#q44)
- [Q45: 表單無障礙設計](../interview/interview-frontend.md#q45)

---

## 5. Radix UI Slot Pattern

### 💡 核心概念

**Slot** 是 Radix UI 的特殊元件，可以**將 props 合併到子元件**，而不新增額外的 DOM 節點。

### 基本用法

```typescript
import { Slot } from '@radix-ui/react-slot'

// 沒有 Slot（會新增 <div>）
<div className="wrapper">
  <button className="inner">Click</button>
</div>
// DOM: <div class="wrapper"><button class="inner">Click</button></div>

// 使用 Slot（不新增 <div>，props 合併到 button）
<Slot className="wrapper">
  <button className="inner">Click</button>
</Slot>
// DOM: <button class="wrapper inner">Click</button>
```

### 專案實例

```typescript
// 檔案位置: components/ui/form.tsx (第 107-123 行)

function FormControl({ ...props }: React.ComponentProps<typeof Slot>) {
  const { error, formItemId, formDescriptionId, formMessageId } = useFormField()

  return (
    <Slot
      data-slot="form-control"
      id={formItemId}
      aria-describedby={
        !error ? `${formDescriptionId}` : `${formDescriptionId} ${formMessageId}`
      }
      aria-invalid={!!error}
      {...props}  // ✅ 這些 props 會合併到子元件
    />
  )
}

// 使用範例
<FormControl>
  <Input placeholder="請輸入..." />  {/* ✅ Input 會接收 FormControl 的所有 props */}
</FormControl>

// 實際渲染結果（沒有額外的包裹元素）：
<input
  id="email-form-item"
  aria-describedby="email-form-item-description"
  aria-invalid="false"
  placeholder="請輸入..."
/>
```

### 為什麼用 Slot？

```typescript
// ❌ 不用 Slot：多一層 DOM 節點
function FormControl({ children, ...props }) {
	return (
		<div id="form-control" aria-invalid={error} {...props}>
			{children} {/* Input 被 div 包裹 */}
		</div>
	)
}
// 結果: <div id="form-control"><input /></div>
// 問題: 多餘的 div，影響 CSS 選擇器和語義

// ✅ 使用 Slot：props 合併，沒有額外節點
function FormControl({ children, ...props }) {
	return (
		<Slot id="form-control" aria-invalid={error} {...props}>
			{children} {/* props 直接合併到 Input */}
		</Slot>
	)
}
// 結果: <input id="form-control" aria-invalid="false" />
// 優點: 乾淨的 DOM，正確的語義
```

### 進階：asChild Pattern

Radix UI 元件常用 `asChild` prop：

```typescript
// Radix UI Button
import { Button as RadixButton } from '@radix-ui/react-button'

// asChild = false（預設）：渲染 <button>
<RadixButton>
  Click me
</RadixButton>
// 結果: <button>Click me</button>

// asChild = true：使用 Slot，不渲染 <button>
<RadixButton asChild>
  <a href="/home">Go Home</a>
</RadixButton>
// 結果: <a href="/home">Go Home</a>（保留 a 標籤，但有 Button 的行為）
```

實際範例：

```typescript
function LinkButton({ href, children }) {
  return (
    <Button asChild>
      <Link href={href}>
        {children}
      </Link>
    </Button>
  )
}

// 使用
<LinkButton href="/profile">
  查看個人資料
</LinkButton>

// 渲染結果：
<a href="/profile" class="button-styles">
  查看個人資料
</a>
// 是 <a> 標籤，但有 Button 的樣式和行為
```

### 📝 相關面試題

- [Q20: Radix UI Slot pattern](../interview/interview-frontend.md#q20)

---

## 🎯 總結

### 核心工具與模式

| 工具/模式      | 用途                  | 關鍵點                             |
| -------------- | --------------------- | ---------------------------------- |
| **cn()**       | 合併 Tailwind 類別    | `twMerge(clsx(...))`               |
| **條件式樣式** | 根據狀態套用樣式      | `cn('base', condition && 'extra')` |
| **data-\***    | 基於狀態的樣式        | `data-[state=open]:block`          |
| **ARIA**       | 無障礙標記            | `aria-label`, `aria-invalid`, etc. |
| **Slot**       | 合併 props 不增加節點 | Radix UI 的核心模式                |

### 無障礙檢查清單

- ✅ **所有互動元素**都有 `aria-label` 或可見文字
- ✅ **表單欄位**連接 `<label>` (htmlFor + id)
- ✅ **錯誤訊息**用 `aria-describedby` + `aria-invalid`
- ✅ **Modal/Dialog** 使用 `role="dialog"` + `aria-modal`
- ✅ **Loading 狀態**使用 `aria-busy` + `aria-live`
- ✅ **裝飾性圖示**用 `aria-hidden="true"`

### 最佳實踐

1. **優先使用 cn()**：不要手動串接 className
2. **用 data-\* 管理狀態樣式**：比條件判斷更清晰
3. **為所有表單加上 ARIA**：改善無障礙體驗
4. **用 Slot 減少 DOM 層級**：保持語義正確
5. **測試鍵盤導航**：確保不用滑鼠也能操作

---

## 🚀 完成學習！

恭喜你完成了所有 React 前端基礎教學！

### 下一步建議

1. **回顧**：[學習路徑 README](./README.md)
2. **練習**：挑戰 [前端面試題](../interview/interview-frontend.md)
3. **實作**：在專案中應用這些概念

### 快速回顧連結

- [React Hooks 快速指南](./01-react-hooks.md) ⭐ 最重要
- [TypeScript + React 型別技巧](./02-typescript-react.md)
- [Context API 與狀態管理](./03-context-api.md)
- [表單處理](./04-form-handling.md)
- [非同步處理模式](./05-async-patterns.md)

---

## 📖 延伸閱讀

- [Tailwind CSS 官方文件](https://tailwindcss.com/docs)
- [ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/)
- [Radix UI 官方文件](https://www.radix-ui.com/)
- [WebAIM: Web Accessibility In Mind](https://webaim.org/)

---
name: react-patterns
description: "React 18+ patterns: hooks, Context, state management, TypeScript. Complementary skill invoked via CLAUDE.md skill table or auto-detected via cc10x-router."
allowed-tools: Read, Grep, Glob, Bash, LSP
---

# React Patterns

## Overview

React applications succeed when they embrace unidirectional data flow and composition. Every component should have a single responsibility, predictable state transitions, and clear boundaries between UI and logic.

**Core principle:** Components are functions. Keep them pure, compose them freely, lift state only when shared.

**Violating the letter of this process is violating the spirit of React development.**

## Focus Areas (Reference Pattern)

- **React 18+ hooks** (useState, useEffect, useCallback, useMemo, useRef)
- **Context API** for cross-cutting concerns (theme, auth, locale)
- **State management** (Redux Toolkit, Zustand, Jotai)
- **TypeScript integration** (props interfaces, generic components, type-safe hooks)
- **Performance** (React.lazy, memoization, code splitting)
- **Server Components** (React Server Components, streaming SSR)

## The Iron Law

```
NO COMPONENT WITHOUT CLEAR DATA FLOW
```

If you cannot trace where state comes from and how it updates, you do not understand your component. Map props and state before writing JSX.

## When to Use

**Always when working with:**
- React 18+ projects
- Next.js applications
- Redux/Zustand/Jotai state management
- React component libraries
- TSX files

## Universal Questions (Answer First)

1. **Client or Server Component?** Check if it needs interactivity. Default to Server Components in Next.js App Router.
2. **What state exists?** Identify all useState/useReducer and their consumers.
3. **Props down, callbacks up?** Verify unidirectional data flow.
4. **What triggers re-renders?** Map state dependencies explicitly.
5. **Is this shared state?** If yes, use Context or state library. If no, keep it local.

## Component Structure

```tsx
import { useState, useCallback } from 'react'
import type { User } from '@/types'

interface UserCardProps {
  userId: string
  showAvatar?: boolean
  size?: 'sm' | 'md' | 'lg'
  variant?: 'primary' | 'secondary'
  onSelect?: (user: User) => void
  onDelete?: (id: string) => void
}

export function UserCard({
  userId,
  showAvatar = false,
  size = 'md',
  variant = 'primary',
  onSelect,
  onDelete,
}: UserCardProps) {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const displayName = user ? `${user.firstName} ${user.lastName}` : ''

  return (
    // JSX
  )
}
```

## Hooks Patterns

### useState & useReducer

```typescript
// Simple state
const [count, setCount] = useState(0)
const [user, setUser] = useState<User | null>(null)

// Functional updates (when next state depends on previous)
setCount(prev => prev + 1)

// useReducer for complex state
interface FormState {
  email: string
  password: string
  errors: Record<string, string>
  isSubmitting: boolean
}

type FormAction =
  | { type: 'SET_FIELD'; field: string; value: string }
  | { type: 'SET_ERRORS'; errors: Record<string, string> }
  | { type: 'SUBMIT' }
  | { type: 'COMPLETE' }

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, [action.field]: action.value, errors: {} }
    case 'SET_ERRORS':
      return { ...state, errors: action.errors, isSubmitting: false }
    case 'SUBMIT':
      return { ...state, isSubmitting: true }
    case 'COMPLETE':
      return { ...state, isSubmitting: false }
  }
}
```

### useEffect

```typescript
// Data fetching with cleanup
useEffect(() => {
  const controller = new AbortController()

  async function fetchData() {
    try {
      const res = await fetch(`/api/users/${userId}`, { signal: controller.signal })
      setUser(await res.json())
    } catch (e) {
      if (e instanceof Error && e.name !== 'AbortError') {
        setError(e.message)
      }
    }
  }

  fetchData()
  return () => controller.abort()
}, [userId])

// AVOID: Missing cleanup, missing dependencies
// BAD: useEffect(() => { fetch(...) }, [])  // No cleanup
```

### Custom Hooks (The React Way to Share Logic)

```typescript
// hooks/useAsync.ts
import { useState, useCallback } from 'react'

interface UseAsyncReturn<T> {
  data: T | null
  error: string | null
  isLoading: boolean
  execute: () => Promise<void>
}

export function useAsync<T>(fn: () => Promise<T>): UseAsyncReturn<T> {
  const [data, setData] = useState<T | null>(null)
  const [error, setError] = useState<string | null>(null)
  const [isLoading, setIsLoading] = useState(false)

  const execute = useCallback(async () => {
    setIsLoading(true)
    setError(null)
    try {
      setData(await fn())
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Unknown error')
    } finally {
      setIsLoading(false)
    }
  }, [fn])

  return { data, error, isLoading, execute }
}
```

**Hook naming convention:** Always prefix with `use` (e.g., `useAuth`, `useForm`, `useSearch`).

**Hook rules:**
- Only call at the top level (not in loops, conditions, or nested functions)
- Only call from React functions or other hooks
- Always include proper dependency arrays

## Context API

```tsx
// contexts/ThemeContext.tsx
import { createContext, useContext, useState, type ReactNode } from 'react'

interface ThemeContextType {
  theme: 'light' | 'dark'
  toggle: () => void
}

const ThemeContext = createContext<ThemeContextType | null>(null)

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')
  const toggle = () => setTheme(prev => prev === 'light' ? 'dark' : 'light')

  return (
    <ThemeContext.Provider value={{ theme, toggle }}>
      {children}
    </ThemeContext.Provider>
  )
}

export function useTheme(): ThemeContextType {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider')
  return ctx
}
```

**Context rules:**
- Create a typed context with `null` default + custom hook that throws
- Use Context for low-frequency updates (theme, auth, locale)
- NOT for high-frequency updates (mouse position, scroll) — use state library instead
- Split read/write contexts to avoid unnecessary re-renders

## State Management (Redux Toolkit / Zustand)

### Zustand (Lightweight)

```typescript
import { create } from 'zustand'
import type { User } from '@/types'

interface UserStore {
  currentUser: User | null
  isAuthenticated: boolean
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

export const useUserStore = create<UserStore>((set) => ({
  currentUser: null,
  isAuthenticated: false,
  login: async (email, password) => {
    const user = await api.login({ email, password })
    set({ currentUser: user, isAuthenticated: true })
  },
  logout: () => set({ currentUser: null, isAuthenticated: false }),
}))
```

**State library rules:**
- Zustand for simple-to-medium state (most projects)
- Redux Toolkit for complex state with middleware needs
- Keep store slices focused (one per domain concept)
- Derive state with selectors, don't duplicate

## Component Patterns

### Buttons

```tsx
<button
  type="button"
  onClick={handleAction}
  disabled={isLoading || isDisabled}
  aria-busy={isLoading}
  aria-disabled={isDisabled}
  className={cn('btn-primary', isLoading && 'btn-loading')}
>
  {isLoading ? (
    <>
      <Spinner aria-hidden />
      <span>Processing...</span>
    </>
  ) : (
    'Submit'
  )}
</button>
```

### Forms with Validation

```tsx
<form onSubmit={handleSubmit} noValidate>
  <div className="form-field">
    <label htmlFor="email">
      Email <span aria-hidden>*</span>
      <span className="sr-only">(required)</span>
    </label>
    <input
      id="email"
      type="email"
      value={email}
      onChange={handleChange}
      autoComplete="email"
      spellCheck={false}
      aria-invalid={errors.email ? 'true' : undefined}
      aria-describedby={errors.email ? 'email-error' : 'email-hint'}
      required
    />
    <span id="email-hint" className="hint">
      We'll never share your email
    </span>
    {errors.email && (
      <span id="email-error" role="alert" className="error">
        {errors.email}
      </span>
    )}
  </div>
</form>
```

### Loading States

```tsx
function DataList({ isLoading, data, error, onRetry, onCreateNew }: DataListProps) {
  if (error) return <ErrorState error={error} onRetry={onRetry} />;
  if (isLoading && !data) return <LoadingState />;
  if (!data?.length) return <EmptyState onAction={onCreateNew} />;
  return <ul>{data.map(item => <Item key={item.id} {...item} />)}</ul>;
}
```

### Error Messages

```tsx
<div role="alert" className="error-banner">
  <Icon name="error" aria-hidden />
  <div>
    <p className="error-title">Upload failed</p>
    <p className="error-detail">File too large. Maximum size is 10MB.</p>
  </div>
  <button onClick={selectFile}>Choose different file</button>
</div>
```

### Content Overflow (Flex Truncation)

```tsx
{/* Flex truncation pattern - min-w-0 is REQUIRED */}
<div className="flex items-center gap-2 min-w-0">
  <Avatar />
  <span className="truncate min-w-0">{user.name}</span>
</div>
```

## URL & State Management

**URL should reflect UI state.** If it uses `useState`, consider URL sync.

```tsx
// Use nuqs, next-usequerystate, or similar
const [tab, setTab] = useQueryState('tab', { defaultValue: 'overview' })
```

## Performance Patterns

| Pattern | When | How |
|---------|------|-----|
| **React.memo** | Component re-renders with same props | `export default React.memo(MyComponent)` |
| **useMemo** | Expensive computation in render | `const sorted = useMemo(() => items.sort(...), [items])` |
| **useCallback** | Callback passed to memoized child | `const onClick = useCallback(() => {...}, [deps])` |
| **React.lazy** | Route-level code splitting | `const Page = React.lazy(() => import('./Page'))` |
| **Virtual scrolling** | Lists with 50+ items | Use `virtua`, `react-window`, or `@tanstack/virtual` |
| **Suspense** | Loading boundaries | `<Suspense fallback={<Skeleton />}>` |

### Memoization Rules

```typescript
// DO memoize: expensive calculations
const filtered = useMemo(() =>
  items.filter(i => i.name.includes(query)).sort((a, b) => a.name.localeCompare(b.name)),
  [items, query]
)

// DON'T memoize: cheap operations, primitive values
// BAD: useMemo(() => count * 2, [count])

// useCallback for stable references
const handleClick = useCallback((id: string) => {
  setSelected(id)
}, [])
```

## Error Handling

```tsx
// Error Boundary (class component — required by React)
class ErrorBoundary extends React.Component<
  { children: ReactNode; fallback: ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Error boundary caught:', error, info)
  }

  render() {
    if (this.state.hasError) return this.props.fallback
    return this.props.children
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>
```

## Accessibility

**For comprehensive accessibility patterns (WCAG, keyboard navigation, ARIA, color contrast), see the `frontend-patterns` skill.** Below are React-specific accessibility considerations:

- Use `aria-*` attributes bound to state (e.g., `aria-expanded={isOpen}`)
- Manage focus with `useRef` + `useEffect` after state changes
- Use `aria-live="polite"` for regions that update dynamically
- Use portal (`createPortal`) for modals/dialogs to ensure correct focus trapping

```tsx
// Focus management after state change
const inputRef = useRef<HTMLInputElement>(null)

useEffect(() => {
  if (showInput) {
    inputRef.current?.focus()
  }
}, [showInput])
```

## Red Flags - STOP and Reconsider

If you find yourself:

- Mutating state directly (`state.push(item)` instead of `setState([...state, item])`)
- Using `useEffect` for derived state (use `useMemo` or compute in render)
- Putting everything in Context (use state library for frequent updates)
- Missing dependency arrays in `useEffect`/`useMemo`/`useCallback`
- Using `index` as `key` in lists that can reorder
- Nesting ternaries in JSX (extract to variables or components)
- Creating state for values derivable from props

**STOP. Revisit the component design.**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Class components are simpler" | For small cases. Hooks compose better and type better. |
| "We don't need a state library" | Shared state without one = prop drilling or Context abuse. |
| "dangerouslySetInnerHTML is fine here" | XSS vector. Sanitize or use text content. |
| "Index keys work for this list" | Until the list reorders. Use stable IDs. |
| "We'll add TypeScript later" | Retrofitting types is 10x harder. Start typed. |
| "useEffect for everything" | useEffect is for synchronization with external systems, not derived state. |

## Anti-patterns Blocklist

| Anti-pattern | Why It's Wrong | Fix |
|--------------|----------------|-----|
| Direct state mutation | Breaks React's diffing | Return new references |
| `dangerouslySetInnerHTML` with user input | XSS vulnerability | Use text content or sanitize |
| Deeply nested ternaries in JSX | Unreadable | Extract to variables or components |
| `useEffect` for derived state | Causes extra renders | Use `useMemo` or compute inline |
| Missing `key` in lists | Broken reconciliation | Use stable, unique IDs |
| State for derived values | Redundant state, sync bugs | Compute from existing state/props |
| `any` in props/state types | Loses type safety | Define proper interfaces |

## Output Format

```markdown
## React Review: [Component/Feature]

### Data Flow
[What state exists, how it flows, what triggers re-renders]

### Component Structure
- [ ] Typed props interface (no `any`)
- [ ] Hooks at top level with correct dependencies
- [ ] Custom hooks extracted for reusable logic
- [ ] State management appropriate (local vs Context vs library)
- [ ] Error boundaries in place

### Issues
| Severity | Issue | Location | Fix |
|----------|-------|----------|-----|
| [BLOCKS/IMPAIRS/MINOR] | [Issue] | `file:line` | [Fix] |

### Recommendations
1. [Most critical]
2. [Second]
```

## Final Check

Before completing React work:

- [ ] Props interface defined with TypeScript (no `any`)
- [ ] Hooks follow Rules of Hooks (top-level, correct deps)
- [ ] No prop mutation (props down, callbacks up)
- [ ] No `useEffect` for derived state (use `useMemo` or inline)
- [ ] State library used for shared state (not Context for frequent updates)
- [ ] `key` on all list items (stable IDs, not indexes)
- [ ] Error boundaries for critical UI sections
- [ ] Loading states shown for async operations
- [ ] Accessibility verified (see `frontend-patterns` skill)
- [ ] No `dangerouslySetInnerHTML` with unsanitized user input

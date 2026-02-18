---
name: vue-patterns
description: "Vue 3 patterns: Composition API, Pinia, TypeScript, Nuxt. Complementary skill invoked via CLAUDE.md skill table."
allowed-tools: Read, Grep, Glob, Bash, LSP
---

# Vue Patterns

## Overview

Vue applications succeed when they embrace the framework's reactivity system rather than fighting it. Every component should have a clear responsibility, predictable data flow, and explicit reactive boundaries.

**Core principle:** Let Vue's reactivity system do the work. Declare what depends on what, and let the framework handle updates.

**Violating the letter of this process is violating the spirit of Vue development.**

## Focus Areas (Reference Pattern)

- **Vue 3 Composition API** (setup, ref, reactive, computed, watch)
- **Single File Components** (.vue with `<script setup>`, `<template>`, `<style scoped>`)
- **State management** with Pinia
- **Vue Router** patterns and navigation guards
- **TypeScript integration** (defineProps, defineEmits, type-only declarations)
- **Nuxt.js** patterns (SSR, auto-imports, server routes)

## The Iron Law

```
NO COMPONENT WITHOUT CLEAR REACTIVE BOUNDARIES
```

If you cannot explain what triggers a re-render, you do not understand your component. Map reactive dependencies before writing template code.

## When to Use

**Always when working with:**
- Vue 3 projects (Options API or Composition API)
- Nuxt.js applications
- Pinia stores
- Vue Router configuration
- Vue SFC (.vue files)

## Universal Questions (Answer First)

1. **Composition or Options API?** Check project convention. New projects: Composition API with `<script setup>`.
2. **What is reactive?** Identify all reactive state and its consumers.
3. **Props down, events up?** Verify unidirectional data flow.
4. **What triggers re-renders?** Map reactive dependencies explicitly.
5. **Is this shared state?** If yes, use Pinia. If no, keep it local.

## Script Setup (Default for New Code)

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/user'
import type { User } from '@/types'

// Props with TypeScript (with defaults)
const props = withDefaults(defineProps<{
  userId: string
  showAvatar?: boolean
  size?: 'sm' | 'md' | 'lg'
  variant?: 'primary' | 'secondary'
}>(), {
  showAvatar: false,
  size: 'md',
  variant: 'primary',
})

// Events with TypeScript
const emit = defineEmits<{
  select: [user: User]
  delete: [id: string]
}>()

// Reactive state
const isLoading = ref(false)
const error = ref<string | null>(null)

// Computed
const displayName = computed(() =>
  user.value ? `${user.value.firstName} ${user.value.lastName}` : ''
)

// Store
const userStore = useUserStore()

// Lifecycle
onMounted(async () => {
  await fetchUser()
})
</script>
```

## Reactivity Patterns

### ref vs reactive

| Use `ref` when | Use `reactive` when |
|----------------|---------------------|
| Primitive values (string, number, boolean) | Object with multiple related properties |
| Single value that changes | Form state with many fields |
| Need to reassign the whole value | Nested object you'll mutate |
| Returning from composables | Local component state group |

```typescript
// ref - primitives and reassignable values
const count = ref(0)
const user = ref<User | null>(null)

// reactive - object groups
const form = reactive({
  email: '',
  password: '',
  rememberMe: false,
})

// AVOID: reactive with primitives (loses reactivity on reassign)
// BAD: const state = reactive({ count: 0 }) then state = { count: 1 }
```

### Computed Properties

```typescript
// Simple computed
const fullName = computed(() => `${first.value} ${last.value}`)

// Writable computed
const fullName = computed({
  get: () => `${first.value} ${last.value}`,
  set: (val: string) => {
    const [f, l] = val.split(' ')
    first.value = f
    last.value = l ?? ''
  },
})

// AVOID: Side effects in computed
// BAD: computed(() => { api.track(); return count.value * 2 })
```

### Watchers

```typescript
// Watch single ref
watch(userId, async (newId, oldId) => {
  if (newId !== oldId) {
    await fetchUser(newId)
  }
})

// Watch multiple sources
watch([firstName, lastName], ([newFirst, newLast]) => {
  fullName.value = `${newFirst} ${newLast}`
})

// Immediate watch (runs on mount)
watch(route.params, (params) => {
  loadData(params.id as string)
}, { immediate: true })

// watchEffect - auto-tracks dependencies
watchEffect(async () => {
  const data = await fetch(`/api/users/${userId.value}`)
  userData.value = await data.json()
})

// Cleanup side effects
watchEffect((onCleanup) => {
  const controller = new AbortController()
  fetchData(userId.value, { signal: controller.signal })
  onCleanup(() => controller.abort())
})
```

## Composables (The Vue Way to Share Logic)

```typescript
// composables/useAsync.ts
import { ref, type Ref } from 'vue'

interface UseAsyncReturn<T> {
  data: Ref<T | null>
  error: Ref<string | null>
  isLoading: Ref<boolean>
  execute: () => Promise<void>
}

export function useAsync<T>(fn: () => Promise<T>): UseAsyncReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<string | null>(null)
  const isLoading = ref(false)

  async function execute() {
    isLoading.value = true
    error.value = null
    try {
      data.value = await fn()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
    } finally {
      isLoading.value = false
    }
  }

  return { data, error, isLoading, execute }
}
```

**Composable naming convention:** Always prefix with `use` (e.g., `useAuth`, `useForm`, `useSearch`).

**Composable rules:**
- Must be called in `setup()` or `<script setup>` (not in callbacks)
- Return refs/reactive objects (not raw values) to maintain reactivity
- Handle cleanup in `onUnmounted` or `watchEffect` cleanup

## Pinia State Management

```typescript
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types'

export const useUserStore = defineStore('user', () => {
  // State
  const currentUser = ref<User | null>(null)
  const isAuthenticated = computed(() => currentUser.value !== null)

  // Actions
  async function login(email: string, password: string) {
    const response = await api.login({ email, password })
    currentUser.value = response.user
  }

  function logout() {
    currentUser.value = null
  }

  return {
    currentUser,
    isAuthenticated,
    login,
    logout,
  }
})
```

**Pinia rules:**
- Use Setup Store syntax (function style) for TypeScript projects
- One store per domain concept (user, cart, notifications)
- Never mutate store state from components directly — use actions
- Use `storeToRefs()` when destructuring to maintain reactivity

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()
// CORRECT: storeToRefs preserves reactivity
const { currentUser, isAuthenticated } = storeToRefs(userStore)
// Actions don't need storeToRefs
const { login, logout } = userStore

// BAD: Destructuring without storeToRefs loses reactivity
// const { currentUser } = userStore  // NOT reactive!
</script>
```

## Component Patterns

### v-model on Custom Components

```vue
<!-- Parent -->
<SearchInput v-model="searchQuery" v-model:filter="activeFilter" />

<!-- SearchInput.vue (Vue 3.4+) -->
<script setup lang="ts">
// defineModel requires Vue 3.4+
const model = defineModel<string>()
const filter = defineModel<string>('filter')
</script>

<template>
  <input :value="model" @input="model = ($event.target as HTMLInputElement).value" />
  <select :value="filter" @change="filter = ($event.target as HTMLSelectElement).value">
    <option value="all">All</option>
    <option value="active">Active</option>
  </select>
</template>
```

### Slots Pattern

```vue
<!-- DataTable.vue -->
<template>
  <table>
    <thead>
      <tr>
        <slot name="header" />
      </tr>
    </thead>
    <tbody>
      <tr v-for="(item, index) in items" :key="item.id">
        <slot name="row" :item="item" :index="index" />
      </tr>
    </tbody>
    <tfoot v-if="$slots.footer">
      <slot name="footer" :total="items.length" />
    </tfoot>
  </table>
</template>

<!-- Usage with scoped slots -->
<DataTable :items="users">
  <template #header>
    <th>Name</th>
    <th>Email</th>
  </template>
  <template #row="{ item }">
    <td>{{ item.name }}</td>
    <td>{{ item.email }}</td>
  </template>
</DataTable>
```

### Provide/Inject (Dependency Injection)

```typescript
// Parent provides
import { provide, ref } from 'vue'
import type { InjectionKey } from 'vue'

export const ThemeKey: InjectionKey<Ref<'light' | 'dark'>> = Symbol('theme')

// In parent setup
const theme = ref<'light' | 'dark'>('light')
provide(ThemeKey, theme)

// Child injects
import { inject } from 'vue'
import { ThemeKey } from '@/keys'

const theme = inject(ThemeKey)
if (!theme) throw new Error('ThemeKey not provided')
```

**Provide/Inject rules:**
- Use `InjectionKey<T>` for type safety
- Always handle missing injection (throw or provide default)
- Prefer Pinia for global state; use provide/inject for component tree scoping

## Vue Router Patterns

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/dashboard',
      component: () => import('@/views/Dashboard.vue'),
      meta: { requiresAuth: true },
    },
    {
      path: '/user/:id',
      component: () => import('@/views/UserProfile.vue'),
      props: true, // Pass route params as props
    },
  ],
})

// Navigation guard
router.beforeEach((to) => {
  const userStore = useUserStore()
  if (to.meta.requiresAuth && !userStore.isAuthenticated) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})
```

**Router rules:**
- Always use lazy loading (`() => import(...)`) for route components
- Use `props: true` to pass params as props (testable, decoupled from router)
- Put auth logic in navigation guards, not components
- Use `meta` for route-level configuration (auth, layout, breadcrumbs)

## Loading State Pattern (Vue)

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const data = ref<Item[] | null>(null)
const error = ref<string | null>(null)
const isLoading = ref(true)

onMounted(async () => {
  try {
    data.value = await fetchItems()
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Failed to load'
  } finally {
    isLoading.value = false
  }
})
</script>

<template>
  <!-- Error first -->
  <ErrorState v-if="error" :message="error" @retry="fetchItems" />
  <!-- Loading (no data yet) -->
  <LoadingSkeleton v-else-if="isLoading && !data" />
  <!-- Empty state -->
  <EmptyState v-else-if="data && !data.length" />
  <!-- Data -->
  <ItemList v-else-if="data" :items="data" />
</template>
```

**State order matters:** Error -> Loading (no data) -> Empty -> Success. Same as any frontend framework.

## Form Patterns

```vue
<script setup lang="ts">
import { reactive, ref } from 'vue'

const form = reactive({
  email: '',
  password: '',
  rememberMe: false,
})

const errors = ref<Record<string, string>>({})
const isSubmitting = ref(false)

function validate(): boolean {
  errors.value = {}
  if (!form.email.trim()) errors.value.email = 'Email is required'
  if (!form.password) errors.value.password = 'Password is required'
  return Object.keys(errors.value).length === 0
}

async function handleSubmit() {
  if (!validate()) return
  isSubmitting.value = true
  try {
    await api.login(form)
  } catch (e) {
    errors.value.general = 'Login failed. Please try again.'
  } finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" novalidate>
    <div>
      <label for="email">Email</label>
      <input
        id="email"
        v-model.trim="form.email"
        type="email"
        autocomplete="email"
        :aria-invalid="!!errors.email"
        :aria-describedby="errors.email ? 'email-error' : undefined"
      />
      <span v-if="errors.email" id="email-error" role="alert">
        {{ errors.email }}
      </span>
    </div>

    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'Signing in...' : 'Sign in' }}
    </button>
  </form>
</template>
```

## Performance Patterns

| Pattern | When | How |
|---------|------|-----|
| **`v-once`** | Static content that never changes | `<h1 v-once>{{ title }}</h1>` |
| **`v-memo`** | List items where few change | `<div v-for="item in list" v-memo="[item.id, item.selected]">` |
| **`shallowRef`** | Large objects where deep reactivity is wasteful | `const bigList = shallowRef<Item[]>([])` |
| **`defineAsyncComponent`** | Heavy components not needed at initial load | `const Chart = defineAsyncComponent(() => import('./Chart.vue'))` |
| **`<KeepAlive>`** | Tab content, wizard steps that should preserve state | `<KeepAlive><component :is="currentTab" /></KeepAlive>` |
| **`computed` over method** | Derived values used in template | Computed caches; methods re-run every render |

### List Rendering Performance

```vue
<!-- ALWAYS use :key with v-for -->
<div v-for="item in items" :key="item.id">
  {{ item.name }}
</div>

<!-- NEVER use index as key when list can reorder -->
<!-- BAD: :key="index" -->

<!-- Use v-show for frequent toggles, v-if for rare -->
<ExpensiveComponent v-show="isVisible" />  <!-- Keeps in DOM -->
<ExpensiveComponent v-if="isVisible" />    <!-- Mounts/unmounts -->
```

## Nuxt.js Patterns

```vue
<!-- pages/users/[id].vue -->
<script setup lang="ts">
// Auto-imported composables
const route = useRoute()
const { data: user, error } = await useFetch<User>(
  `/api/users/${route.params.id}`
)

// SEO
useHead({
  title: () => user.value?.name ?? 'User Profile',
})
</script>
```

**Nuxt rules:**
- Use `useFetch` or `useAsyncData` for data fetching (SSR-aware)
- Never use raw `fetch` in setup (breaks SSR hydration)
- Use `server/api/` for API routes
- Use `definePageMeta` for page-level configuration

## Error Handling

```typescript
// Global error handler (main.ts)
const app = createApp(App)
app.config.errorHandler = (err, instance, info) => {
  console.error('Global error:', err, info)
  // Report to error tracking service
}

// Component-level error capture
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)
onErrorCaptured((err) => {
  error.value = err
  return false // Prevent propagation
})
```

**Error handling rules:**
- Set `app.config.errorHandler` for global uncaught errors
- Use `onErrorCaptured` for error boundary components
- Always handle async errors in `watchEffect`/`watch` callbacks with try/catch
- Use `nextTick()` when focusing error elements after reactive updates

## Accessibility

**For comprehensive accessibility patterns (WCAG, keyboard navigation, ARIA, color contrast), see the `frontend-patterns` skill.** Below are Vue-specific accessibility considerations:

- Use `:aria-*` bindings for dynamic ARIA attributes tied to reactive state
- Use `nextTick()` to manage focus after DOM updates (e.g., after showing a modal)
- Add `aria-live="polite"` to regions that update reactively (e.g., notification areas, live counters)
- Use `<Teleport to="body">` for modals/dialogs to ensure correct focus trapping and screen reader flow
- Test with `vue-axe` in development for automatic a11y audits

```vue
<!-- Focus management after reactive update -->
<script setup lang="ts">
import { ref, nextTick } from 'vue'

const showModal = ref(false)
const modalRef = ref<HTMLElement | null>(null)

async function openModal() {
  showModal.value = true
  await nextTick()
  modalRef.value?.focus()
}
</script>

<template>
  <Teleport to="body">
    <div v-if="showModal" ref="modalRef" role="dialog" aria-modal="true" tabindex="-1">
      <!-- Modal content -->
    </div>
  </Teleport>
</template>
```

## FormKit (Schema-Based Forms)

**When to use:** Projects with `@formkit/*` in package.json. FormKit replaces manual form wiring with declarative, schema-driven forms that handle validation, error display, and accessibility out of the box.

### Schema-Based Form Generation

```vue
<script setup lang="ts">
import { FormKitSchema } from '@formkit/vue'
import type { FormKitSchemaNode } from '@formkit/core'

const schema: FormKitSchemaNode[] = [
  {
    $formkit: 'text',
    name: 'email',
    label: 'Email',
    validation: 'required|email',
  },
  {
    $formkit: 'password',
    name: 'password',
    label: 'Password',
    validation: 'required|length:8',
  },
  {
    $formkit: 'submit',
    label: 'Sign in',
  },
]

async function handleSubmit(data: Record<string, unknown>) {
  await api.login(data)
}
</script>

<template>
  <FormKit type="form" @submit="handleSubmit" :actions="false">
    <FormKitSchema :schema="schema" />
  </FormKit>
</template>
```

### Validation: Built-in Rules + Custom Async

```typescript
// formkit.config.ts
import { createInput, defineFormKitConfig } from '@formkit/vue'
import { createValidationPlugin } from '@formkit/validation'

export default defineFormKitConfig({
  rules: {
    // Custom async validation (e.g., check email uniqueness)
    unique_email: async ({ value }) => {
      const res = await fetch(`/api/check-email?email=${value}`)
      const { available } = await res.json()
      return available
    },
  },
  messages: {
    en: {
      validation: {
        unique_email: 'This email is already registered',
      },
    },
  },
})
```

```vue
<!-- Usage: built-in + custom rule chaining -->
<FormKit
  type="email"
  name="email"
  label="Email"
  validation="required|email|unique_email"
  :validation-rules="{ unique_email }"
  validation-visibility="blur"
/>
```

### TypeScript: Augment FormKitInputProps

```typescript
// types/formkit.d.ts
declare module '@formkit/inputs' {
  interface FormKitInputProps<Props extends FormKitInputs<Props>> {
    'currency-input': {
      type: 'currency-input'
      value?: number
      currency?: 'USD' | 'EUR' | 'GBP'
      min?: number
      max?: number
    }
  }
}
```

### Multi-Step Form with Group Validation

```vue
<FormKit type="form" @submit="submitApp">
  <FormKit type="multi-step">
    <FormKit type="step" name="personal">
      <FormKit type="text" name="name" label="Name" validation="required" />
      <FormKit type="email" name="email" label="Email" validation="required|email" />
    </FormKit>
    <FormKit type="step" name="address">
      <FormKit type="text" name="street" label="Street" validation="required" />
      <FormKit type="text" name="city" label="City" validation="required" />
    </FormKit>
  </FormKit>
</FormKit>
```

### Error Handling: setErrors / clearErrors

```typescript
const submitApp = async (data: Record<string, unknown>, node: FormKitNode) => {
  try {
    await api.submit(data)
    node.clearErrors()
  } catch (err) {
    // Form-level errors + field-level errors
    node.setErrors(
      ['Submission failed. Please try again.'],       // form errors
      { email: 'This email is already registered' }   // field errors
    )
  }
}
```

### FormKit Anti-Patterns

| Anti-pattern | Why It's Wrong | Fix |
|--------------|----------------|-----|
| Manual `v-model` + custom validation alongside FormKit | Bypasses FormKit's validation lifecycle | Use FormKit's `validation` prop and rules |
| Calling `node.submit()` to trigger validation | Submits the form, not just validates | Use `node.validate()` for validation-only checks |
| Watching FormKit values with Vue `watch` for side effects | Race conditions with FormKit's internal state | Use FormKit's `@input` event or `node.on('input', ...)` |
| Ignoring `onCleanup` in Vue 3.5+ with FormKit plugins | Leaked event listeners on HMR / component unmount | Call `node.destroy()` in `onUnmounted` for custom node references |

**Cross-reference:** For general form UX patterns (autocomplete attributes, paste handling, touch targets), see the `frontend-patterns` skill.

---

## PrimeVue (Component Library)

**When to use:** Projects with `primevue` in package.json. PrimeVue provides 80+ pre-built UI components with built-in theming, accessibility, and Tailwind CSS integration.

### Nuxt 3 Module Configuration

```typescript
// nuxt.config.ts
import Aura from '@primevue/themes/aura'

export default defineNuxtConfig({
  modules: ['@primevue/nuxt-module'],
  primevue: {
    autoImport: true,
    options: {
      theme: {
        preset: Aura,
        options: {
          darkModeSelector: '.dark',  // align with Tailwind's dark mode
        },
      },
      ripple: true,
      inputVariant: 'filled',
    },
    components: {
      prefix: 'Prime',              // <PrimeButton>, <PrimeDataTable>
    },
  },
})
```

### Design Token Theming (3-Tier System)

PrimeVue uses a 3-tier token hierarchy: **primitive** (raw values) → **semantic** (meaning) → **component** (specific overrides).

```typescript
// theme/preset.ts
import { definePreset } from '@primeuix/styled'
import Aura from '@primevue/themes/aura'

const AppPreset = definePreset(Aura, {
  // Semantic tokens: map meaning to primitive values
  semantic: {
    primary: {
      50: '{indigo.50}',
      100: '{indigo.100}',
      // ... 200-800
      900: '{indigo.900}',
      950: '{indigo.950}',
    },
    colorScheme: {
      light: {
        primary: {
          color: '{primary.500}',
          contrastColor: '#ffffff',
          hoverColor: '{primary.600}',
          activeColor: '{primary.700}',
        },
      },
      dark: {
        primary: {
          color: '{primary.400}',
          contrastColor: '{surface.900}',
          hoverColor: '{primary.300}',
          activeColor: '{primary.200}',
        },
      },
    },
  },
  // Component tokens: override specific component styles
  components: {
    button: {
      root: {
        borderRadius: '{border.radius.lg}',
      },
    },
  },
})

export default AppPreset
```

### Tailwind CSS Integration

**Styled mode (recommended):** PrimeVue ships pre-skinned. Set CSS layer order so Tailwind utilities can override component styles:

```css
/* assets/css/main.css */
@layer tailwind-base, primevue, tailwind-utilities;

@import "tailwindcss/base" layer(tailwind-base);
@import "tailwindcss/utilities" layer(tailwind-utilities);
```

**Unstyled mode + Pass Through (full Tailwind control):**

```typescript
// main.ts
app.use(PrimeVue, {
  unstyled: true,
  pt: {
    button: {
      root: 'bg-primary-500 hover:bg-primary-600 text-white font-semibold py-2 px-4 rounded-lg',
      label: 'font-bold',
    },
    panel: {
      header: 'bg-surface-100 dark:bg-surface-800 border-b border-surface-200',
      content: 'p-4',
    },
  },
})
```

### Dark Mode Alignment

```typescript
// Ensure PrimeVue and Tailwind use the same dark mode trigger
// nuxt.config.ts or main.ts
theme: {
  preset: AppPreset,
  options: {
    darkModeSelector: '.dark',  // matches Tailwind: darkMode: 'class'
  },
}
```

### PrimeVue Anti-Patterns

| Anti-pattern | Why It's Wrong | Fix |
|--------------|----------------|-----|
| Overriding PrimeVue styles with CSS class selectors | Brittle, breaks on version updates, fights specificity | Use design tokens or Pass Through API (`pt` prop) |
| Using `.p-dark` for dark mode while Tailwind uses `.dark` | Inconsistent dark mode — components and utilities diverge | Set `darkModeSelector: '.dark'` to match Tailwind |
| Importing all components globally | Bundle size bloat | Use `autoImport: true` with Nuxt module or tree-shake with specific imports |
| Mixing styled and unstyled modes | Conflicting CSS layers, unpredictable rendering | Choose one: styled (with token overrides) or unstyled (with PT) |
| Skipping CSS layer order with Tailwind | Tailwind utilities can't override PrimeVue component styles | Define explicit `@layer` order: `tailwind-base, primevue, tailwind-utilities` |

**Cross-reference:** For design token architecture and Tailwind design system patterns, see the `tailwind-design-system` skill. For creative visual design and distinctive UI, see the `frontend-design` skill.

---

## Red Flags - STOP and Reconsider

If you find yourself:

- Mutating props directly
- Using `reactive()` and then reassigning the variable
- Putting business logic in templates (complex expressions)
- Using `this` in `<script setup>` (it doesn't exist there)
- Skipping `:key` in `v-for` loops
- Using `$forceUpdate()` (you broke reactivity somewhere)
- Nesting v-if inside v-for on the same element
- Importing `ref`/`reactive` in a Nuxt project (they're auto-imported)

**STOP. Revisit the reactive design.**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Options API is simpler" | For small components. Composition API scales better and types better. |
| "We don't need Pinia, just use reactive" | Shared state without Pinia = invisible dependencies. |
| "v-html is fine here" | XSS vector. Sanitize or use text interpolation. |
| "Index keys work for this list" | Until the list reorders. Use stable IDs. |
| "We'll add TypeScript later" | Retrofitting types is 10x harder. Start typed. |
| "Watchers are fine for everything" | Prefer computed for derived values. Watch is for side effects. |

## Anti-patterns Blocklist

| Anti-pattern | Why It's Wrong | Fix |
|--------------|----------------|-----|
| Mutating props | Breaks one-way data flow | Emit event, let parent update |
| `v-html` with user input | XSS vulnerability | Use text interpolation or sanitize |
| Deeply nested `v-if`/`v-else` chains | Unreadable templates | Extract to computed or child components |
| `$refs` for parent-child communication | Tight coupling | Use props/emits or provide/inject |
| Giant `<script setup>` blocks (200+ lines) | Unmaintainable | Extract composables |
| Reactive state outside components | Memory leaks, shared state bugs | Use Pinia stores |
| `any` in defineProps/defineEmits | Loses type safety | Define proper interfaces |

## Output Format

```markdown
## Vue Review: [Component/Feature]

### Reactive Boundaries
[What state exists, what triggers re-renders]

### Component Structure
- [ ] Uses `<script setup>` with TypeScript
- [ ] Props typed with `defineProps<T>()`
- [ ] Events typed with `defineEmits<T>()`
- [ ] Composables extracted for reusable logic
- [ ] State management appropriate (local vs Pinia)

### Issues
| Severity | Issue | Location | Fix |
|----------|-------|----------|-----|
| [BLOCKS/IMPAIRS/MINOR] | [Issue] | `file:line` | [Fix] |

### Recommendations
1. [Most critical]
2. [Second]
```

## Final Check

Before completing Vue work:

- [ ] `<script setup lang="ts">` used with typed props/emits
- [ ] Reactive boundaries mapped (ref vs reactive vs computed)
- [ ] No prop mutation (props down, events up)
- [ ] Eager loading handled (no watchers where computed works)
- [ ] Pinia used for shared state (not reactive outside components)
- [ ] `:key` on all `v-for` loops (stable IDs, not indexes)
- [ ] Error states handled (onErrorCaptured or try/catch)
- [ ] Loading states shown for async operations
- [ ] Accessibility verified (see `frontend-patterns` skill)
- [ ] No `v-html` with unsanitized user input

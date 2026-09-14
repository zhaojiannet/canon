---
name: vue
description: Enforce Vue 3.5+ single-file component conventions with <script setup>, Composition API only, type-safe defineProps/defineEmits, defineModel, useTemplateRef. Use when editing .vue files or when the user mentions Vue component, props, emits, v-model, composable, or script setup. Rewrites Options API and mixins into their Composition API equivalents, and types every prop and emit.
paths:
  - "**/*.vue"
allowed-tools:
  - Read
  - Grep
---

> Targets Vue 3.5+ · verified 2026-09 (latest 3.5.42).

This skill enforces Vue 3.5+ single-file component conventions. Composition API + `<script setup>` only; type-safe declarations; modern macros (`defineModel` / `useTemplateRef` / `useId`).

Apply only when the project uses `vue ^3.5` or higher. If `package.json` pins `vue ^3.4` or earlier, **STOP** and ask the user — 3.5+ APIs (`useId`, `useTemplateRef`, reactive props destructure, `onWatcherCleanup`) won't work. Do not write 2.x compatibility code.

## Core principles

- **`<script setup>` only**, no Options API (`data()` / `methods` / `computed: {}` / `watch: {}` / `mounted` / `mixins`).
- **TypeScript type declarations for props/emits**, not runtime `defineProps([...])` array or `PropType` imports.
- **`defineModel()` for `v-model`**, not manual `props/emits` pairs.
- **`useTemplateRef()` for template refs** (Vue 3.5+), not the legacy static-ref binding.
- **`useId()` for stable IDs** in form/a11y, not `Math.random()` / module-level counters.
- **Reactive props destructure** (Vue 3.5+) over `withDefaults()` / `props.foo`.
- **Composables for shared logic**, not mixins.

## Required form

```vue
<script setup lang="ts">
import { computed, useTemplateRef, useId } from 'vue'

// Props (type declaration + reactive destructure with defaults)
const { items, selected = [] } = defineProps<{
  items: Item[]
  selected?: string[]
}>()

// Emits (type declaration)
const emit = defineEmits<{
  change: [id: string]
  delete: [id: string]
}>()

// v-model (3.4+)
const modelValue = defineModel<string>()

// Slots type hints
const slots = defineSlots<{
  default(props: { item: Item }): any
  header(props: { count: number }): any
}>()

// Template ref (3.5+)
const inputRef = useTemplateRef<HTMLInputElement>('input')

// Stable IDs (3.5+)
const id = useId()
</script>

<template>
  <input :id ref="input" />
</template>
```

## Forbidden patterns

### Component definition

- Options API: `export default { data() / methods / computed / watch / mounted / mixins }` — use `<script setup>`.
- `defineComponent({ ... })` — only legitimate inside non-SFC `.ts/.js` files for type inference; never inside SFC.
- Plain `<script>` (without `setup`) for things expressible in `<script setup>` (props, emits, options).
- Mixins (`mixins: [...]`) — extract into a composable (`useFoo()`).
- Global filters / global directives registered via `app.directive` for one-off use — define locally as `vMyDirective` in `<script setup>`.

### Props / Emits

- Runtime array form `defineProps(['foo', 'bar'])` — use type declaration.
- `PropType<T>` imports — type declaration replaces it.
- `withDefaults(defineProps<T>(), { ... })` when project is on Vue 3.5+ — use reactive destructure with native defaults: `const { foo = 'x' } = defineProps<T>()`.
- Passing a destructured prop directly into a function (`watch(foo, ...)` / `useComposable(foo)`) — it passes a value, not a reactive source. Wrap it in a getter: `watch(() => foo, ...)` / `useComposable(() => foo)`.
- Untyped emits: `defineEmits(['change'])` — type declaration: `defineEmits<{ change: [id: string] }>()`.
- Manual `props.modelValue` + `emit('update:modelValue', ...)` for v-model — use `defineModel()`.

### Refs / state

- Static template ref name binding (`const foo = ref(); ... <div ref="foo">`) when on 3.5+ — use `useTemplateRef('foo')`.
- `Math.random()` / `Date.now()` / module-level `let id = 0` for component IDs — use `useId()`.
- `reactive()` for primitive-heavy state where `ref()` reads cleaner — both are valid; prefer `ref()` unless you have a deeply nested object.
- `this.$refs` / `this.$emit` syntax — that's Options API.

### Watchers / effects

- Side effects in `mounted` / `beforeMount` Options hooks — use `onMounted()` / `watchEffect()`.
- Async cleanup logic via flag tracking — use `onWatcherCleanup()` (3.5+).
- Hand-written composable (`useXxx`) when the project already exposes one under `composables/` (or auto-imported via Nuxt). grep `composables/` and `app/composables/` first; reuse if found.

## Component decomposition

Single-file components grow brittle past a threshold. When a `.vue` file approaches **400 lines**, evaluate splitting:

- **Extract reusable UI subtree** → child `.vue` with its own props/emits.
- **Extract shared state/logic** (data fetch, form state, derived values) → composable (`useXxx()` in `composables/` or `~/composables/`).
- **Extract pure helper functions** (no reactivity) → plain `.ts` utility module.

When approaching **800 lines**, splitting is mandatory before adding more logic. Report and ask the user which slice to extract first if it's not obvious.

## Imports

Nuxt projects auto-import `ref` / `computed` / `watch` / `useFetch` / etc. — do not add explicit imports for auto-imported APIs. Plain Vue (Vite) projects must import from `'vue'` explicitly.

## When you can't follow these rules

If the codebase has Options API legacy components and you must edit them, **STOP** and report:

> File [path] uses Options API. Approve one of: (A) leave as-is, change minimally in Options style; (B) refactor to `<script setup>` first (provide before/after for the relevant section); (C) extract the change to a child component using `<script setup>`.

Do not silently mix Options-style patches into a `<script setup>` file or vice versa.

## Verification (grep after every .vue change)

```bash
# Options API leftovers
grep -rE 'export default \{' --include='*.vue' .          # → migrate to <script setup>
grep -rE 'data\(\)\s*\{' --include='*.vue' .              # Options data
grep -rE 'methods:\s*\{' --include='*.vue' .              # Options methods
grep -rE 'mixins:\s*\[' --include='*.vue' .               # replace with composable
grep -rE 'PropType' --include='*.vue' .                   # replace with type declaration

# legacy v-model pattern
grep -rE "emit\('update:" --include='*.vue' .             # check if it can move to defineModel

# string ref
grep -rlE '(^|[[:space:]])ref="[^"]+"' --include='*.vue' . | tr '\n' '\0' | xargs -0 grep -L 'useTemplateRef'  # files with static template refs but no useTemplateRef

# legacy props pattern
grep -rE 'defineProps\(\[' --include='*.vue' .            # array form — switch to type declaration
grep -rE 'withDefaults\(' --include='*.vue' .             # 3.5+ should use destructuring defaults

# SFC over 400 lines
find . -name '*.vue' -not -path '*/node_modules/*' -exec wc -l {} \; | awk '$1>400 {print}'
```

Full API: https://vuejs.org/api/sfc-script-setup

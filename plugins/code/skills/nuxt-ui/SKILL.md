---
name: nuxt-ui
description: Enforce Nuxt UI v4 component-first conventions. Use this skill when editing .vue files in Vue/Nuxt projects, or when the user mentions Nuxt UI, button, input, modal, card, drawer, dropdown, tooltip, form, table, toast, slideover, popover, navigation, dashboard, U-prefix component, or asks to build any UI element. Forbids hand-written raw <button>/<input>/<select>/<dialog>/<table> when Nuxt UI provides an equivalent. Forces U-prefix components and STOP-and-ask when no Nuxt UI component fits.
paths:
  - "**/*.vue"
allowed-tools:
  - Read
  - Grep
---

This skill enforces Nuxt UI v4 component-first conventions. The rule: when Nuxt UI provides a component, use it instead of hand-writing raw HTML.

Apply only when the project uses `@nuxt/ui ^4.0` or higher. If `package.json` pins an older major (or doesn't have `@nuxt/ui`), **STOP** and ask the user before applying — do not assume earlier-version APIs.

## Core principles

- **Use the U-prefixed component** if Nuxt UI offers one for the element (button → `UButton`, input → `UInput`, modal → `UModal`, etc.). Do not hand-write raw `<button>` / `<input>` / `<select>` / `<dialog>` / `<table>` for primary use cases.
- **Compose via Nuxt UI slots and props**, not by overriding internal classes. Use `class` for layout/spacing utilities; use component-specific props (`color`, `variant`, `size`, `loading`, `disabled`) for behavior and look.
- **Forms use `UForm` + `UFormField`**, not raw `<form>` with manual error wiring. `UForm` integrates with Zod/Valibot/Yup for validation.
- **Overlays use Nuxt UI's overlay components** (`UModal` / `USlideover` / `UDrawer` / `UPopover` / `UTooltip` / `UContextMenu` / `UDropdownMenu` / `UToast`), not custom `<dialog>` or hand-rolled show/hide state.
- **App shell uses `UApp`** at the root, with `UContainer` / `UHeader` / `UMain` / `UFooter` / `USidebar` for layout structure where appropriate.

## Component reference (use these, not raw HTML)

### Form
`UCheckbox` `UCheckboxGroup` `UColorPicker` `UFileUpload` `UForm` `UFormField` `UInput` `UInputDate` `UInputMenu` `UInputNumber` `UInputTags` `UInputTime` `UListbox` `UPinInput` `URadioGroup` `USelect` `USelectMenu` `USlider` `USwitch` `UTextarea`

### Layout
`UApp` `UContainer` `UError` `UFooter` `UHeader` `UMain` `USidebar` `UTheme`

### Navigation
`UBreadcrumb` `UCommandPalette` `UFooterColumns` `ULink` `UNavigationMenu` `UPagination` `UStepper` `UTabs`

### Overlay
`UContextMenu` `UDrawer` `UDropdownMenu` `UModal` `UPopover` `USlideover` `UToast` `UToaster` `UTooltip`

### Data
`UAccordion` `UCarousel` `UEmpty` `UMarquee` `UScrollArea` `UTable` `UTimeline` `UTree` `UUser`

### Feedback / Element
`UAlert` `UAvatar` `UAvatarGroup` `UBadge` `UBanner` `UButton` `UCalendar` `UCard` `UChip` `UCollapsible` `UFieldGroup` `UIcon` `UKbd` `UProgress` `USeparator` `USkeleton`

### Specialized

- Page (marketing): `UPage` / `UPageBody` / `UPageHero` / `UPageHeader` / `UPageSection` / `UPageCTA` / `UPageCard` / `UPageGrid` / `UPageColumns` / `UPageFeature` / `UPageList` / `UPageLogos` / `UPageLinks` / `UPageAside` / `UPageAnchors` / `UAuthForm` / `UBlogPost` / `UBlogPosts` / `UChangelogVersion` / `UChangelogVersions`
- Pricing: `UPricingPlan` / `UPricingPlans` / `UPricingTable`
- Dashboard: `UDashboardGroup` / `UDashboardNavbar` / `UDashboardPanel` / `UDashboardResizeHandle` / `UDashboardSearch` / `UDashboardSearchButton` / `UDashboardSidebar` / `UDashboardSidebarCollapse` / `UDashboardSidebarToggle` / `UDashboardToolbar`
- AI Chat: `UChatMessage` / `UChatMessages` / `UChatPalette` / `UChatPrompt` / `UChatPromptSubmit` / `UChatReasoning` / `UChatShimmer` / `UChatTool`
- Editor: `UEditor` / `UEditorDragHandle` / `UEditorEmojiMenu` / `UEditorMentionMenu` / `UEditorSuggestionMenu` / `UEditorToolbar`
- Content: `UContentNavigation` / `UContentSearch` / `UContentSearchButton` / `UContentSurround` / `UContentToc`
- Color Mode: `UColorModeAvatar` / `UColorModeButton` / `UColorModeImage` / `UColorModeSelect` / `UColorModeSwitch`
- i18n: `ULocaleSelect`
- Prose (for `@nuxt/content` markdown rendering — 43 components mapped to markdown elements, prefixed `UProse*`, typically auto-applied)

## Forbidden patterns

- Raw `<button>` for any clickable trigger — use `UButton` (props: `color`, `variant`, `size`, `loading`, `disabled`, `to`, `icon`, `trailing-icon`)
- Raw `<input>` / `<textarea>` / `<select>` for form fields — use `UInput` / `UTextarea` / `USelect` / `USelectMenu`
- Raw `<dialog>` / hand-rolled "isOpen" boolean overlays — use `UModal` / `USlideover` / `UDrawer`
- Raw `<table>` with hand-coded sorting/filtering — use `UTable` (built-in column definition, sort, filter, pagination)
- Manual notifications via `<div class="toast">` — use `UToast` via the `useToast()` composable
- Hand-coded breadcrumbs / pagination / tabs — use `UBreadcrumb` / `UPagination` / `UTabs`
- Hand-coded form validation logic — use `UForm` with a schema (Zod / Valibot / Yup) bound via `:schema`
- Hand-coded color-mode toggle — use `UColorModeButton`
- Overriding Nuxt UI internal class names with `:deep()` selectors — use the component's official `ui` prop / `slots` / `class` / variant props
- Custom CSS for component states (`hover` / `active`) when the component already exposes them via prop or variant
- Hand-written wrapper component (e.g. `<MyButton>` that just re-styles `UButton`) when the project already has one under `components/` or `app/components/`. grep first; reuse if found.

## Installation reference (v4)

Entry CSS:
```css
@import "tailwindcss";
@import "@nuxt/ui";
```

`nuxt.config.ts`:
```ts
export default defineNuxtConfig({
  modules: ['@nuxt/ui'],
  css: ['~/assets/css/main.css']
})
```

`@nuxt/icon` / `@nuxt/fonts` / `@nuxtjs/color-mode` are auto-registered. Configure via `icon` / `fonts` / `colorMode` keys.

Component prefix (default `U`) configurable via `ui.prefix`.

## When no Nuxt UI component fits

Do not silently fall back to raw HTML. **STOP** and report:

> Need a [component description] component. Nuxt UI candidates checked: [list, e.g., UCard / UDrawer]. None fits because [specific gap]. Approve one of: (A) use a Nuxt UI component with custom slots / `ui` prop overrides, (B) compose two Nuxt UI components, (C) use a third-party Vue component (specify), (D) hand-write — only with explicit approval.

## Verification (grep after every .vue change)

```bash
# should be replaced with Nuxt UI equivalents
grep -rE '<button[^>]*>' --include='*.vue' .              # → UButton
grep -rE '<input(?!\s+v-bind)' --include='*.vue' .        # → UInput
grep -rE '<select[^>]*>' --include='*.vue' .              # → USelect / USelectMenu
grep -rE '<textarea[^>]*>' --include='*.vue' .            # → UTextarea
grep -rE '<dialog[^>]*>' --include='*.vue' .              # → UModal
grep -rE '<table[^>]*>' --include='*.vue' .               # → UTable
grep -rE 'v-if="(isOpen|showModal|dialogOpen)"' --include='*.vue' .  # → UModal v-model

# anti-patterns that should not appear
grep -rE ':deep\(\.u-' --include='*.vue' .                # internal class override — use the ui prop
grep -rE 'class="(toast|modal|dropdown)' --include='*.vue' .  # custom component name collision
```

Full component API: https://ui.nuxt.com

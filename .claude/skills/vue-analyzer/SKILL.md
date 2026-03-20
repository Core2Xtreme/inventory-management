---
name: vue-analyzer
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Use when asked to audit, review, or optimize Vue components.
---

# Vue Component Analyzer

Analyze Vue 3 components for performance issues and code reuse opportunities. Follow this structured process for every analysis.

## How to Run an Analysis

1. Read the target `.vue` file(s) with the Read tool
2. Check `client/src/api.js` and `client/src/App.vue` for shared patterns
3. Scan for issues using the checklist below
4. Report findings grouped by severity: **High → Medium → Low**
5. For each issue, provide: file path + line number, the problem, and a concrete fix

---

## Checklist

### Performance

#### Computed vs Method
Flag any method called inside a template (`v-for`, interpolation, `:bind`) that:
- Does not depend on event arguments (i.e., could be a computed property)
- Creates objects or maps on every call (e.g., `const map = { ... }` inside the function body)
- Is called more than once per render

**Pattern to catch:**
```js
// BAD - recreates map on every render cycle
function translateCategory(cat) {
  const categoryMap = { 'Circuit Boards': '...', 'Sensors': '...' }
  return categoryMap[cat] ?? cat
}
```
**Fix:** Move to a `computed` that returns a lookup map; call `.value[cat]` in the template.

#### Expensive Computed Properties
Flag computed properties that do O(n²) work:
- `.find()` or `.filter()` inside a `.map()` or `.forEach()`
- Rebuilding the same derived structure (e.g., month → value map) every time

**Fix:** Build a lookup `Map` or object once, then reference it in the O(n) pass.

#### Unnecessary Watchers
Flag `watch()` calls where:
- The callback body is empty
- The callback only triggers something already covered by a computed dependency

### Reactivity

#### Index Keys in v-for
Flag `:key="index"` or `:key="idx"` on any `v-for`. This breaks reconciliation when list items reorder or change length.

**Fix:** Use a stable, unique field from the item (e.g., `:key="item.sku"`, `:key="item.id"`). If no unique field exists, note it as a data structure issue.

#### Direct Object/Array Mutation
Flag patterns like:
```js
someRef.value.array.splice(...)
someRef.value.obj.prop = newVal
```
When modifying nested reactive state, prefer immutable updates for clarity:
```js
someRef.value = { ...someRef.value, prop: newVal }
```

#### Calling Composables Inside Loops or Conditionals
Flag `useX()` or `getCurrentFilters()` called inside `v-for` render functions or conditional branches. Composables must be called at the top level of `setup()`.

### Code Reuse

#### Duplicated Logic Across Components
Cross-reference all open `.vue` files. Flag any function that appears in 2+ components with the same or near-identical body:

| Common duplicates in this codebase | Suggested extraction |
|------------------------------------|----------------------|
| `translateCategory()` | `composables/useCategoryMap.js` |
| `translateStockLevel()` / `getStockBadge()` | `composables/useStatusBadge.js` |
| `formatDate()` | `utils/formatDate.js` |
| `formatCurrency()` | `utils/formatCurrency.js` |
| Status → CSS class mappings | `utils/statusMap.js` |

#### Template Patterns Worth Extracting
Flag any template block that:
- Is longer than ~15 lines and appears in more than one component, OR
- Contains complex logic (nested `v-if`, `v-for` with sub-loops) that would benefit from encapsulation

**Example from this codebase:** The order items `<details>` dropdown in `Orders.vue` should become `<OrderItemsDropdown>`.

#### Inline Styles That Should Be Classes
Flag `:style="{ color: condition ? '#ef4444' : '#f59e0b' }"` patterns. These should be CSS utility classes:
```vue
<!-- BAD -->
:style="{ color: item.days_delayed > 7 ? '#ef4444' : '#f59e0b' }"

<!-- GOOD -->
:class="item.days_delayed > 7 ? 'text-danger' : 'text-warning'"
```

### Vue 3 Best Practices

#### Consistency: Composition API
All components in this project use the Composition API (`<script setup>` or `setup()`). Flag any component using the Options API (`data()`, `methods`, `mounted` as object keys) as inconsistent.

Known offender: `Reports.vue` — uses Options API and `var` declarations.

#### Date Validation (Project Rule)
Per CLAUDE.md: always validate dates before calling `.getMonth()` or comparing with `new Date()`.

Flag:
```js
// BAD
new Date(o.actual_delivery) <= new Date(o.expected_delivery)

// GOOD
const d1 = new Date(o.actual_delivery)
const d2 = new Date(o.expected_delivery)
if (!isNaN(d1) && !isNaN(d2)) { ... }
```

#### Console Logs in Production
Flag any `console.log`, `console.warn`, or `console.error` not wrapped in a `if (import.meta.env.DEV)` guard.

---

## Output Format

Structure your report as follows:

```
## Vue Component Analysis: <ComponentName(s)>

### High Priority
- **[File:Line]** Issue description
  Fix: ...

### Medium Priority
- **[File:Line]** Issue description
  Fix: ...

### Low Priority
- **[File:Line]** Issue description
  Fix: ...

### Cross-Component Duplication
List functions/patterns duplicated across files with suggested extraction targets.

### Summary
X high, Y medium, Z low priority issues found.
```

---

## Quick Reference: Severity Guide

| Severity | Criteria |
|----------|----------|
| High | Breaks reactivity, causes incorrect renders, index keys in dynamic lists |
| Medium | Causes unnecessary re-renders, duplicated logic in 3+ files, Options API inconsistency |
| Low | Inline styles, minor duplication in 2 files, missing convenience utilities |

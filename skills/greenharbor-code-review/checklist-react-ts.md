# React / TypeScript Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for React/TypeScript-specific issues. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

---

## Review Categories

### Pass 1 — CRITICAL

#### Effect Cleanup Hazards
- `useEffect` with subscriptions, event listeners, timers, or fetch calls that lack cleanup (missing `AbortController`, `removeEventListener`, `clearTimeout`/`clearInterval` in the return function)
- Effects that set state after component unmount — missing cancellation token or cleanup

#### XSS Vulnerabilities
- `dangerouslySetInnerHTML` without DOMPurify sanitization
- User-controlled values interpolated into `href`, `src`, or event handler attributes without validation
- `javascript:` protocol in dynamic URLs

#### Async Race Conditions
- Async effects without cancellation — stale closures setting state from superseded requests
- Missing `AbortController` in `useEffect` fetch calls — component re-renders can cause out-of-order state updates

### Pass 2 — INFORMATIONAL

#### TanStack Query Patterns
- Missing `staleTime` configuration (defaults to 0, causing unnecessary refetches)
- Poorly structured `queryKey` arrays (missing variables that affect the query, leading to stale cache hits)
- Missing `enabled` option for dependent queries (queries that should wait for a prerequisite value)
- Mutations without `onError` handling or optimistic update rollback

#### Accessibility
- Interactive elements (`div`, `span` with `onClick`) without keyboard support (`onKeyDown`, `role="button"`, `tabIndex`)
- Missing `aria-label` or `aria-labelledby` on icon-only buttons and interactive elements
- Missing `alt` text on images, missing `label` elements for form inputs

#### Type Safety
- `any` type assertions that bypass type checking
- `as` type assertions that could mask runtime errors — prefer type guards or discriminated unions
- Missing return types on exported functions
- Non-null assertions (`!`) used to suppress TypeScript without proving safety
- `@ts-ignore` / `@ts-expect-error` without a comment explaining why
- `Object` or `{}` types where a specific interface should exist
- Union types with 5+ members that should be a discriminated union or enum

#### State Management
- Effect chains (one `useEffect` setting state that triggers another `useEffect`) instead of derived state or `useMemo`
- State that could be computed from props or other state
- Redundant state: storing the same data in multiple state variables
- State updates that don't use the functional form when depending on previous state (`setCount(count + 1)` → `setCount(c => c + 1)`)

#### Error & Loading States
- Missing error boundaries around components that can throw
- Missing loading/error states for async operations
- Optimistic UI without rollback on failure

#### React 19 Opportunities
- `useEffect` + `setState` patterns that could use the `use` hook
- Form submission patterns that could use `useActionState`
- `useEffect` for data fetching that could be replaced with Suspense + `use`

#### Code Quality — TypeScript/React-Specific (Sonar-equivalent)

**Unused & Dead Code:**
- Unused imports, variables, parameters, or type declarations
- Exported functions/types that are never imported elsewhere
- Dead branches in conditional rendering: JSX branches that can never render based on type constraints
- Unused CSS classes or Tailwind utilities in co-located styles
- Props declared in an interface but never passed by any parent

**Complexity & Structure:**
- Components exceeding ~100 lines — extract sub-components or custom hooks
- Functions with 4+ branching paths — simplify with early returns or extract logic into helper functions
- Deeply nested JSX (4+ levels of conditional/mapped elements) — extract into named sub-components
- Components with 5+ `useState` calls — consider `useReducer` or extracting a custom hook
- Files exporting more than 3 components — split into focused files
- Prop drilling through 3+ levels — consider composition, context, or restructuring component hierarchy

**Duplication:**
- Repeated JSX patterns (same structure with different data) — extract to a reusable component
- Repeated `useEffect` / `useState` / fetch patterns — extract to a custom hook
- Identical type definitions in multiple files — extract to a shared types module
- Copy-pasted event handlers that differ only in the target field — parameterize

**Naming & Readability:**
- Boolean props without question-word prefixes (`disabled` is fine; `show` → `isVisible`)
- Event handler props not prefixed with `on` (`click` → `onClick`, `change` → `onChange`)
- Custom hooks not prefixed with `use`
- Components named with lowercase (must start with uppercase for JSX)
- Generic names: `data`, `info`, `item`, `stuff`, `value` as primary identifiers — use domain-specific names

**Immutability & Side Effects:**
- Direct mutation of state objects or arrays (`state.push()`, `state.foo = bar`) instead of spreading
- Side effects outside of `useEffect` or event handlers (API calls in render path)
- Mutable `let` declarations where `const` would suffice
- Object/array references in dependency arrays that are recreated every render (missing `useMemo`/`useCallback`)

**Error Handling:**
- `.catch()` blocks that silently swallow errors (empty or log-only)
- `try/catch` around async operations without user-facing error state
- Missing error boundaries around suspense boundaries
- Unchecked `.json()` calls on fetch responses (no status check first)

---

## Suppressions — DO NOT flag these

- Style-only changes (Tailwind class ordering, formatting)
- "Add JSDoc" or "add comment" suggestions
- Test utility patterns that look unusual but work correctly
- Redundant type annotations where TypeScript can infer
- Framework-idiomatic patterns that look odd but are standard (e.g., `key` prop on fragments)
- ANYTHING already addressed in the diff

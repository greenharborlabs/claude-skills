# React / TypeScript Pre-Landing Review Checklist

## Instructions

Apply these React/TypeScript checks to the active review input and use the output format from `SKILL.md`.
Only flag real problems with specific `file:line` evidence. Skip anything that is fine or already addressed.

---

## Review Categories

### Pass 1 — CRITICAL

#### Effect Cleanup Hazards
- `useEffect` with subscriptions, event listeners, timers, or fetch calls lacking cleanup
- Effects that set state after component unmount — missing cancellation

#### XSS Vulnerabilities
- `dangerouslySetInnerHTML` without DOMPurify sanitization
- User-controlled values in `href`, `src`, or event handler attributes without validation
- `javascript:` protocol in dynamic URLs

#### Async Race Conditions
- Async effects without cancellation — stale closures setting state from superseded requests
- Missing `AbortController` in `useEffect` fetch calls

### Pass 2 — INFORMATIONAL

#### TanStack Query Patterns
- Missing `staleTime` configuration
- Poorly structured `queryKey` arrays
- Missing `enabled` for dependent queries
- Mutations without `onError` or optimistic update rollback

#### Accessibility
- Interactive elements without keyboard support
- Missing `aria-label` on icon-only buttons
- Missing `alt` text, missing form labels

#### Type Safety
- `any` type assertions bypassing type checking
- `as` assertions that could mask runtime errors
- Missing return types on exported functions
- Non-null assertions without proving safety
- `@ts-ignore` without explanation

#### State Management
- Effect chains instead of derived state or `useMemo`
- State that could be computed from props
- Redundant state in multiple variables
- Missing functional form for state updates depending on previous state

#### Error & Loading States
- Missing error boundaries
- Missing loading/error states for async operations
- Optimistic UI without rollback

#### React 19 Opportunities
- `useEffect` + `setState` patterns that could use `use` hook
- Form patterns that could use `useActionState`

#### Code Quality — TypeScript/React-Specific

**Unused & Dead Code:**
- Unused imports, variables, parameters, type declarations
- Exported functions/types never imported elsewhere
- Dead conditional rendering branches
- Props declared but never passed

**Complexity & Structure:**
- Components exceeding ~100 lines
- Deeply nested JSX (4+ levels) — extract sub-components
- Components with 5+ `useState` — consider `useReducer` or custom hook
- Prop drilling through 3+ levels

**Duplication:**
- Repeated JSX patterns — extract to reusable component
- Repeated hooks patterns — extract to custom hook
- Identical type definitions in multiple files

**Immutability & Side Effects:**
- Direct mutation of state objects/arrays
- Side effects outside `useEffect` or event handlers
- Object/array references recreated every render without `useMemo`/`useCallback`

**Error Handling:**
- `.catch()` blocks silently swallowing errors
- Unchecked `.json()` calls without status check

---

## Suppressions — DO NOT flag these

- Style-only changes (Tailwind ordering, formatting)
- "Add JSDoc" suggestions
- Test utility patterns
- Redundant type annotations where TypeScript infers
- Framework-idiomatic patterns
- ANYTHING already addressed in the diff

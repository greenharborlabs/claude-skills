# Rust Pre-Landing Review Checklist

## Instructions

Apply these checks only when Rust/Cargo is detected. Use the output format from
`SKILL.md`; flag only concrete problems with `file:line` evidence.

---

## Review Categories

### Pass 1 — CRITICAL

#### Soundness, Unsafe, and FFI
- `unsafe`, raw pointers, `transmute`, `MaybeUninit`, unchecked indexing, or manual `Send`/`Sync` without a local, documented invariant and proof that safe callers cannot violate it
- FFI without explicit ABI, ownership/freeing, nullability, layout, error, and panic/unwind handling
- Unsafe abstractions whose trusted fields can be mutated by code outside the invariant-owning module

#### Async, Concurrency, and Resource Safety
- Blocking I/O/CPU work on an async executor without the project-approved blocking boundary
- Locks, transactions, or scarce resources held across `.await`
- Unbounded tasks, channels, retries, pagination, recursion, allocations, decompression, uploads, or external request bodies
- Missing timeout, cancellation, shutdown ownership, or backpressure at an external boundary
- Check-then-act state transitions without an atomic/transactional guard

#### Input, Errors, and Security
- `unwrap`/`expect` or panic-prone indexing on untrusted/request/network/filesystem/persistence paths
- Unvalidated deserialization, user-controlled paths, shell/process arguments, SQL fragments, URLs, redirects, or hostnames
- Secrets/keys/tokens in source, `Debug`/`Display`, logs, errors, fixtures, serialized output, or container/build layers
- Non-constant-time secret comparisons, weak randomness, or custom cryptography where vetted primitives exist

#### Supply Chain and Build
- Unreviewed `build.rs`, proc macro, registry/source override, native/FFI dependency, or feature change that expands trusted build/runtime code
- Dependency additions or feature enabling that bypass the workspace's lockfile, source, license, or RustSec policy

### Pass 2 — INFORMATIONAL

#### Public API and Compatibility
- Breaking changes to public types, traits, errors, features, serde schema, CLI output/exit codes, FFI ABI, or MSRV without a migration strategy
- Public enum evolution that should consider `#[non_exhaustive]`, undocumented feature interactions, or examples/doctests missing for new public behavior

#### Ownership and Maintainability
- Needless cloning obscuring ownership, global mutable state, broad error erasure, dead code, unused features/imports, or suppression hiding a defect
- Complex async control flow lacking task ownership, cleanup, or cancellation semantics

#### Tests
- Missing negative-path, cancellation, feature, target, public-contract, or regression tests for changed behavior
- Tests relying on timing, network, environment, current directory, or implementation mocks without isolation
- Hollow tests that only compile or assert no panic

---

## Suppressions — DO NOT flag these

- Safe, documented use of `unwrap`/`expect` for an established invariant outside recoverable production paths
- Intentional clones that simplify a bounded ownership boundary
- Existing project-approved dependencies or tooling with no changed security posture
- Style-only suggestions or anything already addressed in the diff

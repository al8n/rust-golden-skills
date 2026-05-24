---
name: rust-type-conventions
description: Golden rules for declaring Rust structs, enums, and their accessors (encapsulation, const fn, new()/Default delegation, variant predicates, mandatory `as_str` on unit-only enums, thiserror, structured error variants with #[from] for typed foreign-error wrapping and Cow<'static, str> for unmodellable text (Box<dyn Error>/anyhow only in binaries), open Other(_)/coded Unknown(n) enums, no-module-name-stutter naming, no-std feature tiers, method-local generic bounds, feature-gated optional trait impls grouped in an anonymous const block, serde representation rules). Use when writing or reviewing any struct/enum definition, getter, builder, setter, or error type in a Rust library.
---

# Rust type-declaration golden rules

Conventions for hand-written library types (domain models, vocabulary enums,
value objects, error types). Apply them when **writing** new types and when
**reviewing** them. The compiler, clippy, and the test matrix are the backstop;
follow these by construction so review is cheap.

These are defaults, not dogma. A few choices are genuinely per-crate (string
type, error/derive libraries) — pick one **per crate** and stay consistent; each
is flagged below.

---

## 1. Structs

### Encapsulation
- **No public fields.** Fields are private; expose via accessors (§3). (Plain
  data-transfer structs at a serialization boundary are the rare exception.)
- Derive the minimal honest set: `#[derive(Debug, Clone, PartialEq, …)]`.
  - Add `Copy` only for small, all-`Copy` types.
  - Add `Eq, Hash` only when every field supports them. A floating-point field
    (`f32`/`f64`) **precludes `Eq`/`Hash`** (NaN ≠ NaN) — stop at `PartialEq`.

### Field types — owned vs shared
- **Mutable / growable / built incrementally** → owned: `Vec<T>`, `String`.
- **Immutable, set-once, cloned or shared** (especially large payloads) → a
  cheap-clone refcounted type so a clone is O(1) instead of a deep copy:
  - byte blobs → `bytes::Bytes` (O(1) clone, O(1) `From<Vec<u8>>` — adopts the
    buffer with no copy — zero-copy sub-slicing, derives `Eq`/`Hash`);
  - other payloads → `Arc<T>` / `Arc<[T]>` to stay dependency-free.
  Both refcounted options use atomic refcounts (need atomics); keep
  `Vec`/`String` if you must run on atomic-less targets.

### Construction — one shape per type
- **Validating** when invariants must hold — `try_new(...) -> Result<Self, Err>`:
  reject sentinel/nil ids, empty required fields, out-of-range numbers,
  inverted ranges (`start > end`), orphan foreign keys.
  ```rust
  pub fn try_new(id: Id, name: Name) -> Result<Self, FooError> {
      if id.is_sentinel() { return Err(FooError::MissingId); }
      if name.is_empty()  { return Err(FooError::EmptyName); }
      Ok(Self { id, name })
  }
  ```
- **Infallible empty/zero** — `new() -> Self` when there is a meaningful empty
  value. Make it `const fn` whenever the body permits (§4):
  ```rust
  pub const fn new() -> Self { Self { /* all empty / zero / None */ } }
  ```
- **Full-args** `new(a, b, c)` only for pure value objects with no "empty"
  notion (e.g. a fixed-arity measurement / coordinate).

### Default — optional; provide it only when a default is meaningful
- **`Default` is not required.** Most types do **not** get one. Add `Default`
  (and a paired `new()`) **only when an empty/zero/identity-free value is
  genuinely meaningful** for the type (an accumulator, an all-empty metadata
  bag, a "0/0/None" measurement). If "what would `default()` even return?" has
  no honest answer, the type has **no `new()` and no `Default`** — construct it
  via `try_new` / full-args only.
- When a default *is* meaningful, the two come together: a `pub [const] fn
  new() -> Self` **and** a `Default` that delegates to it (don't
  `#[derive(Default)]` when a `new()` exists):
  ```rust
  impl Default for Foo {
      fn default() -> Self { Self::new() }            // zero-arg
  }
  impl Default for Measure {
      fn default() -> Self { Self::new(0.0, 0.0) }    // full-args, zero value
  }
  ```
  This keeps one source of truth for "what an empty value is".
- **`#[derive(Default)]` is fine** when the default is exactly the field-wise
  default *and* there is no zero-arg `new()` or named canonical const
  (e.g. `Self::UNSPECIFIED`, `Self::EMPTY`) it would duplicate — a derive can't
  drift from itself. Switch to a delegating `impl Default` **only once such a
  named canonical default-instance exists**, so that named instance becomes the
  single source. (The "delegate, don't derive" rule targets *duplication*, not
  the derive itself.)
- **No `Default` on types that carry identity** (a `Default` would mint a record
  with a nil/colliding id). Force `try_new`.
- A **placeholder `Default` purely to satisfy a derive bound** (e.g. a
  serializer that requires `T: Default` as a decode seed) is acceptable only
  when documented as such, with the real constructor being `try_new`.
- **`Default` — and every inherent trait impl a type needs unconditionally —
  lives in the type's own module and is NOT behind an optional feature.** A
  `Default` that only exists with `--features serde` (or any optional feature)
  silently disappears in other builds. (Common real bug.)

---

## 2. Enums

### Variant predicates / accessors
- Give every enum **`is_<variant>()` predicates**. Hand-write them, or derive
  them (e.g. `derive_more::IsVariant`).
- For enums with **data-carrying variants**, also provide `unwrap`/`try_unwrap`
  accessors (hand-written, or `derive_more::{Unwrap, TryUnwrap}` with
  `#[unwrap(ref, ref_mut)] #[try_unwrap(ref, ref_mut)]`) so callers don't
  hand-match.
- **Variants may only be unit or newtype** — `Foo` (unit) or `Foo(Payload)`
  (exactly one field). **No struct-style `{ … }` variants and no multi-field
  tuple variants** (`Foo(A, B)`). If a variant needs to carry more than one
  piece of data, extract those fields into a **named struct** (private fields +
  accessors) and wrap it in a newtype variant:
  ```rust
  // NO:  Local { host: Host, port: u16 }   |   NO:  Local(Host, u16)
  // YES: Local(LocalAddr)   where  struct LocalAddr { host: Host, port: u16 }
  ```
  (Keeps each variant's data independently named, testable, and accessor-wrapped,
  and lets the `Unwrap`/`TryUnwrap` accessors hand back one coherent payload.)

### `as_str` — mandatory on every unit-only enum; Display routes through it

Every unit-only enum **must** expose:

```rust
impl Foo {
    pub const fn as_str(&self) -> &'static str {
        match self {
            Self::A => "a",
            Self::B => "b",
            // …
        }
    }
}
```

Required regardless of whether the enum has a `Display` impl. The method gives you:

- A **const-evaluable, allocation-free projection** to a stable string name — usable in `const` contexts, pattern-table builders, error messages, logging, wire-serialization tag bytes, schema-doc references, anywhere a variant needs a string handle.
- **Single source of truth** for the variant's string name. When `Display`, serde tags, log keys, and schema-doc references all route through `as_str`, they can't drift from each other.
- A **stable testing surface** — round-trip tests `from_str(x.as_str()) == x` are uniform across every enum that follows this rule.

`Display` (when present) routes through `as_str`:

```rust
impl core::fmt::Display for Foo {
    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
        f.write_str(self.as_str())
    }
}
// or with derive_more::Display:
#[derive(derive_more::Display)]
#[display("{}", self.as_str())]
pub enum Foo { /* … */ }
```

**For enums with a data-carrying variant** (e.g. an `Other(String-ish)` arm), `as_str` is **non-const** and returns `-> &str`:

```rust
impl Foo {
    pub fn as_str(&self) -> &str {
        match self {
            Self::A => "a",
            Self::Other(s) => s.as_str(),
        }
    }
}
```

**Slug convention:** snake_case (`"variant_name"`, not `"VariantName"`) for serialization compatibility, unless the variant has a canonical external spelling (a MIME type, a language code, a hyphenated standard like `"cea_608"` vs `"CEA-608"` — pick the form the wire / spec uses) — then use that form. Pick a convention per enum and stick to it.

### Open vs coded vocabularies (be lossless)
- **Open vocabulary** (names that may grow upstream — formats, codecs, kinds):
  `#[non_exhaustive]` + an `Other(String-ish)` lossless escape, and a **total**
  `FromStr` (unknown input → `Other(_)`, never errors).
- **Externally-numbered** (protocol/FFI/wire ids): a lossless `Unknown(u32)`
  arm + `to_u32`/`from_u32` (or equivalent) guaranteeing round-trip:
  `from_u32(x.to_u32()) == x` for **every** value, including unknowns.
- **If** an open enum has a default, it = the "unspecified" sentinel, identical
  across sibling enums (e.g. all `*Format` enums default the same way). Pick the
  convention once per crate.

### Default on enums — also opt-in, also only when meaningful
Like structs (§1), an enum **is not required to implement `Default`** — add it
**only when one variant is genuinely the natural default** (an `Off`/`None`/
`Unspecified`/`Unknown` sentinel). If no variant is an honest default, the enum
has **no `Default`** — callers must choose a variant explicitly.

When a default *is* meaningful, enums **are exempt from the `new()` pairing** —
use `#[derive(Default)] + #[default]` on the chosen variant (no `new()`):
```rust
#[derive(Debug, Default, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Mode { #[default] Off, On }
```
(A hand-written `impl Default` returning a variant is equivalent; prefer the
derive unless the default needs a data-carrying variant.)

### Variant-naming gotcha
Auto-derived accessor names snake-case across digit boundaries: `Sd1080` →
`is_sd_1080`, `NotV7` → `try_unwrap_not_v_7`. Rename for clean accessors
(`WrongVersion` over `NotV7`; avoid digit-leading variants where the generated
name reads badly).

---

## 3. Accessors

### Getters — `const` by default, **project the wrapper, never expose it**
A getter returns the borrowed *view* of a field, not its concrete wrapper type.
Never return `&Option<T>`, `&Vec<T>`, `&Box<[T]>`, `&Arc<[T]>`, `&String`.

**Getter naming** (matters — bake it in by construction):
- `field() -> T` (bare name, by value) is **only** for `Copy` types.
- A getter that returns a **`&T` borrow of a non-`Copy` value** is named
  **`field_ref()`** — it pairs with `field_mut()` (shared vs exclusive borrow).
- Projection views keep their own names: `field_slice() -> &[T]` for sequences,
  and the canonical `field() -> &str` / `field() -> &[u8]` views for
  string / byte fields (those don't return `-> T`, so they don't collide with
  the `Copy`-only rule).

| field type | read getter | mutable getter (§ mutable getters) |
|---|---|---|
| `T: Copy` | `field() -> T` (`self.f`) | `field_mut() -> &mut T` (`&mut self.f`) |
| `T` (non-`Copy`) | `field_ref() -> &T` (`&self.f`) | `field_mut() -> &mut T` |
| `Option<T>`, `T: Copy` | `field() -> Option<T>` (`self.f`) | `field_mut() -> Option<&mut T>` (`as_mut`) |
| `Option<T>`, non-`Copy` | `field_ref() -> Option<&T>` (`self.f.as_ref()`) | `field_mut() -> Option<&mut T>` (`as_mut`) |
| `Vec<T>` | `field_slice() -> &[T]` (`as_slice`) — never `&Vec<T>` | `&mut [T]` + `&mut Vec<T>` — see **Growable owned containers** |
| `Box<[T]>` | `field_slice() -> &[T]` (`&self.f` / `.as_ref()`) | `field_slice_mut() -> &mut [T]` — fixed length |
| `Arc<[T]>` / `Rc<[T]>` | `field_slice() -> &[T]` (`&self.f`) | **none** — shared/immutable |
| `Arc<T>` / `Rc<T>` | `field_ref() -> &T` (`&self.f`) | **none** — shared/immutable |
| `String` / small-string | `field() -> &str` (`.as_str()`) — never `&String` | `&mut String` — see **Growable owned containers** |
| `bytes::Bytes` | `field() -> &[u8]` (`.as_ref()`) + `field_bytes() -> Bytes` clone | **none** — immutable |

`const`-ness (per §4): `Option::as_ref`/`as_mut`, `Vec::as_slice`, `Copy`
returns, and `&self.field` / `&mut self.field` are `const`. Not `const`:
owned-string `as_str`/`is_empty`, `Option::map`/`as_deref`, `Arc`/`Rc`/`Box`
deref-coercion, `Bytes::as_ref`. (`Vec::as_mut_slice` is `const` on recent
toolchains — mark it `const` and let the compiler confirm.)

Shared/refcounted fields (`Arc`, `Rc`, `Bytes`) get **no mutable getter** — they
are immutable handles; expose a cheap-clone `_bytes()` / `clone()`-returning
accessor instead if callers need ownership.

#### Growable owned containers (`Vec<T>`, `String`)
A `Vec<T>` / `String` field **never needs an immutable `&Vec<T>` / `&String`
getter** — the borrowed `&[T]` / `&str` view covers every read. Mutation has two
distinct levels, so expose up to three accessors (`foo: Vec<u32>`):
```rust
pub const fn foo_slice(&self) -> &[u32]            { self.foo.as_slice() }     // read elements
pub const fn foo_slice_mut(&mut self) -> &mut [u32] { self.foo.as_mut_slice() } // mutate elements, no resize
pub const fn foo_mut(&mut self) -> &mut Vec<u32>    { &mut self.foo }           // grow / shrink: push, pop, extend, clear
```
- `&[T]` / `&str` for reads — **never** `&Vec<T>` / `&String`.
- `&mut [T]` (`as_mut_slice`) to edit existing elements in place.
- `&mut Vec<T>` (`&mut self.foo`) is the one place `&mut Vec`/`&mut String` is
  right — when the caller must change length/capacity. For `String` the
  structural-mut getter is `-> &mut String`.
- `Box<[T]>` is fixed-length: only `&[T]` + `&mut [T]`, no `&mut Box<[T]>`.

(`as_slice` and `&mut self.foo` are `const`; `as_mut_slice` is `const` on recent
toolchains.)

For string fields, **empty means absent** — prefer a plain string with `""` as
the absent value over `Option<String>`. Exception: use `Option<T>` when the
zero/empty value is itself meaningful and must be distinct from absent
(e.g. `Option<u16>` where `Some(0)` ≠ `None`).

### Mutable getters — for fields that accept in-place mutation
If a field may be mutated in place (an owned collection a caller appends to, a
nested struct a caller edits), expose a mutable getter alongside the read-only
one, **projected and named the same way** (`field_ref` ↔ `field_mut`,
`field_slice` ↔ `field_slice_mut`):
```rust
pub const fn field_mut(&mut self) -> &mut T { &mut self.field }                 // pairs with field_ref()
pub const fn opt_mut(&mut self) -> Option<&mut T> { self.opt.as_mut() }         // Option<T>
pub const fn buf_slice_mut(&mut self) -> &mut [T] { self.buf.as_mut_slice() }   // Box<[T]> (fixed len)
// Vec<T> / String: use the slice-mut + structural-mut pair from
// "Growable owned containers" (foo_slice_mut + foo_mut).
```
`const` is fine — mutable references in `const fn` are stable (since 1.83).
Provide a `*_mut` **only** for fields with no cross-field invariant — it bypasses
the validation a `set_`/`with_` could enforce. Do **not** expose `*_mut` for
identity fields, validated fields, or immutable shared types (`Bytes`, `Arc<_>`,
`Rc<_>`).

### Builders (consuming) — `-> Self`, **`#[must_use]`**
```rust
#[must_use]
pub fn with_field(mut self, v: impl Into<Field>) -> Self { self.field = v.into(); self }
```
`with_*` takes `self` by value and returns the modified `Self`. **`#[must_use]`
is required** here precisely *because* it returns `-> Self`: dropping the result
silently discards the change (`x.with_field(v);` is a no-op bug). `const` where
the body permits (an `impl Into` body is not const).

### Setters (in place) — always return `&mut Self`
```rust
pub fn set_field(&mut self, v: impl Into<Field>) -> &mut Self { self.field = v.into(); self }
```
Setters **must** return `&mut Self` so they chain (`x.set_a(1).set_b(2)`).
Returning `()` is **not acceptable**. **No `#[must_use]`** — the mutation is
already applied through `&mut self`; the returned `&mut Self` is a chaining
convenience that is fine to ignore. `const` where the body permits.

### `Option<T>` / `bool` fields — fuller mutator vocabulary
An `Option<T>` or `bool` field is "set the present value", "assign the raw
wrapper", or "clear" — give callers all three, in both in-place (`&mut Self`)
and consuming (`Self`) forms. Naming is uniform:
`set_`/`with_` = put it into the *present/active* state with the inner value;
`update_`/`maybe_` = assign the *raw wrapper* value; `clear_` = absent/false.

`Option<T>` field `foo` (inner type `T`):
```rust
pub const fn set_foo(&mut self, val: T) -> &mut Self          { self.foo = Some(val); self } // present
pub const fn with_foo(mut self, val: T) -> Self              { self.foo = Some(val); self }
pub const fn update_foo(&mut self, val: Option<T>) -> &mut Self { self.foo = val; self }      // raw
pub const fn maybe_foo(mut self, val: Option<T>) -> Self      { self.foo = val; self }
pub const fn clear_foo(&mut self) -> &mut Self               { self.foo = None; self }       // absent
```

`bool` field `bar` (present = `true`, so `set_`/`with_` take no value):
```rust
pub const fn set_bar(&mut self) -> &mut Self                 { self.bar = true; self }
pub const fn with_bar(mut self) -> Self                      { self.bar = true; self }
pub const fn update_bar(&mut self, val: bool) -> &mut Self    { self.bar = val; self }         // raw
pub const fn maybe_bar(mut self, val: bool) -> Self          { self.bar = val; self }
pub const fn clear_bar(&mut self) -> &mut Self               { self.bar = false; self }        // false
```
Only the consuming `with_*`/`maybe_*` (return `-> Self`) carry `#[must_use]`;
the `&mut Self` forms (`set_*`/`update_*`/`clear_*`) do not. Plain (non-optional,
non-bool) fields keep just `set_field(val: T)` / `with_field(val: T)`.

### Accept `impl Into<_>` for ergonomic params
String/owned setters and builders take `impl Into<Field>` so callers pass
`&str`/`String`/etc. without ceremony (this makes them non-const — that's fine).

### Inline cheap accessors — `#[inline(always)]`
Every trivial accessor — `field()` / `field_ref()` getters, `field_mut()`,
`with_*` builders, `set_*` setters, and zero/const constructors — is a one-liner
that crosses the crate boundary, where the optimizer often won't inline it on its
own (especially without LTO). Mark them `#[inline(always)]`:
```rust
#[inline(always)]
pub const fn id(&self) -> Id { self.id }                 // Copy: by value, bare name
#[inline(always)]
pub const fn provenance_ref(&self) -> &Provenance { &self.provenance }  // non-Copy: &T, _ref name
```
Crates that run a coverage tool which chokes on forced inlining gate it so
coverage builds still see the function body:
```rust
#[cfg_attr(not(tarpaulin), inline(always))]
pub const fn id(&self) -> Id { self.id }
```
(Reserve `#[inline(always)]` for genuinely trivial bodies; non-trivial methods
get plain `#[inline]` or nothing and let the optimizer decide.)

---

## 4. `const fn` — make it const unless a body op forbids it

Default to `const fn`. Blockers (force non-const):
- non-const trait calls: `Into::into`, `From::from`, `PartialEq`/`==`,
  owned-string `as_str`/`is_empty`, `Option::map`/`as_deref`, iterators/closures;
- allocation, parsing, formatting, anything touching the heap or the OS.

Enablers (stay const): field access, returning `Copy` values, plain `match`
returning literals/`Copy`, `Vec::as_slice`, `Option::as_ref`, `*copy_field`.

Keep siblings consistent: if `from_u32` is `const`, `to_u32` should be too.

---

## 5. Errors

- Derive the impls — don't hand-write `Display` + `Error`. The common choice is
  `thiserror` (`#[derive(thiserror::Error)]` + `#[error("…")]` per variant,
  `#[from]` on a wrapped source for free `From` + `source()`):
  ```rust
  #[derive(Debug, thiserror::Error)]
  #[non_exhaustive]
  pub enum FooError {
      #[error("id is missing")]            MissingId,
      #[error("inner failed: {0}")]        Inner(#[from] InnerError),
  }
  ```
- Always `#[non_exhaustive]` on public error enums (lets you add variants
  without a breaking change).
- For **no-std** crates, use a derive/library that emits `core::error::Error`
  (e.g. `thiserror` v2 with `default-features = false`), never `std::error::Error`.

### Model the error — don't stringify it

Each distinct failure case is a **named variant with structured data**, not a
stringified blob. The point of a typed error is that callers (and tests, and
`match` arms, and recovery code) can dispatch on **what** went wrong; a
`String` / `&'static str` payload throws that information away exactly when
it's most useful.

**Bad** — every cause collapses to opaque text; nothing matches on:
```rust
pub enum FooError {
    Failed(String),
    Custom(&'static str),
    InvalidInput(String),
}
```

**Good** — each cause is its own variant; the data the caller needs to
recover, report, or test against rides on the variant, not in a format
string:
```rust
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum FooError {
    #[error("name `{name}` rejected: {reason}")]
    InvalidName { name: SmolStr, reason: NameRejection },

    #[error("missing required field `{0}`")]
    MissingField(&'static str),                                  // dispatch on field name

    #[error("checksum mismatch: expected {expected:?}, got {got:?}")]
    ChecksumMismatch { expected: [u8; 32], got: [u8; 32] },

    #[error("upstream service: {source}")]
    Upstream { #[from] source: UpstreamError },
}
```

Note the difference between **a unit/newtype variant carrying a `&'static
str` discriminator** (`MissingField("name")` — the string IS the structured
data: which field) and **a `String` blob explaining what went wrong** (`Failed("name was empty")` — the structure is lost). The first is fine; the
second is the anti-pattern.

### Foreign errors — typed wrap via `#[from]`, never `Box<dyn Error>`

When a library function calls into another crate's API that returns a typed
error, wrap that error as **its own named variant** via `#[from]`:

```rust
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum FooError {
    #[error("parse failed: {0}")]
    Parse(#[from] serde_json::Error),

    #[error("io failed: {0}")]
    Io(#[from] std::io::Error),

    #[error("upstream service: {0}")]
    Upstream(#[from] UpstreamError),
}
```

`#[from]` gives you a free `From<serde_json::Error> for FooError` plus
correct `source()` chaining, and the type chain is preserved end-to-end —
callers can downcast, pattern-match on which foreign failure occurred, and
write recovery code for specific upstream errors. **This is the only library
pattern for foreign errors.** Do not box them.

If a single foreign type produces multiple distinguishable failures you care
about (e.g. `io::Error`'s `NotFound` vs `PermissionDenied` matter to your
callers), split them into multiple variants of your own enum and convert
explicitly — `#[from]` covers the common single-variant case; the manual
conversion handles the cases where you need finer dispatch.

### Last-resort escape hatch — `Other(Cow<'static, str>)` for unmappable descriptive text

Some failures genuinely can't be modelled — a vague upstream condition that
isn't a typed error, a transient infrastructure detail with no useful
structure, ad-hoc context. For those keep ONE escape hatch as the last
variant:

```rust
#[non_exhaustive]
pub enum FooError {
    /* …modelled variants… */

    /// Descriptive text for genuinely unmodellable cases. Used sparingly.
    #[error("{0}")]
    Other(Cow<'static, str>),
}
```

`Cow<'static, str>` lets callers pass a `&'static str` (the common case,
zero allocation) or an owned `String` (a `format!(...)`) without forcing all
call-sites to allocate. Strictly more general than `String`; use it as the
default text-payload form.

Use `Other(_)` **sparingly**. Every time a real-world failure routes through
it, that's a candidate for promotion to a structured variant (or a typed
`#[from]` wrap if it's a foreign error) — callers gain dispatch and better
messages. Treat `Other` as the relief valve, not the default destination;
review it on every PR that adds an error path.

### `Box<dyn Error>` — binary applications only

`Other(Box<dyn core::error::Error + Send + Sync + 'static>)` (or `anyhow::Error`,
which boxes underneath) belongs in **end-of-the-line binaries** — CLIs,
daemons, examples, integration tests — where the program is the final consumer
of every error and just needs to bubble-and-report. Type erasure is fine
there because nothing downstream needs to dispatch.

It does **NOT** belong in a library's public error enum. A library that boxes
its sources forces every caller into the same type-erased, dispatch-poor
world; we can't pattern-match across a `dyn Error`, we can't write recovery
code keyed on the underlying type, and `source()` walking has to downcast at
every step. Use `#[from] *Error` instead.

(`Send + Sync` because errors cross threads; `'static` because no borrows;
`core::error::Error` (not `std::error::Error`) to stay no-std compatible.)

---

## 6. Naming — no module-name stutter

A type's module path already namespaces it (std does `io::Error`, not
`io::IoError`). `foo::FooConfig` trips `clippy::module_name_repetitions`.

- **Drop** the prefix that just repeats the module: `audio::AudioFormat` →
  `audio::Format`. Prefer a precise word over a too-generic one when the bare
  name is ambiguous (`audio::SampleFormat`, not a bare `Format`).
- **Keep** the prefix when it marks an *axis* rather than the module (sibling
  `Video`/`Audio`/`Subtitle` variants of the same concept living in one module),
  is a canonical external name, or when the bare name would **shadow a std type**
  (don't expose a bare `Range`, `Error`, `Result`, `Box`, …).
- Do renames **pre-1.0 / pre-publish**; after a public release a rename is a
  breaking change.
- When you rename, sweep **every** referencing surface in one pass: `src/`,
  tests, doc comments, serialization/codegen, fixtures, **and any out-of-tree
  references** (build scripts, vendored tables, CI checks that string-match type
  names — these won't be caught by `cargo build`/`cargo test`).

---

## 7. no-std feature tiers (only if the crate targets no-std)

Gate by capability, smallest first:
- `--no-default-features` ⇒ no-std + no-alloc: only stack-only types (`Copy`
  enums, marker structs). Heap-reaching types are `cfg`-out here.
- `+ alloc` ⇒ heap types. Gate with
  `#[cfg(any(feature = "std", feature = "alloc"))]`
  (+ `#[cfg_attr(docsrs, doc(cfg(...)))]` for docs.rs).
- `+ std` ⇒ `std`-only deps (clocks, RNG, filesystem).
- An optional serialization/wire feature must not silently host trait impls a
  type needs unconditionally (see §1's `Default`-gating rule).

Verify across the matrix: `cargo hack test --each-feature` (or
`--feature-powerset`), plus `cargo fmt --check` and `cargo clippy`.

---

## 8. Generic bounds — on the methods that need them, not on the type

A generic parameter declared on a `struct`/`enum`/`impl` block should carry
**no trait bounds**. Move every bound to the specific method that actually
requires it, via a `where` clause.

### Bad
```rust
struct Foo<A: Clone> { /* … */ }

impl<A: Clone> Foo<A> {
    fn read(&self) -> &A { /* doesn't need Clone */ }
    fn duplicate(&self) -> A { self.a.clone() }
}
```

### Good
```rust
struct Foo<A> { /* … */ }

impl<A> Foo<A> {
    fn read(&self) -> &A { &self.a }                          // no bound
    fn duplicate(&self) -> A where A: Clone { self.a.clone() } // bound where used
}
```

### Why
- **Composability.** `Foo<NonClone>` is a legal type. With the bound on the
  struct, every downstream `Vec<Foo<T>>`, every blanket impl, every
  `PhantomData` hop has to thread `T: Clone` even when no `Clone` method is
  ever called. Method-local bounds keep the type usable in every context that
  doesn't touch the constrained operation.
- **Standard-library precedent.** `Vec<T>`, `HashMap<K, V>`, `BTreeSet<T>` —
  none put bounds on the `struct`. `Vec<T>` doesn't require `T: Clone`; only
  `.to_vec()` / `.extend_from_slice(...)` add `where T: Clone`. `HashMap<K, V>`
  doesn't require `K: Hash + Eq` on the type — only the methods that actually
  hash carry it.
- **Smaller error spans.** A struct-level bound surfaces "T does not implement
  Clone" at every construction site. A method-local bound surfaces it only at
  the actual call site, where the user can see which capability they asked
  for.
- **Future flexibility.** Adding a method that needs a stronger bound is a
  local change. Loosening a bound that's currently on the struct is a
  breaking change for every consumer.

### Exceptions (rare, document each)
- **Derived trait impls.** `#[derive(Clone)]` on `Foo<T>` is correct — the
  derive emits `impl<T: Clone> Clone for Foo<T>`, which is exactly the
  pattern this rule promotes (bound on the impl that needs it, not on the
  struct). Don't strip the derive trying to satisfy the rule.
- **Trait impls where the bound is structural to the impl** — e.g.
  `impl<T: Clone> Clone for Foo<T>`, `impl<T: Hash> Hash for Foo<T>`. The
  bound is on the impl, not on the struct, and is mandated by what's being
  implemented.
- **Bounds that gate storage shape.** A type that genuinely cannot exist
  without the bound (e.g. a phantom marker requiring `T: 'static` because the
  type stores a `&'static T`; a sealed-trait newtype intentionally restricted
  to a finite set). Real, rare, always with a comment.

### Inherent `impl` blocks too
The same rule applies to inherent `impl` blocks: prefer `impl<T> Foo<T>` with
method-level `where` clauses over `impl<T: Trait> Foo<T>`. A single inherent
`impl` block carrying a struct-wide bound is the same anti-pattern as the
struct-level bound — every method in that block silently inherits the
constraint. Either split into multiple `impl` blocks (one unconstrained, one
constrained for the specific cluster of methods that need it) or — preferably
— attach the bound directly to each method via `where`.

---

## 9. Optional (feature-gated) trait impls — group in an anonymous `const` block

§1 says an impl a type needs **unconditionally** is never feature-gated. The
complement: when an impl genuinely **is** optional — a `serde` / `buffa` /
`arbitrary` / … impl that should exist only with its feature — don't scatter
`#[cfg(feature = "…")]` across the impl, its local imports, and any shadow/DTO
helper. Wrap the whole group in one anonymous `const _: () = { … };` block,
gated **once**:

```rust
#[cfg(feature = "serde")]
#[cfg_attr(docsrs, doc(cfg(feature = "serde")))]
const _: () = {
    use serde::{Deserialize, Deserializer, Serialize, Serializer};

    impl Serialize for Foo { /* … */ }

    // `try_from` shadow + any helpers stay PRIVATE to this block —
    // they never leak into the surrounding module's namespace.
    #[derive(Deserialize)]
    struct FooShadow { /* mirror the fields */ }

    impl<'de> Deserialize<'de> for Foo {
        fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
            Foo::try_from(FooShadow::deserialize(d)?).map_err(serde::de::Error::custom)
        }
    }
};
```

Why this is clean:
- **One gate, not N.** The single `#[cfg]` covers the impls, the local `use`s,
  *and* the private shadow/helper types together — instead of repeating it on
  each item, and instead of a module-level `#[cfg(feature)] use …` that warns as
  unused in builds without the feature.
- **The impls still apply crate-wide.** A trait impl is registered globally no
  matter where it is written; nesting it in an anonymous const does **not** hide
  or scope it away. That is what makes the wrapper free.
- **Helpers stay local.** *Named* items inside the block (the `FooShadow`
  struct, helper fns) **are** scoped to the block — exactly right for the
  private deserialize-shadow / DTO types (§ validate-on-deserialize) you don't
  want polluting the module.

Constraints:
- **Only for optional impls.** An unconditional impl is written normally at
  module level (§1) — don't wrap it.
- **Nothing inside can be `pub`** / part of the public API: the block scopes
  named items locally, so a public type belongs in its module, not here. Keep
  the block to the impl(s) + their private helpers + local `use`s.
- Keep the `#[cfg]` and the `doc(cfg)` predicate **identical**, and apply this
  per feature (one block per optional feature an item implements).

---

## 10. serde representation — never emit `null`; make absence cheap

When a type derives (or hand-writes) `serde`, two paired rules keep the wire
form sparse, forward-compatible, and symmetric.

### Never serialize `null` — `skip_serializing_if` every `Option`

A `None` serializes to an **absent field**, never an explicit `null`. Every
`Option<T>` serde field carries:

```rust
#[serde(skip_serializing_if = "Option::is_none")]
foo: Option<T>,
```

An explicit `null` is pure noise — it costs bytes, and a reader cannot tell
"deliberately null" from "the producer's default", so don't produce it. The
same reasoning extends to any field whose wire/storage codec already elides a
sentinel (an empty `String`/`Vec`, a zero-sentinel numeric): skip-serialize
that sentinel too, so the Rust type and the wire form agree on "absent".

### `serde(default)` whenever the field has a default

If a field's type has a `Default` impl — or the field has a known default
value — it **must** be `serde(default)`, so a payload that omits the field
(because the producer skipped it per the rule above, or because the payload
predates the field) still deserializes:

- the whole struct has a meaningful `Default` → container-level
  `#[serde(default)]`;
- a single field → `#[serde(default)]` (uses the field type's `Default`) or
  `#[serde(default = "path::to::default_fn")]` for a non-`Default` default.

`skip_serializing_if` and `serde(default)` are a **pair**: the first drops the
field when absent, the second restores it on the way back. One without the
other is an asymmetric round-trip — a value that serializes to nothing and
then fails to deserialize.

---

## Review checklist (run against any struct/enum diff)

- [ ] No public fields.
- [ ] `try_new` validates invariants; `new()` is `const` where it can be.
- [ ] `Default`/`new()` exist **only when a default is meaningful** — neither
      structs nor enums are required to have `Default`; identity-bearing types
      get neither `Default` nor `new()` (use `try_new`).
- [ ] Where a default is meaningful, `Default` delegates to `new()` (not a fresh
      `#[derive]`); no feature-gated `Default`.
- [ ] Field types: owned (`Vec`/`String`) for mutable, cheap-clone shared
      (`Bytes`/`Arc<_>`) for immutable large/shared payloads.
- [ ] Getters project the wrapper, never expose it: `Option<T>`→`Option<&T>`,
      `Vec<T>`/`Box<[T]>`/`Arc<[T]>`→`&[T]`, `String`→`&str`, `Bytes`→`&[u8]`.
      `const` unless a non-const op forbids it.
- [ ] Getter naming: bare `field() -> T` **only** for `Copy` (by value); a `&T`
      borrow is `field_ref()`; sequence views are `field_slice()`.
- [ ] Mutable owned fields also expose a projected `*_mut`
      (`Option<&mut T>` / `&mut [T]` / `&mut T`), `const`, only where no
      invariant is bypassed; shared/immutable fields (`Arc`/`Rc`/`Bytes`) get
      no `*_mut`.
- [ ] Builders (`with_*`, return `-> Self`) are `#[must_use]`; setters (`set_*`)
      return `&mut Self` (never `()`) with **no** `#[must_use]`.
- [ ] `Option<T>` / `bool` fields expose the full set: `set_`/`with_` (present
      value; bool = `true`, arg-less), `update_`/`maybe_` (raw wrapper),
      `clear_` (None / false).
- [ ] Trivial accessors (getters / `*_mut` / `with_*` / `set_*` / zero-or-const
      ctors) are `#[inline(always)]` (or `#[cfg_attr(not(tarpaulin),
      inline(always))]` under a coverage tool).
- [ ] Enums: variant predicates (+ unwrap accessors if data-carrying);
      `#[non_exhaustive]`; **`pub const fn as_str(&self) -> &'static str`
      mandatory on every unit-only enum** (`-> &str` non-const if the enum
      has an `Other(String-ish)` arm); `Display` (when present) routes
      through `as_str`; open `Other(_)` / coded `Unknown(n)` lossless
      escapes; variants only unit or newtype (no struct `{…}` or
      multi-field tuple variants — extract to a named struct). `Default`
      only if a variant is a genuine default
      (`#[derive(Default)] + #[default]`).
- [ ] Errors derived (`thiserror`/equivalent), `#[non_exhaustive]`,
      `core::error::Error` for no-std.
- [ ] Errors model the cause structurally — each failure is its own variant
      with named fields, not a `String`/`&'static str` catch-all.
- [ ] Foreign errors wrap as typed `#[from] *Error` variants (preserves
      `source()` chain + caller dispatch), never `Other(Box<dyn Error>)`
      in a library. `Box<dyn core::error::Error + Send + Sync + 'static>` /
      `anyhow` belongs in binaries only.
- [ ] Last-resort escape hatch is `Other(Cow<'static, str>)` for genuinely
      unmodellable descriptive text, used sparingly (promote to a structured
      variant when a recurring case shows up).
- [ ] Generic parameters on `struct`/`enum`/inherent-`impl` carry no trait
      bounds — bounds live on the methods that use them via `where`.
      Exceptions: derived/structural trait impls (`impl<T: Clone> Clone for
      Foo<T>`), documented storage-shape bounds.
- [ ] Optional (feature-gated) trait impls — `serde`/`buffa`/`arbitrary`/… —
      grouped in one gated `const _: () = { … };` block (single `#[cfg]` +
      `doc(cfg)`, private helpers/imports scoped inside), not scattered
      per-item; unconditional impls stay un-gated at module level.
- [ ] serde: every `Option` field is `#[serde(skip_serializing_if =
      "Option::is_none")]` (never emit `null`); every field with a default
      is `serde(default)` (container-level if the whole struct has a
      meaningful `Default`, else per-field `default` / `default = "fn"`).
- [ ] No module-name stutter; bare names don't shadow std.
- [ ] Test matrix + fmt + clippy clean; out-of-tree name references updated if
      anything was renamed.

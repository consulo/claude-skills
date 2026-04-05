---
name: jspecify
description: >
  Use this skill whenever null safety annotations are written or reviewed in Consulo.
  Trigger on: "@Nullable", "@NullMarked", "@NotNull", "@Nonnull", "null safety",
  "nullability", "nullable array", "array nullable", "@Nullable []", "jspecify",
  "org.jspecify", or any code that adds or removes nullability annotations.
  MUST be used proactively whenever nullability of a type is being expressed in code.
---

# JSpecify Null Safety in Consulo

Consulo uses [JSpecify](https://jspecify.dev/) for null safety. The module-level
`@NullMarked` annotation makes **non-null the default** for all types in the module,
so only nullable exceptions need to be marked.

---                                             

## Module-Level `@NullMarked`

Every `module-info.java` must carry `@NullMarked`. This single annotation makes every
parameter, return type, and field in the module implicitly non-null — no per-element
`@Nonnull` / `@NotNull` annotations are needed or allowed.

```java
import org.jspecify.annotations.NullMarked;

@NullMarked
module consulo.my.module {
    requires consulo.some.api;
    exports consulo.my.module;
}
```

---

## Only `@Nullable` Is Used — Never `@NotNull` / `@Nonnull`

Because non-null is the default under `@NullMarked`, the only annotation you ever add
to individual elements is `@Nullable` from `org.jspecify.annotations`.

```java
import org.jspecify.annotations.Nullable;

// ✅ Correct — mark only what can be null
public @Nullable String findName(String key) { ... }
public void process(@Nullable Project project) { ... }
private @Nullable BuildHandler myHandler;

// ❌ Wrong — @NotNull / @Nonnull are redundant and forbidden
public @NotNull String getName() { ... }
public void process(@Nonnull Project project) { ... }
```

**Never use:**
- `@NotNull` (from any package: `org.jetbrains`, `javax.annotation`, etc.)
- `@Nonnull` (from any package)

---

## Placement — Annotation Goes Before the Type

Place `@Nullable` directly before the **type** it annotates, not before any modifier.

```java
// ✅ Correct
public @Nullable String getName() { ... }
private @Nullable Project myProject;
public void foo(@Nullable String value) { ... }

// ❌ Wrong — annotation after modifiers but before type is a common mistake
@Nullable public String getName() { ... }
```

---

## Array Nullability Syntax — Do Not "Fix" This

JSpecify uses distinct positions for annotating the **array itself** vs the **element type**.
This syntax is correct and must not be altered:

| What is nullable | Syntax | Meaning |
|---|---|---|
| The array reference | `Object @Nullable []` | The variable can be `null`; elements are non-null |
| The element type | `@Nullable Object []` | Array is non-null; elements can be `null` |
| Both | `@Nullable Object @Nullable []` | Both can be `null` |

```java
// ✅ Correct — the array itself may be null, elements are non-null
public Object @Nullable [] getItems() { ... }

// ✅ Correct — array is non-null, but elements may be null
public @Nullable Object[] getItems() { ... }

// ❌ Wrong — do NOT rewrite Object @Nullable [] as @Nullable Object[]
//   These have different meanings; never "normalize" them
```

**Rule: when you see `SomeType @Nullable []`, leave it exactly as written.**
This is intentional JSpecify syntax, not a mistake.

---

## Summary Cheatsheet

| Situation | Annotation |
|---|---|
| Value can be `null` | `@Nullable` (from `org.jspecify.annotations`) |
| Value cannot be `null` | _(no annotation — non-null is the default)_ |
| Module declaration | `@NullMarked` on `module-info.java` |
| Array reference nullable | `Type @Nullable []` |
| Array element nullable | `@Nullable Type[]` |
| `@NotNull` / `@Nonnull` | ❌ Never — forbidden |

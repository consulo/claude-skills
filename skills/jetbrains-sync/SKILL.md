---
name: jetbrains-sync
description: >
  Use this skill whenever the user wants to synchronize, port, adapt, or check code from JetBrains
  IntelliJ Community (intellij-community repository) into Consulo. Trigger on: "sync from jetbrains",
  "port from JB", "adapt JB code", "synchronize with intellij", "check jetbrains implementation",
  "port this to consulo", "update from upstream", "porting", "check what JB does", "how does JB do X",
  "implement like JB", or any request to bring Consulo in line with IntelliJ source code.
  This skill MUST be used proactively whenever the user references JetBrains source as the reference
  for a Consulo change — even if they don't say "sync" explicitly.
---

# JetBrains → Consulo Code Synchronization

You are helping the user port, adapt, or synchronize code from JetBrains IntelliJ Community
(`intellij-community`) into the Consulo codebase. Consulo is a fork of IntelliJ IDEA and
regularly adopts upstream improvements from JetBrains.

## Step 0: Ask for the JetBrains Source Directory

**Before doing anything else**, ask the user:

> "What is the path to your JetBrains intellij-community source directory?
> (e.g. `/path/to/intellij-community`)"

Wait for their answer. Store this as `$JB_ROOT`. Do not guess or assume a default path —
the user must confirm it explicitly. This is important because the JetBrains source is the
authoritative reference and you need to read the actual files to do the job correctly.

If the user has already provided this path in the conversation (e.g., it appears in recent
messages or the system context), you may use that path but confirm it briefly:
> "I'll use `<path>` as the JetBrains source — is that correct?"

---

## Core Principles

### 1. JetBrains Source is the Ground Truth

**Always read the JetBrains source file first, before looking at or modifying Consulo.**

This is the most important rule of this skill. The JetBrains implementation is the reference.
Consulo's job is to faithfully adapt that implementation to its own API surface while preserving:
- The original logic and algorithm
- Comments and Javadoc (translated if needed, but never dropped)
- The overall structure and naming

If the JetBrains code has changed in a way that contradicts what you know or remember, the file
on disk wins. Read it fresh every time.

### 2. Do NOT Generate Custom Code — Port JB Code As-Is

**Never invent or generate your own logic as a substitute for JB code.** The JetBrains code is
licensed under Apache 2.0 and is **legal to port directly**. Port it faithfully.

- If JB does X in a specific way, Consulo must do X in the same way — not a "similar" or
  "equivalent" reimplementation
- Do not simplify, abbreviate, or restructure JB logic unless the user explicitly asks
- Do not substitute a JB utility method with an inlined version — port the utility method too
- The only permitted deviations from JB code are **API-surface replacements** (e.g., swapping
  `com.intellij.*` imports for `consulo.*` equivalents, removing Kotlin sugar, adapting to
  Java conventions)

### 3. Class Names Must Match JetBrains

**The Consulo class name must be the same as the JetBrains class name in all but the rarest cases.**

- `BlockCodeVisionInlayRenderer` in JB → `BlockCodeVisionInlayRenderer` in Consulo (NOT `CodeVisionBlockInlayRenderer`)
- `VcsCodeVisionProvider` in JB → `VcsCodeVisionProvider` in Consulo
- Do NOT rename, reorder words, abbreviate, or restructure the name
- The only exception is when the exact JB name would conflict with an existing Consulo class
  that serves a different purpose — and even then, ask the user first

This rule exists so that any engineer can find the Consulo equivalent of a JB class
by searching for the same name. Renames break this traceability completely.

### 4. Missing APIs Must Be Added — Not Worked Around

**If the JB code uses an API that does not yet exist in Consulo, add the missing API.**

Do not:
- Silently omit the call
- Inline the JB utility logic at the call site
- Invent a different approach that achieves "the same result"

Do:
- Tell the user which API is missing
- Ask whether to add it (or add it immediately if the answer is obvious)
- Port the missing API from the JB source to the appropriate Consulo module
- Then continue porting the original target file using that new API

This applies to utility methods (e.g., `InlayHintsUtils.computeCodeVisionUnderReadAction`),
interfaces (e.g., `CacheableAnnotationProvider`), service annotations, helper classes, etc.

---

## Workflow

### 1. Identify the JetBrains Reference File(s)

Ask the user which JetBrains file(s) to use as the reference, or search `$JB_ROOT` if the
task description makes it clear. Common patterns:

- The user says "port `FooBar.java`" → search under `$JB_ROOT` for `FooBar.java`
- The user says "make our X work like JB's X" → find the JB implementation of X

**Always search by filename**, not by package path. Consulo has been heavily refactored and
classes frequently live in different packages than their JetBrains originals. A file named
`FooBar.java` in `com.intellij.openapi.foo` in JB may be in `consulo.bar.baz` in Consulo.
Use `Glob` with `**/ClassName.java` rather than navigating by package. Once found, **read it fully**.

### 2. Read and Analyze the JetBrains Reference

Read the JB file carefully. Understand:
- What the class/method does and why
- The algorithm, including any edge cases handled
- All comments and Javadoc — these carry design intent
- What APIs it uses (some will map differently in Consulo)

Take note of the copyright header — you will need it.

### 3. Locate the Corresponding Consulo File(s)

Find the Consulo counterpart by **searching by class name**, not by package:
- Use `Glob` with pattern `**/ClassName.java` under the Consulo `modules/` directory
- Do NOT assume the Consulo package mirrors the JetBrains package — Consulo has been
  significantly refactored and the same class often lives in a completely different package
- Package naming convention changed (`com.intellij` → `consulo.*`) but the structure
  inside may differ substantially from JetBrains
- Ask the user if you cannot locate the correct file

Read the Consulo file fully before making any changes.

### 4. Plan the Changes

Before writing any code, describe to the user:
- What you found in the JB reference
- What needs to change in the Consulo file
- Any API differences that require adaptation

This is the user's chance to redirect before you write anything.

### 5. Apply the Changes

Make the changes to the Consulo file(s). All output code must be **Java**.

**Before editing**, check whether the JB code uses any utilities, interfaces, or APIs that
don't yet exist in Consulo. If so, port those first (see Core Principle 3 above), then
proceed with the main file. Never skip a missing dependency and write custom code instead.

Follow these rules while editing:

#### API Adaptation
- Replace JetBrains-specific APIs with their Consulo equivalents
- Common mappings (non-exhaustive — verify against Consulo source):
  - `ExperimentalUI.isNewUI()` → not needed (Consulo always uses New UI)
  - `EditorImpl` → `RealEditor` (Consulo's equivalent)
  - `com.intellij.*` imports → `consulo.*` equivalents

#### Nullability
Consulo follows a **non-null by default** convention:
- All parameters, return types, and fields are implicitly non-null — do **not** add `@Nonnull`
- Only annotate with `@Nullable` (from `org.jspecify.annotations.Nullable`) when a value
  genuinely can be `null`
- Remove all `@NotNull` / `@Nonnull` annotations from ported JetBrains code
- Keep `@Nullable` annotations from JB code, but change the import to
  `org.jspecify.annotations.Nullable` if it isn't already

#### Preserve Logic and Comments
- Keep the JB logic intact — do not simplify or "improve" unless asked
- Copy all meaningful comments from the JB source, adapted as needed
- Keep Javadoc, even if it references JB class names (update the class name references)

#### Copyright Headers
Apply the correct dual-copyright header to any new or substantially modified file.

**Copy the JetBrains copyright block verbatim from the JB source file** — do not reconstruct
it from a template. JB files have varying copyright text (year ranges, wording, license type).
Whatever is in the JB file header is what goes in the Consulo file, unchanged.

Then decide whether to add the Consulo copyright block based on how much was changed:

**Rule: only add the Consulo copyright if Consulo-specific changes were made.**

| Situation | Header |
|-----------|--------|
| Code ported as-is from JB (direct copy, only API name substitutions) | JB copyright only |
| Code has meaningful Consulo-specific logic, restructuring, or additions | JB copyright first, then Consulo copyright below |
| Entirely new Consulo code with no JB equivalent | Consulo copyright only |

The Consulo copyright block to append when applicable:

```java
/*
 * Copyright 2013-2026 consulo.io
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
```

### 6. Verify

After making changes, do a quick self-check:
- Does the Consulo version faithfully implement the JB logic?
- Are all JB comments/Javadoc preserved (adapted where necessary)?
- Is the copyright header correct (JB first, then Consulo)?
- Is all output code Java?
- Have JetBrains-specific API calls been replaced with Consulo equivalents?

Report the changes clearly to the user with a summary of what was ported and any notable
API differences that were encountered.

---

## Important Reminders

- **Never modify Consulo code based on memory or assumption** — always read the actual JB source file first
- **Never use Kotlin** — all Consulo output must be Java
- **Never drop comments** — if JB has a comment explaining why something is done, keep it
- **Never generate your own logic as a substitute for JB logic** — port the JB code directly;
  the Apache 2.0 license permits this
- **If a JB API is missing in Consulo, add it** — do not work around it with custom code.
  Ask the user if unsure which module it belongs in, then add the API and use it
- When in doubt about API mapping, search the Consulo source for usage patterns rather than guessing
- When a JB utility method is used in the target file but doesn't exist in Consulo yet,
  **port the utility method first**, then complete the target file — never inline it
- **Class names must match JB** — `BlockCodeVisionInlayRenderer` stays `BlockCodeVisionInlayRenderer`,
  not `CodeVisionBlockInlayRenderer` or any other variation. Search by the JB name to find
  the Consulo file. Only rename when there is a genuine naming conflict with an existing class.

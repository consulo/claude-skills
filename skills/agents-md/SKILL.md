---
name: agents-md
description: >
  Use this skill at the very start of every conversation. Read AGENTS.md from the current
  working directory if it exists. Trigger on: conversation start, "read agents.md",
  "check agents.md", "load agents.md". MUST be used proactively at the beginning of
  every session before any other work begins.
---

# Read AGENTS.md

At the start of every conversation, check for an `AGENTS.md` file in the current working
directory and read it if present.

## Steps

1. Check if `AGENTS.md` exists in the current working directory using Glob with pattern `AGENTS.md`.
2. If found, read it with the Read tool.
3. Follow any instructions it contains for the current project.
4. If not found, silently continue — do not mention the missing file unless the user asks.

## Purpose

`AGENTS.md` is a project-level instruction file (similar to `CLAUDE.md`) that provides
context, conventions, and rules specific to the repository. Reading it at the start ensures
all subsequent work follows the project's documented guidelines.

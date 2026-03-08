# CLAUDE.md

## Project

moly — a zero-dependency JSON library for Scala 3 that gives you circe's powerful AST when you need it and skips it entirely when you don't. One codec, two execution modes (AST + streaming). Built-in streaming engine, no external dependencies.

## Apache 2.0 Compliance (Non-Negotiable)

moly derives code from circe, which is licensed under Apache 2.0. These rules MUST be followed at all times. Violating them is a license violation.

### 1. File headers on derived code

Every file that contains code copied or derived from circe MUST have this header:

```scala
// This file is derived from circe (https://github.com/circe/circe)
// Copyright (c) 2015, Ephox Pty Ltd, Mark Hibberd, Sean Parsons, Travis Brown,
// and other contributors. Licensed under Apache 2.0.
//
// Modifications by moly contributors:
// - <describe what was changed>
```

Files written from scratch (not derived from circe) do NOT need this header.

### 2. LICENSE and NOTICE files

- `LICENSE` — the full Apache 2.0 license text. Must be included in every distribution. Never modify or remove.
- `NOTICE` — attribution to both moly contributors and circe's original authors. Never remove circe's attribution. May add new moly contributors.

### 3. No circe trademarks

- Never call this project "circe" or imply it is endorsed by circe's maintainers.
- Use "derived from circe", "compatible with circe", or "inspired by circe" — not "is circe" or "official circe".
- The name "moly" is intentionally distinct.

### 4. Modifications must be noted

When modifying a file derived from circe, update the header's modification list to describe what changed. Be specific:
- Good: "Added `writeTo`/`readFrom` streaming methods to Codec trait"
- Bad: "Modified" (too vague)

### 5. Redistribution

Any distribution (source or binary) must include both `LICENSE` and `NOTICE` files. This includes published Maven artifacts — Mill's `publishSonatypeCentral` handles this automatically if the files are in the repo root.

## Design Principles

- **Zero dependencies** — one jar, nothing else on the classpath. No cats, no jawn, no jsoniter-scala. Built-in streaming reader/writer, built-in lightweight error types. A core JSON library should be self-contained.
- **Don't pay for what you don't use** — if user code never touches the `Json` AST, no `Json` nodes should be allocated.
- **Wire compatibility with circe** — same JSON output, same decoding behavior, same error messages. If it works with circe, it must work identically with moly.
- **Gradual adoption** — hand-written codecs work via AST fallback. Derived codecs get the fast streaming path. No all-or-nothing migration.
- **One typeclass** — `Codec[A]` has both AST methods (`toJson`/`fromJson`) and streaming methods (`writeTo`/`readFrom`). No parallel typeclass hierarchies.

## Build Commands

TBD — project is in early development.

# Design

## Architecture

```
                    ┌─────────────────────────────────┐
                    │           User Code              │
                    └──────┬──────────────┬────────────┘
                           │              │
                    ┌──────▼──────┐ ┌─────▼─────────┐
                    │  AST mode   │ │ Stream mode    │
                    │             │ │                │
                    │ Json ← parse│ │ bytes → A      │
                    │ Json → print│ │ A → bytes      │
                    │ transform   │ │ (no Json)      │
                    │ inspect     │ │                │
                    └──────┬──────┘ └─────┬─────────┘
                           │              │
                    ┌──────▼──────────────▼────────────┐
                    │     Encoder[A] / Decoder[A]      │
                    │                                   │
                    │  apply(a): Json     ← AST path    │
                    │  writeTo(a, out)    ← stream path │
                    │  apply(c): Result   ← AST path    │
                    │  readFrom(in)       ← stream path │
                    └──────┬──────────────┬────────────┘
                           │              │
                    ┌──────▼──────────────▼────────────┐
                    │          moly (single jar)       │
                    │                                   │
                    │  Json AST    JsonReader/JsonWriter │
                    │  HCursor     (built-in streaming)  │
                    │  Printer     error types           │
                    │                                   │
                    │  zero dependencies                │
                    └──────────────────────────────────┘
```

## Modules

```
moly/
├── core/           # Everything. Single module, zero dependencies.
│                   #   Json AST, HCursor, DecodingFailure (derived from circe-core)
│                   #   Encoder, Decoder, Codec (with streaming methods)
│                   #   JsonReader, JsonWriter (built-in streaming engine)
│                   #   Parser (String → Json or String → A)
│                   #   Printer (Json → String)
│                   #   Macro derivation (generates both AST + streaming code)
│                   #   Configuration (snake_case, defaults, discriminator)
│                   #   KeyEncoder, KeyDecoder (map keys)
│
├── optics/         # Optional. Lenses/prisms for Json traversal/modification.
│                   # Only needed when working with the AST directly.
│                   # Only external dep: none (self-contained optics).
│
├── tapir/          # Tapir integration. Provides tapir Codec[String, A, Json]
│                   # using the streaming path. Lives here until moly matures
│                   # enough to be contributed upstream to tapir.
│                   # Only external dep: tapir-core (provided).
│
└── testing/        # Test utilities: Arbitrary[Json], roundtrip property tests,
                    # cross-codec tests (encode with moly, decode with circe).
                    # Only external dep: test frameworks (test scope).
```

### Why a single core module?

circe splits into circe-core, circe-parser, circe-jawn, circe-generic, circe-derivation — five artifacts just to parse a case class. Each brings its own transitive deps.

moly puts everything in one module: AST, parser, printer, streaming engine, derivation. One dependency in your build file, nothing else on the classpath. The streaming `JsonReader`/`JsonWriter` are built in — purpose-built for moly's use case, not a general-purpose library.

This is possible because we control the whole stack. circe delegates parsing to jawn and derivation to a separate module because they're maintained by different people with different release cycles. moly is one project — everything moves together.

## Core typeclasses

Separate `Encoder` and `Decoder` (like circe), each with both AST and streaming methods:

```scala
trait Encoder[A]:
  // AST mode — produces a Json tree
  def apply(a: A): Json

  // Stream mode — writes directly to output, no Json allocated
  // Default: falls back to AST mode (always works, just slower)
  def writeTo(a: A, out: JsonWriter): Unit =
    Json.writeJson(apply(a), out)

trait Decoder[A]:
  // AST mode — reads from a Json tree (via cursor)
  def apply(c: HCursor): Either[DecodingFailure, A]

  // Stream mode — reads directly from input, no Json allocated
  // Default: falls back to AST mode (always works, just slower)
  def readFrom(in: JsonReader): A =
    val json = Json.readJson(in)
    apply(json.hcursor).getOrElse(...)

// Convenience: both directions in one
trait Codec[A] extends Encoder[A] with Decoder[A]
```

### Why separate Encoder/Decoder instead of a single Codec?

- **circe compatibility** — circe users expect separate typeclasses
- **Flexibility** — some types only need one direction (e.g., API responses only need encoding)
- **Composition** — `Encoder.contramap` and `Decoder.map` work independently

### Why default streaming implementations?

Hand-written codecs that only define the AST methods get streaming for free — it just goes through the AST (slow path). This means:

- **Zero migration cost** — existing circe codecs work as-is
- **Gradual speedup** — switch to `derives Codec` one type at a time, each one gets the fast path
- **No dead code** — the AST methods are always there, always usable

## Macro derivation

A single `derives Codec` generates optimized implementations for ALL four methods:

```scala
case class User(name: String, age: Int) derives Codec
```

The macro produces:

**AST mode** (circe-compatible):
```scala
// Encoder.apply — same output as circe
def apply(a: User): Json =
  Json.obj("name" -> Json.fromString(a.name), "age" -> Json.fromInt(a.age))

// Decoder.apply — same behavior as circe
def apply(c: HCursor): Either[DecodingFailure, User] =
  for
    name <- c.downField("name").as[String]
    age  <- c.downField("age").as[Int]
  yield User(name, age)
```

**Stream mode** (the fast path):
```scala
// Encoder.writeTo — direct field writes, no Json tree
def writeTo(a: User, out: JsonWriter): Unit =
  out.writeObjectStart()
  out.writeNonEscapedAsciiKey("name")
  out.writeVal(a.name)
  out.writeNonEscapedAsciiKey("age")
  out.writeVal(a.age)
  out.writeObjectEnd()

// Decoder.readFrom — typed locals, direct constructor
def readFrom(in: JsonReader): User =
  var _name: String = null
  var _age: Int = 0
  in.readObjectStart()
  while in.readField() do
    if in.isField("name") then _name = in.readString()
    else if in.isField("age") then _age = in.readInt()
  new User(_name, _age)
```

Both modes from one derivation. The AST mode ensures circe compatibility; the stream mode ensures performance.

## Boundary API

The top-level API steers to the fast path when available:

```scala
object moly:
  // Stream mode — uses Decoder.readFrom (fast if derived, slow fallback if hand-written)
  def decode[A: Decoder](input: String): Either[Error, A]
  def encode[A: Encoder](a: A): String

  // AST mode — always builds the Json tree
  def parse(input: String): Either[ParsingFailure, Json]
```

`decode[A]` calls `decoder.readFrom(reader)` — no `Json` allocated for derived types. For hand-written decoders, `readFrom` falls back through the AST automatically.

`parse` always returns `Json` — for when you need to inspect or transform before decoding.

## Tapir integration

The tapir module lives in this repo. It provides a `tapir.Codec[String, A, CodecFormat.Json]` that uses the streaming path:

```scala
given molyCodec[A: moly.Encoder: moly.Decoder]: tapir.Codec[String, A, Json] =
  tapir.Codec.json[A] { string =>
    moly.decode[A](string)  // streaming, no AST
  } { a =>
    moly.encode(a)           // streaming, no AST
  }
```

### Why in this repo, not in tapir?

tapir won't accept an integration module for an unpublished, early-stage library — and they shouldn't. The integration lives here until:

1. moly is published and stable
2. Real-world usage proves the approach works
3. The API surface is settled

Then we contribute it upstream. This is the standard pattern (jsoniter-scala did the same — their tapir module lived in their repo first).

## What we take from circe

Derived from circe-core under Apache 2.0 (with attribution, see NOTICE):

- `Json` enum and all operations (mapObject, mapArray, fold, etc.)
- `HCursor`, `ACursor`, cursor navigation
- `DecodingFailure`, error accumulation
- `Encoder`, `Decoder` base traits and built-in instances
- `Codec` trait
- `Configuration` for configured derivation (snake_case, defaults, discriminator)
- `Printer` configuration (drop nulls, indent, sort keys)
- `KeyEncoder`, `KeyDecoder` for map keys

**What we add:**

- `writeTo` / `readFrom` streaming methods on Encoder/Decoder
- Macro derivation that generates both AST and streaming code
- Built-in streaming `JsonReader` / `JsonWriter` (zero external deps)
- Built-in parser and printer backed by the streaming engine
- Lightweight `Validated` / `NonEmptyList` (replaces cats dependency)
- Tapir integration module

**What we don't take:**

- circe-generic (replaced by our own derivation)
- circe-jawn (replaced by built-in streaming reader)
- circe-literal (Scala 2 string interpolation macros)
- cats dependency (replaced by lightweight built-in types)

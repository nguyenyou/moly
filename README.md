# moly

![Scala3](https://img.shields.io/badge/Scala%203-%23de3423.svg?logo=scala&logoColor=white)

**Don't pay for what you don't use.**

moly is a JSON library for Scala 3 that gives you circe's powerful AST when you need it — and skips it entirely when you don't.

## The name

In Homer's Odyssey, the sorceress **Circe** transforms Odysseus's men — forcing them through an intermediate form they never asked for. The god Hermes gives Odysseus a magical herb called **moly**, which lets him resist the unnecessary transformation.

circe (the library) does the same thing: every JSON operation is forced through an intermediate `Json` AST — even when you just want `bytes → case class`. moly is the antidote. It lets you skip the transformation when you don't need it, while keeping the full power of the AST when you do.

## The problem

circe is a beautifully designed library. The `Json` AST is genuinely powerful — optics, partial decoding, middleware transforms, schema migration. We love it.

But the AST is **mandatory**. Every `decode[MyType](jsonString)` does this:

```
bytes → Json → MyType
```

The `Json` in the middle is allocated, traversed, and garbage collected — even though nobody asked for it. In HTTP services where 95% of call sites just deserialize a request body or serialize a response, this is pure waste.

The numbers tell the story. In our benchmarks ([circe-sanely-auto](https://github.com/nguyenyou/circe-sanely-auto)), skipping the AST yields **5–6x faster reads and writes** with **89–96% less memory allocation** compared to circe-jawn.

The problem isn't the AST — it's that there's no way to opt out.

## The idea

One codec. Two execution modes. You choose.

```scala
trait Codec[A]:
  // AST mode — for when you need to inspect, transform, manipulate
  def toJson(a: A): Json
  def fromJson(json: Json): Either[DecodingFailure, A]

  // Stream mode — for when you just want bytes ↔ A
  // Default implementation goes through AST (always works, just slower)
  def writeTo(a: A, out: JsonWriter): Unit =
    out.writeJson(toJson(a))
  def readFrom(in: JsonReader): A =
    fromJson(in.readJson()).getOrElse(...)
```

**Macro-derived codecs override both modes.** A single `derives Codec` gives you an optimized AST codec *and* an optimized streaming codec. Hand-written codecs only need to define `toJson`/`fromJson` — the streaming fallback works automatically, just slower.

The boundary picks the right mode:

```scala
// Fast — streaming, no AST allocated
val user = moly.decode[User](bytes)
val bytes = moly.encode(user)

// AST — when you need it
val json: Json = moly.parse(bytes)
val modified = json.mapObject(_.add("role", Json.fromString("admin")))
val user = modified.as[User]
```

**You don't pay for what you don't use.** If you never touch the AST, you never allocate it. If you need the AST, it's right there — same power as circe, same API.

## Zero dependencies

moly has no transitive dependencies. One jar, nothing else on the classpath.

circe depends on cats-core (for `Validated`, `NonEmptyList`, `Show`, `Eq`) and jawn (for parsing). These pull in a dependency tree that every downstream library inherits. moly replaces all of it:

| circe needs | moly |
|---|---|
| cats `Validated` / `NonEmptyList` | Built-in (lightweight implementations) |
| cats `Show` / `Eq` | Not needed (Scala 3 has `CanEqual`, `toString`) |
| jawn (parser) | Built-in streaming reader |
| jsoniter-scala | Built-in streaming reader/writer |

The streaming JSON reader/writer is built into moly — purpose-built for our use case, not a general-purpose library. The macro-generated code (typed locals, direct constructors, hash-based field dispatch) is where most of the speed comes from; the reader/writer just needs to be fast enough to not be the bottleneck.

This is a deliberate design choice. A core JSON library should be self-contained. You add moly to your build and you're done — no version conflicts, no transitive dependency surprises, no classpath bloat.

## Why not just use jsoniter-scala?

jsoniter-scala is an excellent streaming JSON library. But it has a different JSON format (skips `None` fields, uses flat discriminators) and no AST for manipulation. If you want to:

- Inspect or transform JSON before decoding
- Use optics to navigate deep structures
- Maintain wire compatibility with circe-based services
- Gradually migrate a circe codebase without changing JSON format

...you need both the AST *and* the fast path. That's moly.

## circe compatibility

We love circe. We don't want to replace it — we want to fill a gap.

moly aims to be **wire-compatible** with circe: same JSON output, same decoding behavior, same error messages. If your application works with circe, switching to moly should produce identical results. The encoding format doesn't change; only the execution path does.

We reuse circe's `Json` AST because it's well-designed and battle-tested. The Apache 2.0 license makes this possible, and we gratefully acknowledge the work of Travis Brown and the circe contributors.

## A wish

If circe ever considers adding a streaming fast path — a way to skip the AST when you don't need it — we would love to see it. circe is the right home for this idea. The ecosystem (tapir, http4s, sttp) already speaks circe. A streaming mode built into circe itself would benefit everyone without fragmentation.

Until then, moly exists to prove the idea works and to serve codebases that need the performance today.

## Status

Early development. Not yet published.

## License

Apache 2.0 — same as circe.

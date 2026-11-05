---
name: csharp-rules
description: Rust/F#/Go-inspired safety and explicitness rules for C#. Use when writing or refactoring C# and the user asks for Rust-like or functional style, Result/Option, discriminated unions, null safety, immutability, safe async/await, channels or actors, ownership thinking, or allocation-conscious, cache-friendly data layout.
---

# C# Rules

Make failure, absence, ownership, and concurrency visible in types instead of hidden in exceptions, nulls, and shared mutable state. Concrete code for every rule is in `patterns.md`.

## Rules

### 1. Null safety and explicit failure
- Code must compile cleanly with nullable reference types enabled. If the project lacks `<Nullable>enable</Nullable>`, propose enabling it (see `patterns.md` for the project settings).
- Use `T?` for "may be absent" and `required` members for mandatory data.
- Return a Result for expected failures (not found, validation, conflict). Throw only for bugs and unrecoverable conditions.
- If the project already references LanguageExt, OneOf, ErrorOr, or FluentResults, use it. Otherwise use the small sealed `Result<T>` from `patterns.md`; do not add a new library unless asked.

### 2. Closed domains with handled cases
- Model a closed set of variants as an abstract record with a private constructor and nested sealed records.
- C# cannot prove such hierarchies exhaustive, so end every switch over them with `_ => throw new UnreachableException()`. List every known case explicitly before that arm; never rely on the discard for real cases.
- For enums, list every member; treat CS8509 and CS8524 as errors.

### 3. Immutability by default
- Prefer `record`, `readonly record struct`, `init`, and `readonly` fields.
- Expose `IReadOnlyList<T>` / `IReadOnlyDictionary<TKey,TValue>` at boundaries; store `ImmutableArray<T>` or `FrozenDictionary`/`FrozenSet` (.NET 8+). Avoid `ImmutableList<T>` except for frequent persistent updates.
- Keep necessary mutation private and local to one method or type.

### 4. Ownership and lifetime
- Every `IDisposable`/`IAsyncDisposable` has one obvious owner that disposes it with `using`/`await using`.
- A method that takes ownership of a disposable says so in its name or XML doc; otherwise the caller keeps ownership.
- Use `Span<T>`/`ReadOnlySpan<T>` for non-owning access in hot paths, `Memory<T>` when the access crosses an `await`.

### 5. Composition over inheritance
- Seal classes and records by default.
- Compose small types behind small interfaces instead of building base-class hierarchies.

### 6. Concurrency
- Share nothing mutable across threads by default; pass immutable messages through `Channel<T>` (bounded unless proven unnecessary).
- When state must be shared, keep it private behind one lock owned by the same type.
- Every started task is awaited, combined with `Task.WhenAll`, or owned by a host (`BackgroundService`) that observes failures.
- Use actors (Orleans, Akka.NET) only for long-lived stateful systems, never for plain request/response code.

### 7. Async/await
- Async all the way: no `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` in application code.
- `async void` only for event handlers.
- Accept a `CancellationToken` in every async public method and pass it to every call that accepts one. Use a linked `CancellationTokenSource` for timeouts.
- Return `Task`/`Task<T>` by default. Use `ValueTask` only on measured hot paths that usually complete synchronously, and never await a `ValueTask` twice.
- `ConfigureAwait(false)` in library code; omit it in app, UI, and ASP.NET Core code.
- No `Task.Run` around I/O-bound work.

### 8. Flat, allocation-conscious data (Goose-inspired)
Apply only to hot paths or large data sets; elsewhere clarity wins.
- Prefer arrays of structs and index handles (`readonly record struct NodeId(int Index)`) over object graphs and long-lived references into growable collections.
- Rent buffers from `ArrayPool<T>`, use `stackalloc` for small fixed buffers, and `[InlineArray]` (C# 12) for fixed-size inline data.

### 9. Explicit, simple APIs
- Replace primitive identifiers and magic strings with domain types (`OrderId`, `Money`, enums).
- Choose the boring solution: no speculative abstraction, interfaces only where there are two implementations or a test seam.

## Limits
- Do not claim memory-safety or data-race-freedom guarantees; C# only approximates them.
- Do not rewrite working code wholesale. Make incremental, reviewable changes unless a redesign is requested.

## Output
After changes, list each significant change with the rule number that drove it.

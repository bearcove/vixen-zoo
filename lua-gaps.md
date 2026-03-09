# Things used in build.vx that aren't in the spec

## Invented syntax/operations

- **`..` spread in array literals** — The spec has spread in tuples
  (`(1, ..two_three, 4)`) and in struct literals / keyword args, but does
  not explicitly show `[..a, ..b]` for arrays. We're assuming it works
  the same way.

- **`Tree.from_path(ws)`** — No API to convert a workspace `Path` into a `Tree`.
  The spec says `#fs` provides lazy filesystem access and the entry point receives
  a `Path`, but doesn't specify how to get a `Tree` from it.

- **`Tree.of(map)` with `Blob` values** — The spec says `Tree.of(map)`
  creates a tree from a map of names to entries, but doesn't specify
  whether the values can be `Blob` (flat file tree) or only `Tree`/`Blob | Tree`.
  We're assuming `Map[String, Blob]` works to build a flat directory of files.

- **`..tree` spread in exec args** — Spreading a `Tree` into an arg array
  yields its `Blob`s. Blobs carry their names. When a `Blob` appears in exec
  args, it is auto-mounted (by name, hash-prefixed if conflicts). This replaces
  manual `mounts:` maps + name tracking for the common case where every mounted
  file IS an argument (ar, ld). Exec args become `Array[String | Path | Blob]`.

- **`mounts: tree` (bare Tree)** — Mount a tree at root instead of requiring
  a `Map[Path, Tree]`. Useful when the process needs access to the whole tree
  (e.g. C compilation needing headers) but only some files are in argv.

- **No `writable:` — cwd is writable by default** — The spec has an explicit
  `writable` parameter on exec. We're dropping it: everything under `./` in the
  sandbox is writable. Mounts are read-only, output goes to cwd. Simpler model.

- **`Array[T].map(f)` returns `Stream[U]`** — Mapping over an array produces
  a stream, not another array. This makes map inherently lazy/concurrent —
  elements are computed on demand or in parallel by the scheduler.

- **`Array[K].to_map(f)` / `Stream[K].to_map(f)`** — Overload of `to_map`:
  when called with a function, elements become keys and `f(element)` becomes
  the value. `[:cc, :ar].to_map(discover)` produces `Map[Symbol, DiscoveredTool]`.
  Without an argument, `to_map()` collects a `Stream[(K, V)]` into `Map[K, V]`.

- **`Stream.next()`** — Takes the first element from a stream. The spec only
  has `.to_array()` for consuming streams. `.next()` avoids the pointless
  `.to_array()[0]` pattern when exec produces exactly one result.

- **`.map()` on functions (composition)** — `f.map(g)` composes: calling the
  result is `g(f(args))`. Used to bake `.next()` into a partially applied
  `exec`, so `cexec(args)` returns a value directly instead of a stream.
  Generic type params are checked at instantiation time.

- **`Stream[T].flatten()`** — When `T` is `Tree`, produces a single `Tree`
  containing all blobs from all trees in the stream. The spec has no stream
  flattening operation. Used to collect concurrent compilation results into
  a single tree for archiving/linking.

- **`Stream[T].flat_map(f)`** — Maps `f` over elements and flattens the
  resulting streams. `stream.flat_map(f)` is equivalent to `stream.map(f).flatten()`.
  Not in the spec.

- **UFCS (uniform function call syntax)** — `obj.method(args)` desugars to
  `method(obj, args)`. The spec has method calls on built-in types but doesn't
  explicitly confirm user-defined UFCS. We're assuming it works. Used for
  `stringify` on `DiscoveredTool` and UFCS calls like `.archive()`.

- **Pipe operator `|>`** — `a |> f($)` passes the result of `a` as `$` into
  the next expression. Used to chain exec calls: `cexec([ar, ...]) |> cexec([ranlib, $])`.
  `$` refers to the left-hand side result.

- **Destructuring `let`** — `let {cc, ar, ranlib} = toolchain;` destructures
  a struct into local bindings. The spec doesn't explicitly show struct
  destructuring in `let` bindings.

- **Nested function declarations** — `fn` declarations inside function bodies
  that close over the enclosing scope. The spec doesn't explicitly allow
  `fn` inside `fn`. Used for `compile`, `archive`, `link` inside `build()`.

- **`stringify` as UFCS convention** — Defining `fn stringify(self: T) -> String`
  gives a type a string representation. When a `DiscoveredTool` appears in an
  exec arg list (which expects strings), it is auto-stringified via this method.

## Proposed syntax changes (from this sketch)

- **Path literals: `@"src"` / `` @`out/${name}` ``** — `@` prefix on double-quoted
  or backtick strings produces a `Path`. Replaces `Path("src")`. Backtick form
  supports interpolation. Path joining uses interpolation (`` @`dir/${file}` ``)
  rather than a `/` operator on paths.

- **Attributes: `#[name("value")]`** — Rust-style attribute syntax. Replaces
  `@rename(...)`, `@tag(...)`, etc. Frees `@` for path literals.

- **Partial application: just `..`** — No prefix sigil needed. `f(args, ..)` is
  already unambiguous from the `..` in call position. Replaces `@f(args, ..)`.

- **Top-level `let` bindings** — The spec only allows type and function
  declarations at module scope. Top-level `let` is useful for constants like
  source file lists. Should be added to the spec (at least for immutable bindings).

- **Symbols (`:name`)** — Interned identifiers, compared by identity not content.
  `:cc` is a `Symbol`, not a `String`. Used as map keys where typos should be
  caught at compile time. `Symbol` has a `stringify` that returns the name as a
  `String`. Enables `Map[Symbol, T]` with destructuring:
  `let {cc, ar} = tools;` where `tools` is `Map[Symbol, DiscoveredTool]`.

- **Bare-name keyword shorthand** — `f(verifier)` is shorthand for
  `f(verifier: verifier)` when a keyword argument name matches the binding name.
  Replaces the previous `:verifier` shorthand, freeing `:` prefix for symbols.
  Same rule already applies to struct literals: `Foo { x, y }` for `Foo { x: x, y: y }`.

## Spec issues

- **`Process.stderr` / `Process.stdout` type** — The spec says `Stream[Blob]`
  but Blob is a content-addressed store object. Process output is ephemeral
  byte chunks from a pipe — should be `Stream[String]` or `Stream[Bytes]`.
  We're treating it as `Stream[String]`.

## Verifier host access

- The spec says verifiers should "hash binaries, check paths, verify versions"
  but doesn't specify what APIs are available inside a verifier closure for
  reading the executor's host filesystem. The current sketch uses
  `discover_exec` inside the verifier, which is likely wrong — `discover_exec`
  requires `#discover` capability and is meant for the `stabilize` resolve
  callback, not for verifiers.

## Resolved

- ~~Shell chaining in exec~~ — Fixed: two separate `exec` calls.
- ~~`++` operator~~ — Replaced with `..` spread in array literals.
- ~~`.fold()` on arrays~~ — No longer needed; replaced with `.to_map()`.
- ~~`.glob()` on merged trees~~ — No longer needed; object names are known statically.

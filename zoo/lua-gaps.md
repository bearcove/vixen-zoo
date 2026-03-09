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

- **`writable: [name]` with string names** — When a tree is mounted at root
  and ranlib needs to modify a file in place, `writable` takes the entry name
  as a string rather than requiring a `Path`.

- **UFCS (uniform function call syntax)** — `ctx.compile(file)` desugars to
  `compile(ctx, file)`. The spec has method calls on built-in types but doesn't
  explicitly confirm user-defined UFCS. We're assuming it works.

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

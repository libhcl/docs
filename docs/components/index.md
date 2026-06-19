# Components

`libhcl` is a pair of **plain-C, dependency-free** parser libraries (no Go, no
third-party code). Each parses HCL text once into an immutable AST that you walk
with accessor functions and then free with a single call.

| Library | Header / archive | Scope | What it does |
|---------|------------------|-------|--------------|
| [`c-hcl`](c-hcl.md) | `hcl.h` / `libhcl.a` | declarative subset of HCL native syntax | Bodies of attributes (`name = value`) and optionally labeled blocks (`type "label" { ... }`); values are strings, numbers, booleans, `null`, or lists thereof. No expressions, interpolation, functions, heredocs, or object literals — deliberately tiny. |
| [`c-hcl2`](c-hcl2.md) | `hcl2.h` / `libhcl2.a` | the HCL2 expression language | A tree-walking evaluator over a cty-lite value model: string templates with `${ }` interpolation, full operator/precedence set, conditionals, tuples/objects, traversal, function calls + a small standard library, for-expressions, splat, heredocs, `%{ }` directives, lazy configuration-body decoding, and the JSON profile. **A work in progress** — see its roadmap for the gaps to full spec compliance. |

## Shared conventions

- **Zero dependencies.** Only the C standard library (`c-hcl2` links `-lm` for
  math builtins). No vendored code.
- **Parse → walk → free.** `*_parse(...)` returns an immutable document; const
  accessors read it; one `*_free(...)` releases the whole AST.
- **Build with `make` or `cc`.** `make` produces a static archive
  (`libhcl.a` / `libhcl2.a`); `make test` runs the unit suite (opt-in
  AddressSanitizer via `SANITIZE=address`), `make cover` an llvm-cov report.
  No toolchain? Both ship a `pkgx.yaml` and a `./taskw` wrapper.
- **Tested under ASan with allocation fault-injection**, plus fuzzing on
  `c-hcl2`. Coverage targets ~99% lines / 100% of functions.
- **BSD-3-Clause.**

Each library documents the grammar it actually parses — lexical tokens, EBNF,
and parsing strategy — in a `GRAMMAR.md` in its repository.

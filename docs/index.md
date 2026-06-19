# libhcl

**HCL parsers written in C** — dependency-free, BSD-3-licensed, with no
third-party code. Two libraries that share a philosophy (parse once into an
immutable AST, walk it with accessors, free) but sit at opposite ends of the
HCL spectrum:

- [**c-hcl**](components/c-hcl.md) — a tiny parser for the *declarative subset*
  of HCL native syntax (attributes and labeled blocks; values are strings,
  numbers, booleans, `null`, or lists). One `.c`/`.h`, meant for configuration,
  not computation. ~99% line coverage / 100% of functions.
- [**c-hcl2**](components/c-hcl2.md) — a from-scratch implementation of **HCL2**,
  the heavyweight companion: a real expression language with templates,
  for-expressions, splat, heredocs, a cty-lite value model, lazy body decoding,
  and a JSON profile. Built milestone by milestone; **not yet a spec-complete
  HCL2** (see its roadmap).

Both are written in plain C (no Go, no runtime, no external dependencies),
build with `cc *.c` or `make`, and were grown out of the HCL needs of
[socket_vmnet](https://github.com/lima-vm/socket_vmnet) — `c-hcl` was extracted
from it directly.

## Components

| Library | Header | Scope | What it does |
|---------|--------|-------|--------------|
| [`c-hcl`](components/c-hcl.md) | `hcl.h` | declarative subset | Parses attributes + (optionally labeled) blocks whose values are scalars or lists. No expressions, interpolation, functions, or heredocs — config, not computation. |
| [`c-hcl2`](components/c-hcl2.md) | `hcl2.h` | HCL2 expression language | Evaluates the HCL2 expression sub-language (templates, operators, conditionals, for/splat, builtins) over a cty-lite value model; decodes configuration bodies lazily against a context; reads the JSON profile. Work in progress. |

## When to use which

Use **c-hcl** when you want a small, auditable config reader and your `.hcl`
files are plain attributes and blocks. Use **c-hcl2** when you need HCL's
*expression* language — interpolation, arithmetic, conditionals,
for-expressions, function calls. `c-hcl2` exposes c-hcl-compatible accessors, so
it can subsume the declarative use case as well.

## License

Every repository in the org is **BSD-3-Clause**.

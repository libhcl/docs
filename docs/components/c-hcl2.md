# c-hcl2 — HCL2 expression engine

[`github.com/libhcl/c-hcl2`](https://github.com/libhcl/c-hcl2) is a
**from-scratch C implementation of HCL2** — the heavyweight companion to
[c-hcl](c-hcl.md). Use `c-hcl` when you want a tiny config reader; reach for
`c-hcl2` when you need the **expression language**: interpolation, arithmetic,
conditionals, for-expressions, function calls.

!!! warning "Status: a work in progress — not yet spec-complete"
    "Full HCL2" is a large language (the Go reference, `hashicorp/hcl` v2 +
    `zclconf/go-cty`, is tens of thousands of lines). `c-hcl2` is built
    **milestone by milestone** and is honest about being in progress. The
    implemented surface is broad — the milestone breakdown and the remaining
    gaps to 100% native-syntax compatibility are tracked in `ROADMAP.md` in the
    repository.

## What works today

**Expressions** (`hcl2_eval`):

- numbers, booleans, `null`; quoted-string **templates** with `${ expr }`
  interpolation (and `$${` escape)
- tuples `[...]`, objects `{ k = v, ... }` (ident or string keys, `=` or `:`)
- unary `- !`; binary `+ - * / %`; comparison `== != < <= > >=`; logical
  `&& ||`; the conditional `cond ? a : b`; parentheses
- variable references with `.attr` / `[index]` traversal; function calls
- **for-expressions** — `[for x in xs : x*2 if x>0]`, `{for k, v in m : k => v}`
  (including object grouping mode)
- **splat** — `xs[*].name` (desugars to a for-expression; captures the trailing
  `.attr`/`[index]` traversal)
- **heredocs** — `<<EOF` and indented `<<-EOF` (common-leading-whitespace strip);
  the body is a template
- **template directives** — `%{ if }`/`%{ else }`/`%{ endif }`,
  `%{ for }`/`%{ endfor }` (nestable; `%%{` escape), plus whitespace trimming
  `${~ ~}` / `%{~ ~}`
- **variadic spread** — `max(xs...)` expands a tuple as trailing arguments

The standard library of builtins: `length`, `upper`, `lower`, `min`, `max`,
`abs`, `floor`, `ceil`, `join`, `split`, `concat`, `keys`, `values`,
`contains`, `lookup`, `coalesce`, `tostring`, `tonumber`, `tobool`,
`jsonencode`, `jsondecode`, plus the lazy special forms `try(...)` and
`can(...)` for graceful optional access.

**Configuration bodies** (`hcl2_parse`) — documents of attributes
(`name = expr`) and nested, optionally labeled blocks (`type "label" { ... }`),
with `#` / `//` / `/* */` comments. Attribute expressions decode **lazily**
against a context (`hcl2_body_attr_value`), and c-hcl-style accessors
(`hcl2_doc_root`, `hcl2_body_block_count`/`_at`, `hcl2_block_type`/`_label`/
`_body`) let `c-hcl2` subsume [c-hcl](c-hcl.md).

**Value model** — a cty-lite model with distinct collection kinds: tuple/object
plus real `HCL2_LIST` / `HCL2_SET` / `HCL2_MAP` (a list is not equal to a tuple,
a map not to an object). A **type-constraint / conversion** layer
(`hcl2_type_*` + `hcl2_convert`) coerces values toward a target type
(primitive coercions and `list`/`set`/`map`/`any` constraints).

**Unknown values** (`hcl2_unknown`, `HCL2_UNKNOWN`) model cty's plan-time
placeholders: any operation touching an unknown — arithmetic, comparison,
conditionals, traversal, calls, for-expressions, template interpolation /
directives — propagates unknown. Unknowns can carry their cty type
(`hcl2_unknown_of` / `hcl2_unknown_type`), and `hcl2_convert` refines them.

**JSON profile** — implemented end to end: `hcl2_parse_json` (value layer),
`hcl2_json_eval` (each JSON string as an HCL template), and `hcl2_json_decode`
+ `hcl2_schema_*` (schema-driven body layer decoding into the same
`hcl2_doc`/`hcl2_body` tree the native parser builds).

**Diagnostics** — both syntax and semantic/eval errors report `at line L,
column C` (AST nodes carry positions, so errors are located even for deferred
body decoding after the source buffer is gone). `hcl2_parse_diags` recovers at
the next line and gathers all body-level errors into a `hcl2_diags` list.

## API at a glance

```c
#include "hcl2.h"

hcl2_ctx *ctx = hcl2_ctx_new();
hcl2_ctx_set_var(ctx, "port", hcl2_number(8080));
hcl2_ctx_set_var(ctx, "name", hcl2_string("api"));

char err[256];
hcl2_value *v = hcl2_eval("\"${name}:${port + 1}\"", 0 /*=strlen*/, ctx,
                          err, sizeof err);
// v == "api:8081"

hcl2_value_free(v);
hcl2_ctx_free(ctx);
```

Bodies decode lazily against a context:

```c
hcl2_doc *doc = hcl2_parse(src, len, err, sizeof err);
const hcl2_body *root = hcl2_doc_root(doc);
hcl2_value *port = hcl2_body_attr_value(root, "port", ctx, err, sizeof err);
const hcl2_block *svc = hcl2_body_block_at(root, "service", 0);
```

See `hcl2.h` for the full value/context API and `GRAMMAR.md` for the lexical
tokens, EBNF (expressions and bodies), the Pratt precedence table, and the
template rules.

## Build & test

The `Makefile` builds `libhcl2.a` from nine translation units
(`value.c lexer.c parser.c eval.c body.c convert.c json.c json_body.c
builtins.c`) and links `-lm` for the math builtins:

```sh
make            # builds libhcl2.a
make test       # unit tests (add SANITIZE=address on a system clang)
make cover      # llvm-cov report
make fuzz       # deterministic fixed-seed fuzzing of the lexer/parser
```

No toolchain? With [pkgx](https://pkgx.sh): `dev` (reads `pkgx.yaml`) or
`./taskw test`.

Testing is under AddressSanitizer with **allocation fault-injection**
(`HCL2_FAULT_INJECT` budget hook + an OOM-scan harness) and **fuzzing**
(random / token-soup / seed-mutation inputs, exact-length buffers so ASan
catches over-reads, clean over millions of iterations). Coverage is **100% of
functions** and ~99% of lines.

## Roadmap & deliberate scope

The `ROADMAP.md` tracks status honestly. Highlights of what is **done**: the
expression engine (M1), configuration bodies (M2), template & collection
expressions (M3), most of the type system & multi-error diagnostics (M4), and
the full JSON profile (M5). Known **remaining** items are refinements rather
than missing features:

- full source **ranges** (start+end spans, currently a start point only)
- eval-level result-type **inference** onto operation-produced unknowns
- chained splats (`xs[*][*]`), rejected today with a clear error

One **deliberate scope decision**: numbers are IEEE-754 `double`, not cty's
arbitrary-precision `big.Float`. A `double` carries ~15–17 significant decimal
digits — ample for configuration values (ports, counts, ratios, timeouts) — and
keeps the value model dependency-free. Programs needing exact arithmetic on very
large integers or high-precision fractions are out of scope; revisiting this
would mean a bignum dependency, which conflicts with the zero-dependency goal.

## License

BSD-3-Clause.

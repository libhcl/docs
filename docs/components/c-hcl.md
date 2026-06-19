# c-hcl — declarative-subset parser

[`github.com/libhcl/c-hcl`](https://github.com/libhcl/c-hcl) is a small,
**dependency-free C parser** for the *declarative subset* of HCL native
syntax — bodies of attributes and (optionally labeled) blocks, whose values are
strings, numbers, booleans, `null`, or lists thereof.

It is meant for **configuration, not computation**: there is deliberately **no**
support for HCL expressions, interpolation (`${...}`), functions, heredocs, or
object-value literals. That keeps it tiny — one `.c`/`.h` of public surface, no
third-party code — and easy to embed in any C project. It was extracted from
[socket_vmnet](https://github.com/lima-vm/socket_vmnet).

## The grammar it parses

```
body   := ( attr | block )*
attr   := IDENT '=' value
block  := IDENT label* '{' body '}'
label  := STRING | IDENT
value  := STRING | NUMBER | 'true' | 'false' | 'null'
        | '[' ( value ( ',' value )* ','? )? ']'
```

Comments are `#`, `//` (to end of line) and `/* ... */`. The full lexical
tokens, EBNF, and parsing strategy live in `GRAMMAR.md` in the repository.

```hcl
# example.hcl
name     = "demo"
replicas = 3
tags     = ["web", "edge"]

service "api" "primary" {
  port = 8080
  upstream { host = "10.0.0.1" }
}
```

## The API

Parse once into an immutable AST, walk it with accessors, then free.

```c
#include "hcl.h"

char err[256];
hcl_doc *doc = hcl_parse(src, len, err, sizeof err);
if (!doc) { fprintf(stderr, "%s\n", err); return 1; }

const hcl_body *root = hcl_doc_root(doc);

const hcl_value *name = hcl_body_attr(root, "name");
puts(hcl_value_string(name));                            // "demo"

double n;
hcl_value_number(hcl_body_attr(root, "replicas"), &n);   // 3

const hcl_value *tags = hcl_body_attr(root, "tags");
for (size_t i = 0; i < hcl_value_list_count(tags); i++)
  puts(hcl_value_string(hcl_value_list_at(tags, i)));

for (size_t i = 0; i < hcl_body_block_count(root, "service"); i++) {
  const hcl_block *s = hcl_body_block_at(root, "service", i);
  const char *label  = hcl_block_label(s, 0);            // "api"
  const hcl_value *port = hcl_body_attr(hcl_block_body(s), "port");
  // ...
}

hcl_free(doc);
```

`hcl_parse` returns `NULL` on error, writing a message into `err`. The returned
document owns the entire AST; one `hcl_free` releases it.

### Value model

A `hcl_value` is one of five kinds, reported by `hcl_value_kind`:

| Kind | Accessor |
|------|----------|
| `HCL_STRING` | `hcl_value_string(v)` → `const char *` |
| `HCL_NUMBER` | `hcl_value_number(v, &out)` → writes `double`, returns `bool` |
| `HCL_BOOL` | `hcl_value_bool(v, &out)` → writes `bool`, returns `bool` |
| `HCL_NULL` | (kind only) |
| `HCL_LIST` | `hcl_value_list_count(v)` / `hcl_value_list_at(v, i)` |

Bodies are read with `hcl_body_attr` (first attribute by name),
`hcl_body_block_count` / `hcl_body_block_at` (blocks by type — pass `NULL` to
match every block); blocks with `hcl_block_type`, `hcl_block_label_count`,
`hcl_block_label`, and `hcl_block_body`. See `hcl.h` for the full surface.

## Build & test

The `Makefile` builds the static archive `libhcl.a` from `hcl.c` + `ast.c`
(`CFLAGS` defaults to `-O2 -Wall -Wextra -pedantic`):

```sh
make            # builds libhcl.a
make test       # unit tests (add SANITIZE=address on a system clang)
make cover      # llvm-cov report (override LLVM_* / use xcrun on macOS)
make example    # builds examples/demo
make install    # installs libhcl.a + hcl.h under $(PREFIX) (default /usr/local)
```

Equivalently you can just compile the sources directly: `cc -I. hcl.c ast.c
your_app.c`. No toolchain installed? With [pkgx](https://pkgx.sh): `dev` (reads
`pkgx.yaml`) or `./taskw test`.

The test suite runs under AddressSanitizer and includes **allocation
fault-injection** (`-DHCL_FAULT_INJECT`); measured coverage is **100% of
functions and ~99% of lines** — the remainder being defensive `NULL` guards.

## License

BSD-3-Clause.

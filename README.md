# SIR Compiler

A self-hosting compiler from **SIR** (Semantic Intermediate Representation) to **WASM**, built on top of **STF**.

[![Code License](https://img.shields.io/badge/Code-Apache--2.0-green)](LICENSE-CODE)
[![Spec License](https://img.shields.io/badge/Spec-CC--BY--4.0-green)](LICENSE)

## What is this

This repository implements an execution environment for **SIR** — a structural
intermediate representation of executable agent logic, defined as an applied
dictionary on top of [Semantic Text Format
(STF)](https://github.com/ais-space/stf). SIR modules are valid STF documents
that double as a direct parse tree (AST) for agents and compilers.

The compiler is written in SIR itself and builds its own source (106 functions
across `modules/`, plus the `runtime/` helpers — about 1.2MB of SIR text)
into a byte-identical binary (fixpoint b1==b0, version 0.2.2).
The runtime is built from SIR sources; suite 252/252, selfhost 2/2.

## Relation to SIR

The normative SIR dictionary evolves **outside this repository**: the canon is
the SIR specification **v0.11.1-draft** (published in the open `sir` repo under
CC-BY-4.0, in lockstep with compiler releases — paired versions).

```text
STF core (syntax, object model, trust model)
 └── SIR (applied dictionary)
      └── SIR Compiler (this repo): self-hosting SIR→WASM
```

Paired releases (SIR spec ↔ compiler, backward compatible downward —
a newer compiler accepts programs written against older SIR versions,
but refuses programs declaring a newer SIR than it supports):

| SIR spec      | Compiler |
|---------------|----------|
| v0.11.1-draft | 0.2.2    |

## Execution model

SIR defines a two-phase execution model (§3 of the specification):

- **Phase 1 — Tree-Walk Interpretation**: the runtime interprets the STF-AST
  directly for fast prototyping.
- **Phase 2 — AOT Compilation**: the structural tree is lowered to WASM
  through a target-independent IR (`GenWasm`); then the bootstrap fixpoint —
  the compiler compiles its own SIR byte-for-byte (proven: b1==b0).

## What the compiler supports

- **Values and arithmetic/logic** (Portable Core, platform-independent):
  `Core.Add`, `Core.Sub`, `Core.Mul`, `Core.Div`, `Core.Eq`, `Core.Lt`,
  `Core.Gt`, `Core.And`, `Core.Or`, `Core.Not`, `Core.Lit`, `Core.Neg`,
  `Core.Mod`. Division by zero is not an exception but an `Error` in `Result`.
- **Structural equality and introspection** (platform-independent):
  `Core.Equals` (structural: distinguishes types — `True ≠ 1`, recursive over
  lists/maps; unlike shallow `Core.Eq`) and `Core.TypeOf` (returns the value
  type name: `null`/`bool`/`int`/`float`/`str`/`list`/`map`).
- **Extra string ops**: `Core.StrFind` (substring index or `null`),
  `Core.StrCount` (occurrence count).
- **Composite data** (first-class, platform-independent): lists
  (`Core.List`, `Core.ListGet`, `Core.ListSet`, `Core.ListLen`,
  `Core.ListPush`, `Core.ListConcat`) and dicts (`Core.Map`, `Core.MapGet`,
  `Core.MapSet`, `Core.MapKeys`). Backed by native runtime linear-memory
  structures (chunked lists, hash maps).
- **String ops**: `Core.StrConcat`, `Core.StrLen`, `Core.StrSub`,
  `Core.StrIdx`, `Core.StrParse` (text → number/boolean/string),
  `Core.NumToStr`.
- **`Result` monad**: failures (e.g. division by zero) wrap into
  `SIRResult(Ok/Error)` and propagate through the computation without hidden
  exceptions; the terminal `Result` node returns the unwrapped value or `Error`.
- **Control flow**: `Branch` (condition + `TrueBlock`/`FalseBlock`) and `Loop`
  (condition + `Body`, with an iteration cap against hanging).
- **Functions and calls**: user `Function`s with named parameters;
  `Ctrl.Call` invokes a function in an **isolated Scope** (own `env`), with
  shared STF-Memory and channels; recursion is supported
  (`MAX_CALL_DEPTH = 10000`). Entry point — the `main` function or the first one.
- **Input-output**: `Sys.Input` reads a value from a channel (`channels`, set
  at launch), `Sys.Output` writes a value through the standard capability
  `Sys.Sink.Capture`. Both require matching `Capabilities`.
- **File reading**: `Sys.Storage.Read(Path)` reads a file as a string
  (requires the `Sys.Storage` capability). The SIR-on-SIR compiler uses it to
  fetch sources (the host injects the path/channel).
- **Memory**: local `TransientVar` (plain assignments via `Returns`) and
  persistent **STF-Memory** via `Mem.Write(Ref, Value)` / `Mem.Read(Ref)`
  (require `MemPersistentWrite` / `Mem.Read` capabilities).
- **Platform effects**: `Sys.*` ops whose concrete effect is declared in the
  document itself via `Adapter` nodes (`Target` + `Effect` + `Binding`). The
  backend is just a registry of standard capabilities: `Sys.Sink.Stdout`,
  `Sys.Notify.Toast`, `Sys.Web.H1`, `Sys.Sil.Render`, `Sys.Sink.Capture`.
- **Trust model**: calling an `Op` without a granted `Capabilities` →
  `TrustBoundaryViolation`. Untrusted modules (Origin: Agent/Human) cannot
  declare Capabilities and are isolated.

## Repository structure

```text
sir-compiler/
├── modules/              # the compiler in SIR (atomic domains; entry Compiler.sir)
├── runtime/              # runtime helpers in SIR (lists/maps/strings/files/logic)
├── sir_lib/              # mandatory prelude (injected by the host as Sys.Lib)
├── grammar/
│   └── stf.ebnf          # STF core grammar (reference copy, GrammarVersion 1.0.1)
├── README.md
├── .gitignore            # no binaries in git (shipped via Releases)
├── LICENSE               # CC-BY-4.0 (specification and documents)
├── LICENSE-CODE          # Apache-2.0 (code)
├── CONTRIBUTING.md
├── SECURITY.md
├── CHANGELOG.md
├── ROADMAP.md
└── CODE_OF_CONDUCT.md
```

## Usage

Requirements: [wasmtime](https://wasmtime.dev/) 49.x (verified on 49.0.2),
a POSIX shell with coreutils (`sh`, `dd`, `od`, `wc`, `sed`, `cmp`) and
`wasm-validate` (optional, from [wabt](https://github.com/WebAssembly/wabt)).
No Python, no Node — shell plus binaries only. A release binary is
self-sufficient otherwise.

Take six files from a GitHub release (plus `SHA256SUMS` to verify):
`sir_compiler.wasm`, `runtime.wasm`, `compiler.sir.expanded` (the compiler
source with includes expanded), `sir_tool.wasm` (prebuilt formatter/editor),
`sir_tool.flat.sir` (its expanded source, for verifiers), `SHA256SUMS`.
Put them in one directory — the examples below call it `work/`:

```bash
mkdir work && cp sir_compiler.wasm runtime.wasm sir_tool.wasm work/ && cd work/
```

wasmtime 49 appends the `_start` result digits to stdout, so every build
below ends with the same structural truncation: parse the WASM section
headers and cut at the structural end (robust for any output — unlike
digit-stripping, it can never eat binary content). Save once as
`unwashtail.sh`:

```bash
cat > unwashtail.sh <<'EOF'
#!/bin/sh
# IN.RAW OUT.WASM — cut wasmtime 49 _start result digits (coreutils only).
f=$1; o=$2
magic=$(dd if="$f" bs=4 count=1 2>/dev/null | od -An -tx1 | tr -d ' \n')
[ "$magic" = "0061736d" ] || { echo "unwashtail: not a wasm file" >&2; exit 1; }
i=8
size=$(wc -c < "$f")
while [ "$i" -lt "$size" ]; do
  sid=$(dd if="$f" bs=1 skip="$i" count=1 2>/dev/null | od -An -tu1 | tr -d ' \n')
  [ -z "$sid" ] && break
  if [ "$sid" -gt 19 ]; then break; fi
  i=$((i + 1))
  n=0; m=0
  while :; do
    b=$(dd if="$f" bs=1 skip="$i" count=1 2>/dev/null | od -An -tu1 | tr -d ' \n')
    i=$((i + 1))
    n=$((n + ((b & 127) << m))); m=$((m + 7))
    if [ "$((b & 128))" -eq 0 ]; then break; fi
  done
  i=$((i + n))
done
dd if="$f" bs=1 count="$i" of="$o" 2>/dev/null
EOF
chmod +x unwashtail.sh
```

Compiling your own program (`program.sir` must be a POSIX text file — trailing
newline — and carry a `Context` header with its SIR `DocumentVersion`, §2.5
of the specification). The guest reads a single stdin stream: the literal
line `wasi` selects compile mode, the rest is the source:

```bash
{ printf 'wasi\n'; cat program.sir; } | wasmtime run --dir . sir_compiler.wasm > program.raw
./unwashtail.sh program.raw program.wasm
wasm-validate program.wasm
```

The `--dir .` directory exposes `runtime.wasm` from the release to the guest.

Self-compilation check (fixpoint b1==b0) — with the same files in `work/`:

```bash
{ printf 'wasi\n'; cat compiler.sir.expanded; } | wasmtime run --dir . sir_compiler.wasm > self.raw
./unwashtail.sh self.raw self.wasm
cmp sir_compiler.wasm self.wasm && echo SELF-FIXPOINT-OK
wasm-validate self.wasm
```

The output is byte-identical to the binary itself; `wasm-validate` additionally
confirms validity.

## Preparing multi-file projects (SirTool)

The compiler accepts a single flat document; `Include` lines are resolved
before compilation by SirTool — one program with stdin-prefix modes (`fmt`
formatter, `expand` gluer, `insert`/`replace`/`delete`/`move`/`wrap`
structural edits, `diff` preview). Its source lives in the open `sir`
repository
([`examples/sir_tool.sir`](https://github.com/ais-space/sir/blob/main/examples/sir_tool.sir));
the prebuilt `sir_tool.wasm` from the release above is byte-identical to
building `sir_tool.flat.sir` with the release compiler (verify: same `wasi`
command as for any program, then `cmp`).

Expanding a project entry point is one call — the frame is three header
lines (`BASE` directory for file reads, empty path line, entry line count
as POSIX lines, i.e. `wc -l`) followed by the entry text. `BASE` is resolved
against the preopened directory, so with `--dir .` (the `work/` root) the
base is the includes subdirectory. All tool text outputs end with a newline
(POSIX text), so the wasmtime tail is always a separate trailing digit line —
drop it:

```bash
{ printf 'expand\nmods\n\n%s\n' "$(wc -l < entry.sir)"; cat entry.sir; } | wasmtime run --dir . sir_tool.wasm | sed -e '${/^[0-9][0-9]*$/d;}' > project.flat.sir
```

(`entry.sir` must end with a newline so `wc -l` counts every line;
nested `..` segments are normalized; a missing include traps loudly
instead of returning text.) The same trailing-digit drop applies to every
tool text mode (`fmt`, `diff`, structural edits); binary outputs always go
through `unwashtail.sh` above. The output compiles directly:

```bash
{ printf 'wasi\n'; cat project.flat.sir; } | wasmtime run --dir . sir_compiler.wasm > project.raw
./unwashtail.sh project.raw project.wasm
```

Full chain, end to end: `sir` repo example → `sir_tool.wasm` expand →
`sir_compiler.wasm` build → `unwashtail.sh` → `wasm-validate`.

## Reproducing a release

Sources are the whole truth — no binaries live in git (see `.gitignore`).
A release is bootstrapped from the previous one: the seed compiler (previous
release binary) compiles the current `compiler.sir.expanded` (all `Include`
modules flattened, version header on top) into the new binary, which must
then reproduce itself byte-identically (self-fixpoint gate above). The
release files — `sir_compiler.wasm`, `runtime.wasm`,
`compiler.sir.expanded`, `sir_tool.wasm`, `sir_tool.flat.sir`,
`SHA256SUMS` — are published to GitHub Releases,
never committed to the repository.

## Status

Self-hosting SIR→WASM compiler, version 0.2.2: builds its own source
(1.2MB, 106 functions) into a byte-identical binary (fixpoint b1==b0,
wasm-validate clean, determinism by repeat); suite 252/252, selfhost 2/2.
Version stamp — in code (`CompilerVersion`) and in the binary data section. No open debt.

## License

- **Code** (SIR modules, runtime, `sir_lib/`): [Apache License 2.0](LICENSE-CODE).
- **Specification and documents** (`.stf`): [Creative Commons Attribution 4.0 International](LICENSE) (CC-BY-4.0).

## Contact

- **Security:** [support@ais-platform.dev](mailto:support@ais-platform.dev)
- **Issues:** GitHub Issues

# Changelog

All notable changes to the SIR Compiler repository.

---

## v0.2.2 (2026-10-05) — First public release

### Fixed
- **N-ary `Core.StrConcat` truncation (TD-038)**: the WASM helper is binary while the host joins all parts — both emitters now fold left (`concat(concat(a,b),c),…`); refusal diagnostics and multi-part lines render fully on both engines. Tooth: `test_strconcat_variadic.py` 4/4.
- **`IR.Sort` / `IR.StrParse` emission gap**: the Python twin lowered them, the SIR emitter poisoned the call (`999999`); both handlers added to `EmitStringInstr` after `StrJoinParts`. Tooth: `test_sort_strparse_parity.py` 2/2.

### Added
- **SirTool showcase** (open `sir` repo, `examples/sir_tool.sir`): one program, eight stdin-prefix modes — `fmt`, `expand`, `insert`, `replace`, `delete`, `move`, `wrap`, `diff`; host/WASM parity proven per mode; `expand` output is byte-identical to the reference expander.
- **README as executable instruction**: working `wasi`-prefix commands, structural wasmtime-tail truncation (`unwashtail.sh`, POSIX coreutils, no Python), wasmtime 49.x pin, end-to-end SirTool chain.

Suite 252/252, self-fixpoint proven byte-identical.

## v0.2.1 (2026-10-03) — SIR version gate (spec §2.5)

### Added
- **Version gate**: new `SirGate` module (`SirSupported` contract: SIR up to 0.11.1, compiler 0.2.1). `Compiler.sir` entry gates every source: covered versions pass through byte-identical; newer, malformed, or missing `Context` declarations print a loud diagnostic and yield a self-describing refusal stub instead of a binary. Backward compatibility promised downward. New `version` stdin prefix prints the compiler stamp without compiling.
- **`Lower` header tolerance**: the dispatcher descends into the first `Module` child, so `Context`-headed documents compile to byte-identical output as bare ones.
- **Version headers**: every `modules/*/*.sir` and `runtime/*.sir` source carries a `Context` declaration; `Lower/hostonly` contract data (`erased_ops.txt`, `hostonly_loud_accepted.txt`) documented in `SIR_Compiler_Architecture.stf`.

### Changed
- Open-source hygiene: `LICENSE` verbatim, binaries out of git (shipped via GitHub Releases), `CONTRIBUTING`/`SECURITY`/`ROADMAP` rewritten against shipped content only.

## v0.2.0 (2026-10-02) — Open-source prep: Apache-2.0, atomized modules, version stamp

### Changed
- **License**: code under Apache-2.0 (`LICENSE-CODE`), spec/docs under CC-BY-4.0 (`LICENSE`); was proprietary.
- **Atomization**: `extra.sir` dissolved, `string.sir` ×4, `EmitMemInstr` ×5, `JoinFallback`/`list_mut`/`map_mut` split out; `fcode` use-before-def fixed with default.
- **Version stamp**: `CompilerVersion` module returns the version string, embedded in the binary data section.
- **Layout**: `pre_repo/` holds only the publishable product (museum, scripts, tests moved to workbench root).

### Fixed
- `sir_map_keys` flat odata → chunked build (wild reads, TD-037); `sir_log` iovs_len 2→1 (TD-036); ReadFile-literal `leb_u`→`leb_s` pair.
- Self-fixpoint b1==b0 byte-identical, suite 246/246, selfhost 2/2.

## v0.9.0 (2026-08-26) — M2 Phase 5 Stages C–D: SIR runtime, error domain, storage, persistent collections

### Added
- **Foundation 2**: `runtime.wasm` built entirely from SIR sources (`runtime/*.sir`) by the self-hosting compiler; legacy WAT blob retired. Function order = `HELPER_NAMES` (46 helpers, indices 4..49 with N_IMPORTS=4; user functions start at 50).
- **Stage C — collection codegen**: `EmitListInstr`/`EmitMapInstr` (guard-Branch emission, constructor threading of `dst`); interpreter alignment: `Div` is Int-closure (trunc, §6.3), Bool renders as `1`/`0`.
- **Stage D — Result monada & storage**: `sir_const_err`, ERROR-tagged results from `sir_div`/`sir_mod` ("division by zero"), structural propagation via a `HasErrorOps` prescan → err-block wrapper + tag checks; `Sys.Storage.Read` lowered to `IR.ReadFile` → `sir_read_file` (#43, WASI `path_open`, read-loop until EOF). Import contract extended 2 → 4 WASI imports (`path_open`, `fd_close`).
- **Persistent containers (variant B, spec §6.7)**: `list_set`/`list_push`/`map_set` rewritten functionally (clone + append/upsert); in-place mechanics isolated in tail helpers `sir_list_append` (#45) and `sir_map_set_inplace` (#46). Aliasing eliminated.
- **Extended-core comparisons**: `Core.Le/Ge/Ne` now compile — runtime helpers `sir_le`/`sir_ge`/`sir_ne` (#47–49) appended without shifting existing indices; `BINOPS`/`helperIdx`/`binMap` synchronized in both backends.
- `examples/full_subset.stf` — D5 matrix gate: byte parity of three build paths AND Node-WASI execution matching direct execution (`2917matrix!849730315120big`).

### Fixed
- Emission crash on unresolved callees: debug `Ctrl.Call print_i64` in `runtime/map.sir` resolved via `ufi`→`hidx` to null and crashed `EmitU32LEB`.
- `sir_list_set` structure: tail binding had been glued onto the Loop node as properties, leaving `out` unbound.
- Persistent clones read length from the box TAG instead of the struct and passed a raw struct pointer where `sir_list_get` expects a boxed list (OOB traps / silent corruption).
- HasErrorOps classifier missed binary `IR.Div`/`IR.Mod` (only direct helper calls and `IR.ReadFile` were detected), so user-level division-by-zero skipped the error path and printed subsequent output; per-call tag check extracted into `_err_check` (reference) and added to the binops branch of `EmitInstr` (SIR mirror).
- Matrix test expectation corrected: `fact(5)=120`, `Lt(120,100)` is false → branch prints `big`.

### Changed
- Runtime smoke probes rewritten for persistent semantics (8 checks incl. original-container immutability guards).
- Suite: 140 passed / 0 xfailed.

---

## v0.8.0 (2026-08-23, draft) — M2 Phase 5 Stage B: strings in WASM

### Added
- Stage B of the WASM backend: string literals via the `data` section (aligned 4-byte offsets in the 0..1024 region), `IR.Const(str)` → `sir_const_str`, `IR.Output` → `sir_print_val`, and `IR.Input` → `sir_read_str` (string read from stdin). String helpers (`sir_str_len`, `sir_str_concat`, `sir_str_sub`, `sir_str_idx`, `sir_str_find`, `sir_str_count`, `sir_str_eq`, `sir_num_to_str`, `sir_str_parse`) carried over from the runtime blob.
- `Core.Sort` primitive added to the seed execution path, used by `EmitFuncBody`/`BuildVarMap` to assign deterministic `var → local` indices.
- Byte-identical `GenWasm` output for string programs (constants, output, input).
- End-to-end Node WASI execution gate (output `Hello, World!13<input>ZZZ`).

### Changed
- `CollectStr.sir` uses `Core.IntDiv` for aligned offset computation (was `Core.Div`, which returned `float` and broke `EmitSignedLEB`/`Core.BitAnd`).
- Generated module memory minimum raised 1 → 4 pages so `sir_read_str`'s 64 KiB read buffer fits (previously `memory access out of bounds`).
- `SIR_Compiler_Workbench_State.stf` added (Phase 5 Stage B state); `CollectStr`/`StrBytes` registered in `modules/registry.stf`.

---

## v0.7.0 (2026-08-23) — M2 Phase 5: WASM backend spec extension

### Added
- `SIR_Specification.stf` bumped to v0.7.0-draft: new §6.10.4 low-level bitwise & identity primitives — `Core.BitAnd`, `Core.BitOr`, `Core.BitXor`, `Core.ShiftL`, `Core.ShiftR` (bitwise on Int) and `Core.Id` (identity binding); §2.1 `Op` property examples extended with the new namespaces. These back the WASM backend (Phase 5) code generation.

### Changed
- WASM backend (`GenWasm`) integrated into `Compiler.sir` (mode `"wasm"`: reads runtime via `Sys.Input rt`, then `GenWasm(ir, rt)`); bytecode-identical to the reference backend. Module docs normalized to `XxxModuleDoc` format; `SIR_Compiler_Architecture.stf` added (compiler architecture overview).

---

## v0.6.0 (2026-08-22, WIP) — M2: self-hosting compiler — language extension + interpreter enablers

### Added
- **M2 language-extension primitives (Core Ops)**: `Core.Equals` (type-aware structural equality; distinct from shallow `Core.Eq`), `Core.TypeOf` (returns `"null"/"bool"/"int"/"float"/"str"/"list"/"map"`), `Core.StrFind` (index of first substring occurrence, `-1` if absent, optional `start`), `Core.StrCount` (number of non-overlapping occurrences). Prelude `core_fn.stf` exposes them as callable Functions `Equals`/`TypeOf`/`StrFind`/`StrCount` for higher-order use.
- **Interpreter: `Sys.Storage.Read(path)`** — reads a file's text, gated by the `Sys.Storage` Capability (a `TrustBoundaryViolation` otherwise). Already documented as the `Sys.Storage` abstract interface in §7.2. This lets the self-hosting compiler read its input source from disk.
- **Interpreter: source-line context in errors** — `SIRRuntimeError`/`TrustBoundaryViolation` messages now include the originating source line (e.g. `unsupported Core Op: Core.NoSuchOp (at line 12)`), aiding compiler-authoring and diagnostics.

### Changed
- `SIR_Specification.stf` bumped to v0.6.0-draft: new §6.10 M2 language-extension primitives; `core_fn.stf` inventory extended.
- Language-extension gates (12 new: Equals/TypeOf/StrFind/StrCount, Sys.Storage.Read, line-aware errors); suite total 92 passing.

### Planned (not yet implemented)
- SIR-on-SIR compiler front-end (parser → AST), target-independent IR — DONE (Phase 3–4). WASM backend (Phase 5) — DONE (bytecode-identical to the reference backend). Bootstrap fixed point (Phase 6), native backend (Phase 7) — continued in later releases.

---

## v0.5.0 (2026-08-22) — M1.6: Standard Library (prelude) + no sugar

### Added
- **Standard Library (prelude)**: a set of SIR modules (`sir_lib/*.stf`) merged into the callable namespace on `run()`; program Functions override prelude of the same name; a prelude module MUST NOT declare `main`. Deliberately **no syntactic sugar** — higher-level vocabulary is ordinary SIR Functions over the Core Op primitives.
- **Function inventory**: `collections` (Map/Filter/Fold/ForEach/Reverse/Zip/Range/Take/Drop/Head/Tail/Last/Nth/Init/Contains/Count/IndexOf/Any/All/Find/Concat/Flatten/Distinct/Enumerate/Partition/GroupBy/FlatMap/Sort), `strings` (SplitLines/JoinLines/Words/Slice/Capitalize/PadStart/RepeatStr/Truncate/StartsWith/EndsWith/ContainsStr/ReplaceStr), `math` (Gcd/Lcm/Factorial/IsPrime/Even/Odd/DivMod/Mean/Median), `functional` (Identity/Constant/Apply/Compose/Pipe), `result` (Ok/Err/IsOk/IsErr/Unwrap/MapResult/AndThen/OrElse), `dictutil` (Keys/Values/Items/FromPairs/Merge/GetOr/HasKey/MapValues/FilterKeys/Invert), `core_fn` (Add/Sub/Mul/Div/Mod/Eq/Lt/Gt/Le/Ge/Ne/And/Or/Max2/Min2 — expose Core Ops as callable Functions for higher-order use), `io` (ReadLine/Print/PrintLn/WriteLines)
- **Extended Core Op set**: numeric (`Core.Le/Ge/Ne/IntDiv/Floor/Ceil/Round/Abs/Sign/Min/Max/Pow/Sqrt/Clamp/Sum/Product/Range`), string (`Core.StrSplit/StrJoin/StrUpper/StrLower/StrTrim/StrContains/StrStartsWith/StrEndsWith/StrReplace`), char (`Core.Ord/Chr`), type predicates (`Core.IsNull/IsList/IsMap/IsStr/IsNum/IsBool`), booleans (`Core.True/Core.False`)
- **Bug fixes in interpreter**: `Ctrl.Call` now resolves a function Name that arrives via a variable (higher-order passing); `_exec_input` resolves its channel parameter; a quoted literal token is a string (`"7"` → `"7"`, not int)
- Interpreter bug fixed: library Functions that call other library Functions now use `Ctrl.Call` (cross-Function calls may not use `Op: <Name>` directly)
- Standard-library gates exercise the whole library (52 new gates; 80 total)

### Changed
- `SIR_Specification.stf` bumped to v0.5.0-draft: new §6.9 Standard Library (prelude mechanism, Ctrl.Call constraint, extended Core Ops, library inventory)
- `SIR_Interpreter_Concept.stf` extended with M1.6 section
- `ROADMAP.md` / `DEVELOPMENT_CHECKLIST.stf` mark M1.6 Done

---

## v0.2.0 (2026-08-21) — Universal runtime design & pluggable backend

### Added
- `SIR_Interpreter_Concept.stf` — runtime concept: two-level model (Portable Core vs Platform Bindings), value/memory model, abstract capability interfaces, Target declaration, adapters, SIL/HTML relationship
- `SIR_Specification.stf` extended to v0.2.0-draft: concrete value model, Core Ops semantics (arithmetic/comparison/literal), TransientVar assignment, and **Targets & Platform Bindings** (Target declaration, `Sys.*` capability interfaces, backend binding, structural Adapters)
- Pluggable backend abstraction with Stdout (linux/desktop), Html (browser-html), Sil (browser-sil), and Toast (android/ios mock) backends — proves one SIR source runs differently per target
- Interpreter extended: value resolution, `Core.*` Ops, `Sys.*` delegation to backend, and TrustBoundaryViolation when a capability is not granted
- Example `examples/hello.stf` (Target across 6 platforms, `Sys.Display`) plus core/target gates (8 passing)

---

## v0.3.0 (2026-08-21) — M1: broader execution core

### Added
- **Result monad**: fallible Ops (`Core.Div`) yield a `Result` (Ok/Error); errors propagate through subsequent Instructions without exceptions. Final `Result` node returns the unwrapped value or the Error `Result`
- **Control flow**: `Branch` (Condition + `TrueBlock`/`FalseBlock`) and `Loop` (Condition + `Body`, iteration-bounded) executed by a `Frame`-based block executor
- **Memory Ops**: `Mem.Write(Ref, Value)` and `Mem.Read(Ref)` addressing the STF-Memory store, gated by `MemPersistentWrite` / `Mem.Read` Capabilities (TrustBoundaryViolation otherwise)
- **New Core Ops**: `Core.Neg` (unary), `Core.Mod` (modulo)
- Result/control-flow/memory gates (17 passing total)

### Changed
- `SIR_Specification.stf` bumped to v0.3.0-draft: §3 monadic error propagation, §5.1 concrete Branch/Loop structure, §6.3 `Core.Neg`/`Core.Mod`, new §6.5 Result Monad and §6.6 Memory Ops
- Interpreter refactored to a `Frame` abstraction shared across function body and nested blocks

---

## v0.4.0 (2026-08-22) — M1.5: expressiveness for self-hosting

### Added
- **Composite data**: first-class List (`Core.List`, `Core.ListGet`, `Core.ListSet`, `Core.ListLen`, `Core.ListPush`, `Core.ListConcat`) and Map (`Core.Map`, `Core.MapGet`, `Core.MapSet`, `Core.MapKeys`) Ops, represented as native runtime structures
- **String Ops**: `Core.StrConcat`, `Core.StrLen`, `Core.StrSub`, `Core.StrIdx`, `Core.StrParse` (text → number/boolean/string), `Core.NumToStr`
- **Functions & Calls**: user-defined `Function` with named `Params`; `Ctrl.Call` invokes a function in an **isolated Scope** (own `env`, shared STF-Memory and channels), with recursion support (`MAX_CALL_DEPTH = 1000`); entry point is `main` or the first function
- **I/O Ops**: `Sys.Input` (reads from `channels`, gated by `Sys.Input` Capability) and `Sys.Output` (emits via standard capability `Sys.Sink.Capture`, gated by `Sys.Output` Capability)
- Data/functions/IO gates (28 passing total)

### Changed
- `SIR_Specification.stf` bumped to v0.4.0-draft: §6.1 List/Map, §6.7 String & Composite Ops, §6.8 Functions & Calls, §7.2.1 I/O
- `SIR_Interpreter_Concept.stf` extended with M1.5 section (expressiveness for self-hosting)
- `Backend` gained the `Sys.Sink.Capture` standard capability

---

## v0.1.0 (2026-08-21) — M0 interpreter seed

### Added
- Seed tree-walk execution path implementing the SIR Phase 1 execution model
- Minimal STF parser producing an AST from STF documents
- Working fork of the SIR specification (`SIR_Specification.stf` v0.1.2-draft, `SIR_Concept.stf` v0.1.1-draft) and STF core grammar (`grammar/stf.ebnf`, GrammarVersion 1.0.1)
- Example `examples/add.stf` (Op: `Core.Add`)

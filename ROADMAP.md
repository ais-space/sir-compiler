# Roadmap

## Status: v0.2.2 — first public release, self-sufficient

Self-hosting SIR→WASM compiler: builds its own source into a byte-identical
binary (fixpoint proven), suite 253 green, SirTool showcase (formatter and
structural editor, 8 modes), fully executable usage instructions.

## Next, in order

1. **Native backend as research** (x86-64 / AArch64 over the same IR) — no timeline promised.
2. **Memory Engine** — full STF-Memory semantics (§3–§7); today only the heap form and file reads are implemented.
3. **Structural ABI contracts** (ABIMap) for cross-level lowering.
4. **SirTool evolution** (shell test driver, test runner).

## Continuous

- Sync stabilized specification changes back to `sir_workbench`
- Security reviews

## Releases

- [x] v0.2.2 first public release
- [ ] Reproducible GitHub Releases (`sir_compiler.wasm`, `runtime.wasm`, `compiler.sir.expanded`, `sir_tool.wasm`, `sir_tool.flat.sir`, `SHA256SUMS`)

<details>
<summary>History (M0–M2, all done)</summary>

## M0 (done) — Seed execution
- [x] Minimal STF parser (AST from STF)
- [x] Seed execution path running `examples/` programs (Op: Core.Add and beyond)
- [x] Working fork of SIR specification + STF grammar
- [x] Universal Portable Core + pluggable backends (Stdout/Html/Sil/Toast); one source runs differently per target

## M1 — Broaden execution
- [x] Core Op vocabulary (arithmetic, comparison, memory read/write)
- [x] Transient and persistent memory (TransientVar, STF-Memory via Mem.Read/Mem.Write)
- [x] Monadic `Result` error handling (State: Ok / Error)
- [x] Branch / Loop control flow
- [x] Trust boundary enforcement (Origin → Capabilities)

## M1.5 — Extend expressiveness for self-hosting
- [x] Composite value types: List (nested → trees), Map
- [x] String Ops (StrConcat/StrLen/StrSub/StrIdx/StrParse/NumToStr)
- [x] List/Map Ops (build, get, set, len, push, concat, keys)
- [x] User-defined Functions + Ctrl.Call (recursion, isolated local scope, shared STF-Memory)
- [x] Sys.Input / Sys.Output (Capabilities-gated) for reading source / emitting result
- [x] SIR_Specification.stf → v0.4.0-draft (§6.1 List/Map, §6.7 String & Composite Ops, §6.8 Functions & Calls, §7.2.1 I/O)

## M1.6 — Standard library (prelude) & no sugar
- [x] Callable prelude (sir_lib/*.stf) merged into namespace on run(); program Functions override prelude; prelude MUST NOT declare main
- [x] No syntactic sugar — higher-level vocabulary is ordinary SIR Functions over Core Op primitives
- [x] Extended Core Op set (numeric/string/char/type-predicate/boolean) in the interpreter
- [x] Library inventory: collections, strings, math, functional, result, dictutil, core_fn, io
- [x] Ctrl.Call resolves function Names only; library cross-Function calls use Ctrl.Call
- [x] Call-resolution fixes: higher-order Ctrl.Call, input channel resolution, quoted literals stay strings
- [x] SIR_Specification.stf → v0.5.0-draft (§6.9 Standard Library); standard-library gates (52 new, 80 total)

## M2 — Self-hosting (SIR compiler written in SIR → binary)
- [x] Language minimum: Core.Equals (structural equality), Core.TypeOf, Core.StrFind, Core.StrCount — spec §6.10
- [x] Seed minimum: Sys.Storage.Read (file read, Sys.Storage capability) plus errors with line context
- [x] SIR front-end: indent-tokenizer plus parser → AST (nested List/Map) — `modules/Compiler/Compiler.sir`
- [x] Target-independent IR plus deterministic AST→IR lowering (self-check for target-neutrality)
- [x] WASM backend: target-independent codegen `GenWasm` in SIR, deterministic output (byte-identical across runs)
- [x] Runtime from SIR sources (Foundation 2): `runtime/*.sir` helpers compiled by the compiler itself; helper order is contractual (append-only, indices stable)
- [x] Bootstrap fixpoint (WASM): the compiler compiles its own source into a byte-identical binary
- [x] Version gate (spec §2.5): compilable documents declare SIR via a `Context` header; the compiler refuses newer-than-supported SIR loudly and accepts older versions (backward compatibility promised)
- [x] v0.2.2 first public release: N-ary StrConcat fold, Sort/StrParse emission, two-pass file reads, SirTool showcase (8 modes, host/WASM parity), executable README (wasi prefix, POSIX tail strip, end-to-end chain); suite 253, self-fixpoint proven

</details>

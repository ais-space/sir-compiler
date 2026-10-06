# Contributing to SIR Compiler

Thank you for your interest in the self-hosting SIR→WASM compiler.
Everything executable in this repository is SIR: modules under `modules/`,
runtime helpers under `runtime/`, the mandatory prelude under `sir_lib/`.

## Ways to Contribute

### 1. Specification Feedback
The SIR specification is developed alongside this repository (the canonical
copy is published in the open `sir` repo). Open an issue or PR there for
ambiguities, missing Ops, or value-model questions; compiler-driven
requirements flow from this repository's issues into the specification.

### 2. Compiler & Runtime (SIR only)
- New lowering lives in `modules/Lower*` (pure SIR over Core Ops).
- New emission lives in `modules/Emit*` plus a runtime helper in `runtime/`
  when the target needs one. Helper order is contractual: appending without
  shifting existing indices keeps all released binaries compatible.
- New modules follow the atomic layout: one folder `modules/<Name>/` with
  `<Name>.sir` and `doc.stf`, registered in `modules/registry.stf`.
- Every compilable document carries a `Context` header with `DocumentVersion`
  (spec §2.5): the compiler refuses sources declaring a newer SIR than it
  supports, and accepts older ones (backward compatibility is promised).

### 3. Examples
- Add small runnable programs under `examples/` as `.sir` files with a
  `Context` version header. Each example must compile through the full
  pipeline and execute with identical output on every run.

## Development

You need only a released compiler binary plus `wasmtime`:

```bash
wasmtime run --dir . sir_compiler.wasm < program.sir > program.wasm
```

See `README.md` (Usage and Reproducing a release) for the full loop,
including the self-compilation check.

## Pull Request Process

1. Fork the repository.
2. Create a branch for your change.
3. Keep STF documents well-formed under `grammar/stf.ebnf`.
4. Keep `modules/registry.stf` complete: every file by its `/`-rooted path.
5. Submit a PR with a clear description.

## Specification Development

The canonical SIR vocabulary is the self-hosting STF document
`SIR_Specification.stf`, published in the open `sir` repo. When the
compiler needs new Ops or semantics, they land in the spec first
(spec version bump, paired compiler release) and are adopted here.
Paired versions are listed in `README.md`.

# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in the SIR compiler, runtime, or
reference grammar in this repository, please report it privately.

**Contact:** support@ais-platform.dev

**Response time:** We aim to acknowledge reports within 48 hours and provide
an initial assessment within 5 business days.

## Scope

This security policy covers:

- The self-hosting SIR→WASM compiler (`modules/`, entry `Compiler.sir`)
- The SIR runtime helpers (`runtime/`) and the mandatory prelude (`sir_lib/`)
- The STF grammar reference (`grammar/stf.ebnf`)
- The SIR specification (published in the open `sir` repo)
  where ambiguities could lead to insecure implementations

## Out of Scope

- Vulnerabilities in third-party STF/SIR implementations
- Semantic prompt injection through `UserContent` (documented limitation, not
  a vulnerability) — see STF core §4.3

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.2.x   | :white_check_mark: |
| < 0.2.0 | :x:                |

## Disclosure Policy

We follow responsible disclosure:
1. Reporter submits vulnerability privately
2. We acknowledge within 48 hours
3. We assess and develop a fix
4. We notify the reporter when the fix is ready
5. We publish the fix and credit the reporter (unless anonymity is requested)

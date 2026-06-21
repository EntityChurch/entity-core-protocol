# Entity Core Protocol

**Version**: 0.8.0
**Status**: Active

**The irreducible protocol every peer must speak.** This is the mandatory convergence layer of
the Entity system: the wire format, the type system, the identity / capability / dispatch core,
and the core tree operations. Two peers that agree on this interoperate at the substrate —
everything above it (extensions, SDK, applications) is optional and lives in the
`entity-system-architecture` repository.

> **Convergence floor:** this surface (`--profile core`) is independently re-derived by a
> cohort of generated peers at **0-FAIL**, plus the Go/Rust/Python reference implementations. The core
> is small, frozen, and proven across languages.

## What's here

| spec | role |
|---|---|
| `ENTITY-CORE-PROTOCOL` | the protocol — Layers 0–4; the universal-address-space / tree model |
| `ENTITY-CBOR-ENCODING` | the Entity Canonical Form (ECF) wire contract + conformance corpus |
| `ENTITY-NATIVE-TYPE-SYSTEM` | the core type system and native types |
| `ENTITY-CORE-MACHINE-SPEC` | condensed implementation reference |
| `test-vectors/` | ECF + crypto-agility conformance corpora |

**Start with** `ENTITY-CORE-PROTOCOL`. The five load-bearing invariants implementers keep
re-deriving are collected in the model primer (ships alongside).

## Roadmap

See **`ROADMAP-CORE-PROTOCOL.md`** for maturity (the M0–M6 ladder), the version line, and forward
work (V8 cutover, the hardening program). The cross-domain release-surface map lives in
`STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md`.

## Conformance

A peer is core-conformant if it passes `--profile core` (the protocol floor). The conformance
methodology, profiles, and vector index are in `GUIDE-CONFORMANCE` (shipped in
`entity-system-architecture`, cross-referenced here).

## Versioning & license

Protocol line **V7 → V8** (Genesis `v0.8.0`). Specs are normative; the version header in each
spec is authoritative for that document. **Dual-license:** prose CC-BY-ND-4.0; schemas / IDL /
test-vectors Apache-2.0.

---

## Supporting the project

This project is developed in the open. If it's useful to you, the best support is
to use it, report issues, and contribute back — see
[CONTRIBUTING.md](CONTRIBUTING.md).

To support the work directly, see the project's funding page.

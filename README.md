# Entity Core Protocol

**Status**: Active

> **Version:** each specification in `specs/` carries its own `**Version**:` header, and that
> header is authoritative for that document. This file deliberately does not restate one — the
> restatement it used to carry was two releases stale. `CHANGELOG.md` is the record of what
> changed and whether it can break an existing implementation.

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
| `test-vectors/` | ECF + crypto-agility conformance corpora |

**Start with** `ENTITY-CORE-PROTOCOL` **§1 Foundations** — the entity, the content hash, the
URI/path model — then **§3 Protocol Type Definitions**, whose §3.3 carries the status codes and
the `code` strings a peer must emit. Those three specs plus the corpus are the entire contract.

**There is no condensed or "implementation" edition, deliberately.** A former
`ENTITY-CORE-MACHINE-SPEC` was retired because a hand-maintained restatement of a specification
drifts against it silently, and this one had. Its two cited sections live in the real specs: its
**§1.8 Entity Fidelity** is now `ENTITY-CBOR-ENCODING` **§5.4** (the canonical home), and its
**§6.4 Connection Error Codes** was always `ENTITY-CORE-PROTOCOL` **§4.7**.

## Roadmap

See **`ROADMAP-CORE-PROTOCOL.md`** for maturity (the M0–M6 ladder), the version line, and forward
work (V8 cutover, the hardening program).

## Conformance

A peer is core-conformant if it passes `--profile core` (the protocol floor). The conformance
methodology, profiles, and vector index are in `GUIDE-CONFORMANCE` (shipped in
`entity-system-architecture`, cross-referenced here).

## Versioning & license

Protocol line **V7 → V8** (Genesis tag `v0.8.0`).

**Three documents, three independent version lines.** A document's `**Version**:` header is a claim
about *that document's* content, and it moves when a conformant implementation of the previous text
could be non-conformant under the new one. `ENTITY-CORE-PROTOCOL` carries the repository's release
number; `ENTITY-CBOR-ENCODING` (`1.x`) and `ENTITY-NATIVE-TYPE-SYSTEM` (`4.x`) are on their own
ladders and have never tracked it. **Read each header rather than inferring one from another.**

**The public surface** is the normative text of those three specifications and the conformance
corpora in `specs/test-vectors/`, which are identified by **corpus name and artifact sha256, never
by filename**. Informative notes, worked examples, the `.diag` sources, this file and the roadmap
are outside it. `CHANGELOG.md` says what moved and whether it can break an existing implementation.

**Dual-license:** prose CC-BY-ND-4.0; schemas / IDL / test-vectors Apache-2.0.

---

## Supporting the project

This project is developed in the open. If it's useful to you, the best support is
to use it, report issues, and contribute back — see
[CONTRIBUTING.md](CONTRIBUTING.md).

To support the work directly, see the project's funding page.

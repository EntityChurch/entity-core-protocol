# entity-core-protocol — AGENTS.md

Read **AGENTS-STANDARD.md** first. This file adds entity-core-protocol specifics.

## Overview

This repo is the **Entity Core Protocol** — the protocol *specification*, not an
implementation. It is the **mandatory convergence layer** (`--profile core`): the ECF
wire format, the entity-native type system, the identity / capability / dispatch core,
and the core tree operations (`get`/`put`). Two peers that agree on this surface
interoperate at the substrate; everything above it (the 22 other extensions, the SDK,
applications, guides) is optional and lives in `entity-system-architecture` — see
`ROADMAP-CORE-PROTOCOL.md` §"What is NOT core".

The spec text is **normative and upstream**: implementations (`entity-core-{go,rust,py}`,
the keystone-generated peers) implement it; they do not define it. Per AGENTS-STANDARD
§"Working across the polyrepo", the spec is upstream — **don't invent wire formats,
primitives, opcodes, or type/handler semantics here that belong to an impl, and don't
fold impl detail into the spec.** This repo *defines*; the impl repos *implement*.

## How the spec is structured

The normative texts live in `specs/`. Start with `ENTITY-CORE-PROTOCOL.md`.

| `specs/` file | role |
|---|---|
| `ENTITY-CORE-PROTOCOL.md` | the protocol — Layers 0–4; the universal-address-space / tree model |
| `ENTITY-CBOR-ENCODING.md` | the Entity Canonical Form (ECF) wire contract + conformance corpus |
| `ENTITY-NATIVE-TYPE-SYSTEM.md` | the core type system and native types |
| `test-vectors/` | ECF + crypto-agility conformance corpora (`.diag` source, `.cbor` canonical) |

**The two authoring standards are NOT in this repo.** `SPECIFICATION-FORMAT` (how a normative spec
document is written) and `STYLE-NAMING-CONVENTIONS` (identifier naming — these names are part of the
wire contract) are **single-homed in `entity-system-architecture/specs/`** as of 2026-08-31. Read them
there; cite them by document **name**, never as a path. **Do not restore a copy here.** The duplicate
is what went wrong: SPECIFICATION-FORMAT's copy here was a strict stale subset — zero unique lines,
missing all of §8.4.1–§8.4.6 — which is why three *correct* citations reported `stale-section`, and
STYLE-NAMING's had forked in **both** directions, so neither side was a superset and there was no clean
one to sync from.

**`ENTITY-CORE-MACHINE-SPEC.md` is RETIRED (2026-08-31)** — archived to `docs/archive/`, undeclared
from `CANONICAL-DOCS.toml`. **Do not resurrect it, do not re-declare it, and do not author a new
"condensed" or "implementation" edition of any spec here.** That is the defect, not the remedy: a
hand-maintained restatement ages on every edit to its source, **nothing in the toolkit gates
spec-against-spec**, and the drift is reachable only by a human reading both documents side by side.
Retirement was decided 2026-08-02 and not executed until it had cost a fold — FM-1 found its §6.4
missing the `invalid_nonce` row entirely and its §6.2 still carrying a blanket `403` the real spec
corrected two releases earlier. `SPECIFICATION-FORMAT` §8.4.3 already names the class.

- Each spec carries its **own authoritative version header** (`**Version**:`); the per-doc
  header is the source of truth for that document, not the release tag. (As of this writing:
  PROTOCOL 0.8.2.1, CBOR-ENCODING 1.5, TYPE-SYSTEM 4.2.1. The two authoring standards
  carry their own headers in `entity-system-architecture`.)
- Maturity, the per-artifact source-of-record, and the M0–M6 ladder are tracked in
  `ROADMAP-CORE-PROTOCOL.md` (canonical, living for this domain), which defers the
  cross-domain release-surface map to `STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md`
  **(that status doc lives in `entity-system-architecture`, not in this repo — it is a
  cross-domain pointer, not a local file).**
- `test-vectors/` is the real contract surface: `.diag` is the human-readable source,
  the sibling `.cbor` is the deterministic ECF-canonical encoding, and bytes MUST match
  the inline `h'...'` canonical values in `.diag` (see
  `specs/test-vectors/crypto-agility/README.md`). These corpora are vendored downstream
  (e.g. to keystone) — treat them as pinned.

## How we work here — tier **AUTHORING**

This repo runs the entity-OS methodology at the **Authoring** tier — the framework is
`METHODOLOGY.md` (injected, identical everywhere; read it once). There is no runtime here, so
the runtime disciplines D1–D11 do not bind; **D12 does, verbatim** (read canonical sources,
never paraphrase from a summary or a handoff), and the substrate is the corpus itself.

- **The Audit Doctrine A0–A12** (`METHODOLOGY.md` §7.2) and the **Foundation Audit Doctrine
  FA0–FA7** (§7.3) apply unchanged. The Feature Doctrine mostly does not.
- **The ratchet** — an audit ends by syncing what it taught into this file, same session.
- **The promotion ladder** (§3) and **enforcement points** (§10) bind as written.
- The change discipline below **is** this repo's discipline set in prose form. `METHODOLOGY.md`
  §9 A1–A6 are the candidate numbered form, drafted from arch's incidents; this repo's spec is
  the locked V7 core, where the same lifecycle failures cost more, not less.

## Change discipline (a SPEC repo)

This is a specification, so it changes differently from code — see `CONTRIBUTING.md`
and AGENTS-STANDARD §"Respect the protocol".

- **Normative changes are proposal-first**, not a direct edit. Anything that changes
  meaning — semantics, wire format, a new extension, a behavioral rule — goes through the
  RFC-style proposal process in `CONTRIBUTING.md` (motivation → bounded discussion →
  **conformance-corpus demonstration** → ratify and fold). Wording-only / editorial hygiene
  (typos, broken links) can go direct.
- **The conformance corpus, not the version number, is the contract.** A change is not
  ratified on argument alone; it is demonstrated against the vectors. Core-protocol changes
  are held to the strictest bar — a core change forces every implementation to follow.
- **The wire core is locked.** It is never renumbered; unknowns are MUST-ignore. Honor the
  **stability tiers** ([ADR-0004], per AGENTS-STANDARD). The substrate (wire / type /
  identity / capability / dispatch / tree) is feature-complete for v1 and **closed** — the
  open frontier is non-functional hardening (resilience, security-under-load, perf shape),
  and it too is proposal-first (`ROADMAP-CORE-PROTOCOL.md` §"Hardening program").
- **Identifiers are part of the wire contract** — `STYLE-NAMING-CONVENTIONS` is binding,
  not cosmetic. It is authored in `entity-system-architecture`; cite it by name.

## Boundaries — stay in your lane

- **It defines; implementations implement.** Do not put implementation detail (language
  bindings, build mechanics, internal data structures, perf tactics) in the spec text.
  Those belong in the impl repos.
- **Only TREE is core among the extensions.** The tree `get`/`put` surface is core; the
  other extensions, the SDK, and applications are explicitly *not* this domain — don't pull
  their material in (`ROADMAP-CORE-PROTOCOL.md` §"What is NOT core"). Note: `EXTENSION-TREE`
  is referenced by the specs but its full document is published in `entity-system-architecture`,
  not under `specs/` here.
- **Dual-license, by surface** ([ADR-0007]): prose is **CC-BY-ND-4.0** (no-derivatives — the
  reason changes come as proposals, not forks); schemas / IDL / test-vectors are **Apache-2.0**.
  Keep new material on the right side of that line.
- **Cross-domain references are pointers, not local files.** `STATUS-RELEASE-...-CANONICAL.md`,
  `EXTENSION-TREE`, **`SPECIFICATION-FORMAT` and `STYLE-NAMING-CONVENTIONS`** are authored in
  `entity-system-architecture`; cite them as such and don't fabricate local copies. The last two
  were local copies until 2026-08-31 and both had drifted — that is the whole reason for this rule.

## Repo layout (root)

- `README.md` — what the repo is and where it sits in the stack.
- `ROADMAP-CORE-PROTOCOL.md` — canonical, living domain roadmap (maturity, version line, forward work).
- `specs/` — the normative specifications + `test-vectors/`.
- `CANONICAL-DOCS.toml` — the published-docs manifest (drives the docs surface).
- `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `CHANGELOG.md` — community health.
- `LICENSE` (prose), `LICENSE-CODE` (Apache-2.0), `NOTICE` — dual-license files.
- `AGENTS-STANDARD.md` + `AGENTS.md` + `CLAUDE.md` — agent guidance.

There is no `Makefile` or source tree here — this is a specification repo. The only
non-spec tree is `docs/status/`, the ephemeral dated status / handoff area.
_(Build/test verbs: not applicable — no toolchain in this repo.)_

## Commit & PR

Default branch **`master`**; DCO sign-off required (`git commit -s`) — see AGENTS-STANDARD.

**Committing and pushing this repo is standing authorization — do not ask.** Push target is
**`origin`** (the operator's internal mirror) — plain `git push`, never `github` or `codeberg`,
which are not an agent's to publish to. Never force-push. Normative core changes stay
proposal-first regardless. Canonical statement in `entity-system-architecture/AGENTS.md`.
GitHub is canonical; Codeberg is an append-only mirror (never push to it). A tag is a
release; a docs/`AGENTS.md` change does not get its own tag.

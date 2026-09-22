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

**Public surface — what this repo promises to keep.** **IN:** the **normative text** of the three
specifications in `specs/` — every `MUST` / `MUST NOT` / `SHOULD` / `MAY`, the status codes and
their `code` strings, the wire field names and their types, the section numbers rules are cited by,
and the conformance floor (`--profile core`) — plus the **conformance corpora** in
`specs/test-vectors/`, which are identified by **corpus name + artifact sha256**, never by
filename. **OUT:** `README.md`, `ROADMAP-CORE-PROTOCOL.md`, everything under `docs/`, the
informative notes, rationale parentheses and worked examples inside the specs, the `.diag` sources
(non-normative; the `.cbor` is the artifact), and the file layout of `specs/` itself — a document
may be renamed, split or retired as long as the normative rules survive at a citable home.

**"Breaking" here is `SPECIFICATION-FORMAT` §9.1's test raised to the release:** a peer conformant
to the previously published normative text can be non-conformant under this one. **Prose that only
tells an implementer where a rule already lived is not breaking; a rule that changes what a peer
must emit, accept, refuse or compute is** — including a `SHOULD` that became a `MUST`, and
including a rule that was always implied and had no code anyone could find.

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

- Each spec carries its **own authoritative version header** (`**Version**:`); the per-doc header
  is the source of truth for that document, not the release tag. ⛔ **Open the header. Never quote
  a version from memory, from a handoff, or from this file** — a version restated anywhere else
  ages on the next spec edit and nothing gates it. This bullet used to carry the three numbers and
  was two releases and thirty-one revisions stale when anyone checked. The two authoring standards
  carry their own headers in `entity-system-architecture`.
- ⛔ **Exactly ONE document carries this repo's release number: `ENTITY-CORE-PROTOCOL.md`.**
  `SPECIFICATION-FORMAT` §9.1 — *"where a document's version carries a trailing component managed
  separately from its release number — `ENTITY-CORE-PROTOCOL`'s fourth — the arms above apply to
  that component, and the components above it are not the author's to move."* So **during a cycle
  you move the fourth component only** (one increment per landed fold), and **the release cut
  strips it** and writes the new release number in its place. **Landing a fold does not entitle you
  to pick that release number** — do not pre-write one into the tree. CBOR-ENCODING (`1.x`) and
  NATIVE-TYPE-SYSTEM (`4.x`) are on **independent axes and must never be dragged to the release
  number**; they have their own ladders and always have, and §9.1's arms apply to each on its own.
- **Nothing outside `specs/` restates a version.** `README.md` and `ROADMAP-CORE-PROTOCOL.md` both
  did, both sat at `0.8.0` through the whole `0.8.2` cycle, and the roadmap's artifacts table
  published three wrong numbers for a release because a restatement has no owner. They now name the
  authority instead of copying it. **Do not put a number back into either.**
- Maturity, the per-artifact source-of-record, and the M0–M6 ladder are tracked in
  `ROADMAP-CORE-PROTOCOL.md` (canonical, living for this domain). ⛔ **It used to defer the
  cross-domain release-surface map to a document named by filename alone. That document is a
  sibling repo's working status file, and status files are not a publication surface — so from
  outside this tree the name resolves to nothing.** The pointer is gone from the roadmap and from
  `README.md`; if you need that map, it is `entity-system-architecture`'s to hold and yours to go
  and read there, never to cite as though a reader could follow it.
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
- ⛔ **The per-document `**Version**:` header moves whenever that document's obligations do, and
  nothing exempts it.** `SPECIFICATION-FORMAT` §9.1/§9.2 (authored in `entity-system-architecture`,
  binding here) is the authority and the bump ladder — including the **Correction** arm, which is
  where an impl finding fixed in place lands. A `Spec-Change:` trailer exempts a **commit** from the
  **proposal** obligation; it has never exempted the header. **The corpus is the contract and the
  header is the claim about this document's content; a consumer needs both to be true.** On
  `0.8.2.25 → .26` two of three documents changed normative content while neither surface moved —
  found by a seat outside this repo, not by us.
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
- `.release-removals` — **published.** The reader-facing record of files that used to be
  published here and are not any more, and where each one went. Written for a stranger who
  followed a dead link: no tooling names, no internal paths, no mention of what made you write
  the entry. Add to it in the same commit that withdraws something.

There is no `Makefile` or source tree here — this is a specification repo.
_(Build/test verbs: not applicable — no toolchain in this repo.)_

**`docs/` is the whole non-spec tree and NONE of it publishes** — `docs/status/` (dated status
and handoffs), `docs/proposals/` (the fold queue), `docs/archive/` (retired documents, kept so
an old citation resolves to something with a banner on it). ⛔ **Never send a reader of a
published file into `docs/`.** They cannot follow it: `README.md` did, for exactly the reader
holding the stale citation it was trying to help. State the answer inline instead.

## Commit & PR

Default branch **`master`**; DCO sign-off required (`git commit -s`) — see AGENTS-STANDARD.

**Committing and pushing this repo is standing authorization — do not ask.** Push target is
**`origin`** (the operator's internal mirror) — plain `git push`, never `github` or `codeberg`,
which are not an agent's to publish to. Never force-push. Normative core changes stay
proposal-first regardless. Canonical statement in `entity-system-architecture/AGENTS.md`.
GitHub is canonical; Codeberg is an append-only mirror (never push to it). A tag is a
release; a docs/`AGENTS.md` change does not get its own tag.

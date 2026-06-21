# Entity System — Normative Specification Format

**Version**: 1.1
**Status**: Active

---

## 1. Purpose

This document defines the format for normative specifications in the entity system. Normative specs contain implementation requirements — what a conforming implementation MUST, SHOULD, and MAY do. They are the authoritative source for building interoperable implementations.

Normative specs are distinct from architectural documents. Architectural documents provide grounding, motivation, design rationale, and system-level understanding. Normative specs reference architectural documents but do not duplicate their content. Implementation requirements live in normative specs; everything else lives elsewhere.

---

## 2. Specification Language

Entity type definitions are the primary specification language. Protocol messages, data structures, and extension types are defined as `system/type` entities — the same type system the protocol itself uses.

### 2.1 Type Definition Notation

```
type_path := {
  fields: {
    field_name: {type_ref: "primitive/string"}
    optional_field: {type_ref: "primitive/uint", optional: true}
    reference_field: {type_ref: "system/hash"}     ; Entity reference
  }
  constraints: {
    field_name: {pattern: "^[a-z]+$"}
  }
}
```

Rules:

- Type paths use forward slashes: `system/protocol/execute`
- Field specs use the `system/type/field-spec` structure from the core protocol
- Comments use `;` (semicolon) — consistent with CBOR diagnostic notation
- Inheritance via `extends` field when applicable
- Entity references use `type_ref: "system/hash"` — the hash value identifies the referenced entity in the envelope's `included` map

### 2.2 Inline Comments

```
system/example := {
  fields: {
    id:     {type_ref: "primitive/string"}    ; Unique identifier
    status: {type_ref: "primitive/uint"}      ; HTTP-style status code
  }
}
; Additional context about the type as a whole.
; Multi-line comments use consecutive ; lines.
```

### 2.3 Primitive Types

Normative specs reference the eight core primitives:

| Type | Description |
|------|-------------|
| `primitive/string` | UTF-8 text |
| `primitive/bytes` | Binary data |
| `primitive/uint` | Unsigned integer |
| `primitive/int` | Signed integer |
| `primitive/float` | IEEE 754 floating point |
| `primitive/bool` | Boolean |
| `primitive/null` | Null |
| `primitive/any` | Unconstrained |

### 2.4 Collection Fields

```
array_field: {array_of: {type_ref: "primitive/string"}}
map_field:   {map_of: {type_ref: "primitive/any"}}        ; Keys always strings
```

### 2.5 Supplementary Structures

Not every structure needs to be a full `system/type`. Inline structures that appear only within one type definition use lowercase names without a registered type path:

```
grant-entry := {
  handlers:   {include: [string], exclude: [string]?}   ; system/capability/scope
  resources:  {include: [string], exclude: [string]?}   ; system/capability/scope
  operations: {include: [string], exclude: [string]?}   ; system/capability/scope
  peers:      {include: [string], exclude: [string]?}   ; system/capability/scope
  constraints: any?                                      ; handler-interpreted
}
```

These are documented alongside the type that contains them. They do not appear in `system/type/*`.

---

## 3. Algorithm Notation

Algorithms use pseudocode with the following conventions:

### 3.1 Function Signature

```
function_name(param1, param2):
  ; body
  return result
```

### 3.2 Control Flow

```
verify_something(input):
  if condition: DENY
  if other_condition:
    do_something()
    do_something_else()

  for item in collection:
    if !check(item): return false

  while condition:
    current = next(current)

  return true
```

### 3.3 Conventions

- `DENY`, `ALLOW`, `REJECT` as terminal outcomes (all caps)
- `lookup(hash, map)` for entity resolution from included maps
- `now()` for current timestamp
- Set operations: `∪` (union), `∈` (membership)
- Indentation for nesting (2 spaces)
- No language-specific syntax (no braces, no `def`, no `fn`)
- Comments with `;`

### 3.4 Error Conditions

```
if invalid_state:
  return error(status_code, "error_code", "Human-readable message")
```

Or for verification algorithms, the pattern `if bad: DENY` with final `ALLOW`.

---

## 4. Requirements Language

Per RFC 2119:

| Keyword | Meaning |
|---------|---------|
| **MUST** | Absolute requirement. Non-compliance breaks interoperability. |
| **MUST NOT** | Absolute prohibition. |
| **SHOULD** | Recommended. Valid reasons to deviate may exist, but implications must be understood. |
| **SHOULD NOT** | Discouraged. |
| **MAY** | Optional. Implementations choosing not to implement remain conformant. |

Keywords appear in **UPPERCASE BOLD** in prose, or uppercase in pseudocode comments.

---

## 5. Document Structure

Every normative spec follows this structure. Sections may be omitted if empty, but the ordering is fixed.

### 5.1 Required Sections

```
# Title — Normative Specification

**Version**: X.Y
**Status**: Draft | Active | Superseded
**Depends**: [list of normative specs this extends or requires]

---

## N. Type Definitions
    ; All types introduced by this spec

## N. Algorithms
    ; Pseudocode for required behaviors

## N. Constants
    ; Status codes, error codes, limits, enumerations

## N. Conformance
    ### N.1 MUST Implement
    ### N.2 SHOULD Implement
    ### N.3 MAY Implement
    ### N.4 Implementation-Defined
```

### 5.2 Optional Sections

Specs MAY include additional sections before Conformance:

- **Foundations** — core concepts and definitions (used by the core protocol spec)
- **Flow diagrams** — ASCII wire diagrams for multi-step interactions
- **Security considerations** — threat model, mitigations
- **Extension points** — how this spec can be extended by other specs

### 5.3 Header Fields

| Field | Required | Description |
|-------|----------|-------------|
| Version | Yes | Semantic version of this spec |
| Status | Yes | Draft, Active, or Superseded |
| Depends | If applicable | Normative specs required as prerequisites |
| Encoding | If applicable | Wire encoding reference (e.g., ECF) |

### 5.4 Section Numbering

- Top-level sections: `## N. Title`
- Subsections: `### N.M Title`
- Cross-references: `§N.M` (e.g., "see §3.2")
- External cross-references: `DOC.md §N.M` (e.g., "see ENTITY-CORE-PROTOCOL.md §5.2")

The full addressing model — canonical reference forms, what a spec may cite, and
how unresolved references are classified — is in §11.

---

## 6. Tables

Tables are used for:

- **Constants and codes** — status codes, error codes, type enumerations
- **Conformance levels** — what each level requires
- **Field summaries** — when a type has many fields and prose is needed per-field
- **Comparison** — when distinguishing similar concepts

Format:

```
| Column | Column | Column |
|--------|--------|--------|
| data   | data   | data   |
```

---

## 7. Examples

### 7.1 Encoding

Examples use CBOR diagnostic notation (RFC 8949 §8), not JSON:

```
{
  "type": "system/protocol/execute",
  "data": {
    "request_id": "req-001",
    "uri": "entity://peer123/some/path",
    "operation": "get",
    "params": null
  }
}
```

When showing binary or encoded output, use hex with annotation:

```
ecfv1-sha256:a1b2c3d4...    ; 64 hex chars
```

### 7.2 When to Use Examples

Examples are supplementary. The type definition IS the specification — examples illustrate but do not define. If an example contradicts a type definition, the type definition wins.

---

## 8. Extension Spec Conventions

System extension specs (inbox, compute, relay, etc.) follow additional conventions:

### 8.1 Dependency Declaration

```
**Depends**: ENTITY-CORE-PROTOCOL.md
```

Extensions MUST declare which normative specs they depend on. Types from dependencies are referenced by path, not redefined.

### 8.2 New Types

Extension specs define new types that layer on core protocol types. Extensions MUST NOT redefine core types. Extensions MAY define types that reference core types via `type_ref`.

### 8.3 New Operations

Extensions that add handler operations define them as `system/handler/operation-spec` entries:

```
my-handler/operations := {
  new_operation: {
    input_type:  "my-extension/input"
    output_type: "my-extension/output"
  }
}
```

### 8.4 Optional Fields on Core Types

Extensions MAY define optional fields that appear on core types (e.g., `deliver_token` on EXECUTE). These are documented in the extension spec, not the core spec. The core spec's Open Types (§2.7) guarantees preservation.

When an extension adds optional fields to a core type, the extension spec MUST:
- Define the field as a type definition
- State which core type it extends
- Specify behavior when the field is absent (default)
- Specify behavior when the field is present

### 8.5 Conformance

Extension specs have their own conformance section. An implementation MAY be conformant to the core protocol without implementing any extensions. Extension conformance is independent.

---

## 9. Document Lifecycle

| Status | Meaning |
|--------|---------|
| Draft | Under development. Subject to change. |
| Active | Stable. Implementations should target this version. |
| Superseded | Replaced by a newer version. Reference only. |

Version bumps:
- **Minor** (X.Y → X.Y+1): Additive changes (new optional fields, new MAY requirements)
- **Major** (X.Y → X+1.0): Breaking changes (new required fields, changed algorithms, removed types)

---

## 10. Relationship to Architectural Documents

Normative specs answer: **what must an implementation do?**

Architectural documents answer: **why does the system work this way?**

| Aspect | Normative Spec | Architectural Document |
|--------|---------------|----------------------|
| Audience | Implementors | Designers, evaluators |
| Content | Type definitions, algorithms, constraints | Rationale, tradeoffs, vision |
| Authority | Binding | Informational |
| Format | This format | Prose, diagrams, free-form |
| Stability | Versioned, breaking changes tracked | Evolves freely |

Normative specs MAY include brief contextual notes (1-2 sentences) explaining why a rule exists. Extended rationale belongs in architectural documents.

---

## 11. Addressing and Cross-References

A spec does not stand alone — it cites other specs, and others cite it. The set
of specs is therefore a **corpus**: a graph of documents laid over each
document's own section tree. This section defines how a spec addresses itself
and others, what it is permitted to cite, and how an unresolved citation is
classified. It is the corpus-level companion to §5.4 (a single document's
internal numbering).

### 11.1 The Addressing Model

Each document is the canonical **host** for the sections it defines; each
`§N.M` is a **path** under that host. A fully-qualified address is therefore:

```
DOC.md §N.M        ; host (the file) + path (the section)
```

This mirrors the entity address space: the file is the host the way a `peer_id`
is the host of an entity tree, and `§N.M` is the path beneath it. The `.md` is
not decoration — it names the host, which is a real file in the corpus. An
address with no host is implicitly hosted by the current document.

### 11.2 Reference Forms

| Reference | Form | Example |
|-----------|------|---------|
| Same-document section | `§N.M` | "see §3.2" |
| Other spec, a section | `DOC.md §N.M` | "see ENTITY-CORE-PROTOCOL.md §5.2" |
| Other spec, whole document | `DOC.md` | "defined in EXTENSION-TREE.md" |

Rules:

- The canonical cross-document citation form is **`DOC.md`** (with `§N.M` when
  pointing at a section). The `.md` suffix is REQUIRED on a citation — it is the
  host identifier. A bare `DOC` with no `.md` is NOT a citation form.
- A target MUST be cited in one form across the corpus. Mixed `DOC` / `DOC.md`
  citations of the same target are **citation-form drift** and MUST be
  normalized to `DOC.md`.
- Section numbers use the **target's** numbering, including any amendment-letter
  form the target uses (`§N.Ma`).
- A prose name is not a citation. "the core protocol spec" is prose; the
  citation is `ENTITY-CORE-PROTOCOL.md`. Specs SHOULD give the citation, not
  only the prose name, wherever a reader would need to find the source.
- Naming a file as a filesystem artifact (e.g. in a code block, a path, or a
  `Depends` field) is not a citation and is exempt from these forms.

### 11.3 What a Spec May Cite

A normative spec references exactly three kinds of target:

| Target | Resolves to | Status |
|--------|-------------|--------|
| Another normative spec | a `.md` file in the corpus | MUST resolve |
| An architectural / guide document | a `.md` document outside the normative set | Informational; allowed |
| Anything else | — | MUST NOT |

The third row is the rule that matters for release: process and team artifacts —
review notes, session handoffs, arch-team update memos, status docs — MUST NOT
be cited from normative text. A ratified proposal MAY be named as historical
provenance, but only in a non-normative note (see §10), never as the source of a
requirement. The requirement lives in the spec; the proposal is how it got
there.

### 11.4 Unresolved References

A citation whose target is not a file in the corpus ("dangling") is one of three
things, and the disposition differs by class:

| Class | Meaning | Disposition |
|-------|---------|-------------|
| **Forward** | A spec planned but not yet written | Permitted ONLY when explicitly marked `(planned)` and confined to non-normative notes or an Extension Points section. An unmarked forward citation is an error. |
| **Stale** | A spec that was removed, renamed, or scope-cut | MUST be removed, or redirected to the superseding spec. |
| **Leak** | A name that is not a spec at all (a process/internal artifact) | MUST be removed — see §11.3. |

A qualified citation `DOC.md §N.M` whose document resolves but whose section
does not (a **stale section ref**) MUST be corrected to a section that exists in
the target.

### 11.5 Tooling

These rules are mechanically checkable. `spec-topology` builds the
canonical-name registry (file stem = host), resolves every citation against it,
and flags citation-form drift, dangling references, and stale section refs. The
addressing standard is the contract; the tool reports deviations from it. A
green corpus is one where every citation resolves, in the canonical form, to a
permitted target.

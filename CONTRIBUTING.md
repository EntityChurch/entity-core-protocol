# Contributing to the specification

Thanks for your interest. A specification is changed differently from code: the
text is licensed no-derivatives on purpose, so the way to change it is to bring
your proposal *here* rather than to fork it.

## Small fixes

Typos, broken links, and obvious editorial errors: open an issue or a small pull
request describing the fix. These are handled directly.

## Substantive changes (the proposal process)

Anything that changes meaning — semantics, wire format, a new extension, a
behavioral rule — goes through a proposal, modeled on the Rust RFC process:

1. **Open a proposal** describing the motivation, the design, the alternatives you
   considered, and the impact on existing implementations.
2. **Discussion** happens in the open on the proposal, with a bounded comment
   period so it converges rather than drifting forever.
3. **Conformance gate.** A change is not ratified on argument alone — it has to be
   demonstrated against the conformance corpus. Core-protocol changes are held to
   the strictest bar, because a core change forces every implementation to follow.
4. **Ratify and fold.** Once accepted and demonstrated, the change is folded into
   the spec and recorded.

The conformance corpus, not the version number, is the real contract — proposals
are evaluated against it.

## Sign-off (DCO)

This project uses the [Developer Certificate of Origin](https://developercertificate.org/)
instead of a CLA. Sign off every commit with `git commit -s`:

```
Signed-off-by: Your Name <you@example.com>
```

## Security

For security-relevant issues in the specification or its reference behavior, follow
[SECURITY.md](SECURITY.md) — do not open a public issue.

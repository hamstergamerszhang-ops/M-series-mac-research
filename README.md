# apple-m5-research

Private research notes on Apple M5-series internals (ANE, GPU, CPU, memory,
power) plus a program of measured optimization work on the same hardware.
**Not yet published** — see the checklist below before this goes public.

## Contents

- [`MANUAL.md`](MANUAL.md) — the full research/optimization manual, as written.
- [`ROADMAP.md`](ROADMAP.md) — open problems, tracked as issues.

## Confidence tags

Carried over from the manual because they're load-bearing for how anything
here should be read:

- **[VERIFIED]** — checked live (registry, disassembly, signatures, byte-level file inspection)
- **[MEASURED]** — benchmarked on this machine, not independently re-run by anyone else
- **[ESTIMATE]** — plausible, never actually measured
- **[BELIEF]** — folklore or string-level inference

None of these have been independently verified by a second party or a
second machine. Until that happens, treat the whole document as one
person's findings on one machine, not an established reference.

## Scope note

One item from the manual's open-problems list is deliberately **not**
tracked as work here: the "no-entitlement" ANE access path via VirtIO
paravirtualization (§5.6 / §9.5 / §14 item 3). Everything else in the
open-problems list is hardware characterization — disassembly, protocol
decoding, a counter-based profiler, ISA feature use. That one item is
different in kind: it's a way to route around an access control Apple put
there on purpose (the manual's own words: "the exclave boundary exists so
a compromised kernel cannot forge ANE work"). It stays documented in
`MANUAL.md` for completeness. It's not on `ROADMAP.md` and has no issue.

## Before this goes public

- [ ] Every `[MEASURED]` claim has been reproduced at least once more, or
      independently spot-checked, before it's presented as fact to a public
      audience.
- [ ] Every `[ESTIMATE]` / `[BELIEF]` tag is still visible in the public
      version — not quietly dropped because it reads more confidently
      without it.
- [ ] §5.5/§5.6/§9.5/§14 (firmware readability, the entitlement gap, the
      VirtIO route) re-reviewed specifically for whether describing them
      crosses from "documents a gap exists" into "how-to." If it's a
      genuine finding Apple would want to know about, consider responsible
      disclosure via Apple Security Bounty before or alongside publishing.
- [ ] No Apple binaries, firmware images, or copied private headers are
      included in the repo — findings and original tooling only, no
      redistributed Apple IP.
- [ ] Publicly-checkable claims (chip identifiers, core/GPU counts, etc.)
      cross-checked against Apple's own spec pages. (Done once already for
      the Mac17,9 / M5 Pro core config — extend to anything else checkable
      before publishing.)
- [ ] If anything here is being framed as enabling interoperability
      (running other software on Apple hardware) rather than pure
      documentation, get an actual legal read on it for your jurisdiction —
      not legal advice, just worth doing once before this is public.

# M-series-mac-research

Research notes on Apple M5-series internals (ANE, GPU, CPU, memory,
power) plus a program of measured optimization work on the same hardware.

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

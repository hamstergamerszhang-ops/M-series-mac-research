# Roadmap

Open problems from `MANUAL.md` §14, tracked as issues. Numbering matches
the manual's own list; item 3 (VirtIO no-entitlement ANE access) is
intentionally excluded — see the scope note in `README.md`.

| # | Issue | Manual ref |
|---|---|---|
| 1 | Disassemble the h17 ANE firmware (`hyperion_j71y`) | §5.5, §14.1 |
| 2 | Decode the 0x948-byte `ProgramSendRequest` struct | §5.3, §14.2 |
| 4 | Research Espresso, the placement engine beneath CoreML | §9.5, §14.4 |
| 5 | Build a GPU profiler on `AGXGPURawCounterBundle` | §4, §14.5 |
| 6 | Write SME/SME2 kernels | §2, §12, §14.6 |
| 7 | Complete the bandwidth measurement sweep | §3, §14.7 |
| 8 | Trace the ANE cache-lease lifecycle | §14.8 |

Issue links get filled in once the issues exist on GitHub.

## Conventions

- Every issue should carry forward the manual's confidence-tag discipline:
  new findings get tagged [VERIFIED]/[MEASURED]/[ESTIMATE]/[BELIEF], same
  as the manual.
- A closed issue means the finding made it back into `MANUAL.md`, not just
  that work happened.

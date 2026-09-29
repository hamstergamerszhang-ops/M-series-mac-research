# Apple M5 Series — Research & Optimization Manual

**Second edition — 2026-09-24.** Compiled from live-system reverse
engineering of an M5 Pro (Mac17,9, macOS 26.5, firmware 18000.120.36)
and from a full program of measured optimization work on the same
machine: custom Metal compute kernels, CoreML/ANE inference
characterization, out-of-core memory management, and power/thermal
analysis. Supersedes the first-edition field guide.

> **Method, stated up front.** Everything here comes from read-only
> probing (`ioreg`, `sysctl`, `kextstat`, `strings`, `nm`, `otool`,
> code-signature dumps) plus measured benchmarks, on stock macOS with
> no security settings changed. Every claim carries a confidence tag:
> **[VERIFIED]** checked live (registry, disassembly, signatures, or
> byte-level file inspection) · **[MEASURED]** benchmarked faithfully,
> not independently re-runnable · **[ESTIMATE]** plausible, never
> actually measured · **[BELIEF]** folklore or string-level inference.
> Optimization numbers additionally carry their measurement conditions
> — on this machine, a concurrent load can distort timings **14×**, so
> a number without its gate is a rumor (see §8.4, the "54 TF/s"
> incident).

---

## Contents

**Part I — The machine**
1. The M5 series at a glance
2. CPU
3. Memory
4. GPU
5. Neural Engine
6. Power, thermal, throttling

**Part II — Optimization guides (new in this edition)**
7. GPU kernel optimization: 0.9 → 5.16 TFLOP/s
8. Measurement methodology (read before any benchmark)
9. ANE optimization: the CoreML contract
10. Hybrid serving architecture
11. Memory: arenas, streaming, and RSS discipline
12. CPU optimization

**Part III — Research frontier**
13. The extraction toolchain
14. Open problems
15. Naming traps and folklore corrections
16. Probe cheat sheet

---

# PART I — THE MACHINE

## 1. The M5 series at a glance

| What | Value | Confidence |
|---|---|---|
| SoC family | t6050-class internal codes; chip-id 0x6050 on Pro parts | [VERIFIED] |
| CPU | 15 cores = 10 "Performance" (Sawtooth) + 5 "Super" (Everest); ARMv9.2 | [VERIFIED] |
| GPU | G17X-class, 2 tiles, 20 physical / 16 active cores, Metal 4 | [VERIFIED] |
| ANE | h17 architecture, 16 cores, control plane in ExclaveOS | [VERIFIED] |
| ANE throughput | "~42 TOPS fp16" — estimate only; Apple publishes nothing | [ESTIMATE] |
| Memory | unified; 307 GB/s on Pro tier (Apple spec) | [VERIFIED] |
| Coprocessors | ~19 RTKit-class engines besides the CPU clusters | [VERIFIED] |

A modern Apple Silicon Mac is a constellation of computers: each
coprocessor has its own firmware, address space, and message-passing
transport. Most of this manual is a map of that constellation, and the
optimization half is about using the three big engines (CPU, GPU, ANE)
without tripping over the other sixteen.

## 2. CPU

**Topology [VERIFIED].** Two core designs, exposed via `hw.perflevel`:

- **Performance (Sawtooth)**, `hw.perflevel1` — two clusters of five,
  each sharing an **8 MB L2**.
- **Super (Everest)**, `hw.perflevel0` — one five-core cluster sharing
  a **16 MB L2**.

Three consequences:

1. **Three L2 domains.** Threads sharing a domain communicate through
   cache; threads split across domains go through fabric. For
   latency-sensitive multithreaded work, pinning to a single L2 domain
   is measurably better than letting the scheduler scatter workers
   (§12).
2. **No SMT.** `hw.ncpu == hw.physicalcpu` (15/15). Nothing to reason
   about.
3. **The mystery 18th core.** CLPC reports 18 cores on this 15-core
   part — plausibly harvested spares or secure-function reserves.
   Unproven either way. [BELIEF]

**Instruction set [VERIFIED].** ARMv9.2-class: **SME, SME2, SME2p1**
(the Scalable Matrix Extension — present, exposed, and almost unused
by public software; the biggest free lunch on the chip), BTI,
PAuth2/FPAC, MTE (running in production — see §15). Notably *absent*:
standalone SVE/SVE2 capability flags. Apple exposes SME, not SVE2, to
userspace — a correction to write-ups that assume ARMv9 implies SVE2.

**Measured [MEASURED].** fp32 GEMM across 15 threads: peak
**1.89 TFLOPS at 2048³**, 1.74 at 4096³, 1.39 at 256³. Below ~512³
thread/cache overhead dominates; the GPU crosses over near 1024³ and
is ~14× faster at 4096³. The CPU is a utility player and a poor GEMM
engine — except through Accelerate BLAS, which self-parallelizes
better than hand-rolled pools (§12).

## 3. Memory

**One pool, three consumers.** CPU, GPU, and ANE address the same RAM
directly — weights load once and are read by whichever engine needs
them. The cost: all three engines compete for one bus.

**Bandwidth.** Trust Apple's spec numbers (153/307/460–614 GB/s by
tier); distrust third-party speed-grade claims. The device tree
reports the type only as "LPDDR5" — too coarse to confirm anything.
The honest measured reference point: a Metal triad kernel (read 2,
write 1) sustains **248 GB/s** at 256 MiB working sets on this part
[MEASURED] — i.e., ~80% of the spec figure, which is the realistic
planning number.

**The arithmetic every local-LLM claim must survive [VERIFIED math].**
For a 3B-parameter model in fp16: weights 6.0 GB, KV cache ~117 MB at
4k context (28 layers × 8 KV heads × 4096 tokens × 128 dim × 2 bytes
× 2; public write-ups quoting "1–2 GB" misplace a factor of ten),
buffers ~0.5 GB. Decode is memory-bound: every token streams the whole
weight set. At 307 GB/s, a 6 GB model has a hard ceiling of
**~51 tokens/second** — no software cleverness beats the bus. Benchmarks
claiming far above that are measuring prefill or something else.

## 4. GPU

**Hardware [VERIFIED].** `AGXAcceleratorG17X` / `AGXG17SDevice`, two
tiles (`num_mgpus=2`), 20 physical cores at 10/tile, 8-of-10 enabled
per tile → **16 active**. USC generation 3. Metal 4. The GPU's
`GPUConfigurationVariable` registry dictionary is a spec sheet Apple
never published.

**Clocks [VERIFIED].** 14 discrete perf states in the device-tree
`perf-states` blob, from deep idle to **1620 MHz @ 1.035 V**.

**Measured [MEASURED], via Metal/MPS, fp16.**

| Workload | Result |
|---|---|
| GEMM 4096³ (MPS) | 24.98 TFLOPS |
| Custom kernel GEMM 2048³ (this manual, §7) | **5.16 TFLOPS** |
| FlashAttention-style, 12k context | ~298k tok/s prefill |
| SwiGLU FFN, 12k context | ~1.46M tok/s |

**The unread treasure [VERIFIED existence].** Apple ships a private
hardware-counter interface — `AGXGPURawCounterBundle`,
`GPURawCounterSelect`, `IOGPUCommandQueueCreate` — that nobody outside
Apple has published a reader for. Meanwhile the IOReport bus (§6)
freely publishes GPU utilization, thermal, power zones, and per-state
residency to unprivileged processes. (For timing individual kernels,
use the command buffer's own GPU timestamps — §8.3.)

**WindowServer shares your GPU [VERIFIED, twice the hard way].** A
wedged GPU stack takes the whole machine down via watchdog force-reset;
there is no per-process sandbox for GPU faults. The operational rules
that keep custom GPU work safe are in §8.5 — read them before running
anything.

## 5. Neural Engine

The most powerful and least documented accelerator on the chip. State
of play: hardware layout mapped, the full software path from app call
to kernel selector mapped (entitlements and daemon names included),
compiler pipeline mapped, firmware — which everyone assumed was
encrypted — **readable all along**. What has never happened anywhere
outside Apple: a hand-written kernel executing on a modern ANE, and a
full firmware disassembly. Both are unblocked (§14).

### 5.1 Hardware, as the OS sees it [VERIFIED]

Device-tree node `ane0@8000000`: `compatible = "ane,t8132exclave"`,
`ane-type = 0x220` (544 little-endian — older write-ups quoting 0x0200
decoded the bytes backwards), `pre-loaded`, `exclave-assigned`. Seven
MMIO regions (32 MB control, 96 KB config, 16 KB status, 256 KB
mailbox, ~16 MB firmware window, 32 MB program memory, 16 KB
interrupts). Driver-reported: 16 cores, architecture "h17", firmware
256.17. Behind its own DART IOMMU with a default mapper plus seven
isolation domains (hypothesis: per-model sandboxing in the secure
world [BELIEF]).

Kernel drivers: `AppleH16ANEInterface` (user client class
**H11ANEIn**), the per-SoC HAL, `AppleANELoadBalancer`, and the
virtualization device `AppleVirtIONeuralEngineDevice`.

**16 KB tensor-tile alignment, fp16-first** — inherited from the ANE
IP block's public lineage (the open-source Linux driver for the h11
generation, `TILE_SIZE 0x4000`).

### 5.2 The journey of a request [VERIFIED, every hop]

```
1. your app → CoreML
2. CoreML/NeuralNetworks — "Espresso" engine decides placement + compiles
3. modelmanagerd — lifecycle: download, verify, stage, cache
4. aned — ANE daemon, holds the entitlements:
     com.apple.private.ane.iokit-user-access
     com.apple.private.ANECompilerService.allow
5. H11ANEInUserClient → AppleH16ANEInterface (kernel)
6. → the ANE exclave (ANEDriverEC protocol) — control plane lives in
   ExclaveOS, not the kernel
```

The design intent: the ANE is not an accelerator you rent, it is a
**service you petition**. Model code is verified, compiled, cached,
and sandboxed before it touches the engine; the exclave boundary
exists so a compromised kernel cannot forge ANE work. Building ANE
tooling means either cooperating with this pipeline (CoreML-mediated —
works today, no privileges) or replacing parts of it (hard; the
entitlement walls are real and now empirically mapped — a direct
H11ANEIn open attempt via ctypes IOKit returns `kIOReturnNotPrivileged`
on a stock system [VERIFIED]).

### 5.3 The kernel ABI [VERIFIED by disassembly of the carved driver]

17 external selectors on `H11ANEInUserClient`; the main entry,
**ProgramSendRequest (selector 2)**, takes a 0x948-byte argument
struct (asynchronous; results via completion callback). Selectors 0–7
are continuous with the classic public Linux ANE driver numbering,
cross-checking the decode. `ProgramChainingPrepare` (selector 9) backs
model chaining. The 0x948 struct's full field map is still open — its
parser hides behind two pointer-authentication-diversified indirect
calls.

### 5.4 Compiler pipeline [VERIFIED via symbols]

CoreML model → **MIL** (942 ops, including SDPA variants and fp8 e5m2
— a strong signal about engine expectations; the "ggml" in the
compiler service's bundle id is an Apple-internal namespace, *not*
llama.cpp's GGML) → **ANECompiler** (~133k symbols; procedures =
model chaining; targets h10–h18 — h18 being an unshipped future
generation visible in the binary) → **Zin programs** (`ZinComputeProgram`,
"command-word v11"), whose runtime lives inside the kernel driver.
Custom-kernel recipe, for whoever finishes it: valid Zin program +
0x948 request + selector 2 + the entitlement.

### 5.5 The firmware — readable all along [VERIFIED byte-level]

`h17_ane0_fw_hyperion_j71y.im4p`, ~1.5 MB, world-readable in the
staged firmware directory. Plain IM4P, payload an **unencrypted
arm64 Mach-O** — no KBAG, no per-device GID encryption. The belief
that coprocessor firmware is key-locked is wrong for these images.
~2,400 strings already name the machinery: **PENA** program format,
**thread domains** (TDs, the execution units), **BAR registers**
(banked register file, `barId < 122`), L2 spill buffers, version-2 IPC
rings, and **95 `CSNE_CMD_*` firmware commands** (~60 undocumented,
including `EXCLAVE_MODE_START/STOP`, `INFERENCE_CALL`,
`DATA_CHAINING_EVENT`). The staged directory holds ~17 variants across
five ANE generations (h13–h17) — a ready-made cross-generation diffing
corpus. Nobody has disassembled any of it. §14, problem 1.

### 5.6 The virtualization route [VERIFIED instruction-by-instruction]

VMs can reach the **host's** ANE through `com.apple.avp.ane`
(ParavirtualizedANE.framework): uint16 command IDs 0x00–0x11 through
an 18-entry jump table — compile, load, evaluate, purge, IOSurface
transport, VM save/restore — bottoming out in the same H11ANEIn user
client. The host side of this protocol is **not** behind the ANE
entitlements. A guest with a determined VirtIO driver gets ANE
execution without anyone holding `com.apple.private.ane.iokit-user-access`.
This is the most promising no-entitlement research path. (Bonus
technique that fell out: ObjC selector references under dyld
chained-fixups resolve as `0x7ff800000000 + 36-bit offset` into the
shared dedup pool — anyone doing static analysis of shared-cache ObjC
code wants that rule.)

### 5.7 What has actually run on the ANE [MEASURED, placement verified via MLComputePlan]

- GPT-2-class FFN prefill: ~395k tok/s @ seq 128.
- Gemma-scale FFN (75.5M params): ~48k tok/s @ seq 4096; 12k contexts
  with 7 of 15 ops ANE-dispatched, agreeing with CPU to ~0.01.
- Full characterization sweep (§9): up to **~1.0M tok/s** at the
  batching knee.

And the honest negative result: **no custom (hand-written) kernel has
ever been demonstrated executing on a modern ANE, and no full LLM has
been shown running end-to-end on the ANE.** Verified ANE execution to
date is CoreML-compiled subgraphs. Treat claims otherwise with
suspicion — the first edition's source material blurred this line
repeatedly.

## 6. Power, thermal, throttling

**The governor [VERIFIED — every number live from the registry].**
`AppleCLPC` (matched on `clpc,t6050`) is four PI controllers:

| Limiter | kp | ki | Guards |
|---|---|---|---|
| package-average | 21475 | 480 | long-term sustained power |
| package-low-peak | 16777 | 5033 | power transients |
| cpu-average | 117441 | 6711 | CPU power long-term |
| cpu-low-peak | 11996 | 122474 | CPU transients |

(Older write-ups quoting "kp=16777, ki=480" as *the* controller are
pairing numbers from two different limiters.) Timescales: 250 ms
thermal, 1000 ms battery, 10 ms peaks, 100 ms low band; six performance
bands; three cluster masks. **Package ceiling ~22.5 W** on this mobile
Pro part. The power split allocates to CPU and GPU but **zero to the
ANE** — the ANE sits outside the package budget on its own accounting
(no dedicated rail; the folklore claim of "dedicated ANE power rail"
was based on nonexistent registry classes).

**What you can change from userspace: almost nothing, by design.**
Writing CLPC properties is rejected. The sanctioned levers are High
Power Mode (`pmset`) and not running hogs. "Anti-throttle" in practice
means *behavioral* countermeasures: keeping the GPU out of deep idle
with periodic work (deep-idle exit latency spikes are real), L2-domain
thread pinning, and killing runaway processes.

**Monitoring without root [VERIFIED].** `powermetrics` needs sudo, but
the **IOReport bus does not**: CPU/GPU/ANE/DRAM/package watts from
"Energy Model" channels, per-cluster frequency, GPU utilization and
thermal, battery gas-gauge, 3,415+ telemetry channels across 52
classes — all readable from an unprivileged process through IOKit.
This is how every watt number in this manual was taken (sampled from a
background thread; in-loop sampling perturbs the timed window).

**The SMC [VERIFIED by disassembly].** Full ABI recovered — three
selectors, `SMCParamStruct` layout (key @ 0x0, size @ 0x1c, command @
0x2a, payload @ 0x30), commands 5/6/8/9, big-endian four-char keys.
On macOS 26 all unentitled SMC I/O returns `kIOReturnBadArgument`;
the entire bus is gated behind `com.apple.private.applesmc.user-access`,
held on a stock system by exactly one binary: Apple's `smcDiagnose`
(whose output is parseable — ~3,300 keys, 194 thermal sensors, both
fans, package power, full battery/charger telemetry).

---

# PART II — OPTIMIZATION GUIDES

*Everything in this part is from the 2026-09 optimization program on
this exact machine: custom MSL kernels compiled at runtime through
PyObjC, CoreML/ANE characterization, arena memory management. Numbers
carry their measurement conditions. The code lives in the workspace
(`tools/gpu_core`, `tools/ane_core`, `tools/unified`); this manual
records the knowledge, not the files.*

## 7. GPU kernel optimization: 0.9 → 5.16 TFLOP/s

The complete progression, with every technique that survived
measurement. All kernels fp32-accumulate unless noted; "bit-exact"
means verified against numpy on the same inputs.

### 7.1 The progression

| Kernel | Techniques | TFLOP/s @2048³ | Notes |
|---|---|---:|---|
| `gemm_tiled` | 16×16 tiles, per-thread FMA | 0.87–1.02 | portable fallback, no alignment constraints |
| `gemm_mma` | simdgroup MMA, 32×32 tiles | 2.32 | the workhorse; M,N%32==0, K%8==0 |
| `gemm_mma2` | + cooperative staging to threadgroup memory | 3.06 | fully coalesced loads |
| `gemm_mma3` | + double buffering (prefetch tile i+1) | 3.00 | hides global-load latency; K%64 |
| `gemm_mma3_f16` | fp16 tiles, fp32 accumulate | 3.17 | half the threadgroup memory and DRAM traffic |
| `gemm_mma4` | 64×32 register blocking, float4 staging | 4.23 (fp32) | route by shape: M ≥ 2048 or K ≥ 4096 |
| `gemm_mma4_f16` | mma4 + fp16 tiles | **5.16** | bit-exact vs fp32-reference-on-fp16-inputs |

Reference points at 2048³: MPS bridge 1.45, numpy/Accelerate 1.39.
The custom path beats MPS **2.9–6.9×** depending on shape and beats
Accelerate 2.0–2.8× at 1024³+. (MPS's own peak of ~25 TFLOPS at 4096³
uses fp16 and Apple's heavily-tuned libraries; the numbers here are
what hand-written MSL achieves — the gap is the cost of visibility.)

**Route by shape**: mma3/mma3_f16 for 1024³-class (tile parallelism
wins), mma4 for tall/deep (M ≥ 2048 or K ≥ 4096). At skinny M=32
(decode GEMMs), all simdgroup-staged kernels degenerate: mma3_f16 runs
0.31–0.42 ms against a ~32 µs memory roofline. Two purpose-built
scalar streaming decode kernels (one thread per column, 32 row
accumulators, half4 loads, 4-deep prefetch) were written and **both
lost** to mma3 — documented negative result: the simdgroup MMA
pipeline is what this GPU is built around. Don't fight it; batch your
decode instead.

### 7.2 simdgroup semantics — established empirically

The MSL simdgroup-matrix API on this G17X/MSL stack, dest-first:

```metal
simdgroup_load(m, ptr, row_stride);            // row-major 8x8
simdgroup_multiply_accumulate(d, a, b, c);     // d = a * b + c
simdgroup_store(m, ptr, row_stride);
```

What does **NOT** work (compiles or silently mis-executes): the 5-arg
`ulong2` load/store overloads (scrambled layouts), `operator+` between
simdgroup matrices, the `acc = a*b + acc` idiom, the 3-arg
`simdgroup_multiply_accumulate`, and the documented-looking
`(a, b, acc, acc` argument order — which **silently zeroes the
result**. The destination is the FIRST argument. Every one of these
cost days to discover; they are recorded so nobody pays again.

Other hard-won MSL rules:

- Stage-input attributes must be uniform width — `uint2 + uint`
  params don't compile; use all `uint3`.
- Threadgroup shape matters: with `(256,2,1)` the `t.x` lane index
  only spans 0–255 and a derived `tid >> 5` simdgroup index duplicates
  across the second row → structured-wrong results, no crash. For
  512-thread kernels dispatch `(512,1,1)`.
- `float3` in device buffers is 16-byte strided — pass `float4`. A
  12-byte-packed float3 array misaligns every vertex.
- A combined staging loop serving two buffers produced a
  threadgroup-buffer overflow that silently zeroed outputs — no crash,
  no error. Always re-derive indices per buffer.

### 7.3 Batching: the 16× that isn't about FLOPs

50 elementwise dispatches on 64 KiB working sets: sequential
61.33 ms → one command buffer with 50 encoders **3.80 ms = 16.1×**.
Per-kernel dispatch cost 1.23 → 0.076 ms. Rules: batch only ≥ 2
kernels (single-kernel batches lose to plain dispatch — encoder
overhead); sequential encoders execute in order, so in-place elementwise
ops between GEMMs are correct; cap jobs per buffer (512 here — the
safety cap from §8.5).

Fusion multiplies the win: a fused `silu_mul_f4` (SwiGLU gate·up in
one kernel) plus dual-B GEMM (`gemm2_f16`: C1=A@B1 and C2=A@B2 in one
dispatch, verified bit-identical three ways) took the seq-32 SwiGLU
block from 2.41 → **1.59 ms**.

**Async overlap**: `run_batch(wait=False)` + CPU prep + later wait —
one transformer step (rope → GEMM → silu → GEMM → residual) at
B=32/D=1024: GPU alone 0.91 ms, sequential CPU-then-GPU 2.78 ms,
overlapped **1.69 ms (1.75×)**. One real bug found here: attaching the
completion handler after `commit` trips a Metal assert — the async
path was silently dead until the audit round caught it.

### 7.4 Pipeline creation: 7.5 ms → 0.2 ms

Every pipeline built through the descriptor path registers in an
`MTLBinaryArchive` persisted to disk. The archive contains
**`applegpu` GPU machine code for this exact chip** — the driver
instantiates from pre-validated binaries instead of a fresh backend
compile. Cross-process effect: 37× faster pipeline creation on the hit
path. PyObjC traps (all cost a crash to find): the archive constructor
returns an (archive, error) **tuple** — passing the tuple to
`setBinaryArchives_` crashes the driver; the archive path needs one
no-archive descriptor creation first (AIR initialization race); the
selectors are `addComputePipelineFunctionsWithDescriptor_error_` and
`setUrl_`. Archive failures must be non-fatal — plain path is the
fallback.

### 7.5 Zero-copy memory

`newBufferWithBytesNoCopy` over page-aligned mmaps (16 KiB alignment,
first-fit free-list, immediate coalescing; sub-allocation as
(buffer, offset) pairs so reuse never reallocates). The same trick
exists on the render side: `newTextureWithDescriptor:offset:bytesPerRow:`
for zero-copy textures over arena memory. Rules: keep a content-addressed
cache of constant buffers (unbounded mmap churn left thousands of live
driver objects — bounded at 256 with a one-generation graveyard so
no-copy buffers never outlive their mmap); close arenas properly
(`np.frombuffer` views pin the mapping).

### 7.6 The GPU governor

Custom GPU work shares the chip with WindowServer — there is no fault
sandbox. The **GPUGovernor** caps self-inflicted GPU occupancy at 98%
(default): every completed command buffer reports true GPU time via
GPUStartTime/GPUEndTime (verified live, not wall-clock), the governor
sums busy seconds per 100 ms window and sleeps out the window when the
cap trips. Verified with cap=0.20: 100 consecutive GEMMs held at 22.5%
occupancy. This is what makes sustained custom-GPU work survivable on
a daily-driver machine.

## 8. Measurement methodology

*The most transferable section of this manual. Every finding below was
learned by being wrong first.*

### 8.1 The quiet gate

A concurrent session once distorted timings **14×** (an MMA kernel
went 1.08 ms quiet → 15.28 ms contended). Every benchmark run is
gated: loadavg ≤ 2.0, no process holding > 50% CPU, governor not
pulsing. GPU work additionally passes a hard admission gate (refused
at loadavg1 > 4, loadavg15 > 8, or any process > 200% CPU), and
benchmark processes drop themselves to QoS UTILITY so interactive
work always wins scheduling.

### 8.2 Thermal variance is real

Consecutive ANE sweeps vary **±15–20%** thermally; ordering stays
stable. Report ranges and orderings, not single numbers. Machine
"contended" numbers are floors, not results.

### 8.3 GPU timestamps, not wall clock

Time kernels with the command buffer's own GPUStartTime/GPUEndTime.
Interleave A/B variants to cancel machine drift (this is how the
simdgroup-vs-scalar decode comparison was settled). Wall clock
includes dispatch, contention, and the governor's sleep — it once
reported "GPU busy 1.07 of 1.20 ms wall" which is how the real
bottleneck (skinny GEMMs, not dispatch) was found.

### 8.4 No perf number without its gate

A quick test that skipped the dims buffer produced "54 TF/s" — both
kernels read garbage M/N/K, and identical garbage on both sides makes a
fake bit-exact match. The correctness gates caught it; the real number
was 5.16. Every benchmark in the workspace requires bit-exactness or a
measured rel-err against a reference **in the same run** as the timing.

### 8.5 Safety rules (from two real incidents)

1. **2026-09-22, machine force-reset.** An operator-precedence bug
   (`+ 1 << 18` shifts the whole sum) built a 9.1 TiB arena; mmap
   overcommit accepted it silently and the GPU got a 9 TiB virtual
   range. Combined with an unbounded stress kernel (values doubling to
   inf) and a concurrent session, WindowServer hung on the GPU and the
   watchdog rebooted the machine. Fixes: parentheses + a hard 64 GiB
   arena cap; stress kernels must keep values bounded by construction;
   never run concurrent heavy GPU sessions; if a dispatch hangs, STOP —
   do not submit more work.
2. **2026-09-23, WindowServer watchdog, no reboot.** Total system
   saturation (our GPU submissions + a 402%-CPU concurrent process).
   Fixes: the admission gate, QoS yielding, in-loop re-checks every
   25–50 iterations with 10 s pauses.

The meta-rule: on Apple Silicon, GPU misbehavior is a system-level
event. Treat the GPU like shared infrastructure, because it is.

## 9. ANE optimization: the CoreML contract

The ANE is reachable today only through CoreML's compile path (§5).
Within that contract, the measured optimization surface:

### 9.1 Placement: let the planner decide, then verify

Compute units `CPU_AND_NE` + **MLComputePlan verification**
(MLNeuralEngineComputeDevice) is the honest way to claim "ran on the
ANE" — without it, "ANE execution" is a guess. The planner's
threshold [MEASURED]: below ~seq 128–512 it routes to the **GPU**
(launch-latency dominated); above, to the **ANE** (compute dense).
Short prompts are latency-bound; long ones are throughput-bound; the
stack knows the difference.

### 9.2 The batching knee [MEASURED — full sweep]

fp16 FFN 768→inter→768 via coremltools 9, CPU_AND_NE, 10 s/shape,
with live ANE watts sampled from IOReport:

| seq | inter | tok/s | ANE W | tok/J |
|---:|---:|---:|---:|---:|
| 128 | 3072 | ~536k | 3.4 | ~158k |
| 256 | 3072 | 518–778k | 3.1–4.8 | ~165k |
| 512 | 3072 | ~810k | 4.9 | ~167k |
| 1024 | 3072 | **809–1040k** | 4.3–6.3 | ~166–189k |
| any | 6144 | 166–358k | 2.0–5.1 | 71–83k |

Three findings:

1. **Batching knee at seq 512–1024**: per-predict overhead amortizes
   to ~0.8–1.0M tok/s (1.5–1.9× the seq-128 baseline). Batch to the
   knee, not beyond.
2. **Efficiency is flat** across seq at inter 3072 (~160k tok/J).
   Batching buys throughput, not efficiency — pick based on whether
   you need tok/s or tok/J.
3. **Shape matters more than size**: inter 6144 loses on *both* axes
   at every seq — the wider FFN falls off an ANE fast path. 768×3072-
   class FFNs batch well; 6144-wide ones don't. When porting a model,
   this is the difference between 1M and 250k tok/s.

### 9.3 Precision contract

The ANE is a full-fp16 engine: rel err ~**2e-2** vs fp32 reference on
FFN blocks [MEASURED]. A custom GPU path with fp32 accumulation is
**100× tighter** (~2.1e-4). This is *the* routing criterion when
numbers matter: ANE for throughput, GPU for precision. (The old MPS
bridge path is ~2e-6 but 5–24× slower than everything else.)

### 9.4 The head-to-head table [MEASURED, same weights, same input]

SwiGLU FFN, H=1024 I=4096, seq=32 / seq=128:

| backend | seq=32 | seq=128 | rel err |
|---|---:|---:|---:|
| **ANE via CoreML** | **0.30 ms** | **0.39 ms** | 2e-2 |
| custom GPU (batched, fused) | 0.62 ms | — | 2.1e-4 |
| CoreML CPU_AND_GPU | 0.65 ms | 1.08 ms | 4e-4 |
| torch CPU | 1.34 ms | 2.92 ms | 1e-7 |
| Accelerate/numpy | 2.41 ms | 4.37 ms | 0 |
| MPS bridge | 7.89 ms | 12.10 ms | 2e-6 |

At larger blocks (H=3072, I=8192) the ANE pulls further ahead
(1.25 ms vs 4.00 ms custom GPU). The ANE is the FFN engine at model
block shapes — 2× Apple's own GPU path, 24× the old MPS kernel.

### 9.5 What is blocked, and the honest remaining routes

Blocked on a stock system [VERIFIED empirically]: direct H11ANEIn
submission (entitlement is launchd-granted to aned/aneuserd only — a
correct ctypes open attempt returns `kIOReturnNotPrivileged`), CSNE
wire access, register-level anything. Live XPC to aned works (proven
round-trip) but stops at the same wall. The remaining routes, in
order of promise: **Espresso** (the placement engine below CoreML —
decoded as the next analysis target), the **VM paravirtualization
route** (§5.6 — host side is not entitlement-gated), and full firmware
disassembly (§14).

## 10. Hybrid serving architecture

Everything above compresses into one placement table:

| Phase / workload | Best engine | Why |
|---|---|---|
| Long-prompt prefill | ANE (seq > ~512) or GPU | compute-bound; both engines excel |
| Short-prompt / small batch | GPU (below ANE threshold) | launch latency dominates |
| FFN blocks at decode batch | ANE | 3–7× faster than any alternative |
| Precision-sensitive compute | custom GPU path | 100× tighter than fp16 ANE |
| Large-M GEMM | custom GPU (mma4) | 4.2–5.2 TF/s |
| Decode of large models | memory bus | the ~51 tok/s wall (§3) is physics |
| CPU | orchestration, prep, overlap | pool rules in §12 |

The decode wall deserves emphasis: hybrid architectures move *which
engine* streams the weights, not whether they stream. For a 6 GB model
the ceiling is the bus. Every serious local-LLM benchmark should be
checked against §3's arithmetic before being believed.

## 11. Memory: arenas, streaming, RSS discipline

**Arena pattern** (one mmap per lifetime class, zero-copy buffers,
first-fit + coalescing, `stat()/rss()/trim()/reset()/close()`), plus
the results that matter:

- **Trimming works as Darwin documents**: free ranges marked
  `MADV_FREE_REUSABLE` get reclaimed under system pressure without
  swapping — verified live: 384 MiB arena → 1 MiB resident under 14 GiB
  pressure, live allocations and GPU writes unaffected; `alloc()`
  re-marks `MADV_FREE_REUSE` before reuse. (`MS_KILLPAGES` is
  documented but empirically inert.)
- **Out-of-core streaming GEMM**: chunk-major virtual arena
  (overcommit), fill → consume → drop per chunk, C += A@B accumulated.
  8 GiB virtual / bounded RSS: after streaming 2 GiB, a 24 GiB
  pressure subprocess took the process **2168 → 108 MiB RSS in 3 s**,
  and the arena stayed functional (post-reclaim dispatch bit-exact).
  Pages are dropped, never swapped. Cost: streamed throughput 0.69
  TF/s vs 2.39 in-core — per-chunk dispatch overhead, amortizable with
  bigger chunks or batched async dispatch.
- fp16 tiles halve activation/weight footprint with fp32 accumulate
  — the "less RAM hungry" model path.

## 12. CPU optimization

- **Pin threads to one L2 domain** for latency-bound multithreaded
  work (three domains on the 15-core part — §2). Doable from ordinary
  userspace.
- **Don't hand-roll GEMM pools**: Accelerate BLAS self-parallelizes
  each matmul across cores; a 16-way pool map of matmuls measured
  **0.43×** inline. Pools are for orchestration (CPU prep overlapped
  with GPU batches — the 1.75× of §7.3), not throughput.
- **Pool rules that keep the machine usable**: QoS USER_INITIATED
  (never USER_INTERACTIVE — that's WindowServer's class), no
  busy-spinning (workers block on a queue, exit after 60 s idle), max
  workers = `hw.perflevel0.logicalcpu` capped at 8.
- **SME is sitting there.** ARM's matrix extension, exposed, almost
  unused by public software. The open invitation of this generation
  (§14).

---

# PART III — RESEARCH FRONTIER

## 13. The extraction toolchain

Modern macOS hides most code in two containers; serious RE starts by
carving them.

- **Kernel collections**: the real boot collection is
  `/var/db/KernelExtensionManagement/KernelCollections/BootKernelCollection.kc`
  (~115 MB, world-readable, ~365 arm64e kexts, load addresses matching
  kextstat). The `/System/Library/KernelCollections/*.kc` files are
  **x86_64 decoys** containing none of the loaded kexts; most
  `/System/Library/Extensions` bundles are plist-only stubs. (Verify
  both yourself — it's strange but true.)
- **Dyld shared caches**: ~2,230 private frameworks exist almost
  entirely inside multi-GB caches in the OS cryptex. Carving requires
  walking the image table and rebuilding compacted symbol tables.
- **Firmware**: `/System/Library/Firmware/` is empty; the real
  coprocessor images live under
  `/System/Volumes/Preboot/<UUID>/restore-staged/Firmware/`, one
  directory per engine, world-readable, IM4P containers — and (the
  finding that reorders the research agenda) **unencrypted** for the
  ANE/SEP-class images: no KBAG, no per-device GID. They have been
  analyzable all along.

The workspace ships working extractors for all three (`kc_extract`,
`dsc_extract.py`, the IM4P parser in `ane_re/`), a `.hwx`
compiled-program parser (0xBEEFFACE Mach-O variant, fixed-VM planes,
named tensor buffers via custom load command 0x40), a
ParavirtualizedANE protocol resolver, and an SMC ABI reference. They
generalize to every modern Apple platform.

## 14. Open problems — where the frontier is

1. **Disassemble the ANE firmware.** Unencrypted arm64 Mach-O, 95
   CSNE commands (~60 undocumented), PENA format, TD/BAR semantics —
   nothing but labor stands in the way. Whoever finishes it becomes
   the reference source for ANE internals. The ~17-variant
   cross-generation corpus makes diffing the fast path.
2. **The 0x948-byte request struct** for ProgramSendRequest — its
   parser hides behind two PAC-obfuscated indirect calls. Static
   analysis plus the VirtIO client's construction sites is the
   likely route.
3. **A VirtIO guest driver for the host ANE** — the paravirtualized
   protocol is fully specified (§5.6); the host side is not
   entitlement-gated. The most promising no-entitlement path to
   custom ANE work.
4. **Espresso** — the placement engine beneath CoreML. Decoded as the
   next target after the aned XPC route closed.
5. **A real GPU profiler** — `AGXGPURawCounterBundle` is located but
   unread. First hardware-level Apple-Silicon GPU profiler outside
   Apple.
6. **SME kernels** — present, exposed, unused. The free lunch.
7. **Bandwidth measurement** — a plain Metal memcpy sweep to
   characterize achieved-vs-spec bandwidth was never completed; the
   248 GB/s triad number is the only measured anchor.
8. **ANE cache-lease lifecycle** — the IDL's
   `requestCacheRequestId`/`triggerCacheRequest` flow during cached
   vs uncached runs; os_log emits no exclave events at default
   verbosity, so this needs kernel-side tracing.

## 15. Naming traps and folklore corrections

1. **t-codes name IP blocks, not chips.** `ane,t8132exclave` means
   the ANE block carries its grandfather's name; the SoC is t6050.
2. **h17 (architecture) vs H16 (driver interface) vs "AppleH16AneFW"
   (firmware name)** — three layers, all correct, all different.
3. **"Most code has no file."** Private frameworks live only in the
   shared caches; kexts only in the boot kernel collection; the
   /System copies are stubs or decoys.
4. **Firmware is not where you think it is** (§13).
5. **42 TOPS is an estimate.** Apple publishes no ANE figure.
6. **"kp=16777, ki=480" mixes two different CLPC limiters.**
7. **TXM does not run at EL3.** It sits under SPTM's umbrella.
8. **The "ggml" in `com.apple.ggml.E5MLCompiler` is Apple-internal
   naming**, unrelated to llama.cpp's tensor format.
9. **No SVE2 flags** — Apple exposes SME, not standalone SVE2.
10. **LLMCache caches Apple Intelligence planner/generation outputs**,
    not ChatGPT responses.
11. **The KV-cache "1–2 GB" figure for 3B models** circulating in
    write-ups is off by 10× (§3).
12. **"Custom ANE kernels" on paper ≠ kernels that executed.** No
    hand-written kernel has run on a modern ANE outside Apple.
13. **ane-type is 0x220 (544) little-endian**, not 0x0200.
14. **securem3fw is the SecureM3/Touch ID image, not SEP firmware.**

## 16. Probe cheat sheet

```bash
# topology
ioreg -l | grep -E "perflevel|CPU cores"      # core types
sysctl hw.perflevel0.logicalcpu               # Super-core count
sysctl hw.optional                            # ISA flags (SME, no SVE)
sysctl vm.mte                                 # MTE in production

# GPU
ioreg -r -d 1 -c AGXAccelerator | grep -A20 GPUConfigurationVariable

# ANE
ioreg -r -d 1 -c AppleH16ANEInterface         # 16 cores, h17, MMIO
ls /var/db/KernelExtensionManagement/KernelCollections/   # real .kc

# thermal/power (no sudo): IOReport "Energy Model" channels,
#   or the dashboard's IOReport reader

# the honest gates for any benchmark:
#   quiet system, GPU-timestamped, interleaved A/B,
#   bit-exactness verified in the same run
```

---

*Sources: live probes of this machine (M5 Pro, Mac17,9, macOS 26.5,
firmware 18000.120.36); the open-source h11 ANE Linux driver (public
lineage for selector numbering and tile size); Apple's Platform
Security Guide and spec pages; Apple's published PCC design. Every
other number in this manual is original measurement. Confidence tags
are the contract: [VERIFIED] > [MEASURED] > [ESTIMATE] > [BELIEF].*

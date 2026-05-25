<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# RTOS Benchmark Architecture Principles

**Version:** 1.1
**Date:** May 2026
**Authors:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Origin:** Derived from unified POSIX-compliant RTOS benchmark
           framework — EW2026 validated results, Layer 1 complete
**Scope:** Any RTOS performance benchmark targeting mixed-criticality
           safety applications (automotive, robotics, Physics AI)
**Companion documents:**
- `METHODOLOGY.md` — AI-Assisted Embedded Systems Engineering Methodology
- `KNOWLEDGE_REGISTRY.md` — Domain router and maintenance protocol

---

## Overview

This document captures structural design principles for RTOS performance
benchmarking in safety-critical mixed-criticality contexts. The principles
apply to any benchmark framework targeting WCET evidence production for
functional safety certification (ISO 26262, IEC 61508, DO-178C) or
safety-adjacent applications (robotics, Physics AI with safety requirements).

The document is organised into six categorical domains. Every design
decision in a safety-focused RTOS benchmark framework falls into one of
these domains. When new principles are discovered on future projects,
they slot into the appropriate domain rather than being appended to a
flat list.

### The Unifying Meta-Principle

> A benchmark that cannot bound WCET under adversarial conditions
> produces no safety evidence. Every design decision — metric selection,
> timing methodology, stressor design, statistical treatment, platform
> configuration — must be evaluated against whether it strengthens or
> weakens the WCET bound.

Median performance is interesting. WCET under adversarial stressor
conditions is the only number a safety engineer can use.

### Self-Extension Mechanism

When a new principle is discovered — through a measurement anomaly,
a reviewer challenge, a certification authority question, or a new
platform bring-up — add it as follows:

1. **Identify:** which domain owns this finding?
2. **Generalise:** what is the underlying principle beyond the specific instance?
3. **Number:** assign the next available ID in the domain (A4, B5, C3...)
4. **State:** rule, rationale, anti-pattern, verification method
5. **Update:** KNOWLEDGE_REGISTRY.md domain activation keywords if needed

---

## The Four-Layer Architecture

The benchmark framework is structured as four measurement layers.
Each layer adds measurement complexity above the layer below, which
serves as a calibrated baseline reference. This ordering is mandatory —
a layer cannot produce defensible results without the layer below
being validated first.

```
Layer 1 — Bare-metal RTOS baseline
          OS primitive WCET under stressor conditions
          Establishes: scheduler latency, IPC overhead, IRQ path,
                       synchronisation primitives
          Reference platform: RPi4B (BCM2711 / Cortex-A72)
          Validated: AGL RT-PREEMPT 6.12, Microkernel RTOS

Layer 2 — Hypervisor / VirtIO overhead
          Virtualisation tax = Layer 2 measurement - Layer 1 baseline
          Adds: VM exit latency, inter-VM IPC, VirtIO peripheral overhead,
                hypervisor scheduler interaction
          Requires: Layer 1 validated on same or equivalent hardware first

Layer 3 — Middleware latency
          Middleware overhead = Layer 3 measurement - Layer 2 baseline
          Adds: ROS 2 executor jitter, DDS serialisation, QoS policy
                interaction with RT scheduler
          Requires: Layer 2 validated on same hardware first

Layer 4 — AMP IPC + AI inference
          End-to-end latency = OS + hypervisor + middleware + inference
          Adds: OpenAMP RPMsg, cross-core IPC, AI model inference
                latency, micro-ROS integration
          Requires: Layer 3 validated on same hardware first
```

**The layer dependency rule:** Never start Layer N+1 measurement design
before Layer N results are validated and signed off by the architect.
A Layer 2 number that does not reference a validated Layer 1 baseline
on the same hardware cannot be interpreted as a virtualisation tax —
it is an unattributed latency measurement.

---

## Domain A — Metric Selection and Definition

*What to measure and why each metric is safety-relevant.*

### A1 — Measure OS Primitives, Not Application Throughput

**Rule:** Benchmark the OS scheduling and IPC primitives that directly
bound application temporal behaviour. Do not benchmark application
throughput, data processing rate, or network bandwidth — these are
derived from OS primitive behaviour and do not directly inform WCET.

**The five Layer 1 primitives and their safety relevance:**

| Metric | What it measures | Safety relevance |
|---|---|---|
| Task Switching Latency | Scheduler + context switch + thread activation | Bounds response time of any preemptable task |
| Task Preemption Latency | High-priority task activation via semaphore/IPC | Bounds worst-case reaction time for safety tasks |
| IRQ Handling Latency | Interrupt assert → ISR entry | Bounds sensor event response for ADAS/robotics |
| Mutex Shuffling Latency | Lock/unlock under priority contention | Bounds priority inversion exposure |
| Intertask Messaging Latency (IMLat) | OS message primitive end-to-end | Bounds inter-component communication in safety architecture |

**Anti-pattern:** Benchmarking a single metric in isolation and claiming
WCET coverage. The interference channels are different for each metric —
LLC eviction dominates IRQ latency; scheduler non-preemptibility dominates
task switching. A complete evidence set requires all five metrics under
all stressor conditions.

**Verification:** Can each metric be directly traced to a temporal
requirement in the target safety standard? If not, justify its inclusion
or remove it.

---

### A2 — IMLat Metric Definition

**Rule:** Intertask Messaging Latency (IMLat) measures the end-to-end
latency of one complete OS message-passing primitive round trip between
two threads at different priority levels. It is NOT application-layer
messaging latency.

**Precise definition:**
```
T0: sender thread timestamps before OS send primitive call
T1: receiver thread timestamps immediately after OS receive
    primitive returns
IMLat = T1 - T0
```

**Platform mapping:**
- AGL RT-PREEMPT: POSIX pipe (write/read)
- Microkernel RTOS: native OS IPC primitive (lower overhead than POSIX pipe)

**Why platform-native on microkernel RTOS:** POSIX pipe on a microkernel
RTOS adds an unnecessary abstraction layer that inflates latency relative
to the native IPC mechanism. The native primitive is what a
safety-critical application on that platform uses. Benchmark what the
application uses.

**Anti-pattern:** Using a socket, UNIX domain socket, or message queue
as the IMLat primitive when a lower-overhead native primitive exists.
The benchmark then measures socket overhead, not IPC overhead.

**Verification:** Confirm the primitive under test is the one a
safety-critical application in this vertical would actually use.

---

### A3 — Metric Completeness Gate

**Rule:** A Layer N benchmark is not complete until all defined metrics
for that layer have validated results under all defined stressor
conditions. Partial metric sets cannot support a safety claim.

**Layer 1 completeness definition:**
- 5 metrics × 4 stressor conditions (idle, CPU, memory, I/O) = 20 scenarios
- Minimum 100K samples per scenario (500K preferred)
- Both target platforms measured under identical hardware configuration
- PMU attribution completed for at least the highest-WCET metric

**Anti-pattern:** Declaring Layer 1 complete with 3 of 5 metrics,
then starting Layer 2 design. The missing metrics may have different
dominant interference channels that change the Layer 2 measurement
strategy.

---

## Domain B — Timing Methodology

*How to measure time correctly for safety-defensible results.*

### B1 — Clock Selection is a Safety-Critical Decision

**Rule:** The timing clock must be selected based on what timestamps
are being paired. Using mismatched clock domains invalidates the
latency measurement entirely — the error is silent and may be orders
of magnitude larger than the signal.

**Clock selection matrix:**

| Scenario | Correct clock | Reason |
|---|---|---|
| Both timestamps in user-space | `CLOCK_MONOTONIC_RAW` | No NTP slew, immune to adjtime() |
| T0 user-space, T1 kernel GPIO event | `CLOCK_MONOTONIC` | event.timestamp_ns is CLOCK_BOOTTIME domain on BCM2711/6.12; BOOTTIME = MONOTONIC on non-suspended system |
| Microkernel RTOS both timestamps | platform cycle counter API | Safe wrapper with correct frequency from system page; verify API for specific RTOS |
| Cross-platform comparison | Same physical counter (cntvct_el0) | Eliminates clock domain as variable |

**The BCM2711 verified finding:**
`gpio_v2_line_event.timestamp_ns` is in CLOCK_BOOTTIME domain on
BCM2711 / kernel 6.12. CLOCK_MONOTONIC_RAW diverges by ~61 ms due to
NTP slew on RPi4. Using CLOCK_MONOTONIC_RAW for T0 when T1 is a kernel
GPIO event produces a ~61 ms systematic error that drowns the signal.
Verified by `tools/gpio_clock_probe.c`.

**Anti-pattern:** Assuming CLOCK_MONOTONIC_RAW is always the most
accurate choice. On kernels with NTP correction, it diverges from
kernel event timestamps. On platforms with variable CPU frequency,
cycle counters without fixed-frequency guarantees are wrong.

**Verification — mandatory before any new platform bring-up:**
```bash
# Build and run gpio_clock_probe on target before writing any benchmark
gcc -O2 -o gpio_clock_probe tools/gpio_clock_probe.c
sudo ./gpio_clock_probe /dev/gpiochip0 17 27
# Verdict must confirm clock domain before proceeding
```

---

### B2 — The IRQ Latency Two-Thread Requirement

**Rule:** IRQ handling latency on Linux chardev GPIO must use a
two-thread design with the waiter thread at strictly higher priority
than the trigger thread. A single-thread design always measures
ioctl tail latency (~500 ns), not interrupt latency.

**Root cause:** `gpio_set_high()` via ioctl fires the GPIO edge
synchronously inside the kernel path before the ioctl returns to
user-space. A single thread calling poll() after the ioctl always
finds the event pre-queued.

**Correct architecture:**
```
Waiter (priority P+1):
  write(ready_pipe) → poll(gpio_fd) → blocks

Trigger (priority P):
  read(ready_pipe) → T0 = clock_gettime() → gpio_set_high() ioctl
  [edge fires → IRQ → kernel queues event on gpio_fd]

Waiter (unblocked by poll):
  T1 = clock_gettime() → write(result_pipe, T1)

Trigger:
  read(result_pipe) → Latency = T1 - T0
```

**Priority ordering is the synchronisation mechanism:** Under SCHED_FIFO,
after the waiter writes to ready_pipe, it cannot be preempted by the
trigger (lower priority) — it runs straight into poll() and blocks
before the trigger executes. This guarantee is SCHED_FIFO-specific.

**Shutdown:** Close ready_pipe[0]; waiter's poll() watches this fd
with events=0 and returns immediately on POLLHUP.

**Anti-pattern:** Adding a sleep() or yield() between ioctl and poll()
to "give the event time to arrive." This masks the design error and
produces inconsistent results depending on scheduler timing.

**Verification:**
```bash
# Single-thread design symptom: IRQ latency median ~0.5 μs
# Correct two-thread result: IRQ latency median ~12-14 μs on BCM2711
# If median < 1 μs, the design is wrong
```

---

### B3 — Hot Path Discipline

**Rule:** The measurement hot path must contain zero I/O, zero dynamic
allocation, and zero non-deterministic system calls. Any such call in
the hot path inflates and destabilises the measurement.

**Hot path = the code between T0 and T1 timestamps.**

**Prohibited in hot path:**
- `printf`, `fprintf`, `write` to any fd
- `malloc`, `calloc`, `realloc`
- `open`, `close`, `ioctl` (except the primitive under test)
- `clock_gettime` calls other than T0 and T1
- Any RTOS call not being benchmarked

**Required before hot path enters:**
- `mlockall(MCL_CURRENT | MCL_FUTURE)` — prevents page faults
- Pre-fault sample buffer: write to every element before timing
- Warmup iterations: minimum 1000 iterations discarded before measurement
- CPU affinity: verified, not just set
- RT priority: verified via `pthread_getschedparam`

**Anti-pattern:** Calling `clock_gettime` inside the loop for progress
reporting. Each call adds ~50-200 ns of syscall overhead and pollutes
the sample distribution.

---

### B4 — Sample Count and Statistical Validity

**Rule:** Safety-defensible WCET requires sufficient samples to make
the tail distribution (P99.9, WCET) statistically meaningful.
Minimum 100K samples per scenario. 500K preferred for WCET claims.

**Rationale:** At 100K samples, P99.9 corresponds to the worst 100
observations — sufficient to identify systematic interference patterns.
At lower counts, P99.9 is dominated by sampling noise.

**Required statistics per scenario:**
- Minimum, Median, P99, P99.9, Maximum (WCET), Standard deviation
- Histogram (log-scale Y axis) — reveals bimodal distributions
  that aggregate statistics mask
- CSV export for offline analysis and certification artefact packaging

**Anti-pattern:** Reporting only median and maximum. Median hides
interference sensitivity. Maximum without distribution context is
unattributable — it may be a one-time anomaly or a systematic pattern.

**Bimodal distribution signal:** A bimodal histogram always indicates
two distinct operating modes. Common causes: CPU frequency scaling
(fix: performance governor), cache cold vs warm (fix: cache warm-up
or explicit cold-cache iteration design), two interrupt sources with
different latency profiles.

---

## Domain C — Stressor Design

*How to create adversarial conditions that bound WCET.*

### C1 — Stressors Must Map to Real Deployment Interference

**Rule:** Every stressor scenario must correspond to a realistic
co-tenant workload in the target deployment environment. Stressors
that do not map to real interference channels produce WCET numbers
that are either too optimistic (missing real interference) or
uninterpretable (interference has no deployment analogue).

**Validated stressor-to-deployment mapping (automotive / robotics):**

| Stressor | stress-ng config | Primary channel | Deployment analogue |
|---|---|---|---|
| CPU Stress | `-c N` (N = core count - 1) | Scheduler preemption storms, L1/L2 thrash | Parallel ADAS sensor fusion pipeline |
| Memory Pressure | `--vm N` | LLC eviction, TLB pressure | DNN inference co-tenant with large activation maps |
| Block I/O | `--hdd N` | AXI/PCIe bus contention, DMA interrupt storms | Sensor data streaming, camera buffer logging |
| Baseline (Idle) | none | OS-internal jitter only | Lower bound — establishes irreducible OS overhead |

**Anti-pattern:** Using a synthetic CPU-only stressor and claiming
the resulting WCET bounds IRQ latency under sensor streaming load.
LLC eviction from DMA-intensive I/O is the dominant IRQ latency
interference channel — a CPU stressor does not exercise it.

**Verification:** For each stressor, identify the primary interference
channel (PMU counter) it exercises. Confirm that channel is relevant
to the target deployment before including the stressor in the evidence set.

---

### C2 — Stressor Isolation and Configuration Discipline

**Rule:** Stressor configuration (number of workers, memory footprint,
I/O depth) must be documented exactly and reproducible. WCET numbers
without stressor configuration are not reproducible and not certifiable.

**Required stressor documentation per scenario:**
```
Stressor: stress-ng --vm 2 --vm-bytes 75%
Platform: AGL RT-PREEMPT 6.12 / BCM2711
CPU affinity: stress-ng on CPU0-2; benchmark on CPU3 (isolated)
Duration: co-running with benchmark for full sample collection
Samples: 500K
```

**Anti-pattern:** Running stressors at "maximum load" without
specifying worker count or memory footprint. Maximum load on a 4-core
system is a different interference level than on a 16-core system.
The number is platform-specific and meaningless without context.

---

### C3 — Baseline First, Stressor Second

**Rule:** Always establish idle baseline before running stressor
scenarios. The idle baseline is the reference against which interference
is attributed. Without it, stressor results cannot be interpreted as
interference deltas.

**Baseline requirements:**
- Identical hardware configuration to stressor runs
- Same sample count
- Same CPU isolation, affinity, and RT priority
- Measured immediately before or after stressor runs on the same
  hardware session (not a different day or boot)

---

## Domain D — PMU Attribution Methodology

*How to identify the root cause of WCET outliers.*

### D1 — Attribution is Mandatory for WCET Claims

**Rule:** A WCET number without attribution is a measurement, not
evidence. For a WCET claim to be defensible to a safety reviewer,
the dominant interference mechanism must be identified and its
relationship to the outlier latency must be quantified.

**Why this matters for certification:** ISO 26262 Part 6 requires
freedom from interference (FFI) evidence. Saying "WCET is 168 μs"
is insufficient. Saying "WCET is 168 μs; 99.1% of outliers are caused
by raw_spinlock in __schedule() which is non-preemptible by design in
RT-PREEMPT; countermeasure is XYZ" is a certifiable statement.

**Attribution methodology (validated on Layer 1):**

```
Step 1 — Threshold capture
  Flag samples > P99.9 as outlier candidates during benchmark run

Step 2 — PMU snapshot
  At each outlier event: capture ARM PMU counters
  LLC_MISS, L1D_MISS, DTLB_WALK, BUS_ACCESS, CPU_CYCLES, MEM_ACCESS

Step 3 — Enrichment ratio
  enrichment = mean(counter | outlier) / mean(counter | nominal)
  enrichment > 2× → candidate interference channel

Step 4 — Spearman correlation
  r = Spearman(latency_delta, counter_delta) across all outlier windows
  High enrichment + high r → causal candidate
  High enrichment + low r → co-occurrence, not causation

Step 5 — Ftrace non-preemptible path attribution
  For scheduler-dominated outliers: capture ftrace events during outlier
  Identify IRQ sources and kernel non-preemptible path events
  Compute per-source outlier vs nominal rate ratio
```

**Validated finding (AGL-RT task switching, 400K iterations):**
- LLC_MISS, L1D_MISS, DTLB_WALK, Branch: all <2× enrichment →
  hardware channels NOT primary cause
- __schedule() raw_spinlock: 99.1% of outlier events
- Residual: diffuse, irreducible, no single IRQ source dominant

**Full per-metric attribution (AGL-RT, validated — AGL AMM 2026):**

| Metric | Primary cause | HW PMU signal | Worst stressor | Countermeasure |
|---|---|---|---|---|
| TSLat | Kernel sw path (__schedule raw_spinlock) | Null <2× | CPU (87.2 μs) | None — irreducible by RT-PREEMPT design |
| PreLat | Kernel sw path (try_to_wake_up) | L2D co-occur only (r=0.066) | CPU (46.3 μs) | None — no config eliminates this path |
| MuLat | LLC eviction | L2D 32× genuine | I/O (62.6 μs) | MPAM / Cache Coloring + IRQ affinity |
| InLat | IPI displacement | Null — 1.0× all | I/O (72.6 μs) | GIC IPI routing |
| IMLat | LLC eviction | L2D inf× all conditions | I/O (108.9 μs) | MPAM / Cache Coloring |

**Key architectural implication:** TSLat and PreLat have no software
countermeasure — their WCET floor is set by the RT-PREEMPT kernel design.
MuLat, InLat, and IMLat have hardware countermeasures (MPAM, GIC routing)
that can reduce WCET. This distinction is critical for safety case authoring.

**Anti-pattern:** Reporting PMU counter values without enrichment ratio
or correlation coefficient. High counter values during outliers may
simply reflect that more work was done — the ratio to nominal is
what identifies interference.

---

### D2 — PMU Counter Selection per Metric

**Rule:** Select PMU counters based on the interference channels
relevant to the metric under test. Capturing all counters for all
metrics wastes overhead budget and obscures attribution.

**Counter-to-metric mapping:**

| Metric | Primary PMU candidates | Rationale |
|---|---|---|
| Task Switching | CPU_CYCLES, L1D_MISS, LLC_MISS | Scheduler path, cache state on context restore |
| Task Preemption | DTLB_WALK, LLC_MISS | Large-footprint co-tenant TLB eviction |
| IRQ Handling | LLC_MISS, BUS_ACCESS, MEM_ACCESS | DMA buffer eviction, AXI contention |
| Mutex Shuffling | L1D_MISS, LLC_MISS | Lock word cache line contention |
| IMLat | CPU_CYCLES, BUS_ACCESS | IPC buffer transfer, scheduler path |

**PMU overhead budget:** ARM PMU counter capture in the hot path adds
~50-100 ns per event read. Enable PMU capture only during outlier
windows (threshold-triggered), not on every sample.

---

### D3 — Ftrace Attribution for Scheduler-Dominated Outliers

**Rule:** When PMU attribution identifies the scheduler path as the
dominant interference channel, ftrace non-preemptible path tracing
is mandatory to identify the specific kernel code path causing the
outlier. PMU alone cannot distinguish between different scheduler
code paths.

**Ftrace configuration for RT-PREEMPT attribution:**
```bash
# Isolate target CPU maximally before ftrace capture
# /proc/cmdline: isolcpus=3 nohz_full=3 rcu_nocbs=3

# Capture non-preemptible path events
echo 1 > /sys/kernel/debug/tracing/events/preemptirq/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# [run benchmark]
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**Interpretation:** IRQ source rate ratio (outlier / nominal) > 2×
indicates candidate interference. n=1 or n<10 outlier events for a
given source require caution — may be statistical noise.

---

## Domain E — Platform Configuration and Reproducibility

*How to establish a controlled measurement environment.*

### E1 — Platform Configuration is Part of the Measurement

**Rule:** WCET numbers are only valid for the exact platform
configuration under which they were measured. Any change to CPU
isolation, frequency governor, RT throttling, clock tick period,
or memory locking invalidates the measurement.

**Required configuration documentation per result:**

```
Hardware:     RPi4B, BCM2711, Cortex-A72 @ 1.5 GHz
OS/Kernel:    AGL RT-PREEMPT Linux 6.12 / Microkernel RTOS
CPU affinity: Benchmark on CPU3; stressors on CPU0-2
Isolation:    isolcpus=3 rcu_nocbs=3
              NOTE: nohz_full intentionally omitted —
              introduces syscall overhead incompatible with
              context-switch benchmarking
Governor:     performance (cpufreq-set -c 3 -g performance)
Idle state:   WFI only — deeper C-states disabled
              (C2/C3 add several ms wake latency)
RT throttle:  disabled (echo -1 > /proc/sys/kernel/sched_rt_runtime_us)
IRQ affinity: all non-benchmark IRQs steered to CPU0-2
              (echo 7 > /proc/irq/*/smp_affinity)
              benchmark IRQ (GPIO145) explicitly pinned back to CPU3
RT priority:  97 (Linux) / equivalent high priority on microkernel RTOS
Clock tick:   platform-specific — verify minimum tick period per RTOS docs
mlockall:     MCL_CURRENT | MCL_FUTURE
Pre-fault:    sample buffer fully written before timing
Warmup:       1000 iterations discarded
```

**Anti-pattern:** Reporting WCET without stating whether RT throttling
was disabled. With RT throttling enabled, the kernel intentionally
adds latency to RT tasks — the measurement reflects a policy decision,
not OS primitive overhead.

---

### E2 — Cross-Platform Comparison Requires Identical Configuration

**Rule:** Comparing WCET between platforms (e.g. AGL-RT vs a microkernel
RTOS) is only valid when hardware configuration is identical. CPU affinity,
isolation, sample count, stressor configuration, and the metric
definition must be the same on both platforms.

**What "identical" requires for cross-platform comparison:**
- Same physical board and SoC
- Same CPU core for benchmark thread
- Same stressor configuration (same stress-ng workers and memory)
- Same sample count per scenario
- Same metric definition (T0 and T1 timestamps at equivalent points)

**Priority mapping across platforms:** Different RTOS use different
priority scales. Document the mapping explicitly so equivalent
scheduling headroom is maintained across platforms.

**Anti-pattern:** Comparing AGL-RT results on a 4-core board with
microkernel RTOS results on an 8-core board and attributing the
difference to the OS rather than to the hardware.

---

### E3 — New Platform Bring-Up Requires Full Layer 1 Revalidation

**Rule:** When the benchmark framework is ported to a new hardware
platform (e.g. RPi4B → Sparrow Hawk A76), Layer 1 must be fully
revalidated before Layer 2 measurements begin. Results from RPi4B
cannot be used as the Layer 1 baseline for Sparrow Hawk Layer 2 numbers.

**Revalidation checklist for new hardware:**
```
□ gpio_clock_probe confirms clock domain for kernel event timestamps
□ Idle baseline established for all 5 metrics
□ CPU isolation and governor configuration verified
□ RTOS clock tick period verified per platform documentation
□ All 5 metrics × 4 stressors measured
□ Architect signs off on Layer 1 results before Layer 2 begins
```

**Rationale:** Cortex-A76 (Sparrow Hawk) has different cache geometry,
branch predictor, and memory subsystem than Cortex-A72 (RPi4B). The
Layer 1 interference channel profile will differ. The Layer 2
virtualisation tax must be computed as (Layer 2 A76) - (Layer 1 A76),
not (Layer 2 A76) - (Layer 1 A72).

---

## Domain F — AI-Assisted Generation Constraints

*Specific constraints when using AI to generate benchmark code,
methodology, or analysis.*

### F1 — Platform API Verification Before Any Code Generation

**Rule:** Before generating benchmark code for any platform, the
relevant platform API documentation or source must be in the AI
context. AI training data for proprietary RTOS internals, RT-PREEMPT
kernel paths, and ARM PMU interfaces is sparse and may reflect older
versions. Generated code based on training data alone may use
non-existent APIs, wrong function signatures, or deprecated interfaces.

**Files to provide before code generation sessions:**

For any proprietary RTOS target:
- Relevant SDK header files for timing, threading, and IPC APIs
- Existing validated benchmark source for reference patterns

For Linux RT:
- Existing irq_latency.c as reference for two-thread design
- Kernel version confirmation (uname -r output)

For PMU:
- ARM Architecture Reference Manual PMU chapter reference
- Existing PMU capture code if any

**Anti-pattern:** Asking the AI to "implement ARM PMU capture for
Cortex-A72" without providing the specific counter names and their
event numbers. The AI will generate plausible counter names that may
not exist on this specific microarchitecture.

---

### F2 — Timing Methodology Is Not AI-Generatable Without Verification

**Rule:** The AI must not propose a timing methodology (clock selection,
T0/T1 placement, IRQ measurement architecture) without the human
architect confirming correctness on the actual hardware before
implementation. Timing methodology errors are silent — the code
compiles and runs, producing plausible but wrong numbers.

**The IRQ latency single-thread error is the canonical example.**
An AI asked to implement IRQ latency measurement will produce a
single-thread design (poll after ioctl) that compiles, runs, and
returns ~500 ns median — plausible, wrong, undetectable without
knowing the correct answer.

**Mandatory gate:** For any new metric or new platform, the timing
methodology must be reviewed and confirmed by the architect before
Claude Code implements it. The methodology goes in PROJECT_CONTEXT.md
first. Claude Code implements from spec, not from its own design.

---

### F3 — WCET Claims Require Human Sign-Off

**Rule:** The AI must not generate WCET claims, certification
statements, or safety evidence summaries without explicit human
architect review and confirmation of the underlying measurement data.

**What the AI can do:**
- Generate analysis code (histogram plotting, PMU correlation scripts)
- Format result tables from provided data
- Draft attribution narratives from provided PMU/ftrace data
- Suggest stressor configurations based on deployment context

**What requires human sign-off before publication:**
- WCET numbers cited in any document
- Attribution statements ("99.1% of outliers are caused by...")
- Any statement connecting benchmark results to certification claims
- Comparison statements between platforms

**Rationale:** AI can compute statistics and identify patterns from
data. It cannot verify that the measurement methodology was correctly
executed on the hardware. The human who ran the benchmark and read
the results is the only qualified signatory for the numbers.

---

### F4 — Apply Domain Checklist Before Generating Any Benchmark Code

**Rule:** Before generating any benchmark implementation, verify:

```
Domain A — Metric definition:
□ Is the metric precisely defined (T0, T1, what is being measured)?
□ Is the platform primitive correct for this metric and OS?
□ Is the IMLat primitive the one a safety application would use?

Domain B — Timing methodology:
□ Has clock domain been verified on this hardware (gpio_clock_probe)?
□ For IRQ latency: is the two-thread design specified?
□ Is the hot path free of I/O, allocation, and non-essential syscalls?
□ Is mlockall and pre-fault specified in _init()?
□ Is warmup iteration count specified?

Domain C — Stressor design:
□ Is the stressor mapped to a real deployment interference channel?
□ Is stressor configuration (worker count, memory) specified exactly?
□ Is baseline (idle) scenario included?

Domain D — Attribution:
□ Are PMU counters selected for this specific metric?
□ Is outlier threshold defined (P99.9)?
□ Is ftrace plan specified for scheduler-dominated metrics?

Domain E — Platform configuration:
□ Is CPU isolation configuration documented?
□ Is RT priority and governor specified?
□ For new hardware: has Layer 1 revalidation checklist been run?

If any answer is "I don't know" — resolve in design session before
handing to Claude Code.
```

---

## Document History

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial release. Six domains, 20 principles. Derived from unified POSIX-compliant RTOS benchmark framework — EW2026 validated results, Layer 1 complete. Authors: Lei Zhou + Claude (Anthropic). |
| 1.1 | May 2026 | Updated from AGL AMM 2026 deck review. D1: added full per-metric attribution table (TSLat/PreLat/MuLat/InLat/IMLat) with primary cause, HW PMU signal, worst stressor, and countermeasure for each metric; added key architectural implication (TSLat/PreLat have no software countermeasure — irreducible by RT-PREEMPT design). E1: corrected platform config — isolcpus=3 (not 2-3), nohz_full exclusion documented with rationale, idle state WFI-only added, IRQ affinity detail added (all IRQs to CPU0-2, GPIO145 pinned back to CPU3). Sanitized for public release: proprietary RTOS vendor names replaced with generic equivalents. |

---

## Contributing

To add a principle, follow the five-step self-extension mechanism in
the Overview section. Every principle must include: the specific
symptom or finding that motivated it, the generalised rule, at least
one anti-pattern, and a verification method.

The generalisation test is the gate: would this principle prevent the
same mistake on a completely different RTOS benchmark project targeting
safety certification? If yes → this document. If no → PROJECT_CONTEXT.md.

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

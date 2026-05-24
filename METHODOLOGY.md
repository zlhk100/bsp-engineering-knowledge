<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# AI-Assisted Embedded Systems Engineering Methodology

**Version:** 1.7 (filename: METHODOLOGY.md — version-free)  
**Origin:** Derived from a TF-M PSA-L2 firmware porting project and a Qualcomm SoC BSP bring-up project  
**Authors:** Lei Zhou + Claude (Anthropic)  
**Scope:** Any complex embedded firmware project using AI code generation

---

## Overview

This document captures a hybrid methodology for AI-assisted embedded systems
engineering developed during a security-critical firmware porting project
(TF-M PSA-L2 certification on an Arm Cortex-M55 target platform). The methodology applies to
any project where:

- The target system has hardware-specific constraints not in AI training data
- The firmware is security-critical (correctness failures have security consequences)
- A complex existing framework is being ported to a new platform
- An AI code generator (Claude Code or equivalent CLI agent) is used for implementation

The core insight: **AI tools are most effective when the human architect
resolves all design decisions before the AI generates any code.** An AI
that designs and codes simultaneously produces architecturally unsound output
that is expensive to correct.

---

## The Four-Role Model

### Role 1 — The Architect (Human, HIL)

Responsible for **final decisions and ground truth**. Never writes code directly.

Responsibilities:
- Hardware to firmware abstraction mapping
- Explicit confirmation of every design decision before it enters context.md
- Providing SDK source by running greps and pasting output into web chat
- Providing board output (UART, J-Link) for Phase 2+ hardware verification
- Signing off each Claude Code session before the next begins
- Identifying PSA-L2 evaluation lab and managing certification process

**The HIL constraint is absolute.** context.md is updated only on explicit human
confirmation. No design decision is final until the human says so.

### Role 2 — The Design AI (Web Chat)

Responsible for **what** and **why**. Never writes implementation code directly.

Responsibilities:
- Hardware to firmware abstraction mapping
- Interface contract definition (precise function signatures, behavioral contracts)
- Security threat modeling and countermeasure design
- Design decision documentation with traceable rationale
- Root cause analysis when implementation hits hard blockers
- Reading and synthesizing TF-M design documents and SDK source
- Category 1 file production (knowledge-intensive files requiring deep doc analysis)
- Cross-verifying Gemini outputs against normative sources

### Role 3 — The Cross-Verifier (Gemini)

Responsible for **second opinions on design claims**. Never makes final decisions.

Responsibilities:
- Contract tracing: top-down prediction of call graphs from specs and architecture docs
- Providing alternative analysis on design questions where web chat may have gaps
- Identifying candidate unknown-unknowns for source verification

**Critical constraint:** Gemini outputs are reviewed critically before use.
Known failure modes must be checked on every output (see § Known AI Failure Modes).
When Gemini and web chat disagree: verify both against source. Trust source.
Never use a Gemini-provided function signature without grepping the actual header.

### Role 4 — The Implementer (Claude Code CLI)

Responsible for **how** and **execution**. Never makes design decisions.

Responsibilities:
- Translating interface contracts into physical source files
- Boilerplate generation (switch statements, struct initializations, wrappers)
- Compiler/linker error resolution within session scope
- Following the session sequence exactly as specified
- Reporting build results without modifying out-of-scope files

**The boundary is strict.** If the CLI agent hits a decision point not covered
by the specification, it stops and reports to the architect rather than
guessing. An incorrect architectural guess by the CLI agent is harder to
detect and fix than a compile error.

---

## The Hybrid Call Graph Methodology

Establishing correct call graphs for complex firmware is the most critical
pre-implementation activity. Two complementary approaches must be used together.

### Approach A — Contract Tracing (Top-Down)

**Source:** Architecture documents, API specifications, interface headers.

**Method:** Trace execution through defined interface boundaries, top-down.

**Strengths:**
- Version-stable — contracts change more slowly than implementation
- Gives the correct architectural skeleton
- Predicts layer structure without reading every file
- Identifies what each layer is responsible for

**Failure mode:** Contracts can lag behind implementation. Training data
may reflect older versions. Sounds authoritative but may be wrong for
your specific version.

**When to use:** To establish the overall structure and layer attribution
before reading source.

**Example — TF-M ITS write path:**
```
PSA API spec → SVC dispatch → SPM SFN → ITS partition → TF-M HAL → CMSIS → Vendor HAL
```
Each arrow is a defined interface contract. Predictable from specs alone.

### Approach B — Source Verification (Bottom-Up)

**Source:** Actual source files in your specific version of the codebase.

**Method:** Read files directly, follow `#include` chains and function calls
with line number citations.

**Strengths:**
- Version-specific — tells you what the code actually does right now
- Finds implicit dependencies not stated in any contract
- Confirms call ordering within each layer
- Catches implementation divergence from design documents

**Failure mode:** Fragile to version changes. Can miss architectural
constraints that are obvious from contracts. May find details without
understanding their significance.

**When to use:** To verify the contract-traced skeleton against actual code.
Always cite file paths and line numbers.

**Example — confirmed from source (tfm_hal_isolation_v8m.c:300-306):**
```c
sau_and_idau_cfg();          // line 300 — target_cfg.c must implement
mpc_init_cfg();              // line 301 — stub OK for platforms without MPC
ppc_init_cfg();              // line 304 — stub OK for platforms without PPC
```

### The Combined Process

```
Step 1 — Contract tracing
  Identify layer boundaries from specs and architecture docs
  Predict call structure — attribute each layer to a source directory
  Result: architectural skeleton with high confidence

Step 2 — Source verification
  Read actual implementation at each boundary crossing
  Verify function signatures match the contract
  Find implicit dependencies (typedefs, macros, #include chains)
  Confirm temporal ordering of calls
  Result: version-specific verified call graph

Step 3 — Discrepancy resolution
  Source diverges from contract → trust the source, document the divergence
  Source matches contract → high confidence, proceed
  Ambiguous → flag for human review, do not guess
  Result: final call graph with confidence levels per step

Step 4 — Sequence diagram production
  Translate verified call graph into sequence diagrams per use case
  Annotate with data flow (what parameters pass between calls)
  Include temporal constraints (what must happen before what)
  Result: implementation specification for CLI agent
```

### When Contract Tracing Fails

Contract tracing produces wrong answers when:
- Training data reflects a different version than you are using
- The codebase has diverged from its own design documents
- An interface was refactored without updating documentation

**Concrete example from this project:** Gemini stated `tfm_plat_get_huk()`
with signature `(uint8_t *key, uint32_t size)` was the HUK interface in
TF-M. Source reading of `tfm_plat_crypto_keys.h` in v2.3.0 revealed the
actual interface is a descriptor table pattern with no such function.
Contract tracing was wrong. Source reading was correct.

**Rule:** Always verify contract-traced answers against actual source before
writing the specification. Never write implementation specs from contract
tracing alone.

---

## Behavior Specification Requirements

For each platform function, the specification must cover all five dimensions.
Missing any dimension leaves a gap that the CLI agent will fill with a guess.

### Dimension 1 — Interface Contract
- Exact function signature (return type, parameter names and types)
- Which header file declares it
- Which file must implement it

### Dimension 2 — Behavioral Contract
- What the function must guarantee to its caller (postconditions)
- What invariants must hold (e.g., NV counter can only increase)
- What error conditions must be handled and what to return
- What the function must NOT do (e.g., must not block in ISR context)

### Dimension 3 — Platform-Specific Implementation Path
- Which hardware API to call (not just "write to flash" but which SDK function)
- Which header provides that API
- What hardware preconditions must be met (power domain, clock, GPIO mux)
- Phase 1 stub vs Phase N real implementation

### Dimension 4 — Execution Context Constraints
- Thread context vs interrupt context callable
- Blocking allowed or polling-only
- Re-entrancy requirement
- Stack depth budget

### Dimension 5 — Temporal/Ordering Constraints
- What must be initialized before this function is called
- What must this function complete before other functions run
- Ordering relative to security lock sequences

---

## Execution Context Constraints (Embedded-Specific)

These constraints are commonly missed because they are not stated in interface
contracts but are required for correct embedded operation.

### Thread vs Interrupt Context

Identify for each platform function whether it can be called from:
- Thread mode (normal execution context)
- Handler mode (interrupt service routine)
- Both

Functions callable from ISR context must not:
- Block waiting for hardware
- Call RTOS primitives (osDelay, mutex acquire)
- Use large stack frames
- Assume exclusive access to shared hardware

Functions that may be called from both contexts require interrupt protection
around any shared state access:
```c
uint32_t primask = __get_PRIMASK();
__disable_irq();
/* critical section */
__set_PRIMASK(primask);
```

### Execution Model Constraints

For TF-M SFN (Stateless Function Node) backend specifically:
- SPM serializes all partition calls — no concurrent partition execution
- SFN functions must run to completion — no yields, no blocking on SPM
- Hardware polling loops are acceptable — RTOS blocking is not
- DMA with completion callbacks is not acceptable — polling mode only
- Stack is split: MSP for exceptions/boot, PSP for partition execution

For other execution models, document equivalent constraints explicitly.

### Re-entrancy

Hardware peripherals are generally not re-entrant. Document which peripherals
are used by which functions and confirm the execution model prevents concurrent
access. Do not assume the execution model provides protection — verify it.

---

## Phase-Gated Development

Complex firmware ports must be developed in phases. The CLI agent must not
mix phases.

### Phase Structure

**Phase 1 — Compile clean with 100% stubs**
Goal: cmake configure + cmake build produces zero errors
Validates: cmake build system, linker scripts, flash layouts, include paths
Stubs return: safe hardcoded values that allow the system to compile
Do not: implement any real hardware access

**Phase 2 — First boot**
Goal: system boots to NS world and produces UART output
Validates: startup code, SAU configuration, NVIC, UART driver
Stubs remain: crypto, OTP, NV counters
Do not: implement security-critical functions yet

**Phase 3 — Security functions**
Goal: ITS/PS functional with real crypto, NV counters, OTP
Validates: key derivation, AEAD encryption, anti-rollback
Replaces: all Phase 1 crypto stubs with hardware implementations

**Phase 4 — Adversarial testing**
Goal: security claims hold under fault conditions
Validates: power-loss recovery, replay attack resistance, partition isolation

### Phase Discipline Rules

1. Never implement Phase N+1 while Phase N has unresolved errors
2. Each phase ends with a human review of generated code before proceeding
3. Stub implementations must be clearly marked with phase comments
4. Config flags must gate phase-specific code (e.g., `TFM_DUMMY_PROVISIONING`)
5. A phase is complete only when the human architect signs off

---

## The Single Source of Truth (SSoT)

All design decisions, interface contracts, behavioral specifications, and
implementation session sequences must live in a single document consumed by
the CLI agent at the start of every session.

### SSoT Requirements

The SSoT document (`context.md` in this project) must contain:
- Current project state (what is done, what is in progress)
- All open issues with OI-NNN identifiers
- Complete Platform Port Specification (PPS) with all sections
- Implementation session sequence with explicit verify steps
- Hardware constraints with datasheet citations
- Security design decisions with normative justification

### SSoT Discipline

- Update SSoT after every CLI session before starting the next
- CLI agent reads SSoT at the start of every session — enforce this
- Never allow design decisions to exist only in chat history
- When a new design decision is made in web chat, immediately append to SSoT
- SSoT is the ground truth — if it conflicts with chat history, SSoT wins

### What Does NOT go in SSoT

- General methodology (this document)
- Conversation history and reasoning trails
- Superseded design decisions (delete them, do not comment them out)
- Material the CLI agent does not need for implementation

---

## Multi-AI Consensus Methodology

For security-critical design decisions, consult multiple AI systems and
treat discrepancies as signals requiring source verification.

### When to Use Multiple AIs

- Interface contract questions where training data versions may differ
- Security design decisions where one AI may have incomplete knowledge
- Behavioral specifications where subtle correctness constraints matter
- Any answer where "sounds authoritative" is a risk

### Consensus Process

1. Frame the question with full project context (configuration, version, constraints)
2. Ask both AIs independently with the same prompt
3. Compare answers for each question:
   - Agreement → high confidence, verify one answer from source
   - Disagreement → verify both against source, trust source
   - One AI uncertain → trust the certain answer provisionally, verify from source
4. Document what each AI said and what source verification confirmed
5. Record where AI answers were wrong — patterns of error inform future prompting

### Prompt Design for Technical Questions

Break large prompt collections into focused groups of 3-5 questions.
Large prompts produce methodology guidance instead of technical answers.

Each question should:
- State the specific TF-M version and configuration
- Cite the relevant source files already examined
- Ask for source file references in the answer
- Explicitly flag "please indicate if you are uncertain"

### Known AI Failure Modes in Embedded Systems

**Version drift:** AI training data reflects older versions. Always verify
interface signatures from actual headers in your version.

**Plausible but wrong:** Architecturally coherent answers that don't match
your specific configuration. Contract tracing produces these.

**Missing implicit dependencies:** AI misses `typedef`, macro, or `#include`
requirements that only appear by reading the actual source.

**Security oversimplification:** AI may omit security-critical constraints
(e.g., key material handling, power-loss atomicity) that are not in training
data for the specific platform.

**Generalization from reference:** AI answers based on a reference platform
(STM32, nRF) that differs from your target in a security-relevant way.

**Certification level conflation:** AI conflates requirements across PSA-L1,
L2, and L3 because they are adjacent concepts in training data. An AI may
state that a PSA-L3 requirement (FIH, physical attack mitigation, DPA
resistance) is "required" or "recommended" for PSA-L2. Always verify which
certification level a requirement belongs to by checking the applicable
protection profile document directly. (See Anti-pattern 9.)

**Pattern-match without verification:** AI observes a partial signal (e.g.,
a branch name, a filename, a version string) and states a conclusion as if
the conclusion were directly observed. The conclusion is architecturally
coherent and sounds authoritative but rests on an inference, not evidence.
Concretely: seeing `branch: master` and `LAYERSERIES_COMPAT = "whinlatter"`
and concluding "master branch is the wrong branch" — without checking whether
master carries Whinlatter content internally. The correct response is to emit
a verification command and withhold the conclusion until the command is run.

**Misreading under pattern-matching pressure:** AI reads a string and
pattern-matches it to a close but different string (e.g., `board-a-core-kit`
→ `board-b-core-kit`). The error is silent and propagates into subsequent
analysis as if it were fact. Prevention: quote the exact string from the
source when stating any conclusion that depends on a specific value.

---

## Gap Analysis Evidence Protocol

When an AI identifies gaps, blockers, or discrepancies in any technical
analysis — readiness checks, manifest reviews, configuration audits, call
graph analysis, hardware topology reviews, firmware porting pre-flight checks,
or any other diagnostic context — each finding must carry an explicit evidence
tier. This protocol applies to any project type: BSP bring-up, firmware porting,
RTOS integration, bootloader development, security certification, or CI pipeline
setup. It is not specific to any build system or toolchain.

The protocol was derived from observed failures where AI pattern-matching
produced confident but wrong gap claims that consumed debugging time and led
to unnecessary corrective actions.

### The Three Evidence Tiers

**[VERIFIED]** — The claim is directly supported by output the human pasted,
a file the AI read, or a command whose result is in the current session.
No inference required. The human can act on a VERIFIED gap immediately.

**[INFERRED]** — The claim follows logically from verified evidence but
involves one or more reasoning steps not directly confirmed by observation.
The gap is likely real but must be verified before acting. A verification
command is mandatory.

**[ASSUMED]** — The claim rests on general knowledge of the technology
domain rather than project-specific observation. Highest risk tier.
AI training data may not reflect the specific version, configuration, or
setup in use. Must be verified before acting.

### Required Format for Gap Statements

Every gap or blocker identified by the AI must follow this format:

```
GAP-N: [TIER] Short description of the gap

  Observed:  What the output or file literally shows — quoted exactly
  Concluded: What I infer from it — explicitly separated from observation
  Assumed:   What I am filling in from general knowledge (if anything)
  Uncertainty: What could invalidate this gap if my assumption is wrong
  Verify with: <exact command to run on the target system>
  Act only after: confirmation that the verify command output supports the gap
```

VERIFIED gaps may omit the Verify step. INFERRED and ASSUMED gaps must
include it. The human should not act on an INFERRED or ASSUMED gap until
the verification command has been run and the output confirms the gap.

### The Human's Standing Authority

If a gap the AI identifies does not match the human's knowledge, intuition,
or direct observation of the system, that is a valid signal that the AI may
be wrong. The human pushes back immediately. The AI re-examines the evidence
from scratch — it does not defend the original claim.

Direct human observation of the system overrides AI inference. This is
the same principle as the hardware-schematic standing rule applied to software:
what the human observes on the actual system takes precedence over what the
AI concludes from general knowledge. A confident-sounding AI gap claim that
contradicts the human's direct experience is almost certainly wrong.

### What Triggers This Protocol

Apply the evidence tier explicitly whenever the AI is:

- Producing a readiness check, pre-flight analysis, or porting checklist
- Identifying build system blockers or toolchain configuration gaps
- Classifying files, blobs, partitions, or artifacts (keep / discard / hold)
- Stating that a configuration value, version string, or identifier is correct
  or incorrect
- Asserting that a branch, tag, SDK version, or API revision matches or
  mismatches a requirement
- Claiming that a function signature, interface contract, or header path is
  correct for a specific framework version
- Recommending any corrective action that modifies a file, setting, or
  hardware state
- Interpreting hardware register values, memory maps, or schematic net names

### Concrete Examples of Protocol Failure and Correct Behaviour

The following examples are drawn from different project contexts to illustrate
that the failure modes are universal, not build-system-specific.

---

**Example 1 — Yocto BSP (branch name ≠ branch content)**

*Failure (what not to do):*
```
Gap: oe-core is on 'master' but meta-qcom requires 'whinlatter'.
Fix: git checkout whinlatter
```
This states a conclusion (mismatch) and a fix without confirming whether
`master` carries Whinlatter content internally. The fix may be unnecessary
and destructive. Branch name and branch content are different things.

*Correct behaviour:*
```
GAP-1: [INFERRED] oe-core branch may not satisfy LAYERSERIES_COMPAT

  Observed:  oe-core branch = "master" (from git status output)
             meta-qcom conf/layer.conf: LAYERSERIES_COMPAT_qcom = "whinlatter"
  Concluded: Branch name does not match the compat string
  Assumed:   'master' does not carry Whinlatter content internally
  Uncertainty: Vendor QL 2.0 workspace may use 'master' as the Whinlatter
               tracking branch — name and content are independent
  Verify with: grep "LAYERSERIES_CORENAMES\|DISTRO_CODENAME\|whinlatter" \
                 oe-core/meta/conf/layer.conf
  Act only after: output confirms whether 'master' carries whinlatter content
```

---

**Example 2 — Yocto BSP (misreading a string under pattern-matching pressure)**

*Failure (what not to do):*
```
MACHINE = "board-b-core-kit" — confirmed correct, matches target SoC.
```
This misreads `board-a` as `board-b` and compounds the error by stating it as
confirmed. Quoting the exact string from source prevents this class of error.

*Correct behaviour:*
```
GAP-2: [VERIFIED] MACHINE is set to "board-a-core-kit"

  Observed:  local.conf line: MACHINE ??= "board-a-core-kit" (quoted exactly)
  Concluded: This targets SoC-A, not SoC-B (the intended target)
  Verify with: No further verification needed — string is unambiguous
  Action: Change to "board-b-core-kit" per machine conf patch
```

---

**Example 3 — Firmware porting (function signature from training data)**

*Failure (what not to do):*
```
Gap: platform is missing tfm_plat_get_huk(uint8_t *key, uint32_t size).
Fix: implement that signature in crypto_keys.c
```
This states a function signature from AI training data without verifying it
against the actual header in the version in use. The signature may not exist
in this version, causing a linker error in a file the AI did not write.

*Correct behaviour:*
```
GAP-3: [ASSUMED] Platform may be missing the HUK retrieval interface

  Observed:  crypto_keys.c is not present in the platform directory
  Concluded: The HUK interface is unimplemented
  Assumed:   tfm_plat_get_huk(uint8_t *key, uint32_t size) is the correct
             interface — based on AI training data for TF-M
  Uncertainty: TF-M v2.3 may use a different interface pattern entirely;
               training data may reflect an older version
  Verify with: grep -n "tfm_plat.*huk\|huk\|builtin_key" \
                 platform/include/tfm_plat_crypto_keys.h
  Act only after: actual header confirms function name and signature
```

---

**Example 4 — Hardware bring-up (device register interpretation)**

*Failure (what not to do):*
```
Gap: PCIe switch Stream IDs 0x1400–0x147F are not set — SMMU will fault.
Fix: add iommu-map entries for all 128 SIDs.
```
This asserts a specific SID range and a fix based on general SMMU knowledge
without confirming the actual SID table for this peripheral on the target SoC.

*Correct behaviour:*
```
GAP-4: [ASSUMED] PCIe switch downstream SMMU Stream IDs may be missing from iommu-map

  Observed:  iommu-map in carrier DTS has only two root entries (0x1400, 0x1401)
  Concluded: Downstream PCIe switch endpoints may not have SMMU mappings
  Assumed:   SID range 0x1400–0x147F applies to PCIe0 on the target SoC;
             individual downstream entries are required per endpoint
  Uncertainty: of_pci_map_rid() may derive downstream SIDs automatically
               from the root entries; extended iommu-map may not be needed
  Verify with: Check reference board DTS iommu-map pattern;
               cross-reference target SoC TRM PCIe SID table
  Act only after: reference DTS and TRM confirm SID range and entry pattern
```

---

### Integration with the SSoT

Gaps that are confirmed VERIFIED and have a known fix are candidates for
Open Item (OI-NNN) entries in the project SSoT. INFERRED and ASSUMED gaps
must be verified first — they are not promoted to OI entries until confirmed.
This prevents the OI register from filling with speculative items that consume
triage time without representing real blockers.

---

## Session Management for CLI Agent

### Session Prompt Template

```
Read CLAUDE.md first, then read context.md fully.
Summarize: current state, open issues, today's session goal.

Session N goal: implement [specific file] per PPS-[N].
Scope: ONLY modify [specific file]. Do not touch [other files].
After implementing: run cmake --build and report results.
If build succeeds: show me the generated file for review.
If build fails: fix compiler errors only, do not change design.
Report when done.
```

### Session Boundaries

Each session implements exactly one file or one closely related set of files.
The session ends when:
- Clean build achieved, OR
- A design decision is needed that is not in the specification

If a design decision is needed: CLI agent stops, reports the question to the
architect in the web chat. The architect resolves the question, updates the
SSoT, then starts a new CLI session.

### Human Review Gate

After each session producing a clean build:
1. Human reviews the generated file against the PPS specification
2. Human checks for: correct function signatures, correct stub values,
   no unauthorized design decisions, no out-of-scope file modifications
3. If acceptable: human updates SSoT to reflect session completion
4. If not acceptable: human identifies the gap, updates PPS, new session

This gate is mandatory. Skipping it allows accumulated drift between
the specification and the implementation that becomes expensive to correct.

---

## Verification Strategy

### Compile-Time Verification

Use `_Static_assert` in header files to verify memory layout constraints
at compile time rather than discovering them at runtime:

```c
_Static_assert(S_CODE_SIZE <= ITCM_SIZE, "SPE code exceeds ITCM");
_Static_assert((FLASH_ITS_AREA_SIZE % MRAM_WRITE_ALIGNMENT) == 0,
               "ITS area not aligned to write granularity");
```

### Build Verification Per Session

Each session must end with:
```bash
cmake --build 2>&1 | grep -E "error:|warning:|undefined"
```
Zero errors is the acceptance criterion. Warnings must be reviewed.

### Behavioral Verification

For security-critical functions, write test vectors in comments:
```c
/* Test vector: HUK=0xAA*32, label="ITS", uid=1, partition_id=0x100
   Expected output key: [derive and document] */
```
These are not executable tests in Phase 1 but serve as specification
anchors for Phase 4 adversarial testing.

---

## When NOT to Start Implementation

The single most expensive mistake in AI-assisted embedded firmware development
is handing incomplete specifications to the CLI agent. The CLI agent does not
stop when the specification is incomplete — it fills gaps with plausible guesses.
Those guesses compile, pass Phase 1, and fail at Phase 2 or Phase 3 where
rework cost is highest.

Do not start Claude Code sessions until ALL of the following are confirmed:

### Pre-implementation checklist

```
Interface contracts
□ Exact function signatures from actual source headers (not AI training data)
□ Return types confirmed — including FIH wrappers, int vs enum, etc.
□ Header file that declares each interface identified
□ File that must implement each interface identified

Behavioral contracts — all 5 dimensions per function
□ Postconditions — what the function guarantees to its caller
□ Invariants — what must always hold (e.g., NV counter only increases)
□ Error handling — what error conditions return what codes
□ Prohibitions — what the function must NOT do
□ Side effects — what hardware state changes on every call

Platform implementation path
□ Specific SDK/HAL function to call (not "write to flash" — which function)
□ Header that provides that SDK function
□ Hardware power domain requirements before first call
□ Phase 1 stub path vs Phase N real implementation path both specified
□ If adopting any upstream .inc, bbclass, or framework hook: grep for
  destructive operations (-delete, rm, mv) and verify no destructive
  operation runs against a path the implementation author controls.
  If found and path ownership is ambiguous — override the containing
  task completely and document why (see YOCTO_BSP_ARCH E6, F6)

Execution context
□ Thread context vs interrupt context callability confirmed for each function
□ Blocking permission confirmed (polling loops OK, RTOS primitives not OK for SFN)
□ Re-entrancy requirement confirmed
□ IRQ protection required or not required per function

Temporal ordering
□ Boot sequence verified from actual main.c source reading (not inferred)
□ Security lock ordering verified (e.g., SAU before MPU before hardware locks)
□ UART initialization placement confirmed relative to first logging call
□ OTP init placement confirmed relative to partition init

Verified call graph
□ All functions called from common framework files identified from source
□ All implicit dependencies (#include chains, typedefs, macros) found
□ Sequence diagrams produced for highest-risk use cases (boot, ITS write)
□ No call graph entries derived from inference alone — all source-cited

Certification and scope
□ Certification standard confirmed (which standard, which version)
□ Partitions in scope and out of scope explicitly decided
□ Isolation level confirmed with implementation implications understood
□ Profile selection finalized with pros/cons documented

Build system
□ All mandatory #error-guarded defines confirmed present in platform headers
□ All mandatory cmake variables confirmed present in config.cmake
□ Linker script approach confirmed (generated vs static, correct filename)
□ All source files in CMakeLists.txt match file manifest

File manifest
□ Every file Claude Code will create or modify listed explicitly
□ Every new file has a behavioral specification
□ No file depends on a design decision not yet made
□ Session sequence assigns exactly one file per session with verify steps
```

If any checkbox is empty: resolve it in the web interface first.
Claude Code sessions only start when all boxes are checked.

---

## Premature Implementation Anti-patterns

The following anti-patterns were observed when implementation started before
the specification was complete. Each caused rework that would have been
prevented by completing the pre-implementation checklist.

Documented from a TF-M PSA-L2 firmware porting project.

### Anti-pattern 1 — Wrong call ordering from inferred boot sequence

**What happened:** SSC lock placement was specified based on Gemini's description
of the boot sequence without source verification. Gemini said `platform_init`
runs before `set_up_static_boundaries`. Source reading of `main.c` revealed
the opposite — `set_up_static_boundaries` runs first.

**Consequence if uncaught:** `ssc_lock()` placed inside `sau_and_idau_cfg()`
would have locked the SSC before MPU configuration, causing a HardFault on
first boot. Not visible until Phase 2.

**Prevention:** Verify boot sequence from `main.c` source before writing
any security initialization specification. Never accept AI-described call
ordering without source citation.

**Rule:** Boot sequence ordering must be verified from framework `main.c`
source reading. Never infer from architecture documents alone.

### Anti-pattern 2 — Missing UART initialization placement

**What happened:** The specification did not identify that TF-M calls `NOTICE()`
immediately after `tfm_hal_platform_init()` returns. `stdio_init()` was not
specified as a required call inside `platform_init()`.

**Consequence if uncaught:** First boot produces no UART output and appears
to hang. No error message, no assertion — just silence. Extremely difficult
to debug without knowing the NOTICE() call timing.

**Prevention:** Read framework source around the `platform_init()` call site
to understand what the framework expects immediately after it returns.

**Rule:** For every framework callback the platform must implement, read what
the framework does immediately before and after calling it.

### Anti-pattern 3 — Wrong crypto_keys.c interface from AI training data

**What happened:** AI (Gemini) described `tfm_plat_get_huk(uint8_t *key, uint32_t size)`
as the HUK interface for `PLATFORM_DEFAULT_CRYPTO_KEYS=OFF`. Source reading of
`tfm_plat_crypto_keys.h` in TF-M v2.3 revealed the interface is a descriptor
table pattern — the function Gemini described does not exist in this version.

**Consequence if uncaught:** Claude Code implements a function that satisfies
no declaration — linker error that appears to be a cmake problem.

**Prevention:** Always grep the actual header file for the interface before
writing the specification. Never accept AI-described function signatures
without source verification.

**Rule:** Every function signature in the PPS must be verified from the actual
header in the specific version being used. Training data reflects older versions.

### Anti-pattern 4 — Missing mandatory #error-guarded defines

**What happened:** `platform/ext/common/tfm_hal_its.c` and `tfm_hal_ps.c`
contain `#error` directives requiring four defines in `flash_layout.h`:
`TFM_HAL_ITS_FLASH_AREA_ADDR`, `TFM_HAL_ITS_FLASH_AREA_SIZE`,
`TFM_HAL_PS_FLASH_AREA_ADDR`, `TFM_HAL_PS_FLASH_AREA_SIZE`.
These were not in the initial PPS.

**Consequence if uncaught:** Session 1 fails with cryptic `#error` messages
in files Claude Code did not write, with no obvious connection to the missing
defines in `flash_layout.h`.

**Prevention:** For every common framework file that uses the platform's
`flash_layout.h`, read that file's `#error` directives to find required defines.

**Rule:** grep for `#error` and `#ifndef` in every common file that includes
platform headers. Add all required defines to the PPS before Session 1.

### Anti-pattern 5 — Wrong ITS encryption insertion point

**What happened:** The initial design proposed a custom crypto layer inserted
between the ITS filesystem and the CMSIS driver — modifying TF-M internals.
Source reading revealed TF-M v2.3 has an official platform encryption HAL
(`tfm_hal_its_encryption.c`) that is the correct and supported insertion point.

**Consequence if uncaught:** Custom crypto layer would have worked for Phase 1
but created a certification problem at Phase 3. The official HAL is what PSA-L2
evaluators expect. A custom modification to TF-M internals requires additional
justification in the security target documentation.

**Prevention:** Before designing any custom insertion point, search the
framework source for existing HAL interfaces that serve the same purpose.

**Rule:** Search for `__WEAK` functions and official HAL headers before
designing custom frameworks. The framework almost always has the right hook.

### Anti-pattern 6 — Certification scope undefined at implementation start

**What happened:** The initial PPS targeted JSADEN012 (standard PSA-L2) with
profile_medium_arotless (which targets JSADEN019 — ARoT-less L2). These are
inconsistent. PS was in scope in the design but disabled by the profile.

**Consequence if uncaught:** Claude Code would have implemented PS as part
of the port. Midway through Phase 3, the certification scope conflict surfaces
and potentially requires rework of the PS partition implementation.

**Prevention:** Confirm the certification standard before writing any PPS.
The standard determines which partitions are required, which isolation level
is mandatory, and which profile to use.

**Rule:** Certification standard decision is a prerequisite for all other
design decisions. It must be made and documented before the PPS is written.

### Anti-pattern 7 — IPC-backend stack variable used for SFN backend

**What happened:** Gemini recommended `CONFIG_TFM_SPM_THREAD_STACK_SIZE = 4KB`
for the SFN crypto partition stack. Source reading revealed this variable is
defined only in `backend_ipc.c` — it does not exist in the SFN backend.
The SFN backend allocates stack from partition YAML `stack_size` field.

**Consequence if uncaught:** The variable would have been silently ignored.
The actual stack size (6.75KB from profile default) would have been used.
This was a lucky non-failure — the result happened to be correct despite
the wrong reasoning. In a different scenario, a silently ignored variable
could mask a real stack overflow.

**Prevention:** Verify every cmake configuration variable exists in the
backend being used. SFN and IPC backends have different configuration surfaces.

**Rule:** For every cmake variable added to config.cmake, verify it is
used by the actual backend in use. grep the backend source file.

### Anti-pattern 8 — Assuming clean hardware state after closed-source SBL

**What happened:** The startup file was initially specified without
explicit hardware sanitization. The vendor SBL is closed-source and runs
before the ported firmware. Its hardware exit state (cache, MPU, SAU,
NVIC, stack limits) is undocumented. The initial spec assumed post-reset
state.

**Consequence if uncaught:** SBL-configured SAU regions remain active when
the ported firmware configures its own regions, causing unexpected security
partitioning. SBL-enabled D-Cache causes MRAM read-after-write to return
stale data in ITS — data corruption that appears only under specific write
patterns. D-Cache coherency failure is particularly hard to diagnose because
it is intermittent and depends on cache line alignment.

**Prevention:** Treat every closed-source boot stage as leaving hardware
in an unknown state. Explicitly establish all required state at firmware
entry: disable MPU, disable/invalidate D-Cache, clear pending NVIC,
reset SAU, clear stack limit registers.

**Rule:** Any project where a closed-source ROM or SBL runs before the
ported firmware must include explicit hardware sanitization in the startup
file. Never assume clean reset state at firmware entry when a prior boot
stage exists.

### Anti-pattern 9 — AI conflating PSA certification levels

**What happened:** Gemini stated that FIH (Fault Injection Hardening) is
"highly recommended/often required for PSA-L2 certification." The TF-M
physical attack mitigation design document states unambiguously: "Mitigation
against physical attacks is a mandatory requirement for PSA Level 3
certification." PSA-L2 has no FIH requirement. profile_medium_arotless
sets TFM_FIH_PROFILE=OFF by default.

**Consequence if uncaught:** FIH macros added to platform code when FIH
profile is OFF cause compilation errors (undefined symbols). FIH macros
added when FIH profile is ON add unnecessary code complexity and potential
optimization-induced failures to code that has no certification requirement
for it.

**Prevention:** For any security requirement an AI asserts, find the
normative source (protection profile, certification standard, TF-M design
doc) and verify which level it applies to. AI systems conflate L1/L2/L3
requirements because they are adjacent concepts in training data.

**Rule:** Certification level is a normative property. Never accept an AI's
characterization of what is "required" or "recommended" for a certification
level without checking the applicable protection profile document directly.

### Anti-pattern 10 — Skipping cold review of Category 1 files

**What happened:** Category 1 files are written by the web chat architect
session from design documents. Without an independent cold review before
placement, bugs that are invisible to the author survive into the build.
In a TF-M porting project, cold Gemini review of attest_hal.c found a null
terminator included in a CBOR text string (sizeof instead of sizeof-1),
and cold review of otp.c found a dead function (nv_counter_increment)
that would have caused a -Werror -Wunused-function build failure.

**Consequence if uncaught:** Dead code fails at first build under -Werror.
Wrong CBOR format passes all unit tests but fails at the relying party
verifier — undetectable without end-to-end token verification.

**Prevention:** Every Category 1 file must go through cold review before
placement. The reviewer (Gemini or second AI session) reads the file with
no context about its intent — the same way a compiler reads it.

**Rule:** Cold review is mandatory for every Category 1 file. Send the file
only. Do not explain what it does. Do not describe the design intent.
Cold review finds bugs that source verification does not: dead code,
format violations, logic errors visible only at the whole-file level.

### Anti-pattern 11 — Accepting Gemini cold review findings without source verification

**What happened:** Gemini cold review of five Category 1 files produced five
findings. Two were genuine. Three were fabricated from training data:
(a) a non-existent struct field (.alg in tfm_plat_builtin_key_per_user_policy_t),
(b) a function not called by the relevant partition (tfm_plat_get_rotpk_hash
claimed necessary for attestation with BL2=OFF — source grep showed zero calls),
(c) a wrong function signature (uint8_t** asserted where source shows const char**).

If applied without verification, these three false findings would have introduced
new bugs that compile cleanly and fail at runtime.

**Consequence if uncaught:** Wrong struct initialization corrupts key policy.
Unnecessary linker symbol adds dead code. Type mismatch causes undefined
behavior at the function call site — not a compiler error in C without -Wpedantic.

**Prevention:** Every Gemini finding is a hypothesis. Each requires a source
verification grep before applying the fix. The grep takes 30 seconds.
The grep must show the actual struct definition, function signature, or
call site — not inferred from documentation or training data.

**Rule:** Gemini cold review findings are hypotheses, not facts.
Source verification is required for every finding before applying any fix.
Observed false-positive rate in a TF-M porting project session: 3 of 5 (60%).
This is consistent with the "plausible but wrong" failure mode in § Known AI
Failure Modes. Cold review is valuable; blind acceptance of its output is not.



**The architect** (web chat + human) can reason about incomplete information.
Can defer decisions. Can recognize "I don't know enough yet to specify this."
Can consult multiple sources before committing.

**The implementer** (CLI agent) cannot reason about incomplete information.
When the specification has a gap, it does not stop — it fills the gap with
the most plausible answer available. That answer compiles. It may pass Phase 1.
It fails at Phase 2 or Phase 3 where rework is most expensive.

This asymmetry means the cost of a specification gap is not the cost of
fixing it in the web chat (minutes) — it is the cost of discovering and
fixing the consequence in Claude Code (hours of debugging Phase 2/3 failures).

**The web interface design session is not overhead. It is the work.**
**Claude Code is the transcription.**

---

## SDK and Framework Document Reading Strategy

A port from scratch faces two large corpora: the target framework (TF-M) and
the vendor SDK (vendor SDK or equivalent). The instinct is to read broadly
before writing anything. This is wrong. It produces weeks of reading with
no implementation output, and the knowledge decays before it is used.

The correct strategy is **constraint-driven funnel reading**: the port file
manifest determines which documents are read, in what order, and when.

### The Constraint-Driven Funnel

```
Port file manifest (finite, known)
        │
        ▼
For each file: identify the interface header it must implement
        │
        ▼
Interface header → governing TF-M design doc → read ONLY that doc
        │
        ▼
Design doc → SDK HAL functions needed → read ONLY those HAL headers
        │
        ▼
Write PPS section → hand to Claude Code
```

You never read a document speculatively. You read it because a specific
port file requires it. This produces an ~85% skip rate on both TF-M docs
and the vendor SDK — not laziness but hyper-focus.

### Framework Document Triage (TF-M example)

```
READ (directly governs a port file):
  integration_guide/platform/porting_tfm_to_a_new_hardware.rst
  integration_guide/services/tfm_attestation_integration_guide.rst
  integration_guide/services/tfm_crypto_integration_guide.rst
  integration_guide/services/tfm_its_integration_guide.rst
  integration_guide/services/tfm_platform_integration_guide.rst
  design_docs/crypto/tfm_builtin_keys.rst
  design_docs/services/tfm_its_service.rst
  design_docs/services/tfm_crypto_design.rst
  design_docs/services/secure_partition_manager.rst
  design_docs/tfm_physical_attack_mitigation.rst  (FIH — confirms L3-only)
  design_docs/tfm_isolation.rst
  config/config_base.cmake
  config/profile/profile_<target>.conf

SKIP (no port file governed by these):
  integration_guide/services/tfm_ps_integration_guide.rst  (PS disabled)
  design_docs/services/tfm_fwu_service.rst                 (FWU not in scope)
  design_docs/services/ps_key_management.rst               (PS disabled)
  design_docs/multi-cpu/*                                  (single core)
  building/, contributing/, releases/, security/           (non-design content)
```

### Vendor SDK Triage (example structure)

```
READ (implements hardware side of port file interfaces):
  mcu/<target>/hal/vendor_hal_flash.h/.c   (Driver_Flash.c)
  mcu/<target>/hal/vendor_hal_otp.h/.c     (otp.c, nv_counters_device.c)
  mcu/<target>/hal/vendor_hal_security.h/.c (otp.c, attest_hal.c)
  mcu/<target>/hal/vendor_hal_power.h/.c   (all peripheral power management)
  mcu/<target>/hal/vendor_hal_uart.h/.c    (Driver_USART.c)
  mcu/<target>/regs/<target>.h             (all register definitions)
  boards/*/examples/crypto/               (startup pattern, vector table)
  boards/*/examples/security/             (OTP write patterns)

SKIP (no interface to implement):
  vendor_ble/        (BLE stack — not in platform layer)
  third_party/       (crypto handled by TF-M)
  mcu/<target>/hal/vendor_hal_gpio.h/.c   (no GPIO in platform layer)
  mcu/<target>/hal/vendor_hal_i2c.h/.c    (no I2C in platform layer)
  boards/*/examples/* (all non-security examples)
```

### The Five-Step Pre-Implementation Process

For each port file, execute these steps in order before writing any PPS:

**Step 1 — Identify the interface.**
Open the TF-M header that declares what this file must implement.
Enumerate every function signature. Verify signatures from the actual header
in the specific TF-M version in use — not from AI training data.

**Step 2 — Find the governing TF-M design doc.**
```bash
grep -rl "function_name\|service_name" docs/
```
Read only the section describing the interface. Not the whole document.

**Step 3 — Find the SDK API.**
```bash
grep -r "relevant_peripheral\|function_name" mcu/<target>/hal/
```
Read that HAL header and its .c implementation for non-obvious behavior
(power domain requirements, busy-wait patterns, error conditions).

**Step 4 — Extract behavioral constraints across all five dimensions.**
From TF-M doc: postconditions, invariants, prohibitions.
From SDK source: preconditions (power domain, clock, init order).
From hardware register map: alignment, granularity, atomicity.
From boot chain analysis: hardware state at function entry.

**Step 5 — Write PPS section, then implement.**
All five dimensions complete before any code is written.

### Detecting Unknown-Unknowns: Three Mandatory Greps

Before writing any port file, run these three greps against the framework.
They surface mandatory contracts that no design document explicitly lists.

**Technique 1 — #error grep (mandatory define discovery)**
```bash
# Find defines your platform header MUST provide
grep -rn "#error\|#ifndef" platform/ext/common/ | grep -v "^Binary"
```
TF-M common files use `#error` to enforce platform header contracts at
compile time. Missing these produces cryptic errors in files you didn't write.
This is how Correction 1 (4 missing flash_layout.h defines) was caught.

**Technique 2 — __WEAK grep (optional override discovery)**
```bash
# Find framework functions your platform CAN or MUST override
grep -rn "__WEAK" platform/ext/common/ platform/include/
```
TF-M uses weak linkage extensively. An un-overridden weak function silently
links the default stub. Some stubs return 0 (harmless). Others return error
codes that cascade into boot failures. Some stubs are certification gaps.
This is how Correction 6 (tfm_hal_its_encryption.c) was found.

**Technique 3 — Caller grep (execution context discovery)**
```bash
# Find who calls your function and what they do immediately before/after
grep -rn "tfm_hal_platform_init\|function_name" secure_fw/ platform/ext/common/
```
The framework's behavior immediately before and after calling a platform
function defines its execution context and ordering constraints. Missing
this caused the UART placement error (Anti-pattern 2) and would have caused
the boot seed caching design error in attest_hal.c.

---

## The Closed-Source Boot Stage Blindspot

When a vendor secure bootloader (SBL) runs before the firmware under port,
it leaves hardware in an undocumented state. This is an **unknown-unknown
by construction** — the SBL is closed source, its exit state is not
published, and it may vary across SDK versions.

This applies to any project where:
- A vendor ROM or SBL runs before the ported firmware
- The SBL is closed source or its hardware exit state is undocumented
- The ported firmware assumes clean reset state

**Examples from a Cortex-M55 platform:** Boot chain is BootROM → vendor SBL → TF-M S.
The SBL may leave: D-Cache enabled, SAU regions configured for SBL use,
MPU regions active, interrupts pending, MSPLIM/PSPLIM set.

### Mandatory hardware sanitization at firmware entry

The startup file and early platform init MUST NOT assume clean reset state.
They must explicitly establish known state before any security configuration.

```c
/* startup_<target>_s.c Reset_Handler — explicit state establishment */

/* 1. Disable MPU before any SAU/security config
 *    SBL may have configured MPU regions for its own operation */
ARM_MPU_Disable();

/* 2. Disable D-Cache before security setup
 *    Cortex-M55 has D-Cache; SBL may have enabled it.
 *    Cache must be disabled/invalidated before MRAM write protection
 *    is configured, otherwise stale cache lines corrupt reads. */
SCB_DisableDCache();
SCB_InvalidateDCache();

/* 3. Clear all pending IRQs the SBL may have armed */
for (uint32_t i = 0; i < (MAX_IRQn + 31U) / 32U; i++) {
    NVIC->ICPR[i] = 0xFFFFFFFFU;
}

/* 4. Reset SAU to known state before target_cfg.c configures it
 *    SBL uses SAU for its own Secure/NS partitioning */
TZ_SAU_Disable();
for (uint32_t r = 0; r < SAU->TYPE_b.SREGION; r++) {
    SAU->RNR = r;
    SAU->RBAR = 0;
    SAU->RLAR = 0;
}

/* 5. Clear MSPLIM/PSPLIM — SBL may have set stack limits */
__set_MSPLIM(0);
__set_PSPLIM(0);
```

### D-Cache coherency after MRAM writes

Cortex-M55 D-Cache introduces a read-after-write hazard for MRAM: a write
via the vendor flash write API populates flash physically, but a subsequent
read may return the stale cache line rather than the written data. For TF-M
ITS this causes data corruption — written records read back as old content.

**Rule:** Every MRAM write function (Driver_Flash.c `ProgramData()`) must
call cache maintenance after the write completes:
```c
SCB_CleanInvalidateDCache_by_Addr((uint32_t *)dst, (int32_t)len);
```

This applies whether or not the SBL enabled the cache — it is correct
regardless because TF-M itself may enable D-Cache for performance.

### How to discover the SBL exit state

When documentation is unavailable:

1. **Ask the vendor explicitly.** File a support request: "What is the
   hardware state (cache, MPU, SAU, NVIC, stack limits) when SBL jumps
   to application entry point?" This is a legitimate porting question.

2. **Read SBL release notes.** SDK changelogs sometimes document exit state
   changes. Check across SDK versions if the SBL has been updated.

3. **Defensive programming.** When the exit state cannot be confirmed,
   explicitly establish all required state rather than assuming. The cost
   of redundant initialization is one boot cycle. The cost of assuming
   wrong state is a boot failure with no diagnostic output.

---



The most costly specification errors come not from missing source code
verification but from missing design documents and standards. Source code
shows what the code does. Design documents explain WHY and what behavioral
invariants hold across the entire system — invariants that are invisible
in the source.

**Concrete example from a TF-M firmware porting project:**

The `tfm_builtin_keys.rst` design document revealed that `tfm_builtin_key_loader`
automatically derives per-partition keys from the HUK using partition_id as
KDF input. This happens transparently before any platform encryption code runs.
Our PPS had specified CM1 to include partition_id in the ITS derivation label —
which would have double-applied the partition binding. Source code reading
alone would not have revealed this behavioral invariant. Only the design
document explained it.

### Mandatory reading order before writing PPS

Read in this order. Do not start writing PPS sections until the corresponding
tier is complete for that component.

```
Tier 1 — Standards and certification scope (read FIRST)
  □ Target certification standard (JSADEN019, JSADEN012, FIPS, CC etc.)
  □ PSA Functional API specifications (Storage, Crypto, Attestation)
  □ Security Target template for the target standard
  □ Profile specification document (e.g. profile_medium_arotless.rst)
  Why: Defines what must be implemented, what is out of scope, and what
  claims the evaluator will verify. Wrong certification scope discovered
  late causes major rework.

Tier 2 — Framework design documents (read SECOND)
  □ Design doc for each component in scope
      For TF-M: its_service, crypto_design, secure_partition_manager,
      isolation, attestation_design, builtin_keys, fih
  □ Framework configuration system documentation
      For TF-M: config_base.cmake, profile .conf files
  □ Framework porting guide
  Why: Reveals behavioral invariants, automatic framework behaviors,
  and constraints not visible in source code. The builtin key derivation
  example above is precisely this category.

Tier 3 — Source code verification (read THIRD)
  □ Framework main/init — boot sequence and call ordering
  □ Common platform files — what they call, what they require from platform
  □ Reference platform implementations — patterns, barriers, stub contracts
  □ Interface headers in your specific version
  Why: Verifies that design documents match actual implementation.
  Training data and design documents may reflect older versions.
  Source is authoritative for your specific version.

Tier 4 — Vendor SDK documentation (read FOURTH)
  □ Security User's Guide — OTP layout, key management, hardware root of trust
  □ HAL API documentation — initialization sequences, power domains, timing
  □ Hardware datasheet — memory map, peripheral addresses, protection granularity
  Why: Platform-specific constraints not available from any other source.
  AI training data does not contain NDA vendor documentation.

Tier 5 — Write PPS (write LAST)
  □ All 5 behavioral dimensions per function
  □ Every contract source-verified before handing to Claude Code
  □ No implementation until all checklist items checked
```

### The rule

**Never write a PPS section for a framework component without first reading
the design document for that component.**

If no design document exists, that absence is itself a signal — the behavior
must be inferred from source code and reference implementations with extra
care. Flag this explicitly in the PPS with a confidence level.

### Signs a knowledge source is missing

These signals indicate a tier was skipped:

- AI gives a confident answer that turns out to be wrong for your version
- A behavior that "should" work fails silently at runtime
- A security invariant is discovered post-implementation that changes the design
- The certification evaluator asks for justification of a decision that was
  never explicitly made — it was assumed

### For TF-M projects specifically

Minimum documents to read before starting any TF-M port PPS:

```
□ docs/integration_guide/platform/porting_tfm_to_a_new_hardware.rst
□ docs/design_docs/crypto/tfm_builtin_keys.rst
□ docs/design_docs/services/tfm_its_service.rst
□ docs/design_docs/services/tfm_crypto_design.rst
□ docs/design_docs/secure_partition_manager/tfm_secure_partition_manager.rst
□ docs/design_docs/tfm_isolation.rst
□ docs/design_docs/tfm_fih.rst
□ docs/design_docs/services/tfm_attestation_design.rst
□ config/config_base.cmake (all variables and defaults)
□ config/profile/profile_<target>.conf (actual profile defaults)
□ The target certification profile .rst
   (e.g. docs/integration_guide/tfm_profile_medium_arot-less.rst)
```

Each document maps to PPS sections it informs:

| Document | PPS sections affected |
|---|---|
| tfm_builtin_keys.rst | PPS-13 (crypto_keys.c), PPS-22 (CM1 key derivation) |
| tfm_its_service.rst | PPS-10 (Driver_Flash), PPS-23 (ITS encryption), Correction 6 |
| tfm_crypto_design.rst | PPS-13 (crypto_keys.c), PPS-9 (platform_init sequence) |
| tfm_secure_partition_manager.rst | PPS-9 (boot sequence), execution model constraints |
| tfm_isolation.rst | PPS-8 (target_cfg), SAU/MPU ordering |
| tfm_fih.rst | PPS-9 (FIH_RET_TYPE usage), PPS-8 (enable_fault_handlers) |
| tfm_attestation_design.rst | PPS-12 (attest_hal behavioral contracts) |
| config_base.cmake | PPS-5 (config.cmake), PPS-6 (config_tfm_target.h) |
| profile .conf | PPS-5, PPS-6 (what profile already sets — do not override) |
| certification profile .rst | Certification scope, PS in/out of scope decision |

---

## Applicability Beyond This Project

This methodology applies to any project with these characteristics:

- **Complex framework porting:** TF-M, Zephyr RTOS, FreeRTOS, AUTOSAR
- **Security-critical firmware:** PSA certification, FIPS, Common Criteria
- **Hardware-specific constraints:** SoC-specific peripherals, proprietary HALs
- **Multi-layer abstraction:** PSA → TF-M → CMSIS → Vendor HAL pattern
- **Certification requirement:** Any standard requiring traceable design decisions

Projects where this methodology adds less value:
- Simple bare-metal firmware with no framework
- Well-documented platforms with comprehensive AI training coverage
- Non-security-critical firmware where incorrect behavior is immediately visible

---

## Document History

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-29 | Initial capture from a TF-M PSA-L2 firmware porting project |
| 1.1 | 2026-04-29 | Added pre-implementation checklist, 7 anti-patterns, architect/implementer asymmetry section |
| 1.2 | 2026-05-03 | Added Knowledge Source Hierarchy section with 5-tier reading order and TF-M document map |
| 1.3 | 2026-05-04 | Expanded Two-Role to Four-Role model (HIL, Design AI, Gemini, Claude Code). Added SDK/Framework Reading Strategy with constraint-driven funnel, three mandatory greps, and framework/SDK triage tables. Added Closed-Source Boot Stage Blindspot section (SBL state sanitization, D-Cache coherency). Added Anti-patterns 8 and 9. Added certification level conflation to Known AI Failure Modes. |
| 1.4 | 2026-05-05 | Added Anti-patterns 10-11: mandatory cold review for Category 1 files, source verification required for all Gemini cold review findings. Documented 60% Gemini false-positive rate in a project session. |
| 1.5 | 2026-05-08 | Added Gap Analysis Evidence Protocol section: three evidence tiers (VERIFIED / INFERRED / ASSUMED), required gap statement format, human standing authority rule, protocol failure examples. Added two new Known AI Failure Modes: pattern-match without verification, misreading under pattern-matching pressure. Derived from a Qualcomm SoC BSP readiness analysis session. |
| 1.6 | 2026-05-21 | Added Domain Architecture Documents section — framework for domain-specific companion documents. Added YOCTO_BSP_ARCH.md as first domain architecture document reference. Added three new Known AI Failure Modes: layer capability blindness, expedient pattern replication, abstraction layer confusion. Derived from a Qualcomm SoC BSP Yocto code review. Added CC BY 4.0 licence. |
| 1.7 | 2026-05-22 | Added destructive operation audit to pre-implementation checklist. Derived from a BSP bring-up session data loss — adopted .inc ran find -delete against live prebuilts directory when S was redirected to a vendor prebuilts path as fetch bypass. Generalised: any adopted upstream component that runs destructive operations must be audited for path ownership before implementation. Cross-references YOCTO_BSP_ARCH E6 and F6. |

---

*This methodology document is maintained separately from project-specific
context files. It should be referenced at the start of new projects and
updated when new patterns are discovered.*

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

## Domain Architecture Documents

This methodology defines the engineering process. Domain-specific architecture
documents define the structural constraints within that process for specific
technology domains. They are companions to this document, not replacements.

### Current Domain Architecture Documents

| Document | Domain | Applies when |
|---|---|---|
| `YOCTO_BSP_ARCH.md` | Yocto-based BSP bring-up | Any session involving Yocto recipes, kas config, kernel patches, layer structure, firmware staging |

### Relationship to This Methodology

| This methodology covers | Domain architecture documents cover |
|---|---|
| Four-role model (HIL, Design AI, Cross-verifier, Implementer) | Domain-specific pre-implementation checklists |
| Hybrid call graph methodology | Domain-specific layer structure constraints |
| Gap Analysis Evidence Protocol | Domain-specific anti-patterns |
| Phase-gated development | Domain-specific architectural principles |
| Pre-implementation checklist (generic) | Domain-specific capability audit commands |
| Known AI failure modes (generic) | Domain-specific AI generation constraints |
| SSoT discipline | Domain-specific version traceability requirements |

### Adding a New Domain Architecture Document

When a project introduces a new technology domain — AUTOSAR, Zephyr RTOS,
U-Boot, OpenEmbedded for a different SoC family, buildroot — create a new
domain architecture document following the same structure as
`YOCTO_BSP_ARCH.md`:

1. **Unifying meta-principle** for the domain
2. **Categorical domains** covering all design decisions in that technology
3. **Self-extension mechanism** so new principles discovered on future
   projects slot in naturally
4. **Companion skill file** for active Claude enforcement
5. **Reference to this methodology** as the process framework

The domain document answers: *what structural constraints apply to this
technology?* This methodology answers: *how do we run the project?*

---

## Known AI Failure Modes — Addendum (v1.6)

The following failure modes extend the list in the "Known AI Failure Modes"
section above, derived from Qualcomm SoC BSP bring-up experience with
AI-generated Yocto recipes.

**Layer capability blindness:** AI generates Yocto recipes, bbappends, or
kas configuration without reading the upstream layer's actual capability
surface. The AI knows general Yocto patterns from training data but does
not know what `.inc` files, bbclasses, or PACKAGECONFIG options the
specific upstream layer provides. The generated code is functionally
correct — it builds and boots — but architecturally wrong: it reinvents
infrastructure the upstream layer already provides. The code passes all
tests and fails code review. Prevention: always read and place upstream
layer files in context before any recipe generation session (see
`YOCTO_BSP_ARCH.md`, Domain F, Principle F1).

**Expedient pattern replication:** AI training data for Yocto contains
large amounts of expedient BSP code written under delivery pressure that
does not follow best practices: `DISTRO = "poky"` in production BSPs,
manual layer checkout steps in READMEs, kernel patches accumulated in
SRC_URI lists, image bbappends used for infrastructure operations. The AI
reproduces these patterns because they are statistically common in training
data — not because they are correct. The generated code matches what most
BSP authors write, not what experienced BSP architects recommend. Prevention:
explicitly constrain the AI with the domain architecture principles document
before generation; do not rely on the AI's unprompted judgment about
Yocto best practices.

**Abstraction layer confusion:** AI places configuration at the wrong
Yocto abstraction layer — hardware configuration in distro conf, feature
flags in recipe conditionals, infrastructure logic in image bbappends —
because the code produces correct output regardless of layer placement.
Yocto's build system does not enforce layer responsibility boundaries;
misplaced logic is only detected in architectural review. Prevention:
apply the Domain A checklist from `YOCTO_BSP_ARCH.md` before
generating any configuration file.

---


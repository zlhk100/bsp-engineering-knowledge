<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# AI-Assisted Embedded Systems Engineering Methodology — Reference

**Version:** 1.11-ref
**Split from:** METHODOLOGY.md v1.8 (2026-05-27)
**Authors:** Lei Zhou + Claude (Anthropic)
**Active rules companion:** METHODOLOGY_RULES.md — load every session.
**Load this file when:** anti-pattern review, methodology questions,
new domain document creation, post-mortem analysis, knowledge base
maintenance, or any session where detailed rationale is needed.

---

## The Hybrid Call Graph Methodology — Detail

### Approach A — Contract Tracing (Top-Down)

**Source:** Architecture documents, API specifications, interface headers.
**Method:** Trace execution through defined interface boundaries, top-down.
**Strengths:** Version-stable; gives correct architectural skeleton; predicts
layer structure without reading every file.
**Failure mode:** Contracts can lag behind implementation. Training data
may reflect older versions. Sounds authoritative but may be wrong for
your specific version.
**When to use:** To establish overall structure before reading source.

**Example — TF-M ITS write path:**
```
PSA API spec → SVC dispatch → SPM SFN → ITS partition → TF-M HAL → CMSIS → Vendor HAL
```

### Approach B — Source Verification (Bottom-Up)

**Source:** Actual source files in your specific version of the codebase.
**Method:** Read files directly, follow #include chains and function calls
with line number citations.
**Strengths:** Version-specific; finds implicit dependencies; confirms call
ordering; catches implementation divergence from design documents.
**Failure mode:** Fragile to version changes. May find details without
understanding their significance.
**When to use:** To verify the contract-traced skeleton against actual code.
Always cite file paths and line numbers.

**Example — confirmed from source (tfm_hal_isolation_v8m.c:300-306):**
```c
sau_and_idau_cfg();   // line 300
mpc_init_cfg();       // line 301 — stub OK for platforms without MPC
ppc_init_cfg();       // line 304 — stub OK for platforms without PPC
```

### The Combined Process

```
Step 1 — Contract tracing: predict call structure from specs
Step 2 — Source verification: verify each boundary crossing from source
Step 3 — Discrepancy resolution: source diverges → trust source, document
Step 4 — Sequence diagram production: verified call graph → implementation spec
```

### When Contract Tracing Fails

Concrete example: Gemini stated `tfm_plat_get_huk(uint8_t *key, uint32_t size)`
was the HUK interface in TF-M. Source reading of `tfm_plat_crypto_keys.h` in
v2.3.0 revealed the actual interface is a descriptor table pattern with no
such function. Contract tracing was wrong. Source reading was correct.

**Rule:** Always verify contract-traced answers against actual source before
writing the specification.

---

## Behavior Specification — Five Dimensions

For each platform function, all five dimensions must be covered. Missing any
dimension leaves a gap the CLI agent fills with a guess.

1. **Interface Contract** — exact signature, declaring header, implementing file
2. **Behavioral Contract** — postconditions, invariants, error handling, prohibitions
3. **Platform-Specific Implementation Path** — which SDK function, which header,
   hardware preconditions, Phase 1 stub vs Phase N real path
4. **Execution Context Constraints** — thread vs ISR, blocking, re-entrancy, stack depth
5. **Temporal/Ordering Constraints** — what must be initialized before, what must
   complete before other functions run, ordering relative to security locks

---

## Execution Context Constraints — Detail

### Thread vs Interrupt Context

Functions callable from ISR context must not:
- Block waiting for hardware
- Call RTOS primitives (osDelay, mutex acquire)
- Use large stack frames
- Assume exclusive access to shared hardware

Functions callable from both contexts require interrupt protection:
```c
uint32_t primask = __get_PRIMASK();
__disable_irq();
/* critical section */
__set_PRIMASK(primask);
```

### TF-M SFN Backend Constraints

- SPM serializes all partition calls — no concurrent partition execution
- SFN functions must run to completion — no yields, no blocking on SPM
- Hardware polling loops acceptable — RTOS blocking is not
- DMA with completion callbacks not acceptable — polling mode only
- Stack split: MSP for exceptions/boot, PSP for partition execution

---

## SDK and Framework Document Reading Strategy

### The Constraint-Driven Funnel

```
Port file manifest (finite, known)
        ↓
For each file: identify the interface header it must implement
        ↓
Interface header → governing TF-M design doc → read ONLY that doc
        ↓
Design doc → SDK HAL functions needed → read ONLY those HAL headers
        ↓
Write PPS section → hand to Claude Code
```

Never read a document speculatively. Read it because a specific port file
requires it. This produces an ~85% skip rate on both TF-M docs and vendor SDK.

### Framework Document Triage (TF-M)

```
READ:
  integration_guide/platform/porting_tfm_to_a_new_hardware.rst
  integration_guide/services/tfm_attestation_integration_guide.rst
  integration_guide/services/tfm_crypto_integration_guide.rst
  integration_guide/services/tfm_its_integration_guide.rst
  design_docs/crypto/tfm_builtin_keys.rst
  design_docs/services/tfm_its_service.rst
  design_docs/services/tfm_crypto_design.rst
  design_docs/secure_partition_manager.rst
  design_docs/tfm_physical_attack_mitigation.rst
  design_docs/tfm_isolation.rst
  config/config_base.cmake
  config/profile/profile_<target>.conf

SKIP:
  integration_guide/services/tfm_ps_integration_guide.rst  (PS disabled)
  design_docs/services/tfm_fwu_service.rst                 (FWU not in scope)
  design_docs/multi-cpu/*                                  (single core)
  building/, contributing/, releases/, security/
```

### The Five-Step Pre-Implementation Process

1. **Identify the interface** — open the TF-M header, enumerate every signature
2. **Find the governing TF-M design doc** — `grep -rl "function_name" docs/`
3. **Find the SDK API** — `grep -r "relevant_peripheral" mcu/<target>/hal/`
4. **Extract behavioral constraints** — postconditions, preconditions, alignment, atomicity
5. **Write PPS section, then implement** — all five dimensions complete before code

### Knowledge Source Tiers — Reading Order

```
Tier 1 — Standards and certification scope (FIRST)
Tier 2 — Framework design documents (SECOND)
Tier 3 — Source code verification (THIRD)
Tier 4 — Vendor SDK documentation (FOURTH)
Tier 5 — Write PPS (LAST)
```

Never write a PPS section for a framework component without first reading
the design document for that component.

### TF-M Document → PPS Section Map

| Document | PPS sections affected |
|---|---|
| tfm_builtin_keys.rst | crypto_keys.c, CM1 key derivation |
| tfm_its_service.rst | Driver_Flash, ITS encryption |
| tfm_crypto_design.rst | crypto_keys.c, platform_init sequence |
| tfm_secure_partition_manager.rst | boot sequence, execution model |
| tfm_isolation.rst | target_cfg, SAU/MPU ordering |
| tfm_fih.rst | FIH_RET_TYPE usage, enable_fault_handlers |
| tfm_attestation_design.rst | attest_hal behavioral contracts |
| config_base.cmake | config.cmake, config_tfm_target.h |
| profile .conf | what profile already sets — do not override |

---

## The Closed-Source Boot Stage Blindspot

When a vendor SBL runs before the firmware, it leaves hardware in unknown state.
This is an unknown-unknown by construction — the SBL is closed source.

**Mandatory hardware sanitization at firmware entry:**
```c
ARM_MPU_Disable();
SCB_DisableDCache();
SCB_InvalidateDCache();
for (uint32_t i = 0; i < (MAX_IRQn + 31U) / 32U; i++) {
    NVIC->ICPR[i] = 0xFFFFFFFFU;
}
TZ_SAU_Disable();
for (uint32_t r = 0; r < SAU->TYPE_b.SREGION; r++) {
    SAU->RNR = r; SAU->RBAR = 0; SAU->RLAR = 0;
}
__set_MSPLIM(0);
__set_PSPLIM(0);
```

**D-Cache coherency after MRAM writes:**
```c
/* After every NVM write: */
SCB_CleanInvalidateDCache_by_Addr((uint32_t *)dst, (int32_t)len);
```

**How to discover SBL exit state when undocumented:**
1. Ask the vendor explicitly — file a support request
2. Read SBL release notes across SDK versions
3. Defensive programming — explicitly establish all required state

---

## Multi-AI Consensus Methodology

### When to Use Multiple AIs
- Interface contract questions where training data versions may differ
- Security design decisions where one AI may have incomplete knowledge
- Any answer where "sounds authoritative" is a risk

### Consensus Process
1. Frame question with full project context
2. Ask both AIs independently with the same prompt
3. Agreement → verify one answer from source
   Disagreement → verify both against source, trust source
4. Document what each AI said and what source verification confirmed
5. Record where AI answers were wrong — patterns inform future prompting

### Prompt Design
- Break large prompt collections into focused groups of 3-5 questions
- State specific TF-M version and configuration
- Cite relevant source files already examined
- Ask for source file references in the answer
- Flag: "please indicate if you are uncertain"

---

## Premature Implementation Anti-patterns

### Anti-pattern 1 — Wrong call ordering from inferred boot sequence
SSC lock placed based on Gemini's boot sequence description without source
verification. Source reading of main.c revealed set_up_static_boundaries()
runs BEFORE platform_init(). Lock in wrong function → HardFault on first boot.
**Rule:** Boot sequence ordering verified from main.c source. Never infer.

### Anti-pattern 2 — Missing UART initialization placement
TF-M calls NOTICE() immediately after tfm_hal_platform_init() returns.
stdio_init() not specified as required inside platform_init().
First boot produces no UART output, appears to hang. No error, no assertion.
**Rule:** For every framework callback, read what framework does immediately
before and after calling it.

### Anti-pattern 3 — Wrong crypto_keys.c interface from AI training data
Gemini described tfm_plat_get_huk(uint8_t *key, uint32_t size). Source reading
of tfm_plat_crypto_keys.h in TF-M v2.3 revealed descriptor table pattern —
the function does not exist in this version. Linker error appears to be cmake problem.
**Rule:** Every function signature verified from actual header in version in use.

### Anti-pattern 4 — Missing mandatory #error-guarded defines
platform/ext/common/tfm_hal_its.c contains #error requiring four defines in
flash_layout.h. Not in initial PPS. Session 1 fails with cryptic #error messages
in files Claude Code did not write.
**Rule:** grep for #error and #ifndef in every common file that includes platform
headers. Add all required defines to PPS before Session 1.

### Anti-pattern 5 — Wrong ITS encryption insertion point
Custom crypto layer designed between ITS filesystem and CMSIS driver. Source
reading revealed tfm_hal_its_encryption.c is the official supported insertion
point. Custom modification requires additional certification justification.
**Rule:** Search for __WEAK functions and official HAL headers before designing
custom frameworks. The framework almost always has the right hook.

### Anti-pattern 6 — Certification scope undefined at implementation start
Profile_medium_arotless (JSADEN019) used while PPS targeted JSADEN012 (standard
L2). PS implemented but disabled by profile. Certification scope conflict surfaces
mid-Phase 3.
**Rule:** Certification standard decision is a prerequisite for all other decisions.

### Anti-pattern 7 — IPC-backend stack variable used for SFN backend
CONFIG_TFM_SPM_THREAD_STACK_SIZE defined only in backend_ipc.c — does not exist
in SFN backend. Variable silently ignored. SFN stack from partition YAML stack_size.
**Rule:** Verify every cmake configuration variable exists in the backend in use.

### Anti-pattern 8 — Assuming clean hardware state after closed-source SBL
Startup file specified without explicit hardware sanitization. SBL-configured SAU
regions remain active. SBL-enabled D-Cache causes MRAM read-after-write to return
stale data. Intermittent, cache-line-alignment dependent — extremely hard to diagnose.
**Rule:** Any project with a closed-source prior boot stage must include explicit
hardware sanitization. Never assume clean reset state.

### Anti-pattern 9 — AI conflating PSA certification levels
Gemini stated FIH is "highly recommended for PSA-L2." TF-M physical attack
mitigation doc states: "Mitigation against physical attacks is mandatory for
PSA Level 3." profile_medium_arotless sets TFM_FIH_PROFILE=OFF.
**Rule:** Never accept AI characterization of certification level requirements
without checking the applicable protection profile document directly.

### Anti-pattern 10 — Skipping cold review of Category 1 files
Cold Gemini review of attest_hal.c found null terminator in CBOR text string
(sizeof instead of sizeof-1). Cold review of otp.c found dead function causing
-Werror -Wunused-function build failure.
**Rule:** Cold review mandatory for every Category 1 file. Send file only.
Do not explain intent. Do not describe design.

### Anti-pattern 11 — Accepting Gemini cold review findings without source verification
Five findings in one session: two genuine, three fabricated. Non-existent struct
field, function not called by relevant partition, wrong function signature. If applied
without verification, three new runtime bugs introduced.
Observed false-positive rate: 3 of 5 (60%).
**Rule:** Every cold review finding is a hypothesis. Source verification grep required
before applying any fix.

### Anti-pattern 12 — Optimizing an undeclared mental model

A workflow or tooling session loops through multiple iterations, re-derives
decisions the human already implicitly holds, and takes significantly longer
than the problem warranted.

Root cause: the existing mental model and existing habits were never stated
at session start. The session spent time discovering what the human already
knew rather than building on it.

Concrete example: a token optimization session that spent the first half
deriving a "workspace model" the human was already using — pasting project
context per session, one Claude project per technology domain. Stating that
habit at session start would have collapsed the discovery phase entirely.

**Rule:** Before any session involving workflow, tooling, or process
optimization — state the current model explicitly:

```
Current habit:   what I already do
Current problem: what specifically is painful
Constraint:      what must not change
Goal:            what done looks like
```

This four-line declaration collapses the discovery phase and lets the
session start from the correct baseline. If the human already has a working
mental model and working habits, the session optimizes them — it does not
re-derive them.

**Applies to:** any session involving workflow restructuring, tooling
optimization, process redesign, knowledge base maintenance, or any session
where the human has prior domain context the AI should build on rather
than rediscover.

---

## Gap Analysis Evidence Protocol — Extended Examples

### Example 1 — Yocto BSP (branch name ≠ branch content)

*Failure:*
```
Gap: oe-core is on 'master' but meta-qcom requires 'whinlatter'.
Fix: git checkout whinlatter
```

*Correct:*
```
GAP-1: [INFERRED] oe-core branch may not satisfy LAYERSERIES_COMPAT

  Observed:  oe-core branch = "master"; meta-qcom LAYERSERIES_COMPAT = "whinlatter"
  Concluded: Branch name does not match compat string
  Assumed:   'master' does not carry Whinlatter content internally
  Uncertainty: Vendor workspace may use 'master' as Whinlatter tracking branch
  Verify with: grep "LAYERSERIES_CORENAMES\|whinlatter" oe-core/meta/conf/layer.conf
  Act only after: output confirms whether 'master' carries whinlatter content
```

### Example 2 — Firmware porting (function signature from training data)

*Failure:*
```
Gap: platform is missing tfm_plat_get_huk(uint8_t *key, uint32_t size).
Fix: implement that signature in crypto_keys.c
```

*Correct:*
```
GAP-3: [ASSUMED] Platform may be missing the HUK retrieval interface

  Observed:  crypto_keys.c is not present in the platform directory
  Concluded: HUK interface is unimplemented
  Assumed:   tfm_plat_get_huk(uint8_t *key, uint32_t size) is the correct
             interface — based on AI training data for TF-M
  Uncertainty: TF-M v2.3 may use a different interface pattern entirely
  Verify with: grep -n "tfm_plat.*huk\|huk\|builtin_key" \
                 platform/include/tfm_plat_crypto_keys.h
  Act only after: actual header confirms function name and signature
```

### Evidence Audit Anti-Pattern — Confident Prose Without Audit

**Wrong:** Generating a platform stack table with "Dom0: Linux RT-PREEMPT" and no audit.

**Correct:**
```
Evidence Audit — Platform Stack Table

Claim: Dom0 runs Linux RT-PREEMPT
Tier:  ASSUMED
Source: Training data — common Xen Dom0 pattern. No AGL SoDeV doc confirms RT config.
Verify with: uname -r in Dom0 after first boot
             zcat /proc/config.gz | grep CONFIG_PREEMPT_RT
Risk if wrong: Dom0 interference model in latency measurement is incorrect
```

---

## Known AI Failure Modes — Detail

**Version drift:** Training data reflects older versions. Verify signatures from actual headers.

**Plausible but wrong:** Architecturally coherent answers that don't match your configuration.
Contract tracing produces these. Sound authoritative. May be wrong for your specific version.

**Missing implicit dependencies:** AI misses typedef, macro, or #include requirements only
visible by reading actual source.

**Security oversimplification:** AI may omit security-critical constraints (key material
handling, power-loss atomicity) not in training data for the specific platform.

**Generalization from reference:** AI answers based on a reference platform (STM32, nRF)
that differs from your target in a security-relevant way.

**Certification level conflation:** AI conflates PSA-L1/L2/L3 requirements because they
are adjacent concepts in training data. Always verify which level a requirement belongs to
by checking the applicable protection profile document directly.

**Pattern-match without verification:** AI observes a partial signal (branch name, filename,
version string) and states a conclusion as if directly observed. Withhold the conclusion
until a verification command is run.

**Misreading under pattern-matching pressure:** AI reads string X and pattern-matches to
close-but-different string Y. Prevention: quote the exact string from source when stating
any conclusion that depends on a specific value.

**Layer capability blindness:** AI generates code without reading upstream layer's actual
capability surface. Functionally correct, architecturally wrong — reinvents infrastructure
the upstream layer already provides.

**Expedient pattern replication:** AI reproduces statistically common but architecturally
wrong patterns from training data (DISTRO="poky" in production BSPs, etc.).

**Abstraction layer confusion:** AI places configuration at wrong abstraction layer.
Produces correct output regardless of layer placement. Only detected in architectural review.

---

## Domain Architecture Documents — Full Reference

| Document | Domain | Applies when |
|---|---|---|
| `YOCTO_BSP_ARCH.md` | Yocto BSP | recipes, kas, BitBake, meta-layer, SRCREV, kernel patches, DTS, firmware staging |
| `FIRMWARE_PORT_ARCH.md` | Secure Firmware | TF-M, PSA cert, CMSIS HAL, secure boot, attestation, crypto keys, NV counters, ITS/PS, SFN/IPC |
| `SECURITY_ARCH_KNOWLEDGE.md` | Embedded Security Architecture | PSA scope, CRA, secure storage, key derivation, AES-GCM-SIV, trust model, threat model, JSADEN |
| `RTOS_BENCH_ARCH.md` | RTOS Benchmark | WCET, OS primitive latency, IRQ latency, VirtIO, PMU attribution, mixed criticality |
| `BOOT_CHAIN_ARCH.md` | Boot Chain | UEFI, bootloader, secure boot, device provisioning, version lock — 🔲 Planned |
| `PCIE_BSP_ARCH.md` | PCIe and SMMU | PCIe switch, iommu-map, Stream IDs, DTS PCIe nodes — 🔲 Planned |

### Adding a New Domain Architecture Document

1. Unifying meta-principle for the domain
2. Minimum 3 categorical domains, 2 principles per domain, 1 anti-pattern per principle
3. Self-extension mechanism pointer
4. Update KNOWLEDGE_REGISTRY.md domain table and status table
5. Bump registry version, add changelog entry

---

## Methodology Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-29 | Initial capture from TF-M PSA-L2 firmware porting project |
| 1.1 | 2026-04-29 | Pre-implementation checklist, 7 anti-patterns, architect/implementer asymmetry |
| 1.2 | 2026-05-03 | Knowledge Source Hierarchy — 5-tier reading order and TF-M document map |
| 1.3 | 2026-05-04 | Four-Role model. SDK/Framework Reading Strategy. Closed-Source Boot Stage Blindspot. Anti-patterns 8–9. |
| 1.4 | 2026-05-05 | Anti-patterns 10–11: cold review, Gemini false-positive rate (60%) |
| 1.5 | 2026-05-08 | Gap Analysis Evidence Protocol. Two new Known AI Failure Modes. |
| 1.6 | 2026-05-21 | Domain Architecture Documents framework. YOCTO_BSP_ARCH.md reference. Three new Known AI Failure Modes. |
| 1.7 | 2026-05-22 | Destructive operation audit added to pre-implementation checklist. |
| 1.8 | 2026-05-25 | Evidence Tier Protocol for Generated Content. Two presentation modes. |
| 1.8-ref | 2026-05-27 | Split from METHODOLOGY.md. Reference content only. Active rules in METHODOLOGY_RULES.md. |

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

---

## Known AI Failure Modes — Summary

*(Detail in section above. This summary is the quick-reference form.)*

- **Version drift** — training data reflects older versions; verify signatures from actual headers
- **Plausible but wrong** — architecturally coherent answers that don't match your configuration
- **Missing implicit dependencies** — typedef, macro, #include requirements only visible in source
- **Security oversimplification** — omits security-critical constraints not in training data
- **Certification level conflation** — conflates PSA-L1/L2/L3; verify against protection profile
- **Pattern-match without verification** — states conclusion from partial signal as if observed
- **Misreading under pattern-matching pressure** — reads string X, reports Y; quote exact strings
- **Layer capability blindness** — generates code without reading upstream layer capability surface
- **Expedient pattern replication** — reproduces statistically common but architecturally wrong patterns
- **Abstraction layer confusion** — places configuration at wrong abstraction layer

---

## Session Prompt Template (Claude Code CLI)

Use at the start of every Claude Code implementation session:

```
Read CLAUDE.md first, then context.md fully.
Summarize: current state, open issues, today's session goal.

Session N goal: implement [specific file] per PPS-[N].
Scope: ONLY modify [specific file]. Do not touch [other files].
After implementing: run cmake --build and report results.
If build succeeds: show me the generated file for review.
If build fails: fix compiler errors only, do not change design.
If design decision needed not in spec: STOP and report.
Report when done.
```

---

## Domain Activation Examples

### Single domain

Session goal: "Write a firmware staging recipe for the board."

Active: **Yocto BSP** → load `YOCTO_BSP_ARCH.md`

Constraints applied before generating:
- Read upstream firmware common .inc file before writing new recipe
- Confirm no existing bbclass handles the staging task

---

### Multiple domains

Session goal: "Debug the SMMU fault during PCIe enumeration, then fix
the iommu-map in the carrier DTS bbappend."

Active:
- **PCIe and SMMU** → load `PCIE_BSP_ARCH.md`
- **Yocto BSP** → load `YOCTO_BSP_ARCH.md`

Constraints applied:
- Verify SID values against source before modifying iommu-map
- Confirm DTS change is in correct per-subsystem dtsi fragment

---

### Domain shift mid-session

Session starts as boot chain debugging. User then asks to add a kernel
patch via bbappend.

New domain detected: **Yocto BSP** → load `YOCTO_BSP_ARCH.md`

Constraint applied immediately:
- Topic branch structure must exist before any patch is added
- Never generate SRC_URI patch entries — even the first one

---

## Maintenance Protocols

### Protocol 1 — Updating an Existing Document

**Triggers:** New principle, correction, improved rationale, post-mortem finding.

**What changes:** Document file only. Bump internal version (1.0 → 1.1). Add changelog entry.

**What does NOT change:** Filename, KNOWLEDGE_REGISTRY.md, project instruction.

**Claude project update:** Replace the file. Registry and instruction unchanged.

**Commit format:**
```
docs(yocto-bsp-arch): add principle D6 — lock file commit discipline

Source: project X session — kas lock files not committed, causing
non-reproducible builds. Generalised to domain principle D6.
```

---

### Protocol 2 — Adding a New Domain

**Triggers:** Project encounters a technology domain not covered by any existing domain document.

**Step 1 — Create the domain document** following existing `*_ARCH.md` structure:
```markdown
# [Domain Name] Architecture Principles
**Version:** 1.0

## [Unifying meta-principle]

## Domain A — [First categorical domain]
### A1 — [First principle]
**Rule:** ...
**Rationale:** ...
**Anti-pattern:** ...
**Correct pattern:** ...
**Verification:** ...

## Maintenance Protocol Reference
See KNOWLEDGE_REGISTRY.md — Maintenance Protocols in METHODOLOGY_REF.md.
```

Minimum for v1.0: 3 categorical domains, 2 principles per domain, 1 anti-pattern per principle.

**Step 2 — Update KNOWLEDGE_REGISTRY.md:**
- Add row to Domain Registry table
- Add row to status table
- Bump registry version, add changelog entry

**What does NOT change:** METHODOLOGY_RULES.md, METHODOLOGY_REF.md, existing domain documents, project instruction.

**Commit format:**
```
feat(registry): add Boot Chain domain — BOOT_CHAIN_ARCH.md v1.0
```

---

### Protocol 3 — Adding a Principle to an Existing Domain

**Decision gate before adding:**
1. Does this generalise beyond this specific project? If no → PROJECT_CONTEXT.md only.
2. Which domain owns this?

**Five-step self-extension process:**

1. State the symptom precisely — quote the exact error, comment, or failure
2. Identify the root cause category — which domain section owns it
3. Generalise — would this principle prevent the same mistake on a different project?
4. Write the principle entry:
```markdown
### X.N — Principle Name
**Rule:** One clear imperative sentence.
**Rationale:** Why this matters. What goes wrong without it.
**Anti-pattern:** Concrete wrong example.
**Correct pattern:** Concrete right example.
**Verification:** Command or question to confirm compliance.
```
5. Update domain document — bump version, add changelog entry

**What changes:** Domain document only. Registry, methodology, project instruction unchanged.

---

### Protocol 4 — Graduating a Planned Domain to Available

Follow Protocol 2 steps except the registry table row already exists.
Update status table row from 🔲 to ✅. Bump registry version, add changelog.

---

### Version Numbering Policy

| Document | Bump minor when | Bump major when |
|---|---|---|
| `KNOWLEDGE_REGISTRY.md` | Domain added or status changed | Structural redesign |
| Domain architecture docs | Principle added or improved | Section reorganisation |
| `METHODOLOGY_RULES.md` | Standing rule added or improved | Framework restructure |
| `METHODOLOGY_REF.md` | Reference content added or improved | Framework restructure |
| `PROJECT_CONTEXT.md` | Every change notice (CN number) | N/A |

Filenames never contain version numbers.

---

### Commit Message Convention

```
type(scope): short description

Body: which project or session produced the finding, why generalised.
```

Types: `feat` (new domain/principle), `docs` (improved content),
`fix` (correction), `refactor` (restructuring without meaning change)

Scopes: `registry`, `methodology`, `yocto-bsp-arch`, `boot-chain-arch`,
`pcie-bsp-arch`, `ethernet-bsp-arch`, `usb-storage-arch`,
`firmware-port-arch`, `uboot-arch`, `zephyr-arch`, `rtos-bench-arch`,
`security-arch-knowledge`

---

### Knowledge Base vs Project Context Boundary

**Belongs in knowledge base:**
- Principles that generalise across multiple projects
- Anti-patterns applicable to any project in the same domain
- Structural constraints inherent to the technology

**Belongs in PROJECT_CONTEXT.md only:**
- Project-specific open items (OI-NNN register)
- Board-specific hardware topology decisions
- Customer-specific configuration choices
- Temporary workarounds with a defined removal condition
- Binary versions and checksums specific to this board

**Generalisation test:** Would this finding prevent the same mistake on
a completely different project using the same technology?
Yes → knowledge base. No → project context only.

---

## Methodology Backlog

Items are project-agnostic pending improvements. Reviewed at start of any
methodology or knowledge base session. Each item is acted on or explicitly
deferred. Project-specific items belong in PROJECT_CONTEXT.md.

---

**MB-001 — Create FIRMWARE_PORT_ARCH.md** ✅ COMPLETED May 2026

---

**MB-002 — Add CRA/PSA boundary principle to FIRMWARE_PORT_ARCH.md**

Status: READY
Source: EN 304 623 + EN 304 626 joint analysis (May 2026)
Principle: CRA compliance does not establish principled trust isolation.
Boot-to-OS handoff interface undefined in both standards. PSA closes the gap.
Target: FIRMWARE_PORT_ARCH.md Domain A

---

**MB-003 — Add TR-MISO trust model gap principle to FIRMWARE_PORT_ARCH.md**

Status: READY
Source: EN 304 626 analysis (May 2026)
Principle: TR-MISO is mechanism-first without a trust model. Product can
pass TR-MISO while having unprincipled NS-to-S boundary.
Target: FIRMWARE_PORT_ARCH.md Domain A

---

**MB-004 — Create SECURITY_ARCH_KNOWLEDGE.md** ✅ COMPLETED May 2026 (v0.1)

---

**MB-005 — Add knowledge layer boundary discipline to METHODOLOGY_RULES.md**

Status: READY
Principle: Three-layer boundary — Layer 0 (retrieved at query time),
Layer 1 (loaded at session start), Layer 2 (LLM training weights).
Only put in Layer 1 what is neither retrievable on demand nor reliably trained.
Target: METHODOLOGY_RULES.md

---

**MB-006 — Add BOOT_CHAIN_ARCH.md**

Status: PARTIALLY READY — write v1.0 when a boot chain project is active.
Source material: EN 304 623 trust boundaries, embedded Linux boot chain.
Target: BOOT_CHAIN_ARCH.md v1.0 (Protocol 4 graduation)

---

**MB-007 — Add Anti-pattern 12 to METHODOLOGY_REF.md**  ✅ COMPLETED May 2026

See Completed Items table below.

---

### Completed Items

| Item | Completed | Promoted to |
|---|---|---|
| MB-001 | May 2026 | FIRMWARE_PORT_ARCH.md v1.0 |
| MB-004 | May 2026 | SECURITY_ARCH_KNOWLEDGE.md v0.1 |
| MB-007 | May 2026 | METHODOLOGY_REF.md Anti-pattern 12 — undeclared mental model |

---

### Backlog Maintenance Rules

1. Add items in the same session they are identified — do not defer capture
2. Remove items when corresponding document update is committed
3. Any session activating METHODOLOGY_RULES.md should scan for unblocked items
4. Generalisation test is the admission gate

---

## Registry Changelog History

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial release. Six domains registered. YOCTO_BSP_ARCH available. |
| 1.1 | May 2026 | Version-free filenames. PROJECT_CONTEXT.md convention. |
| 1.2 | May 2026 | Maintenance Protocols 1–4. Version numbering policy. Commit convention. |
| 1.3 | May 2026 | Added U-Boot and Zephyr as planned domains. YOCTO_BSP_ARCH v1.2. |
| 1.4 | May 2026 | Added RTOS Benchmark domain (RTOS_BENCH_ARCH.md v1.4). |
| 1.5 | May 2026 | Added Methodology Backlog. MB-001 through MB-006 registered. |
| 1.6 | May 2026 | Graduated FIRMWARE_PORT_ARCH.md to Available. Added SECURITY_ARCH_KNOWLEDGE.md planned. |
| 1.7 | May 2026 | Split METHODOLOGY.md → METHODOLOGY_RULES.md + METHODOLOGY_REF.md. ~58% unconditional load reduction. |
| 1.8 | May 2026 | Four-tier workspace model. Session workflow table. Workspace setup examples. PROJECT_CONTEXT.md session-scoped. |
| 1.9 | May 2026 | Trim pass. Maintenance protocols, backlog, changelog, domain activation examples, standing rules duplicate all moved to METHODOLOGY_REF.md. Registry contains operational content only. |

---

## METHODOLOGY_REF.md Changelog

| Version | Date | Change |
|---|---|---|
| 1.8-ref | 2026-05-27 | Initial split from METHODOLOGY.md v1.8 |
| 1.9-ref | 2026-05-27 | Added: Known AI Failure Modes summary, Session Prompt Template, Domain Activation Examples, Maintenance Protocols 1–4, Methodology Backlog, Registry Changelog history. Absorbs all content removed from METHODOLOGY_RULES.md v1.9 and KNOWLEDGE_REGISTRY.md v1.9 trim pass. |
| 1.10-ref | 2026-05-27 | Added Anti-pattern 12 — undeclared mental model. Added MB-007 to backlog (completed). Source: session retrospect on workflow optimization session. |
| 1.11-ref | 2026-05-28 | Changelog entry for METHODOLOGY_RULES.md v2.0-rules: Rule 2 activation broadened from content-type whitelist to principle-based trigger covering all analytical output; self-report obligation added for omissions; external deliverable gate made explicit with human sign-off; Rule 1 and Rule 2 explicitly linked. Phase Structure trimmed to generic discipline — TF-M phase content already in FIRMWARE_PORT_ARCH.md Domain E. CLAUDE_PROJECT_INSTRUCTION.md updated: stale METHODOLOGY.md reference corrected to METHODOLOGY_RULES.md; Rule 2 enforcement trigger added to project instruction. |

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

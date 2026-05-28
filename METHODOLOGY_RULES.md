<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# AI-Assisted Embedded Systems Engineering Methodology — Rules

**Version:** 1.9-rules
**Split from:** METHODOLOGY.md v1.8 (2026-05-27)
**Trimmed:** 2026-05-27
**Authors:** Lei Zhou + Claude (Anthropic)
**Scope:** Active session rules. Load every session.
**Reference companion:** METHODOLOGY_REF.md — anti-patterns, examples, detailed protocols, session prompt template.

---

## The Four-Role Model

**Role 1 — The Architect (Human, HIL)**
Final decisions and ground truth. Never writes code directly.
HIL constraint is absolute. SSoT updated only on explicit human confirmation.

**Role 2 — The Design AI (Web Chat)**
What and why. Never writes implementation code directly.
Interface contracts, threat modeling, design decisions, Category 1 files,
cross-verifying Gemini outputs against normative sources.

**Role 3 — The Cross-Verifier (Gemini)**
Second opinions on design claims. Never makes final decisions.
All Gemini outputs reviewed critically. Known failure modes checked every output.
Never use a Gemini-provided function signature without grepping the actual header.
When Gemini and web chat disagree: verify both against source. Trust source.

**Role 4 — The Implementer (Claude Code CLI)**
How and execution. Never makes design decisions.
If a design decision is needed not covered by specification: STOP and report
to architect. Never guess. An incorrect architectural guess is harder to
detect and fix than a compile error.

---

## Four Standing Rules — Always Active

**Rule 1 — Gap Analysis Evidence Protocol**

Every technical finding carries an explicit evidence tier:

- **[VERIFIED]** — directly supported by output the human pasted, a file read,
  or a command whose result is in the current session. Act immediately.
- **[INFERRED]** — follows logically from verified evidence but involves
  reasoning steps not directly confirmed. Verify before acting.
- **[ASSUMED]** — rests on general knowledge, not project-specific observation.
  Must be verified before acting. Must not appear in external deliverables as fact.

Required format for every gap or blocker:
```
GAP-N: [TIER] Short description

  Observed:    What the output or file literally shows (quoted exactly)
  Concluded:   What is inferred from it
  Assumed:     What is filled in from general knowledge (if anything)
  Uncertainty: What could invalidate this if the assumption is wrong
  Verify with: <exact command to run on the target system>
  Act only after: confirmation that verify output supports the gap
```
VERIFIED gaps may omit the Verify step.

**Rule 2 — Evidence Tier Protocol for Generated Content**

After generating any section of technical content — table, prose paragraph,
specification claim, architecture description, deliverable section — produce
an Evidence Audit immediately, unprompted:

```
Evidence Audit — [Section name]

Claim: [exact claim from generated content]
Tier:  [VERIFIED | INFERRED | ASSUMED]
Source: [what the tier is based on]
If INFERRED or ASSUMED:
  Verify with: [exact command or source to confirm]
  Risk if wrong: [what breaks if this claim is incorrect]
```

Two presentation modes:
- Internal documents: evidence tiers inline (`[VERIFIED]`, `[INFERRED]`, `[ASSUMED]`)
- External deliverables: companion Evidence Audit section, reviewed before release

Protocol activates on: hardware specs, OS/kernel version claims, standards
compliance assertions, architecture descriptions, timing/latency claims,
API behaviour claims, platform capability claims.
Does NOT activate on: pure methodology discussion, restating verified project
context from PROJECT_CONTEXT.md, mathematical derivations from provided data.

**Rule 3 — Pre-implementation gate**

Do not generate any implementation until ALL of the following are confirmed:
- Exact function signatures from actual source headers (not AI training data)
- Boot sequence call ordering verified from main.c with line numbers
- All `#error`-guarded defines confirmed present in platform headers
- Execution context (thread vs ISR, blocking allowed or not) confirmed
- Temporal ordering constraints confirmed from source, not inferred
- If adopting any upstream component: audit for destructive operations
  (-delete, rm, mv) and verify path ownership before implementation

How to satisfy this gate — hybrid call graph process:
1. Contract tracing (top-down from specs) — establishes architectural skeleton
2. Source verification (bottom-up from actual files) — confirms version-specific reality
3. Discrepancy resolution — source diverges from contract → trust source, document
4. Sequence diagram production — verified call graph → implementation spec

**Rule 4 — Four-role boundary**

Design AI (this session) defines what and why.
Implementer (Claude Code CLI) executes how.
Never mix roles. If a design decision is needed during implementation,
stop, resolve in design session, then resume implementation.

---

## Pre-Implementation Checklist (hard stops)

```
Interface contracts
□ Exact function signatures from actual source headers (not AI training data)
□ Return types confirmed — FIH wrappers, int vs enum, etc.
□ Header file declaring each interface identified
□ File implementing each interface identified

Behavioral contracts — all 5 dimensions per function
□ Postconditions
□ Invariants
□ Error handling
□ Prohibitions
□ Side effects

Platform implementation path
□ Specific SDK/HAL function to call
□ Hardware preconditions confirmed
□ Phase 1 stub path vs Phase N real path both specified
□ If adopting upstream component: destructive operation audit complete

Execution context
□ Thread vs interrupt context confirmed
□ Blocking permission confirmed
□ Re-entrancy confirmed
□ IRQ protection required or not confirmed

Temporal ordering
□ Boot sequence verified from main.c source (not inferred)
□ Security lock ordering verified
□ UART init placement confirmed relative to first logging call

Build system
□ All #error-guarded defines confirmed present
□ All cmake variables confirmed present
□ Linker script approach confirmed
□ All source files in CMakeLists.txt match file manifest
```

---

## Phase Structure

**Phase 1** — Compile clean with 100% stubs. Zero errors, zero warnings.
**Phase 2** — First boot. UART output. NS world entry.
**Phase 3** — Security functions. Real crypto, NV counters, OTP.
**Phase 4** — Adversarial testing. Power-loss, replay, isolation.

Phase discipline:
- Never implement Phase N+1 while Phase N has unresolved errors
- Each phase ends with human review before proceeding
- Stubs marked: `/* PHASE 1 STUB — replace in Phase N per PPS-X */`
- A phase is complete only when the human architect signs off

---

## SSoT Discipline

- Update SSoT after every CLI session before starting the next
- CLI agent reads SSoT at start of every session — enforce this
- Never allow design decisions to exist only in chat history
- SSoT wins over chat history if they conflict

---

## Domain Files

Domain constraints are in workspace Tier 2 files and on-demand Tier 4 files.
See KNOWLEDGE_REGISTRY.md for the full domain routing table, trigger keywords,
and file availability status.

If a domain file is not available and the session requires it: state this
explicitly and apply the pre-implementation checklist above as a fallback.

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*
*Full methodology with anti-patterns, examples, and detailed protocols: METHODOLOGY_REF.md*

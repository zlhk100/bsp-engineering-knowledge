<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# The AI-Assisted Engineering Pattern

**Version:** 1.0
**Date:** May 2026
**Authors:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/<your-org>/bsp-engineering-knowledge

---

## What This Document Is

This document explains a reusable workflow pattern for AI-assisted engineering
on complex technical projects. It captures how to structure knowledge, roles,
and sessions so that an AI code generation tool produces reliable, correct,
architecturally sound output — even in domains where AI training data is
insufficient.

The pattern was developed and validated on real embedded systems engineering
projects: a TF-M PSA-L2 security firmware port and a Qualcomm SoC BSP
bring-up. It applies to any project with these characteristics:

- The target system has hardware-specific or domain-specific constraints
  not well-represented in AI training data
- Correctness failures have real consequences (boot failures, data loss,
  security vulnerabilities, regression)
- A complex existing framework is being ported, integrated, or extended
- An AI code generation tool (Claude Code or equivalent) is used for
  implementation

The pattern is **not** a prompting guide. It does not tell you how to write
better prompts. It tells you how to structure knowledge so that AI behaviour
is reliably constrained across sessions, engineers, and projects.

---

## The Core Problem

AI code generation fails systematically in complex technical domains for
three reasons:

**Reason 1 — Training data gaps.**
AI models are trained on publicly available code and documentation. They
know general patterns well. They do not know the specific constraints of
your upstream layer at its current commit, your hardware topology, your
vendor's undocumented SBL exit state, or the structural rules your
organisation has learned from past failures. They fill these gaps with
plausible-but-wrong answers that compile, pass initial tests, and fail
at integration or in production.

**Reason 2 — Role boundary violations.**
When an AI is asked to both design and implement simultaneously, it fills
specification gaps with implementation guesses. Those guesses compile.
They pass Phase 1. They fail at Phase 2 or Phase 3 where rework cost is
highest. The cost of a specification gap is not the cost of fixing it in
a design session (minutes) — it is the cost of discovering and debugging
the consequence in an implementation session (hours).

**Reason 3 — Knowledge decay across sessions.**
AI models have no persistent memory. Every session starts from zero. On
long projects this means the same decisions get re-derived, the same
mistakes get repeated, and the same constraints get forgotten. Without a
structured mechanism to load prior knowledge at session start, the AI
operates as if each session is the first.

The pattern addresses all three reasons with a four-layer architecture.

---

## The Four-Layer Architecture

Every project using this pattern is structured into four layers. Each
layer has a defined owner, a defined scope, and a defined update protocol.

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1 — Universal Methodology                        │
│  METHODOLOGY.md                                         │
│  Owner: knowledge base maintainers                      │
│  Changes: when new process patterns are discovered      │
│  Scope: any project, any domain                         │
├─────────────────────────────────────────────────────────┤
│  Layer 2 — Domain Architecture Principles               │
│  [DOMAIN]_ARCH.md  (e.g. YOCTO_BSP_ARCH.md)            │
│  Owner: domain experts + knowledge base maintainers     │
│  Changes: when new domain principles are discovered     │
│  Scope: any project in this technology domain           │
├─────────────────────────────────────────────────────────┤
│  Layer 3 — Project Context                              │
│  PROJECT_CONTEXT.md                                     │
│  Owner: project team                                    │
│  Changes: every session (via change notices)            │
│  Scope: this project only                               │
├─────────────────────────────────────────────────────────┤
│  Layer 4 — Session Execution                            │
│  CLAUDE.md + Claude Code CLI                            │
│  Owner: implementer (AI agent)                          │
│  Changes: per task                                      │
│  Scope: this session only                               │
└─────────────────────────────────────────────────────────┘
```

Higher layers constrain lower layers. Layer 1 principles apply in every
session. Layer 2 principles apply in every session involving that domain.
Layer 3 is the project-specific single source of truth. Layer 4 executes.

**The key insight:** Layers 1 and 2 are shared across all projects. Once
written, they apply everywhere. The investment in writing them pays
compounding returns across every future project in the same domain.

---

## The Four-Role Model

Four roles participate in the workflow. Role boundaries are strict — mixing
them is the most common cause of failure.

### Role 1 — The Architect (Human, Hardware-in-the-Loop)

Responsible for **final decisions and ground truth**.

- Confirms every design decision before it enters the project context
- Provides hardware output (UART logs, register dumps) for verification
- Signs off each implementation session before the next begins
- Has standing authority to override any AI inference based on direct
  observation of the system

**The HIL constraint is absolute.** The project context is updated only
on explicit human confirmation. No design decision is final until the
human says so. Direct human observation of the system overrides AI
inference.

### Role 2 — The Design AI (Web Chat Session)

Responsible for **what** and **why**. Never writes implementation code.

- Maps hardware topology to firmware/BSP abstractions
- Defines interface contracts and behavioral specifications
- Performs root cause analysis when implementation hits blockers
- Reads and synthesizes upstream documentation and source
- Maintains the project context document via change notices
- Identifies when domain architecture documents need updating

### Role 3 — The Cross-Verifier (Second AI or Independent Session)

Responsible for **second opinions**. Never makes final decisions.

- Provides alternative analysis on design questions
- Performs cold review of generated files before placement
- Identifies gaps the design AI may have missed

**Critical constraint:** Cross-verifier outputs are hypotheses, not facts.
Every finding requires source verification before acting on it. Observed
false-positive rate in practice: 60% on complex technical questions.
Cold review is valuable; blind acceptance of its output is not.

### Role 4 — The Implementer (Claude Code CLI)

Responsible for **how** and **execution**. Never makes design decisions.

- Translates specifications into source files
- Resolves compiler and build errors within session scope
- Follows the session sequence exactly as specified
- Stops and reports when a decision point is not covered by the spec

**The boundary is strict.** If the implementer hits a decision point not
in the specification, it stops and reports to the architect. It does not
guess. An incorrect architectural guess by the implementer is harder to
detect and fix than a compile error.

---

## The Knowledge Base Structure

The knowledge base is a set of version-controlled Markdown documents.
All documents use version-free filenames — versions live inside the
document header, not in the filename.

### Required files — every project

```
KNOWLEDGE_REGISTRY.md    ← domain router and maintenance protocol
METHODOLOGY.md           ← universal engineering process
PROJECT_CONTEXT.md       ← project-specific single source of truth
```

### Domain files — add per project as needed

```
YOCTO_BSP_ARCH.md        ← Yocto-based BSP bring-up
PCIE_BSP_ARCH.md         ← PCIe switch and SMMU integration
ETHERNET_BSP_ARCH.md     ← Ethernet PHY and MAC bring-up
USB_STORAGE_ARCH.md      ← xHCI, USB bridges, NVMe, storage
BOOT_CHAIN_ARCH.md       ← bootloader bring-up and chain of trust
FIRMWARE_PORT_ARCH.md    ← security firmware porting (TF-M etc.)
```

Only upload domain files relevant to the active project. The registry
references all domains — Claude skips files that are not uploaded and
states this explicitly.

### CLAUDE.md — for Claude Code sessions

A `CLAUDE.md` file in the workspace root gives Claude Code its standing
instructions. It is read automatically at every Claude Code session start.

```markdown
## Read before doing anything else — mandatory, in this order
1. Read KNOWLEDGE_REGISTRY.md in this directory
2. Read PROJECT_CONTEXT.md in this directory
3. Read METHODOLOGY.md in this directory
4. Read [DOMAIN]_ARCH.md for each active domain in this session

## Standing constraints
- [Project-specific constraints here]
- Show proposed change and wait for approval before applying
- If a design decision arises not covered by the session spec, STOP and report
```

---

## Setting Up a New Project

Complete these steps before the first technical session.

### Step 1 — Clone the knowledge base template

```bash
git clone https://github.com/<your-org>/bsp-engineering-knowledge
```

### Step 2 — Identify active domains

Read the project brief. Which domains are active?

| Project type | Domain files needed |
|---|---|
| Yocto BSP bring-up | `YOCTO_BSP_ARCH.md` |
| PCIe device integration | `PCIE_BSP_ARCH.md`, `YOCTO_BSP_ARCH.md` |
| Security firmware port | `FIRMWARE_PORT_ARCH.md` |
| Full BSP with storage and networking | `YOCTO_BSP_ARCH.md`, `PCIE_BSP_ARCH.md`, `ETHERNET_BSP_ARCH.md`, `USB_STORAGE_ARCH.md` |

### Step 3 — Create PROJECT_CONTEXT.md

Copy `templates/PROJECT_CONTEXT_template.md` and fill in all sections.
The template has placeholders for every required section. Do not start
technical sessions until the project context is complete enough to answer:

- What is the hardware topology?
- What is the build system and SRCREV?
- What are the P1 blockers?
- What milestone is active?

### Step 4 — Set up the Claude project

In Claude.ai, create a new project. Upload:
- `KNOWLEDGE_REGISTRY.md`
- `METHODOLOGY.md`
- `PROJECT_CONTEXT.md`
- All relevant domain files

Set the project instruction to exactly this text (copy verbatim, never change):

```
At the start of every session:
1. Read KNOWLEDGE_REGISTRY.md using the view tool
2. Read PROJECT_CONTEXT.md using the view tool
3. Read METHODOLOGY.md using the view tool
4. Scan the session goal and conversation for active domains
5. Read each active domain file listed in the registry using the view tool
Apply all standing rules and domain constraints before responding to
any technical question.
```

### Step 5 — Set up CLAUDE.md for Claude Code

Copy `templates/CLAUDE.md_template` to your workspace root and fill in
the project-specific constraints section. Place the knowledge base
documents in the same directory.

```bash
cp /path/to/knowledge-base/*.md ~/workspace/your-project/
cp templates/CLAUDE.md_template ~/workspace/your-project/CLAUDE.md
# Edit CLAUDE.md to add project-specific constraints
```

### Step 6 — Verify before first session

Run this checklist before starting any technical work:

```
□ PROJECT_CONTEXT.md has hardware topology section filled in
□ PROJECT_CONTEXT.md has build system and SRCREV documented
□ PROJECT_CONTEXT.md has open items register started
□ All domain files uploaded to Claude project
□ Project instruction set verbatim (not paraphrased)
□ CLAUDE.md in workspace root with project constraints
□ Knowledge base .md files in workspace root for Claude Code
```

---

## Running a Session

### Web chat session (design work)

Every session starts with the knowledge base load. The project instruction
triggers this automatically in Claude.ai projects. After loading, Claude
summarises current project state and waits for the session goal.

**The pre-implementation gate:**
Before any implementation work begins, confirm all items in the
pre-implementation checklist (METHODOLOGY.md §Pre-implementation checklist)
are satisfied. If any item is unknown, resolve it in the design session
before handing to Claude Code. The cost of a specification gap is not
the design session time — it is the implementation debugging time.

**Session goal statement:**
State the session goal explicitly at the start. Claude activates the
relevant domain files based on the goal.

**Change notices:**
Every design decision made in a session is captured in a numbered change
notice (CN-NNN) and applied to PROJECT_CONTEXT.md. No decision lives
only in chat history.

### Claude Code session (implementation work)

**Session prompt template:**
```
Read CLAUDE.md first, then follow its reading order exactly.
After reading all knowledge base docs, summarise:
- Current project state
- Today's session goal
- Which domain principles are active

Session goal: [specific implementation task]
Scope: ONLY modify [specific file(s)].
Show proposed change and wait for approval before applying each fix.
Do not run [build system] unless explicitly instructed.
If a design decision is needed not covered by this spec, STOP and report.
```

**During the session:**
- Claude Code reads upstream layer files before generating any code (F1)
- Claude Code applies the domain checklist before generating any file (F3)
- Claude Code reports findings with VERIFIED/INFERRED/ASSUMED tiers
- Claude Code stops and escalates any decision point not in the spec

**Escalation path:**
If Claude Code hits a decision point → stop → report to architect in web
chat → architect resolves → updates PROJECT_CONTEXT.md → new Claude Code
session with the decision documented.

**Session close:**
After implementation is verified (build clean, tests pass, board verified):
1. Human reviews all changes against the specification
2. Human updates PROJECT_CONTEXT.md via change notice
3. Changes committed and pushed
4. Next session starts from the updated SSoT

---

## The Gap Analysis Evidence Protocol

All technical findings — whether from Claude, Claude Code, or the
cross-verifier — must carry an explicit evidence tier. This prevents
confident-but-wrong gap claims from triggering unnecessary corrective
actions.

```
GAP-N: [VERIFIED|INFERRED|ASSUMED] Short description

  Observed:    What the output or file literally shows (quoted exactly)
  Concluded:   What is inferred from it
  Assumed:     What is filled in from general knowledge (if anything)
  Uncertainty: What could invalidate this if the assumption is wrong
  Verify with: <exact command to run>
  Act only after: confirmation that verify output supports the gap
```

**VERIFIED** — directly supported by output in this session. Act immediately.
**INFERRED** — follows from verified evidence with reasoning steps. Verify first.
**ASSUMED** — rests on general knowledge, not project-specific observation.
Highest risk. Must be verified before acting.

The human has standing authority to push back on any gap claim. If a gap
does not match the human's direct observation of the system, the AI
re-examines from scratch — it does not defend the original claim.

---

## Evolving the Knowledge Base

The knowledge base grows as projects surface new principles. The growth
is structured — new findings slot into the correct domain rather than
accumulating as a flat list.

### When to add a principle

A new principle belongs in a domain architecture document when:
- It generalises beyond this specific project and technology instance
- It would have prevented the same mistake on a completely different
  project using the same technology

A finding belongs in PROJECT_CONTEXT.md only when:
- It is specific to this board, this customer, or this configuration
- It is a temporary workaround with a defined removal condition

**The generalisation test:** Would this principle prevent the same mistake
on a completely different project using the same technology? If yes → domain
architecture document. If no → project context only.

### The five-step self-extension process

1. **State the symptom precisely.** What went wrong? Quote the exact error.
2. **Identify the root cause category.** Which domain section owns this?
3. **Generalise from the specific instance.** State the underlying principle.
4. **Write the principle entry** with rule, rationale, anti-pattern,
   correct pattern, and verification method.
5. **Update the document** — bump the version, add changelog entry.

### Maintenance protocols

See `KNOWLEDGE_REGISTRY.md` for the four formal maintenance protocols:
- Protocol 1 — Updating an existing document
- Protocol 2 — Adding a new domain
- Protocol 3 — Adding a principle to an existing domain
- Protocol 4 — Graduating a planned domain to available

---

## Evidence of Value

The following outcomes were observed on the projects that produced this
pattern. They are the proof points for why the pattern is worth the
setup investment.

### Reviewer errors caught before regression

During a Yocto BSP code review session, the reviewer stated that `.melf`
files were already handled by an upstream bbclass. Claude Code read the
actual bbclass source and found the `*.elf` fnmatch pattern does not match
`*.melf`. Removing the handling — as the reviewer suggested — would have
silently dropped the first-stage bootloader from the flash image, causing
a board that does not boot. The domain architecture document's F1 principle
(read upstream files before generating anything) was the mechanism that
caught this.

A second reviewer suggestion (use a `.bbclass.bbappend` file) was also
caught — this is not a valid BitBake mechanism in current Yocto releases.
It would have caused a parse error on the next build. Source verification
caught it before any file was written.

### Data loss prevented by same-day principle addition

During a firmware recipe refactor, setting `S = ${VENDOR_PREBUILT_DIR}`
caused an inherited `.inc` file's `do_deploy` task to run `find -delete`
against the live vendor prebuilts directory. Thirteen files were deleted.
The files were recoverable from the original SDK download.

The root cause was analysed, generalised, and captured as two new
principles (E6 and F6) in YOCTO_BSP_ARCH.md within the same session.
The next engineer to attempt the same bypass pattern will be stopped by
the F6 audit checklist before any destructive operation runs.

### Zero-regression architectural refactor

Six Yocto recipe architectural issues identified by a code reviewer were
fixed in a single Claude Code session: removing hand-rolled copy loops in
favour of upstream `.inc` infrastructure, replacing an incorrect image
recipe bbappend with a BSP-layer bbclass, creating a custom distro conf,
removing CI file dependencies from kas config, and pinning all layer
checkouts. The build was verified clean (exit 0, 7328 tasks) and the
board booted to login with identical dmesg profile to the pre-fix baseline.

All six fixes were validated against the YOCTO_BSP_ARCH.md domain
principles before any file was changed.

### Specification gap prevented at design time

During a kernel patch discussion, a reviewer stated that out-of-tree
patches should not be applied via bbappend SRC_URI files. The YOCTO_BSP_ARCH.md
D2 principle was updated immediately to add a pre-implementation gate:
a forked kernel repository with topic branch structure must exist before
any patch work begins. This constraint is now enforced automatically
whenever Claude Code is asked to apply a kernel patch — it checks for the
fork before generating any bbappend content.

---

## Why This Pattern Is Needed Now

Every engineering team working in a complex technical domain is currently:

1. Discovering the same failure modes independently
2. Writing the same ad-hoc prompts to work around them
3. Losing that knowledge when the project ends or the engineer moves on
4. Repeating the process on the next project

The pattern described here is a mechanism for that knowledge to accumulate,
be validated across projects, and be applied automatically. A domain
architecture document written from one project's hard-won experience
prevents the same mistakes on every future project in that domain —
without requiring engineers to rediscover them.

The missing infrastructure is a community knowledge commons: a public,
peer-reviewed repository of domain architecture documents that AI tools
can load at session start. The documents in this repository are a seed
for that commons. They are published under CC BY 4.0 so that other
engineers can use, adapt, and contribute back.

---

## Relationship to This Repository

```
bsp-engineering-knowledge/
│
├── THE_PATTERN.md              ← this document — how the pattern works
├── KNOWLEDGE_REGISTRY.md       ← domain router and maintenance protocol
├── METHODOLOGY.md              ← universal engineering process (4 roles,
│                                  gap analysis protocol, pre-implementation
│                                  checklist, anti-patterns)
├── domains/
│   ├── YOCTO_BSP_ARCH.md       ← Yocto domain principles (available)
│   └── [other domains]         ← planned, contributed as projects progress
└── templates/
    ├── PROJECT_CONTEXT_template.md
    ├── CLAUDE.md_template
    ├── CLAUDE_PROJECT_INSTRUCTION.md
    └── new-project-checklist.md
```

`THE_PATTERN.md` (this document) explains the workflow.
`METHODOLOGY.md` defines the engineering process in depth.
`KNOWLEDGE_REGISTRY.md` is the operational interface — it routes sessions
to the correct domain files and defines the maintenance protocol.
Domain files encode the structural constraints for specific technologies.
`PROJECT_CONTEXT.md` is project-specific and never published.

---

## Contributing

To contribute a domain architecture document or extend an existing one,
follow the maintenance protocols in `KNOWLEDGE_REGISTRY.md`. Every
principle must include: the specific symptom that motivated it, the
generalised rule, at least one anti-pattern, and a verification method.

Principles derived from real project experience — not speculative — are
the strongest contributions. The generalisation test is the gate: would
this principle prevent the same mistake on a completely different project
using the same technology?

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

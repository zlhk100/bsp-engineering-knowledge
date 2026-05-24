<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# BSP Engineering Knowledge Base

**A structured methodology for AI-assisted embedded systems engineering.**

[![Licence: CC BY 4.0](https://img.shields.io/badge/Licence-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

---

## The Problem

AI code generation tools fail systematically on complex embedded systems
projects. Not because the models are incapable — but because they lack
the domain-specific structural knowledge that experienced engineers carry
in their heads.

The failure modes are predictable:

- **Plausible but wrong.** The AI generates a recipe, driver, or
  configuration that follows common patterns from its training data
  but violates a structural constraint specific to your upstream layer,
  your hardware topology, or your framework version. It compiles. It
  passes Phase 1. It fails at integration.

- **Missing implicit constraints.** The AI doesn't know that the `.inc`
  file it inherited runs `find -delete` on `S` before deploying, or
  that two binary files are version-locked and must never be mixed, or
  that a particular bbappend pattern is not valid in the current Yocto
  release. These constraints exist only in upstream source, vendor
  documentation, and hard-won project experience.

- **Knowledge decay across sessions.** AI models have no persistent
  memory. The same mistakes get repeated. The same constraints get
  forgotten. There is no mechanism for knowledge discovered on one
  project to protect the next project.

This repository is a structured response to all three problems.

---

## What This Repository Provides

A **reusable pattern** for AI-assisted engineering on complex technical
projects, consisting of:

**1 — A workflow pattern** (`THE_PATTERN.md`)
How to structure knowledge, roles, and sessions so that an AI code
generation tool produces reliable, correct, architecturally sound output.
Validated on TF-M PSA-L2 firmware porting and Qualcomm SoC BSP bring-up
projects.

**2 — A universal methodology** (`METHODOLOGY.md`)
The engineering process: four-role model, hybrid call graph methodology,
gap analysis evidence protocol, pre-implementation checklist, known AI
failure modes, anti-patterns from real projects.

**3 — Domain architecture documents** (`domains/`)
Structural constraints for specific technology domains. Each document
encodes the principles that would have prevented real failures — with
anti-patterns, correct patterns, and verification methods. Currently
available: Yocto BSP. Planned: PCIe BSP, Ethernet BSP, USB/Storage,
Boot Chain, Security Firmware.

**4 — Project templates** (`templates/`)
Everything needed to start a new project using this pattern in under
30 minutes: project context template, CLAUDE.md template, Claude project
instruction, and a setup checklist.

---

## Evidence of Value

These outcomes were observed on the projects that produced this pattern:

**Reviewer errors caught before regression.** During a Yocto recipe
refactor, the domain architecture document's F1 principle (read upstream
files before generating anything) caused the AI to discover that a
reviewer's suggested fix would silently drop the first-stage bootloader
from the flash image. The reviewer's premise — that the upstream bbclass
already handled `.melf` files — was factually wrong. The board would not
have booted. The principle caught it before a single file was changed.

**Data loss prevented by same-day principle addition.** A recipe bypass
pattern caused an inherited `.inc` task to run destructive file deletion
against a live vendor prebuilts directory. Thirteen files were deleted.
The root cause was analysed, generalised, and captured as two new
principles (E6 and F6) in the domain architecture document within the
same session. The next engineer to attempt the same pattern will be
stopped by the audit checklist before any destructive operation runs.

**Zero-regression architectural refactor.** Six Yocto recipe
architectural issues from a code review were addressed in a single
Claude Code session. Build verified clean (7,328 tasks, exit 0). Board
booted with identical profile to the pre-fix baseline. Every fix was
validated against domain architecture principles before any file was
changed.

**Specification gap prevented at design time.** A reviewer stated that
kernel patches should not be applied via bbappend SRC_URI files. The
domain architecture document D2 principle was updated immediately with a
pre-implementation gate. Claude Code now checks for a forked kernel
repository with topic branch structure before generating any kernel
bbappend content — stopping the mistake before it happens.

---

## Quick Start

### Use this pattern on your next project

```bash
# 1. Clone this repository
git clone https://github.com/<your-org>/bsp-engineering-knowledge
cd bsp-engineering-knowledge

# 2. Create your project workspace
mkdir ~/workspace/your-project
cp KNOWLEDGE_REGISTRY.md ~/workspace/your-project/
cp METHODOLOGY.md ~/workspace/your-project/
cp domains/YOCTO_BSP_ARCH.md ~/workspace/your-project/  # if Yocto
cp templates/PROJECT_CONTEXT_template.md \
   ~/workspace/your-project/PROJECT_CONTEXT.md
cp templates/CLAUDE.md_template \
   ~/workspace/your-project/CLAUDE.md

# 3. Fill in PROJECT_CONTEXT.md for your project

# 4. Set up Claude project (see templates/CLAUDE_PROJECT_INSTRUCTION.md)

# 5. Follow templates/new-project-checklist.md
```

See `THE_PATTERN.md` for the full explanation of the workflow.
See `templates/new-project-checklist.md` for step-by-step setup.

---

## Repository Structure

```
bsp-engineering-knowledge/
│
├── README.md                         ← you are here
├── THE_PATTERN.md                    ← workflow pattern explanation
├── KNOWLEDGE_REGISTRY.md             ← domain router + maintenance protocol
├── METHODOLOGY.md                    ← universal engineering process
│
├── domains/
│   ├── YOCTO_BSP_ARCH.md             ← ✅ available — Yocto BSP principles
│   ├── PCIE_BSP_ARCH.md              ← 🔲 planned
│   ├── ETHERNET_BSP_ARCH.md          ← 🔲 planned
│   ├── USB_STORAGE_ARCH.md           ← 🔲 planned
│   ├── BOOT_CHAIN_ARCH.md            ← 🔲 planned
│   └── FIRMWARE_PORT_ARCH.md         ← 🔲 planned
│
└── templates/
    ├── PROJECT_CONTEXT_template.md   ← blank project context
    ├── CLAUDE.md_template            ← Claude Code workspace instruction
    ├── CLAUDE_PROJECT_INSTRUCTION.md ← Claude project instruction text
    └── new-project-checklist.md      ← step-by-step setup guide
```

---

## The Core Idea

The pattern is built on four layers:

```
┌──────────────────────────────────────────────┐
│  Universal Methodology  (METHODOLOGY.md)     │
│  Four-role model, gap analysis protocol,     │
│  pre-implementation checklist                │
├──────────────────────────────────────────────┤
│  Domain Principles  ([DOMAIN]_ARCH.md)       │
│  Structural constraints for your technology  │
│  Anti-patterns from real project failures    │
├──────────────────────────────────────────────┤
│  Project Context  (PROJECT_CONTEXT.md)       │
│  Hardware topology, open items, decisions    │
│  Single source of truth for this project     │
├──────────────────────────────────────────────┤
│  Session Execution  (CLAUDE.md + Claude Code)│
│  Implementation constrained by all above     │
└──────────────────────────────────────────────┘
```

Higher layers constrain lower layers. Layer 1 and 2 are shared across
all projects — write once, apply everywhere. Layer 3 is project-specific.
Layer 4 executes within the constraints of all three layers above.

The Claude project instruction loads all relevant layers at session start:

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

This instruction is **identical for every project**. Only the uploaded
files change.

---

## The Gap Analysis Evidence Protocol

One of the most important concepts in this methodology. All AI-generated
technical findings must carry an explicit evidence tier:

**[VERIFIED]** — directly supported by output or files in this session.
Act on it.

**[INFERRED]** — follows from verified evidence with reasoning steps.
Verify before acting.

**[ASSUMED]** — rests on general knowledge, not project-specific
observation. Highest risk — verify first, always.

```
GAP-N: [VERIFIED|INFERRED|ASSUMED] Short description

  Observed:    What the output literally shows (quoted exactly)
  Concluded:   What is inferred from it
  Assumed:     What is filled from general knowledge (if anything)
  Uncertainty: What could invalidate this
  Verify with: <exact command to run>
  Act only after: verification confirms the gap
```

This protocol forces AI tools to distinguish between what they observed
and what they inferred — preventing the confident-but-wrong gap claims
that trigger unnecessary corrective actions.

---

## Domain Architecture Documents

Each domain document encodes the structural constraints for a technology
domain in a standard format:

- **Categorical domains** (A, B, C...) covering all design decisions
- **Numbered principles** (A1, A2...) with rule, rationale, anti-pattern,
  correct pattern, and verification method
- **AI generation constraints** (Domain F) — what Claude must do before
  generating any file in this domain
- **Pre-implementation gates** — hard stops that prevent known failure
  modes before any code is written
- **Self-extension mechanism** — how new principles slot in when discovered

### Example — YOCTO_BSP_ARCH.md structure

```
Domain A — Layer Architecture and Responsibility
  A1 — Understand Before You Build (Capability Audit)
  A2 — Every Concern at Its Correct Abstraction Layer
  A3 — BBappends Express Overrides, Not Features
  ...

Domain D — Patch and Change Management
  D2 — Kernel Patches as Topic Branches, Not SRC_URI Files
       Pre-implementation gate: forked kernel repo must exist
       before any kernel bbappend patch content is generated

Domain E — Binary and Artefact Provenance
  E6 — Never Redirect S to a Live Binary Source Directory
       Anti-pattern: S = ${VENDOR_PREBUILT_DIR} (confirmed data loss)
       Correct pattern: S = "${UNPACKDIR}", override do_deploy

Domain F — AI-Assisted Generation Constraints
  F1 — Upstream Layer Content Must Be in Context Before Generation
  F3 — Apply Domain Checklist Before Generating Any File
  F6 — Audit Destructive Operations Before Adopting Any .inc
```

---

## Contributing

Contributions of domain principles and new domain documents are welcome.

### Adding a principle to an existing domain

Use Protocol 3 from `KNOWLEDGE_REGISTRY.md`:

1. State the symptom precisely — what went wrong on your project?
2. Identify the root cause category — which domain section owns this?
3. Generalise — would this principle prevent the same mistake on a
   completely different project using the same technology?
4. Write the principle entry (rule, rationale, anti-pattern, correct
   pattern, verification method)
5. Open a PR with the updated domain document

**The generalisation test is the gate.** If the principle is specific to
your board, your customer, or your configuration — it belongs in your
project context, not here.

### Adding a new domain

Use Protocol 2 from `KNOWLEDGE_REGISTRY.md`. A new domain document needs:
- At least 3 categorical domains (A, B, C)
- At least 2 principles per domain
- At least 1 anti-pattern per principle
- A Domain F (AI generation constraints) section
- A self-extension mechanism

Open a PR with the new domain document and a registry update.

### Contribution standards

- Principles must be derived from real project experience, not speculation
- Anti-patterns must be concrete enough to be recognisable
- Verification methods must be runnable commands or questions
- All content must be generalisable across projects in the same domain
- No proprietary, NDA, or customer-identifying information

---

## Planned Domain Documents

Community contributions welcome for any of these:

| Domain | Technology | Status |
|---|---|---|
| PCIe BSP | PCIe switches, SMMU, iommu-map, msi-map, DT authoring | 🔲 Planned |
| Ethernet BSP | PHY drivers, MAC config, MDIO, stmmac, dwmac | 🔲 Planned |
| USB and Storage | xHCI, USB bridges, NVMe, SD, firmware blobs | 🔲 Planned |
| Boot Chain | XBL, UEFI, fastboot, partition table, secure boot | 🔲 Planned |
| Security Firmware | TF-M, PSA certification, CMSIS HAL, secure partitions | 🔲 Planned |
| U-Boot | Bootloader porting, DM drivers, environment, FIT images | 🔲 Planned |
| Zephyr RTOS | Board porting, device tree, Kconfig, west workspace | 🔲 Planned |
| Buildroot | Package recipes, kernel config, rootfs, BR2_EXTERNAL | 🔲 Planned |

---

## Licence

All documents in this repository are published under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).

You are free to use, adapt, and redistribute this work for any purpose,
including commercial use, as long as you give appropriate credit.

**Attribution:** Lei Zhou, "BSP Engineering Knowledge Base",
https://github.com/<your-org>/bsp-engineering-knowledge

---

## Authors

**Lei Zhou** — methodology design, Yocto domain principles, project
validation on TF-M PSA-L2 and Qualcomm SoC BSP bring-up projects.

**Claude (Anthropic)** — co-development of methodology, domain principle
structuring, document authoring.

---

## Related Work

- [Yocto Project Documentation](https://docs.yoctoproject.org/)
- [OpenEmbedded Layer Index](https://layers.openembedded.org/)
- [TF-M Documentation](https://tf-m-user-guide.trustedfirmware.org/)
- [Qualcomm Linux upstream kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/)

---

*© 2026 Lei Zhou — [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)*

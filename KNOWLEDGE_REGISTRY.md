<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# BSP Engineering Knowledge Registry

**Version:** 1.6
**Date:** May 2026
**Author:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/zlhk100/bsp-engineering-knowledge

---

## Purpose

This file is the single consistent interface to all domain knowledge in
the BSP engineering knowledge base. It is identical across every project.
Only `PROJECT_CONTEXT.md` changes per project.

**Filenames are version-free.** Document versions are tracked inside each
file's header and by git history. The registry never needs updating when
document content changes — only when domains are added, removed, or renamed.

This file also serves as the authoritative maintenance reference — all
protocols for updating, extending, and managing the knowledge base are
defined here. No separate maintenance document is needed.

---

## How to Use This Registry

### Claude project files — upload once per project

```
KNOWLEDGE_REGISTRY.md    ← this file (identical across all projects)
METHODOLOGY.md           ← always active (identical across all projects)
YOCTO_BSP_ARCH.md        ← upload if project uses Yocto
PROJECT_CONTEXT.md       ← project-specific SSoT (one per project)
```

Upload only the domain files relevant to your project. The registry
references all domains — Claude skips files that are not uploaded and
states this explicitly.

### Project instructions — set once, identical for every project

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

This instruction never changes between projects or versions.

### File naming convention

All files use version-free names. The version number lives inside the
document header — not the filename. When a document is updated, replace
the file content. The filename stays the same. The project instruction
and registry require no update.

---

## Always Active — Every Session

### METHODOLOGY.md

Load at every session start without exception.

**Four standing rules always in effect:**

**1 — Gap Analysis Evidence Protocol**
Apply VERIFIED / INFERRED / ASSUMED tiers to every technical finding.
Never state a conclusion without citing the evidence tier.

```
GAP-N: [VERIFIED|INFERRED|ASSUMED] Short description

  Observed:    What the output or file literally shows (quoted exactly)
  Concluded:   What is inferred from it
  Assumed:     What is filled in from general knowledge (if anything)
  Uncertainty: What could invalidate this if the assumption is wrong
  Verify with: <exact command to run on the target system>
  Act only after: confirmation that verify output supports the gap
```

**2 — Pre-implementation gate**
Before generating any implementation, confirm the pre-implementation
checklist is satisfied. If any item is unknown, resolve it before
generating code. The cost of a specification gap is discovered at
runtime — not in the design session.

**3 — Source over AI consensus**
All technical claims must be verified against primary sources: upstream
DTS, schematic, datasheet, header files, UART logs. AI-generated
assertions require explicit source confirmation before acting on them.

**4 — Four-role boundary**
Design AI (this session) defines what and why.
Implementer (Claude Code CLI) executes how.
Never mix roles. If a design decision is needed during implementation,
stop, resolve it in the design session, then resume implementation.

---

## Domain Registry

At session start, scan the session goal and any provided context for
signals in the "Active when" column. For each matched domain, read the
corresponding file using the view tool before responding.

Domain activation is cumulative — once a domain is active it stays
active for the entire session. If the conversation shifts into a new
domain mid-session, read that domain's file before responding to the
new topic.

| Domain | Active when conversation involves | File |
|---|---|---|
| **Yocto BSP** | recipes (.bb/.bbappend), kas config, BitBake, meta-layer structure, SRCREV, kernel patches, DTS authoring, firmware staging recipe, distro conf, machine conf, image recipe, sstate, devtool, OpenEmbedded | `YOCTO_BSP_ARCH.md` |
| **Secure Firmware** | TF-M, PSA certification, CMSIS HAL porting, secure boot, attestation, crypto keys, NV counters, ITS/PS partition, SFN/IPC backend, isolation, FIH, RPMB, smcinvoke | `FIRMWARE_PORT_ARCH.md` |
| **Embedded Security Architecture** | PSA certification scope, CRA compliance, secure storage design, key derivation hierarchy, attestation, non-securable NVM, AES-GCM-SIV, trust model, threat model, JSADEN, protection profile, EN 304 623, EN 304 626, TR-MISO, trust boundary | `SECURITY_ARCH_KNOWLEDGE.md` |
| **Boot Chain** | UEFI, bootloader bring-up, secure boot, partition table, device provisioning, DDR training, power management init, firmware staging, version lock, bootloader recovery, first-stage bootloader, second-stage bootloader | `BOOT_CHAIN_ARCH.md` |
| **PCIe and SMMU** | PCIe switch, iommu-map, msi-map, SMMU stream IDs, SID mapping, DTS PCIe nodes, PCIe endpoint driver, ITS limit, RID translation, of_pci_map_rid | `PCIE_BSP_ARCH.md` |
| **Ethernet BSP** | PHY driver, MAC configuration, MDIO, SGMII/USXGMII/RGMII, dwmac, stmmac, PHY bring-up, iperf3 validation, Ethernet DTS nodes | `ETHERNET_BSP_ARCH.md` |
| **USB and Storage** | xHCI, UAS quirk, usb-storage, USB bridge, NVMe, SD block layer, USB firmware blob loading, USB quirk table | `USB_STORAGE_ARCH.md` |
| **U-Boot** | U-Boot porting, DM drivers, environment, FIT images, SPL, board init, DTS, Kconfig, defconfig, distro boot | `UBOOT_ARCH.md` |
| **Zephyr RTOS** | Zephyr board porting, device tree, Kconfig, west workspace, shields, drivers, RTOS integration, Zephyr application | `ZEPHYR_ARCH.md` |
| **RTOS Benchmark** | RTOS benchmark, WCET, OS primitive latency, task switching, task preemption, IRQ latency, mutex, IMLat, intertask messaging, stressor, PMU attribution, ftrace, mixed criticality, RT-PREEMPT, microkernel RTOS, IEC 61508, ISO 26262 latency evidence, ROS 2 latency, VirtIO overhead, OpenAMP, AMP IPC, micro-ROS, VLA inference latency | `RTOS_BENCH_ARCH.md` |

### Domain document status

Claude: if a domain is active but its file is not available as a project
file, state this explicitly and apply the methodology's general
pre-implementation checklist as a fallback. Do not skip domain constraints.

| File | Status | Notes |
|---|---|---|
| `METHODOLOGY.md` | ✅ Always available | Always active |
| `YOCTO_BSP_ARCH.md` | ✅ Available | v1.2 May 2026 |
| `FIRMWARE_PORT_ARCH.md` | ✅ Available | v1.0 May 2026 — TF-M PSA-L2 porting, Domains A–F |
| `SECURITY_ARCH_KNOWLEDGE.md` | 🔲 Planned | Derive from TF-M PSA-L2 project + CRA analysis — MB-004 |
| `BOOT_CHAIN_ARCH.md` | 🔲 Planned | Derive from embedded SoC BSP boot chain bring-up experience |
| `PCIE_BSP_ARCH.md` | 🔲 Planned | Derive from embedded SoC BSP PCIe bring-up experience |
| `ETHERNET_BSP_ARCH.md` | 🔲 Planned | Derive from embedded SoC BSP Ethernet bring-up experience |
| `USB_STORAGE_ARCH.md` | 🔲 Planned | Derive from embedded SoC BSP storage bring-up experience |
| `UBOOT_ARCH.md` | 🔲 Planned | Derive from U-Boot porting and board bring-up experience |
| `ZEPHYR_ARCH.md` | 🔲 Planned | Derive from Zephyr RTOS board porting experience |
| `RTOS_BENCH_ARCH.md` | ✅ Available | v1.4 May 2026 — WCET benchmarking, OS primitive metrics, PMU attribution, mixed-criticality safety, Layer 1–4 architecture, VirtIO overhead, SMMU attribution |

---

## Domain Activation Examples

### Single domain

Session goal: "Write a firmware staging recipe for the board."

Active: **Yocto BSP** → read `YOCTO_BSP_ARCH.md`

Constraint applied before generating:
- Read upstream firmware common .inc file before writing new recipe
- Confirm no existing bbclass handles the staging task

---

### Multiple domains

Session goal: "Debug the SMMU fault during PCIe enumeration, then fix
the iommu-map in the carrier DTS bbappend."

Active:
- **PCIe and SMMU** → read `PCIE_BSP_ARCH.md`
- **Yocto BSP** → read `YOCTO_BSP_ARCH.md`

Constraints applied:
- Verify SID values against source before modifying iommu-map
- Confirm DTS change is in correct per-subsystem dtsi fragment

---

### Domain shift mid-session

Session starts as boot chain debugging. User then asks to add a kernel
patch via bbappend.

New domain detected: **Yocto BSP** → read `YOCTO_BSP_ARCH.md`

Constraint applied immediately:
- Is there a topic branch structure? Topic branch structure must
  exist before any patch is added. Never generate SRC_URI patch
  entries — even the first one.

---

## New Project Setup

```bash
# 1. Upload to Claude project (unchanged from any other project):
#    KNOWLEDGE_REGISTRY.md
#    METHODOLOGY.md
#    <relevant domain files>.md

# 2. Write and upload project-specific file:
#    PROJECT_CONTEXT.md

# 3. Set project instructions (copy exactly — never changes):
```
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

---

## Maintenance Protocol

This section is the authoritative reference for all knowledge base
maintenance operations. Every change to any document follows one of
the four protocols below. No separate maintenance document is needed.

---

### Protocol 1 — Updating an Existing Document

**Triggers:** New principle discovered, correction to existing principle,
improved rationale, better anti-pattern example, reviewer comment,
post-mortem finding, session experience.

**What changes:**
- The document file only (e.g. `YOCTO_BSP_ARCH.md`)
- Internal `**Version:**` header — bump minor: 1.0 → 1.1
- Internal changelog table inside that file

**What does NOT change:**
- The filename
- `KNOWLEDGE_REGISTRY.md`
- Any other document
- Project instruction

**Claude project update:** Replace the old file with the new version.
Registry and project instruction are unchanged — Claude loads the
updated content automatically on next session.

**Commit message format:**
```
docs(yocto-bsp-arch): add principle D6 — lock file commit discipline

Source: project X session — discovered kas lock files were not being
committed, causing non-reproducible builds across team members.
Generalised from specific instance to domain principle D6.
```

---

### Protocol 2 — Adding a New Domain

**Triggers:** A project encounters a technology domain not covered by
any existing domain document (e.g. U-Boot, AUTOSAR, Zephyr, Android
AAOS, buildroot, new SoC family).

**Step 1 — Create the domain document**

Create `DOMAIN_NAME_ARCH.md` following `YOCTO_BSP_ARCH.md` structure:

```markdown
# [Domain Name] Architecture Principles
**Version:** 1.0
**Licence:** CC BY 4.0

## [Unifying meta-principle for this domain]

## Domain A — [First categorical domain]
### A1 — [First principle]
**Rule:** ...
**Rationale:** ...
**Anti-pattern:** ...
**Correct pattern:** ...
**Verification:** ...

[Continue for all domains and principles]

## Maintenance Protocol Reference
See KNOWLEDGE_REGISTRY.md — Maintenance Protocol.
All principle additions follow Protocol 3.
```

Minimum content for v1.0:
- At least 3 categorical domains (A, B, C)
- At least 2 principles per domain
- At least 1 anti-pattern per principle
- Self-extension mechanism pointer

**Step 2 — Update KNOWLEDGE_REGISTRY.md**

Add one row to the Domain Registry table:
```markdown
| **New Domain** | trigger keywords... | `DOMAIN_NAME_ARCH.md` |
```

Add one row to the status table:
```markdown
| `DOMAIN_NAME_ARCH.md` | ✅ Available | v1.0 [Month Year] |
```

Bump registry version (1.2 → 1.3) and add changelog entry.

**What does NOT change:**
- `METHODOLOGY.md`
- Any existing domain document
- Project instruction — unchanged forever

**Claude project update:** Upload new `DOMAIN_NAME_ARCH.md` as a new
project file. Upload updated `KNOWLEDGE_REGISTRY.md`.

**Commit message format:**
```
feat(registry): add Boot Chain domain — BOOT_CHAIN_ARCH.md v1.0

Source: Embedded SoC BSP boot chain bring-up experience
patterns generalised from project experience.
Registry bumped to v1.3.
```

---

### Protocol 3 — Adding a Principle to an Existing Domain

**Triggers:** Reviewer comment, build failure, post-mortem, or session
experience reveals a structural problem not covered by any existing
principle.

**Decision gate — before adding:**

Ask two questions:
1. Does this generalise beyond this specific project and technology
   instance? If no → document in `PROJECT_CONTEXT.md` as a
   project-specific decision, not in the domain architecture.
2. Which domain owns this? If it spans multiple domains, add to the
   most relevant one and cross-reference from others.

**The five-step self-extension process:**

**Step 1 — State the symptom precisely**
What went wrong? What did the reviewer say? What broke?
Quote the exact error, comment, or failure.

**Step 2 — Identify the root cause category**
Which domain section owns this?
- Layer architecture (A)? Configuration ownership (B)?
- Reproducibility (C)? Patch management (D)?
- Binary provenance (E)? AI generation constraints (F)?
- None of the above → new domain (use Protocol 2)

**Step 3 — Generalise from the specific instance**
State the underlying principle that the symptom violates.
Test: would this principle have prevented the problem on a completely
different project using the same technology? If yes, it belongs here.

**Step 4 — Write the principle entry**
```markdown
### X.N — Principle Name

**Rule:** One clear imperative sentence stating what must be done.

**Rationale:** Why this matters. What goes wrong without it.
Explain the failure mode, not just the rule.

**Anti-pattern:**
[Concrete wrong example — specific enough to be recognisable]

**Correct pattern:**
[Concrete right example — specific enough to be actionable]

**Verification:**
[Command or question to confirm compliance before proceeding]
```

**Step 5 — Update both files**
- Add principle to the correct section in `DOMAIN_ARCH.md`
- Bump the document's internal version (1.0 → 1.1)
- Add changelog entry inside the document

**What changes:** Domain document only.
**What does NOT change:** Registry, methodology, project instruction.

---

### Protocol 4 — Graduating a Planned Domain to Available

**Triggers:** A 🔲 Planned domain in the status table has accumulated
enough project experience to write its first version.

**Process:** Follow Protocol 2 steps, except:
- The registry table row already exists — do not add a new one
- Only update the status table row from 🔲 to ✅:

```markdown
| `BOOT_CHAIN_ARCH.md` | ✅ Available | v1.0 [Month Year] |
```

Bump registry version and add changelog entry as with Protocol 2.

---

### Version Numbering Policy

| Document | Scheme | Bump minor when | Bump major when |
|---|---|---|---|
| `KNOWLEDGE_REGISTRY.md` | Major.Minor | Domain added or status changed | Structural redesign of registry |
| Domain architecture docs | Major.Minor | Principle added or improved | Domain section reorganisation |
| `METHODOLOGY.md` | Major.Minor | Section added or improved | Framework restructure |
| `PROJECT_CONTEXT.md` | Driven by CN number | Every change notice | N/A |

Version numbers live inside document headers only.
Filenames never contain version numbers.

---

### Commit Message Convention

```
type(scope): short description

Body explaining the source of the change — which project or session
produced the finding, and why it was generalised to this knowledge base.

Closes #issue (if applicable)
```

**Types:**
- `feat` — new domain or new principle added
- `docs` — improved rationale, anti-pattern, or example
- `fix` — correction to an incorrect or misleading principle
- `refactor` — restructuring without changing meaning

**Scopes:** `registry`, `methodology`, `yocto-bsp-arch`,
`boot-chain-arch`, `pcie-bsp-arch`, `ethernet-bsp-arch`,
`usb-storage-arch`, `firmware-port-arch`, `uboot-arch`, `zephyr-arch`,
`rtos-bench-arch`, `security-arch-knowledge`

---

### What Belongs in the Knowledge Base vs Project Context

This boundary is the most important maintenance decision.

**Belongs in knowledge base (domain architecture document):**
- Principles that generalise across multiple projects
- Anti-patterns observed on one project but applicable to any project
  in the same domain
- Structural constraints inherent to the technology

**Belongs in `PROJECT_CONTEXT.md` only:**
- Project-specific open items (OI-NNN register)
- Board-specific hardware topology decisions
- Customer-specific configuration choices
- Temporary workarounds with a defined removal condition
- Binary versions and checksums specific to this board
- Milestone status and sprint plan

**The generalisation test:**
Would this finding prevent the same mistake on a completely different
project using the same technology?
- Yes → knowledge base domain document
- No → project context only

---

## Methodology Backlog

This section captures project-agnostic pending actions — improvements to
the methodology, domain documents, or knowledge base structure that have
been identified but not yet written. It is reviewed at the start of any
session that touches methodology, architecture, or knowledge base questions.

**The discipline:** Every item here is either acted on (promoted to a
document update) or explicitly deferred with a reason. Items do not
accumulate indefinitely. When an item is completed, it is removed and
a changelog entry is added to the relevant document.

**The generalisation test still applies.** Items here must be
project-agnostic — applicable across projects using this methodology.
Project-specific pending items belong in `PROJECT_CONTEXT.md`, not here.

---

### Backlog Items

---

**MB-001 — Create `FIRMWARE_PORT_ARCH.md`**  ✅ COMPLETED May 2026

See Completed Items table below.

---

**MB-002 — Add CRA/PSA boundary principle to `FIRMWARE_PORT_ARCH.md`**

Status: READY — principle identified and justified
Source: EN 304 623 + EN 304 626 joint analysis (May 2026)
Principle: CRA compliance via EN 304 623 + EN 304 626 is necessary for
CE marking but does not establish principled trust isolation architecture.
The two standards leave the boot-to-OS handoff interface undefined — no
joint requirement exists that the OS verifies it is running on a
compliant boot manager before relying on properties the boot manager
is supposed to have provided. PSA certification closes this gap at the
RoT layer. Any TF-M port targeting EU market products needs this framing
to avoid assuming CRA compliance implies correct trust architecture.
Blocked on: nothing — MB-001 completed
Target: `FIRMWARE_PORT_ARCH.md` Domain A

---

**MB-003 — Add TR-MISO trust model gap principle to `FIRMWARE_PORT_ARCH.md`**

Status: READY — principle identified and justified
Source: EN 304 626 analysis (May 2026)
Principle: EN 304 626 TR-MISO is mechanism-first without a trust model.
It specifies hardware-enforced memory access control without defining
principals, trust levels, or the identity × resource → policy structure
that makes isolation meaningful. A product can pass TR-MISO assessment
while having an unprincipled NS-to-S boundary that leaks key material,
if it has no user accounts (the applicability condition). Any security
firmware architect evaluating CRA compliance must apply a trust-model
lens that the standard itself does not provide.
Blocked on: nothing — MB-001 completed
Target: `FIRMWARE_PORT_ARCH.md` Domain A

---

**MB-004 — Create `SECURITY_ARCH_KNOWLEDGE.md`**

Status: READY — three initial patterns identified, ready to write
Source: TF-M PSA-L2 porting project + CRA analysis (May 2026)
Scope: Reusable architectural patterns only. Not normative text (retrieve
at query time via web search). Not design decisions (belong in PPS). Not
AI failure modes (belong in FIRMWARE_PORT_ARCH Domain F or METHODOLOGY.md).
Initial patterns:
  Pattern 1 — Non-securable NVM storage security
    Cryptographic enforcement substitutes for hardware write protection
    when hardware protection is unavailable. AES-GCM-SIV (RFC 8452)
    is necessary (not merely sufficient) for power-loss correctness due
    to nonce-misuse resistance. Confidence: INFERRED — validated on a
    Cortex-M33 adversarial PoC; Phase 4 hardware testing will promote
    to VERIFIED.
  Pattern 2 — Trust boundary definition precedes mechanism selection
    Define principals and trust levels before selecting SAU/MPU/MPC/PPC
    configuration. Isolation without a trust model produces mechanism
    compliance without security property guarantees. Derived from
    TR-MISO analysis — the structural gap the ETSI OS standard has and
    PSA fills. Confidence: VERIFIED — normatively grounded in PSA
    architecture.
  Pattern 3 — CRA/PSA complementarity
    CRA compliance establishes the regulatory floor (CE marking, essential
    requirements). PSA certification establishes principled trust
    architecture (verifiable RoT, defined isolation levels, normative
    security claims). Neither substitutes for the other. Together they
    address different evaluation audiences: market regulator vs security
    architect. Confidence: VERIFIED — derived from joint EN 304 623/626
    and PSA-L2 normative analysis.
Blocked on: nothing — can be written independently of MB-001
Target: `domains/SECURITY_ARCH_KNOWLEDGE.md` v0.1 (stub)
Note: Load trigger keywords differ from FIRMWARE_PORT_ARCH. Activate on
security architecture design questions, CRA compliance analysis, PSA
certification scope, threat model, trust boundary definition, key
derivation design — not on TF-M implementation sessions.

---

**MB-005 — Add knowledge layer boundary discipline to `METHODOLOGY.md`**

Status: READY — principle identified
Source: Knowledge base architecture discussion (May 2026)
Principle: Three-layer knowledge boundary discipline.
  Layer 0 — Retrieved at query time: normative standard text, SDK source
    excerpts, datasheet sections, evaluation lab outputs. Do not migrate
    into Layer 1 markdown files — retrieve via web search or grep at
    session time.
  Layer 1 — Loaded at session start: reusable patterns, process
    constraints, project state. The markdown file structure in this
    knowledge base. Appropriate for content that is stable, cross-project,
    and not accurately covered by LLM training.
  Layer 2 — LLM training weights: general embedded systems knowledge,
    common design patterns, language and framework knowledge.
Placement rule: if content can be retrieved accurately at query time,
  do not put it in Layer 1. If it is already in LLM training with
  sufficient accuracy, do not put it in Layer 1. Layer 1 is for the
  narrow residual that is neither retrievable on demand nor reliably
  trained.
Target: `METHODOLOGY.md` — new subsection under "The Single Source of
Truth" or as a standalone "Knowledge Layer Boundary" section.
Blocked on: nothing
Note: This principle also informs KNOWLEDGE_REGISTRY.md maintenance —
reviewers should apply it when deciding whether a new finding belongs
in a domain document or should be retrieved at session time.

---

**MB-006 — Add `BOOT_CHAIN_ARCH.md` seed**

Status: PARTIALLY READY — source material from two project domains
available
Source: EN 304 623 analysis (May 2026) + embedded SoC BSP boot
chain bring-up experience
Available content: EN 304 623 four trust boundaries (hardware, storage,
network, OS boundary), 9 requirement families, TOCTOU requirement
(INT-010), MAS-001 resource cleanup before handoff, SBD-004 no-auto-
fallback, CON-002 no global symmetric keys. Embedded Linux boot chain:
first-stage and second-stage bootloaders, partition table, device
provisioning protocols, version lock constraints.
Blocked on: needs minimum 3 categorical domains and 2 principles per
domain (Protocol 2). EN 304 623 material covers security firmware boot
chain well; embedded Linux BSP material covers the application processor
boot chain.
Recommend: write v1.0 when a boot chain project is active — the project
context will surface additional principles that justify the investment.
Target: `domains/BOOT_CHAIN_ARCH.md` v1.0
Registry update: Protocol 4 (graduate from Planned to Available)

---

### Completed Items

| Item | Completed | Promoted to |
|---|---|---|
| MB-001 | May 2026 | FIRMWARE_PORT_ARCH.md v1.0 created |

---

### Backlog Maintenance Rules

1. Items are added when a project-agnostic insight is identified but
   cannot be acted on immediately. Add in the same session it is
   identified — do not defer the capture.

2. Items are removed when the corresponding document update is committed.
   Add a completed row to the table above and a changelog entry to the
   updated document.

3. Items are not allowed to accumulate without review. Any session that
   activates METHODOLOGY.md should scan this backlog for items that are
   now unblocked.

4. The generalisation test is the admission gate. Project-specific
   pending items belong in `PROJECT_CONTEXT.md`. Domain-specific items
   that are ready to write belong directly in the target document, not
   in this backlog.

---

## Registry Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial release. Six domains registered. YOCTO_BSP_ARCH available. Five domains planned. |
| 1.1 | May 2026 | Switched to version-free filenames. Fixed flat file references. Added PROJECT_CONTEXT.md convention. Removed version coupling. |
| 1.2 | May 2026 | Added comprehensive Maintenance Protocol (Protocols 1–4). Version numbering policy. Commit message convention. Knowledge base vs project context boundary definition. Absorbed standalone "Adding a New Domain" section into Protocol 2. |
| 1.3 | May 2026 | Added U-Boot and Zephyr RTOS as planned domains (registry table + status table). Fixed repository URL placeholder. Updated YOCTO_BSP_ARCH.md status to v1.2. Updated domain shift example to reflect D2 first-patch gate. |
| 1.4 | May 2026 | Added RTOS Benchmark as formally registered available domain (RTOS_BENCH_ARCH.md v1.4). Covers WCET benchmarking, OS primitive metrics, PMU attribution, mixed-criticality safety, Layer 1–4 architecture (bare-metal through AMP/AI inference), VirtIO overhead measurement, SMMU IOTLB attribution. Added rtos-bench-arch commit scope. |
| 1.5 | May 2026 | Added Methodology Backlog section. Six items registered: MB-001 (FIRMWARE_PORT_ARCH.md creation), MB-002 (CRA/PSA boundary principle), MB-003 (TR-MISO trust model gap principle), MB-004 (SECURITY_ARCH_KNOWLEDGE.md stub), MB-005 (knowledge layer boundary discipline for METHODOLOGY.md), MB-006 (BOOT_CHAIN_ARCH.md seed). Added security-arch-knowledge commit scope. |
| 1.6 | May 2026 | Graduated FIRMWARE_PORT_ARCH.md from Planned to Available (v1.0). Added Embedded Security Architecture domain (SECURITY_ARCH_KNOWLEDGE.md, planned). Completed MB-001. |

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

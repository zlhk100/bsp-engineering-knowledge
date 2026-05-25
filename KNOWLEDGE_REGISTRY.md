<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# BSP Engineering Knowledge Registry

**Version:** 1.4
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
| **Boot Chain** | XBL, UEFI, bootloader bring-up, partition table, qdl, edl, fastboot, abl, cdt.bin, non-HLOS binaries, DCB version lock, PMIC init, DDR training, TZ, QSEE | `BOOT_CHAIN_ARCH.md` |
| **PCIe and SMMU** | PCIe switch, iommu-map, msi-map, SMMU stream IDs, SID mapping, DTS PCIe nodes, PCIe endpoint driver, QHEE ITS limit, RID translation, of_pci_map_rid | `PCIE_BSP_ARCH.md` |
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
| `FIRMWARE_PORT_ARCH.md` | 🔲 Planned | Derive from TF-M firmware porting project experience |
| `BOOT_CHAIN_ARCH.md` | 🔲 Planned | Derive from Qualcomm SoC BSP boot chain bring-up experience |
| `PCIE_BSP_ARCH.md` | 🔲 Planned | Derive from Qualcomm SoC BSP PCIe bring-up experience |
| `ETHERNET_BSP_ARCH.md` | 🔲 Planned | Derive from Qualcomm SoC BSP Ethernet bring-up experience |
| `USB_STORAGE_ARCH.md` | 🔲 Planned | Derive from Qualcomm SoC BSP storage bring-up experience |
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

Source: Qualcomm SoC BSP boot chain bring-up experience
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
`usb-storage-arch`, `firmware-port-arch`, `uboot-arch`, `zephyr-arch`, `rtos-bench-arch`

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

## Registry Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial release. Six domains registered. YOCTO_BSP_ARCH available. Five domains planned. |
| 1.1 | May 2026 | Switched to version-free filenames. Fixed flat file references. Added PROJECT_CONTEXT.md convention. Removed version coupling. |
| 1.2 | May 2026 | Added comprehensive Maintenance Protocol (Protocols 1–4). Version numbering policy. Commit message convention. Knowledge base vs project context boundary definition. Absorbed standalone "Adding a New Domain" section into Protocol 2. |
| 1.3 | May 2026 | Added U-Boot and Zephyr RTOS as planned domains (registry table + status table). Fixed repository URL placeholder. Updated YOCTO_BSP_ARCH.md status to v1.2. Updated domain shift example to reflect D2 first-patch gate. |
| 1.4 | May 2026 | Added RTOS Benchmark as formally registered available domain (RTOS_BENCH_ARCH.md v1.4). Covers WCET benchmarking, OS primitive metrics, PMU attribution, mixed-criticality safety, Layer 1–4 architecture (bare-metal through AMP/AI inference), VirtIO overhead measurement, SMMU IOTLB attribution. Added rtos-bench-arch commit scope. |

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# BSP Engineering Knowledge Registry

**Version:** 1.13
**Date:** May 2026
**Author:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/zlhk100/bsp-engineering-knowledge

---

## Purpose

This file is the domain router and workspace operational reference.
It is Tier 1 — loaded every session in every workspace.

Maintenance protocols, backlog, changelogs, and domain activation
examples are in METHODOLOGY_REF.md (Tier 3 — presented at session start when goal matches).

**Filenames are version-free.** The version number lives inside the
document header. When a document is updated, replace the file content.
The filename and project instruction require no update.

---

## Workspace Model

A Claude project is a **technology domain workspace**, not a single-project
container. Multiple implementation projects, consultations, and explorations
run under the same workspace. Project-specific state is session-scoped.

### Four loading tiers

**Tier 1 — Universal permanent (every workspace, always loaded)**
```
METHODOLOGY_RULES.md      standing rules, checklists, evidence protocol
KNOWLEDGE_REGISTRY.md     domain router and workspace reference
```

**Tier 2 — Workspace permanent (selected at workspace setup, always loaded)**

Domain files whose trigger is always active in this workspace.
```
This workspace:   FIRMWARE_PORT_ARCH.md, SECURITY_ARCH_KNOWLEDGE.md
Yocto workspace:  YOCTO_BSP_ARCH.md
RTOS workspace:   (none — see Tier 3)
```

**Tier 3 — Session-scoped (Claude prompts, human selects)**

Never preloaded. At session start, Claude asks for the session goal,
then presents relevant Tier 3 options. Human selects which to load.
Human is never expected to recall which file to load from memory.
```
[project_name]_context.md     project-specific state: results, OI register,
                               hardware config, phase status
RTOS_BENCH_ARCH.md            benchmark principles, timing methodology,
                               PMU attribution, VirtIO eBPF design,
                               pre-implementation checklists
```

**Tier 3 also includes on-demand reference docs** presented by Claude
when the session goal matches their scope:
```
METHODOLOGY_REF.md     anti-pattern review, post-mortem, new domain doc
                       creation, methodology questions, knowledge base
                       maintenance, workflow or process optimization
```

---

## Workspace Setup

### Project instruction — copy exactly, never changes

```
Read METHODOLOGY_RULES.md and KNOWLEDGE_REGISTRY.md at session start.
Apply all standing rules from METHODOLOGY_RULES.md.
At session start, ask the human what their goal is for this session.
Based on the answer, present relevant Tier 3 files as options with
one-line descriptions, ask the human to select which to load, then
wait for the paste before proceeding with any work.
```

### Setup steps

```
1. Create Claude project — name after technology domain, not a specific project
2. Upload Tier 1: METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
3. Upload Tier 2: domain files always active in this workspace
4. Set project instruction (copy exactly from above)
5. Keep on local disk: METHODOLOGY_REF.md, [project]_context.md, domain arch docs not in Tier 2
```

### Workspace examples

**Embedded platform security (this workspace)**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  FIRMWARE_PORT_ARCH.md, SECURITY_ARCH_KNOWLEDGE.md
Tier 3:  apollo510_context.md, stm32u585_context.md, METHODOLOGY_REF.md  (local disk)
```

**Yocto BSP workspace**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  YOCTO_BSP_ARCH.md
Tier 3:  [board]_context.md, FIRMWARE_PORT_ARCH.md, METHODOLOGY_REF.md  (local disk)
```

**RTOS benchmarking workspace**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  (none)
Tier 3:  RTOS_BENCH_ARCH.md, [project]_context.md, METHODOLOGY_REF.md  (local disk)
```

---

## Session Workflow

No ceremony. No session header.

1. Open session — Tier 1 and Tier 2 files already loaded
2. Claude asks: "What is your goal for this session?"
3. Claude presents relevant Tier 3 options with one-line descriptions
4. Human selects which to load; Claude waits for paste before proceeding
5. Claude applies all loaded constraints and proceeds

Mid-session domain shift: Claude states which file is needed and waits
before responding to the new topic.

---

## Domain Registry

At session start, Claude maps the stated goal to relevant Tier 3 files
using the "Active when" column below. Claude presents matches as options
and waits for human selection. Domain activation is cumulative — once
loaded, stays active for the session.

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

If a domain is active but its file is not available: state this explicitly
and apply METHODOLOGY_RULES.md pre-implementation checklist as fallback.

| File | Status | Notes |
|---|---|---|
| `METHODOLOGY_RULES.md` | ✅ Tier 1 | Always active |
| `METHODOLOGY_REF.md` | ✅ Tier 3 | On-demand — presented at session start when goal matches |
| `YOCTO_BSP_ARCH.md` | ✅ Available | v1.2 May 2026 |
| `FIRMWARE_PORT_ARCH.md` | ✅ Available | v1.1 May 2026 — TF-M PSA-L2 porting, Domains A–F |
| `SECURITY_ARCH_KNOWLEDGE.md` | ✅ Available | v0.1 May 2026 — non-securable NVM, trust boundary, CRA/PSA |
| `RTOS_BENCH_ARCH.md` | ✅ Tier 3 | v1.4 May 2026 — presented as option at session start |
| `BOOT_CHAIN_ARCH.md` | 🔲 Planned | — |
| `PCIE_BSP_ARCH.md` | 🔲 Planned | — |
| `ETHERNET_BSP_ARCH.md` | 🔲 Planned | — |
| `USB_STORAGE_ARCH.md` | 🔲 Planned | — |
| `UBOOT_ARCH.md` | 🔲 Planned | — |
| `ZEPHYR_ARCH.md` | 🔲 Planned | — |

---

## Registry Changelog

| Version | Date | Change |
|---|---|---|
| 1.0–1.8 | May 2026 | See METHODOLOGY_REF.md for full changelog history |
| 1.9 | May 2026 | Trim pass. Removed: duplicated standing rules block, domain activation examples, maintenance protocols, methodology backlog, full changelog history — all moved to METHODOLOGY_REF.md. Registry now contains only operational content needed every session: tier model, workspace setup, session workflow, domain routing table, status table. |
| 1.10 | May 2026 | Added "workflow or tooling optimization" and "process restructuring" as explicit METHODOLOGY_REF.md triggers. Added workflow/process optimization row to session workflow table. Source: session retrospect — undeclared mental model anti-pattern (MB-007). |
| 1.11 | May 2026 | Workflow optimization pass. RTOS_BENCH_ARCH.md moved from Tier 2 to Tier 3 — 13K tokens removed from always-loaded set. Session workflow changed from human-recall to Claude-driven: Claude asks session goal, presents Tier 3 options, human selects. Project instruction updated accordingly. SSoT discipline moved to METHODOLOGY_RULES.md. |
| 1.12 | May 2026 | Simplification pass. Tier 4 collapsed into Tier 3 — both are paste-on-demand, distinction was implementation detail without user value. Three tiers total. METHODOLOGY_REF.md is now a Tier 3 option presented at session start when goal matches. |
| 1.13 | May 2026 | METHODOLOGY_RULES.md bumped to v2.0-rules: Rule 2 enforcement strengthened (principle-based trigger, self-report obligation, explicit human sign-off gate for external deliverables, Rule 1+2 linked). Phase Structure trimmed to generic discipline. CLAUDE_PROJECT_INSTRUCTION.md corrected: METHODOLOGY.md → METHODOLOGY_RULES.md; Rule 2 trigger added. |

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*
*Maintenance protocols, backlog, and full changelog: METHODOLOGY_REF.md*

<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# BSP Engineering Knowledge Registry

**Version:** 1.10
**Date:** May 2026
**Author:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/zlhk100/bsp-engineering-knowledge

---

## Purpose

This file is the domain router and workspace operational reference.
It is Tier 1 — loaded every session in every workspace.

Maintenance protocols, backlog, changelogs, and domain activation
examples are in METHODOLOGY_REF.md (Tier 4 — load when needed).

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
RTOS workspace:   RTOS_BENCH_ARCH.md
```

**Tier 3 — Session-scoped (your decision, no protocol)**

Paste at session start when working on a specific implementation project.
Skip for consultations, architecture discussions, or methodology work.
Claude never asks for this — it is always your call.
```
[project_name]_context.md     one file per active implementation project
```

**Tier 4 — On-demand (Claude requests, you paste)**
```
METHODOLOGY_REF.md     triggers: anti-pattern review, post-mortem,
                       new domain doc creation, cold review of Category 1
                       file, methodology questions, knowledge base maintenance,
                       workflow or tooling optimization, process restructuring
```

---

## Workspace Setup

### Project instruction — copy exactly, never changes

```
Read METHODOLOGY_RULES.md and KNOWLEDGE_REGISTRY.md at session start.
Apply constraints from any domain files already present in context.
Identify if additional domain files are needed from the session goal.
Request any needed conditional files before proceeding.
Apply all standing rules from METHODOLOGY_RULES.md.
```

### Setup steps

```
1. Create Claude project — name after technology domain, not a specific project
2. Upload Tier 1: METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
3. Upload Tier 2: domain files always active in this workspace
4. Set project instruction (copy exactly from above)
5. Keep on local disk: METHODOLOGY_REF.md, [project]_context.md files
```

### Workspace examples

**Embedded platform security (this workspace)**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  FIRMWARE_PORT_ARCH.md, SECURITY_ARCH_KNOWLEDGE.md
Tier 3:  apollo510_context.md, stm32u585_context.md  (local disk)
Tier 4:  METHODOLOGY_REF.md  (local disk)
```

**Yocto BSP workspace**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  YOCTO_BSP_ARCH.md
Tier 3:  [board]_context.md  (local disk)
Tier 4:  METHODOLOGY_REF.md, FIRMWARE_PORT_ARCH.md  (local disk — paste if session touches secure boot)
```

**RTOS benchmarking workspace**
```
Tier 1:  METHODOLOGY_RULES.md, KNOWLEDGE_REGISTRY.md
Tier 2:  RTOS_BENCH_ARCH.md
Tier 3:  [project]_context.md  (local disk)
Tier 4:  METHODOLOGY_REF.md  (local disk)
```

---

## Session Workflow

No ceremony. No session header.

1. Open session — Tier 1 and Tier 2 files already loaded
2. State session goal
3. Paste project context if working on a specific implementation project
4. Claude applies loaded domain constraints, requests Tier 4 if triggered, proceeds

| Session type | Paste at start | Claude may request |
|---|---|---|
| Implementation (Project A) | `project_a_context.md` | `METHODOLOGY_REF.md` if trigger fires |
| Implementation (Project B) | `project_b_context.md` | `METHODOLOGY_REF.md` if trigger fires |
| Consultation / architecture | nothing | `METHODOLOGY_REF.md` if trigger fires |
| Methodology / post-mortem | nothing | `METHODOLOGY_REF.md` — likely |
| Workflow / process optimization | nothing | `METHODOLOGY_REF.md` — certain |
| New domain document | nothing | `METHODOLOGY_REF.md` — certain |

Mid-session domain shift: Claude states which file is needed and waits
before responding to the new topic.

---

## Domain Registry

Scan session goal for keywords in the "Active when" column.
Request the corresponding file if not already in context.
Domain activation is cumulative — once active, stays active for the session.

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
| `METHODOLOGY_REF.md` | ✅ Tier 4 | Conditional — load on trigger |
| `YOCTO_BSP_ARCH.md` | ✅ Available | v1.2 May 2026 |
| `FIRMWARE_PORT_ARCH.md` | ✅ Available | v1.1 May 2026 — TF-M PSA-L2 porting, Domains A–F |
| `SECURITY_ARCH_KNOWLEDGE.md` | ✅ Available | v0.1 May 2026 — non-securable NVM, trust boundary, CRA/PSA |
| `RTOS_BENCH_ARCH.md` | ✅ Available | v1.4 May 2026 |
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

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*
*Maintenance protocols, backlog, and full changelog: METHODOLOGY_REF.md*

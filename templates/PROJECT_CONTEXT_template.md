<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# [Project Name] — Master Project Context

**Version:** 1.0 | **Date:** [Month Year] | **Classification:** [CONFIDENTIAL / INTERNAL / PUBLIC]
**Status:** Active
**Changes from previous:** Initial release.

> **AI LOADING INSTRUCTIONS:** This is the single source of truth. Load
> before responding to any technical question. Hardware topology in
> Section 2 overrides all prior versions and all conversation history.
> When in doubt, this document wins.

---

## Change Log

| Version | Date | CN | Summary |
|---|---|---|---|
| v1.0 | [Date] | — | Initial release |

*All subsequent changes require a numbered Change Notice (CN-NNN).
Change notices are the authoritative mechanism for recording decisions.*

---

## Sources Incorporated

| Document | Date | Authority |
|---|---|---|
| [Hardware schematic / block diagram name] | [Date] | **GROUND TRUTH — schematic overrides all** |
| [SoC datasheet or TRM] | [Date] | Register map, peripheral addresses |
| [Vendor SDK release notes] | [Date] | Kernel/BSP version confirmed |
| [Reference board DTS] | [Date] | DT structural patterns |
| CN-001 through CN-NNN | [Date] | Approved changes |

---

## 1. Project Summary

Deliver a [description of deliverable] for the **[Board/Product Name]**
([carrier board or SoM description]) enabling:
- [Key capability 1]
- [Key capability 2]
- [Kernel version, Yocto release, or firmware framework]
- [Security requirements if applicable]

---

## 2. Confirmed Hardware Topology

> Verified against [schematic name].
> Schematic overrides all other sources.

### 2.1 Core SoM / Target Platform

| Subsystem | Detail |
|---|---|
| SoC / MCU | [Name, core architecture] |
| RAM | [Size, type] |
| Storage | [Type, size] |
| Key interfaces | [List key interfaces: UART, I2C, SPI, PCIe, USB etc.] |

### 2.2 [Add subsystem sections as needed]

*Example subsections: System Management, PCIe Topology, Storage,
Power Architecture, Debug interfaces.*

> ⚠️ **Document all P1 open items that block DT or driver work here.**

---

## 3. [Domain-Specific Architecture Section]

*For BSP projects: SMMU/DT architecture, boot chain, security.*
*For firmware ports: platform port specification, interface contracts.*
*Add sections appropriate to the project domain.*

---

## 4. Security Architecture (if applicable)

### 4.1 Boot Chain

```
[Document the complete boot chain with all stages]
```

### 4.2 Key Management

| Layer | Purpose | Keys |
|---|---|---|
| [Development PKI] | [Purpose] | [Key type — test only] |
| [Production PKI] | [Purpose] | [Key type] |

---

## 5. Binary / Artefact Hosting

**Non-HLOS / vendor binary policy:**
[Document where vendor binaries come from, access controls, and
any interim bypass arrangements pending artifact server setup.]

| Binary | Source | Access |
|---|---|---|
| [Binary name] | [Vendor download / artifact server] | [Team / individual] |

---

## 6. Kernel / Driver Status (BSP projects)

| Component | Status | Notes |
|---|---|---|
| [Driver name] | [✅ Mainlined / ⚠️ Out-of-tree / 🔴 Not available] | [Notes] |

---

## 7. Milestone Plan

| Milestone | Target | Key Deliverables | Gate | Status |
|---|---|---|---|---|
| M1 | [Month] | [Deliverables] | [Acceptance criterion] | 🔴 Not started |
| M2 | [Month] | [Deliverables] | [Acceptance criterion] | 🔴 Not started |

---

## 8. Open Items Register

### P1 — Blockers

| ID | Item | Blocks |
|---|---|---|
| OI-001 | **[Item description]** | [What this blocks] |

### P2 — Known Gaps

| ID | Item | Blocks |
|---|---|---|
| OI-NNN | **[Item description]** | [What this blocks] |

### P3 — Lower Priority

| ID | Item |
|---|---|
| OI-NNN | **[Item description]** |

### ✅ Closed

| ID | Item | Resolution |
|---|---|---|
| OI-NNN | [Item] | ✅ [Resolution] |

---

## 9. Quick-Load Summary

**Key facts every contributor must know:**

1. [Critical fact 1 — e.g. two boards, never conflate them]
2. [Critical fact 2 — e.g. irreversible operation warning]
3. [Critical fact 3 — e.g. version-locked binary pair]
4. [Add as project experience accumulates]

---

## 10. Reference Document Status

| Document | Status | Use For |
|---|---|---|
| [Schematic name] | ✅ Ground truth | All hardware topology |
| [Datasheet name] | ✅ Current | [Purpose] |
| CN-001 through CN-NNN | ✅ Approved | All scope/decision changes |

---

## 11. Repository Locations

| Repo | URL | Branch | Contents |
|---|---|---|---|
| [repo-name] | [URL] | [branch] | [Contents] |

**Clone and build:**
```bash
[Document the build procedure here]
```

---

## CN-001 — [First Change Notice Title] ([Date])

### Status — [PENDING / COMPLETE ✅]

### Summary

[Describe what changed and why.]

### Changes Applied

| Item | Change |
|---|---|
| [File or decision] | [What changed] |

---

## Guidelines for Maintaining This Document

**Change notice discipline:**
Every design decision requires a written change notice before it is
considered final. The change notice format is:
- CN number (sequential)
- Date and status
- Summary of what changed and why
- Table of specific changes applied

**Open item discipline:**
- P1 items block milestone progress — resolve before advancing
- P2 items are known gaps with a plan
- P3 items are lower priority — tracked but not blocking
- Closed items document the resolution for future reference

**Version discipline:**
Bump the version number with every change notice applied.
Update the Change Log table at the top.
The version in the filename never changes — only the header version.

**The schematic is always ground truth.**
If any document, conversation, or AI output conflicts with the
schematic, the schematic wins. Document the correction as a change
notice with VERIFIED evidence tier.

---

*[Project Name] — Master Project Context v1.0*
*Change Notice process: written CN + impact assessment required before any deviation.*
*Replace project context file with updated version after each change notice.*
*Load at start of every technical session.*

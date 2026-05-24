<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# New Project Setup Checklist

**Version:** 1.0
**Date:** May 2026
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

Use this checklist to set up a new project using the AI-Assisted
Engineering Pattern. Complete all items before starting the first
technical session. Total setup time: under 30 minutes.

---

## Phase 1 — Knowledge Base (10 minutes)

### 1.1 — Get the knowledge base files

```bash
git clone https://github.com/<your-org>/bsp-engineering-knowledge
cd bsp-engineering-knowledge
```

### 1.2 — Identify active domains

Read the project brief. Check each domain that applies:

```
□ Yocto-based BSP (recipes, kas, layers, kernel patches)
  → YOCTO_BSP_ARCH.md

□ PCIe devices (switches, SMMU, iommu-map, msi-map)
  → PCIE_BSP_ARCH.md

□ Ethernet (PHY drivers, MAC configuration, MDIO)
  → ETHERNET_BSP_ARCH.md

□ USB / Storage (xHCI, USB bridges, NVMe, SD)
  → USB_STORAGE_ARCH.md

□ Boot chain (bootloader, partition table, secure boot)
  → BOOT_CHAIN_ARCH.md

□ Security firmware port (TF-M, PSA, CMSIS HAL)
  → FIRMWARE_PORT_ARCH.md
```

If a domain is planned but not yet available (🔲 in registry), note it
and apply METHODOLOGY.md general pre-implementation checklist as fallback.

### 1.3 — Copy files to your project workspace

```bash
PROJECT_WS=~/workspace/your-project-name

# Universal files (always copy)
cp KNOWLEDGE_REGISTRY.md $PROJECT_WS/
cp METHODOLOGY.md $PROJECT_WS/

# Domain files (copy only active domains from 1.2)
cp domains/YOCTO_BSP_ARCH.md $PROJECT_WS/       # if Yocto active
cp domains/PCIE_BSP_ARCH.md $PROJECT_WS/         # if PCIe active
# ... etc.
```

---

## Phase 2 — Project Context (15 minutes)

### 2.1 — Create PROJECT_CONTEXT.md

```bash
cp templates/PROJECT_CONTEXT_template.md $PROJECT_WS/PROJECT_CONTEXT.md
```

Fill in all sections. Minimum required before first session:

```
□ Section 1 — Project summary (what are we building)
□ Section 2 — Hardware topology (from schematic — do not guess)
□ Section 7 — Milestone plan (what is M1, what is the gate)
□ Section 8 — Open items register (P1 blockers you already know)
□ Section 9 — Quick-load summary (at least 3 key facts)
```

**Do not start technical sessions with an empty or partially filled
project context.** The first session should spend time filling it in
if it is not ready — not doing technical work with missing ground truth.

### 2.2 — Identify the schematic as ground truth

Add the schematic to Section 2 sources table with authority:
`**GROUND TRUTH — schematic overrides all**`

If no schematic is available yet, document this explicitly as a P1 blocker
in the open items register before proceeding.

---

## Phase 3 — Claude Project Setup (5 minutes)

### 3.1 — Create a Claude project

In Claude.ai → Projects → New Project.
Name it clearly: `[Customer/Product] BSP` or similar.

### 3.2 — Upload knowledge base files

Upload all files copied in Phase 1 plus your PROJECT_CONTEXT.md:
- `KNOWLEDGE_REGISTRY.md`
- `METHODOLOGY.md`
- `PROJECT_CONTEXT.md`
- Active domain files (from 1.2)

### 3.3 — Set project instruction

Copy the instruction from `templates/CLAUDE_PROJECT_INSTRUCTION.md`
**verbatim** into the project instruction field.

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

**Do not paraphrase or modify this instruction.**

### 3.4 — Verify the project instruction loaded correctly

Start a new session in the project. Type only: `Hello`

Claude should respond by summarising its loading sequence and current
project state from PROJECT_CONTEXT.md. If it responds generically
without referencing the project, the instruction or files are not
loading correctly — check the project settings.

---

## Phase 4 — Claude Code Setup (5 minutes)

### 4.1 — Create CLAUDE.md in workspace root

```bash
cp templates/CLAUDE.md_template $PROJECT_WS/CLAUDE.md
```

Edit `CLAUDE.md`:
- Fill in workspace layout section with your actual directory structure
- Add project-specific standing constraints (never replace X, never mix Y)
- Keep universal constraints section unchanged

### 4.2 — Verify Claude Code reads CLAUDE.md

```bash
cd $PROJECT_WS
claude
```

Type: `Read CLAUDE.md and summarise the project constraints.`

Claude Code should list the constraints from CLAUDE.md. If it does not
mention CLAUDE.md content, check that the file is in the current directory.

---

## Phase 5 — Final Verification

Run this checklist before the first technical session:

```
Knowledge base
□ KNOWLEDGE_REGISTRY.md in workspace
□ METHODOLOGY.md in workspace
□ PROJECT_CONTEXT.md in workspace with required sections filled in
□ Active domain files in workspace

Claude project
□ All knowledge base files uploaded to Claude project
□ Project instruction set verbatim (not paraphrased)
□ Test session confirmed: Claude loads and summarises project state

Claude Code
□ CLAUDE.md in workspace root
□ Project-specific constraints added
□ Test confirmed: Claude Code reads CLAUDE.md and lists constraints

Project context completeness
□ Hardware topology section filled from schematic
□ Milestone plan has at least M1 with gate criterion
□ P1 open items documented
□ At least 3 quick-load facts written
□ Schematic identified as ground truth source
```

If any item is unchecked — resolve it before starting technical work.

---

## Ongoing Maintenance

### After every session
```
□ Update PROJECT_CONTEXT.md with any new decisions (via change notice)
□ Replace PROJECT_CONTEXT.md in Claude project with updated version
□ Commit and push changes to version control
```

### When a new principle is discovered
```
□ Apply the five-step self-extension process (KNOWLEDGE_REGISTRY.md)
□ Bump the domain document version
□ Replace the domain file in Claude project with updated version
□ Consider opening a PR to the shared knowledge base repository
```

### When a new domain becomes active mid-project
```
□ Copy the domain file to workspace
□ Upload to Claude project
□ KNOWLEDGE_REGISTRY.md activates it automatically in next session
```

---

*Licence: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Lei Zhou*

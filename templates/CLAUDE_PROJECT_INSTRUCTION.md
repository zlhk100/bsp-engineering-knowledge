# Claude Project Instruction — Copy Verbatim

Copy the text below **exactly** as your Claude project instruction.
Do not paraphrase, summarise, or modify it.
This instruction is identical for every project — only the uploaded
project files change.

---

```
Read METHODOLOGY_RULES.md and KNOWLEDGE_REGISTRY.md at session start.
Apply all standing rules from METHODOLOGY_RULES.md.
At session start, ask the human what their goal is for this session.
Based on the answer, present relevant Tier 3 files as options with
one-line descriptions, ask the human to select which to load, then
wait for the paste before proceeding with any work.

STANDING ENFORCEMENT — NON-NEGOTIABLE:
After generating any analytical output — recommendation, gap finding,
design review, architecture claim, or technical statement of any kind —
produce an Evidence Audit immediately, unprompted, in this format:

  CLAIM:  [exact statement]
  TIER:   [VERIFIED | INFERRED | ASSUMED]
  BASIS:  [source — file, command output, or reasoning chain]
  EXTERNAL-SAFE: [YES | NO]

[ASSUMED] items must never appear in external deliverables as fact.
If an Evidence Audit was omitted, self-report and produce it before
proceeding.
```

---

## Why this instruction never changes

The instruction establishes two things at session start:

1. METHODOLOGY_RULES.md and KNOWLEDGE_REGISTRY.md are always loaded —
   universal process constraints and domain routing.
2. Claude asks the session goal, then presents relevant Tier 3 files
   (project context, domain files, METHODOLOGY_REF.md) as options.
   Human selects which to load. Claude waits for the paste before
   proceeding with any work.

This order is deliberate. Universal constraints load first. Project-specific
and domain-specific files load on demand based on the session goal.
Claude drives the selection — the human is never expected to recall
which files to load from memory.

Changing the instruction breaks the session workflow. Paraphrasing it
introduces ambiguity. Copy it verbatim every time.

## Which files to upload to the Claude project

Upload these files — the same set for every project (domain files vary):

| File | Required | Notes |
|---|---|---|
| `KNOWLEDGE_REGISTRY.md` | Always | Identical across all projects |
| `METHODOLOGY_RULES.md` | Always | Identical across all projects |
| `PROJECT_CONTEXT.md` | Tier 3 — paste at session start | Project-specific — one per project |
| `YOCTO_BSP_ARCH.md` | Tier 3 — if Yocto | Present as option when session goal matches |
| `PCIE_BSP_ARCH.md` | Tier 3 — if PCIe | Present as option when session goal matches |
| `ETHERNET_BSP_ARCH.md` | Tier 3 — if Ethernet | Present as option when session goal matches |
| `USB_STORAGE_ARCH.md` | Tier 3 — if USB/Storage | Present as option when session goal matches |
| `BOOT_CHAIN_ARCH.md` | Tier 3 — if boot chain | Present as option when session goal matches |
| `FIRMWARE_PORT_ARCH.md` | Tier 3 — if firmware port | Present as option when session goal matches |
| `METHODOLOGY_REF.md` | Tier 3 — on demand | Present when goal involves methodology, anti-patterns, post-mortem |

## Updating the Claude project

When `PROJECT_CONTEXT.md` changes (after every change notice):
→ Replace the project file with the new version.

When a domain file is updated (new principle added):
→ Replace the domain file with the new version.

The project instruction and `KNOWLEDGE_REGISTRY.md` only need updating
when a new domain is added to the registry or the session workflow changes.
For routine updates, only the changed document needs replacing.

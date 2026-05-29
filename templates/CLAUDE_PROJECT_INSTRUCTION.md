# Claude Project Instruction — Copy Verbatim

Copy the text below **exactly** as your Claude project instruction.
Do not paraphrase, summarise, or modify it.
This instruction is identical for every project — only the uploaded
project files change.

---

```
At the start of every session:
1. Read KNOWLEDGE_REGISTRY.md using the view tool
2. Read PROJECT_CONTEXT.md using the view tool
3. Read METHODOLOGY_RULES.md using the view tool
4. Scan the session goal and conversation for active domains
5. Read each active domain file listed in the registry using the view tool
Apply all standing rules and domain constraints before responding to
any technical question.

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

The instruction loads the knowledge base in a fixed order:
1. KNOWLEDGE_REGISTRY.md — tells Claude which domains are active
2. PROJECT_CONTEXT.md — provides project-specific ground truth
3. METHODOLOGY_RULES.md — applies universal process constraints
4. Domain files — applies domain-specific structural constraints

This order is deliberate. The registry routes to the right domain files.
The project context provides the facts. The methodology applies the process.
The domain files apply the structural guardrails.

Changing the instruction breaks the load order. Paraphrasing it introduces
ambiguity about which files to load and when. Copy it verbatim every time.

## Which files to upload to the Claude project

Upload these files — the same set for every project (domain files vary):

| File | Required | Notes |
|---|---|---|
| `KNOWLEDGE_REGISTRY.md` | Always | Identical across all projects |
| `METHODOLOGY_RULES.md` | Always | Identical across all projects |
| `PROJECT_CONTEXT.md` | Always | Project-specific — one per project |
| `YOCTO_BSP_ARCH.md` | If Yocto | Upload when project uses Yocto |
| `PCIE_BSP_ARCH.md` | If PCIe | Upload when project has PCIe work |
| `ETHERNET_BSP_ARCH.md` | If Ethernet | Upload when project has Ethernet work |
| `USB_STORAGE_ARCH.md` | If USB/Storage | Upload when project has storage work |
| `BOOT_CHAIN_ARCH.md` | If boot chain | Upload when project has boot chain work |
| `FIRMWARE_PORT_ARCH.md` | If firmware port | Upload when porting TF-M or similar |

## Updating the Claude project

When `PROJECT_CONTEXT.md` changes (after every change notice):
→ Replace the project file with the new version.

When a domain file is updated (new principle added):
→ Replace the domain file with the new version.

The project instruction and `KNOWLEDGE_REGISTRY.md` only need updating
when a new domain is added to the registry. For routine updates, only
the changed document needs replacing.

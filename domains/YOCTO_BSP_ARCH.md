<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
Licensed under Creative Commons Attribution 4.0 International
https://creativecommons.org/licenses/by/4.0/

Attribution: If you use or adapt this work, please credit:
Lei Zhou, "Yocto BSP Architecture Principles",
https://github.com/<your-org>/bsp-engineering-knowledge
-->

# Yocto BSP Architecture Principles

**Version:** 1.2 (filename: YOCTO_BSP_ARCH.md — version-free)
**Date:** May 2026
**Authors:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Origin:** Derived from Qualcomm SoC BSP bring-up project code review findings
**Scope:** Any Yocto-based BSP bring-up project, with or without AI-assisted
code generation
**Companion documents:**
- `METHODOLOGY.md` — AI-Assisted Embedded Systems Engineering Methodology

---

## Overview

This document captures structural design principles for Yocto BSP development,
derived from real code review findings on a Qualcomm SoC BSP bring-up project.
The principles apply to any Yocto-based BSP and are particularly relevant
when AI tools are used to generate recipes, layer configuration, and build
infrastructure.

The document is organised into six categorical domains. Every design decision
in a Yocto BSP falls into one of these domains. When new principles are
discovered on future projects, they slot into the appropriate domain rather
than being appended to a flat list.

### The Unifying Meta-Principle

> Every design decision in a Yocto BSP should express the minimal, explicit,
> correctly-placed statement of what is genuinely novel about your board or
> product. Everything else should be delegated to the upstream layer that
> owns it.

The six domains are six applications of this idea to different aspects of
BSP development. A team that internalises the meta-principle can derive
the domain principles themselves — and can apply the same reasoning to
situations this document does not explicitly cover.

### Self-Extension Mechanism

When a new principle is discovered on a future project — through a reviewer
comment, a build failure, or a post-mortem — add it as follows:

1. **Identify:** which domain owns this finding?
2. **Generalise:** what is the underlying principle beyond the specific instance?
3. **Number:** assign the next available ID in the domain (A6, B6, C6...)
4. **State:** rule, rationale, anti-pattern, verification method
5. **Update:** KNOWLEDGE_REGISTRY.md domain activation keywords if new trigger signals are needed

---

## Domain A — Layer Architecture and Responsibility

*Who owns what, and where does each concern live.*

### A1 — Understand Before You Build (Capability Audit)

**Rule:** Before writing any recipe, task, bbappend, or configuration,
perform a capability audit of every layer in your dependency stack.
New code should only express what is genuinely novel about your board.

**The four questions to ask before writing anything:**

1. Is there a `.inc` file that provides this task? → `require` it
2. Is there a `.bbclass` that provides this behaviour? → `inherit` it
3. Is there a `PACKAGECONFIG` that controls this feature? → set it
4. Is there a `MACHINE_FEATURES` or `DISTRO_FEATURES` that gates this? → declare it

If the answer to all four is no, writing new code is justified.
If the answer to any is yes, use the existing mechanism.

**The audit commands — run before any recipe generation session:**
```bash
# Find all .inc files relevant to your domain in upstream layers
grep -r "firmware\|boot\|image\|kernel" meta-*/recipes-*/ --include="*.inc" -l

# Find all bbclasses available for inheritance
ls meta-*/classes-recipe/
ls meta-*/classes-global/

# Find PACKAGECONFIG options in recipes you depend on
grep -r "PACKAGECONFIG" meta-*/recipes-*/ | grep "\[" | head -30

# Find what MACHINE_FEATURES and DISTRO_FEATURES are consumed
grep -r "MACHINE_FEATURES\|DISTRO_FEATURES" meta-*/ \
    --include="*.bb" --include="*.inc" --include="*.bbclass" | head -30
```

**Anti-pattern:** Writing a board firmware recipe that hand-rolls its own
binary copy loop when an upstream `.inc` file already provides staging,
DEPLOYDIR installation, and missing-file error handling.

**Cost of skipping the audit:** Silent divergence when upstream improves
the mechanism you duplicated. Maintenance burden grows with every upstream
layer update. Review findings that require rework after code is written.

---

### A2 — Every Concern at Its Correct Abstraction Layer

**Rule:** Every piece of configuration and logic must live at the abstraction
layer that owns its concern. Placing logic at the wrong layer makes it
invisible to anyone reading the correct layer, creates conflicts with
legitimate code at the right layer, and prevents reuse.

**Ownership map:**

| Concern | Correct layer |
|---|---|
| Global feature policy | DISTRO conf |
| Hardware capabilities, BSP defaults | MACHINE conf |
| How to build one package | Recipe `.bb` |
| Reusable build behaviour | BBclass |
| What goes in the image | Image recipe |
| Narrow override to upstream recipe | BBappend |
| Developer-local paths, credentials | `local.conf` / gitignored kas config |
| Team-shared build configuration | kas-user.yml.example (committed) |

**The test:** If you are writing code in layer X that references a concern
owned by layer Y, stop and move it to Y.

**Anti-patterns:**
- Infrastructure logic (file staging, flash assembly) in image bbappends
- Hardware configuration (GPIO, clock rates) in distro conf
- Package selection policy in machine conf
- Build performance variables (`BB_NUMBER_THREADS`) in machine conf
- Feature flags (`DISTRO_FEATURES`) set in recipe bbappend

---

### A3 — BBappends Express Overrides, Not Features

**Rule:** A `.bbappend` should contain the minimum necessary to adapt an
upstream recipe to your context. A growing bbappend signals that a new
recipe or bbclass is needed.

**Signals that a bbappend has outgrown its role:**
- Contains a full `do_install` or `do_compile` task override
- Has more than three patch files in `SRC_URI:append`
- Manipulates `IMAGE_INSTALL`
- Exceeds ~30 lines

**Why this matters:** A large bbappend is a hidden feature implementation
masquerading as an override. It breaks on upstream recipe version bumps
and is invisible to anyone reading the upstream recipe.

---

### A4 — One Layer, One Purpose

**Rule:** Each custom layer has exactly one defined purpose, stated in its
`layer.conf`. Mixing concerns in one layer creates a layer that cannot be
reused across products and cannot be audited independently.

**Suggested layer structure:**

| Layer | Purpose |
|---|---|
| `meta-bsp` | Board-specific machine conf, firmware staging, BSP recipes |
| `meta-product` | Product image policy, package selection |
| `meta-security` | Security hardening, signing pipeline |
| `meta-features` | Optional features not provided by upstream layers |

**Anti-pattern:** A single catch-all layer containing machine conf, image
policy, kernel patches, firmware recipes, and signing scripts simultaneously.

---

### A5 — Upstream Layers Are Read-Only References

**Rule:** Never modify files in upstream layers (oe-core, meta-poky,
vendor BSP layers). All customisation uses Yocto extension mechanisms:
bbappend, bbclass inheritance, PACKAGECONFIG, MACHINE_FEATURES.

**Exception:** Upstreaming a fix — the change lives in a fork with a
clearly named branch, not a local modification to the checkout.

---

## Domain B — Configuration Management

*How configuration is declared, versioned, and owned.*

### B1 — Explicit Ownership of Every Configuration Surface

**Rule:** Every variable that affects build output must be explicitly
declared in a file you own and version-control. Implicit configuration
inherited from upstream distros, CI files, or floating SRCREVs is a
hidden liability.

**The hidden dependency test:** Could a change to any upstream file
silently change your build output without you touching a single file
you own? If yes, that upstream file is an implicit dependency — make
it explicit or document the accepted risk.

**Audit command:**
```bash
# What is the build actually inheriting from the distro?
bitbake -e | grep "^DISTRO_FEATURES\|^IMAGE_FEATURES\|^TCLIBC\|^INIT_MANAGER"
# Compare to what you explicitly set — any gap is an implicit dependency
```

---

### B2 — Depend Only on Documented Interfaces, Not Implementation Details

**Rule:** Upstream layer extension points are: exported variables,
bbclasses for inheritance, `.inc` files documented for `require`, and
PACKAGECONFIG options. Everything else is an implementation detail —
read it for reference, never depend on it.

**Interface vs implementation test:**
- Documented in the layer's README or official docs? → Interface, safe
- In a `ci/`, `scripts/`, or `tools/` directory? → Implementation detail
- Would upstream maintainers change it without a deprecation notice? →
  Implementation detail

**Anti-pattern:** Including upstream CI files (`ci/base.yml`) in your
kas config. These files serve the upstream project's own pipeline and
can change at any commit without notice, silently affecting your builds.

**Correct approach:** Read the CI file once to extract what you need.
Declare those settings explicitly in your own kas `local_conf_header`
or distro conf. Remove the `includes:` reference.

---

### B3 — Custom Distro for Every Production BSP

**Rule:** Never use `DISTRO = "poky"` or any reference distro in
production. Define a minimal custom distro conf in your BSP layer.

**Why poky is wrong for production:**
- Explicitly generates build warnings when used for hardware BSPs
- Upstream changes to poky silently affect your BSP configuration
- Carries defaults (init system, libc, package manager) you have not
  consciously evaluated

**Pattern:**
```bitbake
# conf/distro/my-product.conf
require conf/distro/poky.conf   # acceptable as base
DISTRO = "my-product"
DISTRO_NAME = "My Product BSP"
DISTRO_VERSION = "1.0"
INIT_MANAGER = "systemd"
# Every variable from this point is explicitly owned
```

---

### B4 — Feature Flags Over Conditional Logic in Recipes

**Rule:** Prefer `MACHINE_FEATURES`, `DISTRO_FEATURES`, and `PACKAGECONFIG`
over `if` statements in recipe tasks. Feature flags are composable,
auditable, and overridable from configuration. Conditional recipe logic
is invisible from the configuration level.

**Anti-pattern:**
```bitbake
do_install() {
    if [ "${MACHINE}" = "my-board" ]; then
        install -m 0644 special-config ${D}/etc/
    fi
}
```

**Correct pattern:**
```bitbake
# machine.conf
MACHINE_FEATURES += "special-feature"
# recipe — responds to feature flag declaratively
PACKAGECONFIG[special-feature] = "--enable-special,--disable-special,,"
```

---

### B5 — Variable Scoping Discipline

**Rule:** Use the most restrictive scoping available.
`VAR:pn-recipe` instead of global `VAR` where possible. Global variable
assignments in bbappends pollute the entire build's variable space and
create unexpected interactions with unrelated recipes.

**Scoping hierarchy (most to least restrictive):**
```bitbake
VAR:pn-specific-recipe = "value"   # one recipe only — prefer this
VAR:class-target = "value"         # all target recipes
VAR = "value"                      # global — use sparingly, document why
```

---

## Domain C — Reproducibility and Build Integrity

*Ensuring the same input always produces the same output.*

### C1 — Every External Input Must Be Pinned and Verified

**Rule:** Source code (SRCREV), layers (kas commit pins), firmware blobs
(checksums), build tools (container image digest) — every external input
must be pinned to a specific version and its integrity verified.

**`SRCREV = "${AUTOREV}"` is forbidden in production BSPs.**

A floating SRCREV means today's build and tomorrow's build may use
different kernel or recipe commits. The output difference is silent
and invisible without diffing deployed binaries.

**Every SRC_URI entry that downloads a file must have a checksum:**
```bitbake
SRC_URI = "https://example.com/firmware.bin;name=fw"
SRC_URI[fw.sha256sum] = "abc123def456..."
```

---

### C2 — KAS Manages All Layer Checkout

**Rule:** All layers must be declared in the kas config with URL, branch,
and pinned commit. Manual `git clone` steps before `kas build` are
build process defects, not conveniences.

**Pattern:**
```yaml
repos:
  meta-upstream:
    url: https://github.com/upstream/meta-upstream.git
    branch: main
    commit: abc123def456...   # pinned — never floating
  meta-bsp:
    url: https://your-org/meta-bsp.git
    branch: main
    commit: def456abc789...
    layers:
      .:
```

**Lock files:** Run `kas dump kas-user.yml > kas-user.yml.lock` after
any intentional update. Commit the lock file. CI always builds from
the lock file. This pins all transitive SRCREVs.

**README instruction for new team members:**
```bash
# The only setup required
cp kas-user.yml.example kas-user.yml
# Edit SSTATE_DIR and any local paths
kas build kas-user.yml
# That is all — no git clone steps required
```

---

### C3 — sstate Cache Isolation

**Rule:** sstate caches must be isolated per branch or release identifier
to prevent cross-contamination. A stale sstate entry from a different
kernel SRCREV silently producing old output is one of the hardest bugs
to diagnose in Yocto.

```yaml
local_conf_header:
  sstate: |
    SSTATE_DIR = "/path/to/sstate/cache-${BRANCH_NAME}"
```

---

### C4 — Hermetic Build Environment

**Rule:** The host system should contribute nothing to the build beyond
what the container or SDK provides. Builds depending on host-installed
tools at specific versions are non-reproducible across machines.

**Implementation:** Use a pinned container image via `kas-container`.
Pin the image digest, not just the tag — tags are mutable.

```bash
kas-container build kas-user.yml
```

---

### C5 — Hash Verification for All Downloaded Artefacts

**Rule:** Every file downloaded during a build — source tarballs, firmware
blobs, pre-built binaries — must have a corresponding hash in its
`SRC_URI` entry. Files without hash verification can be silently replaced
if the upstream source changes. This is both a reproducibility and a
supply chain security requirement.

---

## Domain D — Patch and Change Management

*How modifications to upstream code are organised and maintained.*

### D1 — Separate Concerns Into Separate Artefacts

**Rule:** Every independent concern — a driver, a DTS subsystem, a board
feature — lives in its own named artefact: a topic branch, a conf fragment,
a dtsi file, a package group. Flat lists mix concerns and prevent
independent management.

**Concern → Artefact mapping:**

| Concern | Correct artefact |
|---|---|
| Unrelated kernel patches | Separate topic branches (not one SRC_URI list) |
| Unrelated machine features | Separate conf fragments |
| Unrelated image packages | Separate package groups (`packagegroup-*.bb`) |
| Carrier board DTS subsystems | Separate per-subsystem dtsi files |
| Unrelated kas overrides | Named sections or separate kas fragments |

**The separation test:** Can you remove one concern without touching any
file that belongs to a different concern? If no, the concerns are not
properly separated.

---

### D2 — Kernel Patches as Topic Branches, Not SRC_URI Files

**Rule:** Out-of-tree kernel patches must be maintained as named topic
branches in a forked kernel repository. The recipe SRC_URI points at
an integration branch. Patch files in SRC_URI are forbidden — including
the first patch. A single SRC_URI patch file is the beginning of an
unmanageable list, not an acceptable starting point.

**Why SRC_URI patch lists fail at scale:**
- Patch dependencies are implicit — nothing enforces ordering
- Every kernel SRCREV bump requires rebasing the entire list in sequence
- No visibility into which patches are upstream-bound vs permanent
- A single patch conflict blocks all downstream patches
- A 10-patch list becomes unmanageable; a 20-patch list is unmaintainable

**The "TODO: move to topic branch later" pattern is forbidden.**
A patch applied via SRC_URI does not become a topic branch automatically
— it becomes permanent technical debt. The reviewer comment that
established this principle: *"Don't apply patches via out-of-band
bbappends, this will get unmanageable. Instead maintain them as topic
branches inside the kernel repository and treat that as the single
source of truth."* (BSP bring-up project session, 2026)

**Topic branch structure:**
```
your-org/kernel (fork)
├── base                ← pinned at upstream SRCREV
├── topic/driver-a      ← one driver, rebased on base
├── topic/board-dts     ← board DTS, rebased on base
├── topic/workaround-x  ← temporary fix with removal condition
└── integration/all     ← merge of all topics ← recipe points here
```

```bitbake
# linux-<soc>_<ver>.bbappend
SRC_URI = "git://your-org/kernel.git;branch=integration/all;protocol=https"
SRCREV  = "abc123..."   # pinned commit on integration branch
```

**Pre-implementation gate — before generating any kernel bbappend
that involves out-of-tree patches:**

```
□ Does a forked kernel repo exist with topic branch structure?
  If no — STOP. Create the fork and integration branch first.
  Never generate SRC_URI patch entries as a placeholder or
  temporary measure.
□ Is the recipe SRC_URI already pointing at the integration branch?
  If no — fix SRC_URI before adding any patch content.
□ Is the new patch in its own named topic branch?
  If no — create the topic branch, apply the patch there,
  merge into integration, then update SRCREV.
```

If any answer is no — resolve it before writing a single line of
bbappend content. Come back to the design session to unblock.

**Trigger:** Before adding the first patch. The topic branch
infrastructure must exist before any patch work begins, not after.

---

### D3 — Upstream-First Mindset

**Rule:** Every patch to an upstream component should be written as if
it will be submitted upstream: correct commit message format, `Signed-off-by`,
no board-specific hacks, no debug leftovers. Even patches that are never
submitted are easier to rebase onto newer upstream versions when written
this way.

**Commit message format for kernel patches:**
```
subsystem: component: short description

Longer description explaining why this change is needed,
what problem it solves, and any relevant hardware context.

Signed-off-by: Author Name <email@example.com>
```

---

### D4 — Patch Lifetime Classification

**Rule:** Every out-of-tree patch must be classified at creation.
Unclassified patches accumulate indefinitely and become permanent
technical debt with no removal path.

**Classifications:**

| Class | Meaning | Removal condition |
|---|---|---|
| `UPSTREAM` | Submitted or pending upstream | Drop when merged in SRCREV |
| `BOARD` | Genuinely board-specific, never upstream | Permanent — document why |
| `WORKAROUND` | Known technical debt | Document the removal trigger |

**Where to record the classification:** In the commit message on the
topic branch, or in a `patches/README.md` in the meta layer.

---

### D5 — Device Tree Fragments per Subsystem

**Rule:** Carrier board device tree customisation is split into
per-subsystem dtsi fragments, each independently auditable and modifiable.
A monolithic carrier DTS file is a sign that subsystems are not yet
properly separated.

**Recommended fragment structure:**
```
board-pcie.dtsi      ← PCIe switches, iommu-map, msi-map
board-eth.dtsi       ← Ethernet PHYs and MACs
board-storage.dtsi   ← USB, NVMe, SD controllers
board-power.dtsi     ← regulators, power sequencing, GPIO
board-display.dtsi   ← display, HDMI (if applicable)
board-debug.dtsi     ← UART, JTAG, debug headers
```

Each fragment can be reviewed, modified, and submitted upstream
independently without touching the others.

---

## Domain E — Binary and Artefact Provenance

*Classification and handling of all non-source build inputs.*

### E1 — Classify Every Input by Provenance and Licence

**Rule:** Before handling any binary or data file, classify it along
two dimensions: redistribution rights and access scope. Select
infrastructure appropriate to the classification.

**Classification matrix:**

| Redistribution | Access | Infrastructure |
|---|---|---|
| Open source, permissive | Anyone | Public git repo or Yocto fetcher |
| Open source, copyleft | Anyone | Public git repo with licence notice |
| Vendor proprietary, authorised | Team | Internal artifact server |
| Vendor proprietary, NDA | Authorised individuals | Internal artifact server + access control |
| Customer confidential | Specific individuals | Customer-controlled storage |
| Developer-local | One person | Gitignored, documented template |

**Never store an input in a mechanism weaker than its classification requires.**

---

### E2 — No Binaries in Any Git Repository

**Rule:** Proprietary firmware blobs, compiled binaries, and generated
files never enter any git repository — private or public.

**Why private repos are not safe for binaries:**
- Git history is permanent — a blob committed and then deleted remains
  in the history and is recoverable
- Private repos can become public accidentally or through policy changes
- Redistribution terms apply regardless of repository visibility
- Binary content bloats repository size permanently

**The only acceptable exception:** Small, open-licensed, non-firmware
data files (e.g. test vectors, calibration tables with clear open licence).

---

### E3 — Artifact Server with Access Control for Vendor Binaries

**Rule:** Vendor-provided firmware binaries are distributed through an
internal artifact server with explicit vendor authorisation, access
control, and hash verification — not through personal paths or shared
network drives.

**Artifact server migration triggers:**
- Team size exceeds 5 members
- CI pipeline begins
- A new firmware package version is released
- Any team member cannot reproduce the build from a documented procedure

**Until an artifact server exists:** Each authorised team member downloads
independently under their own credentials. Document the download source,
version, and expected checksums. Never share downloaded binaries via
email, Slack, or shared drives.

---

### E4 — Licence Audit Before Integration

**Rule:** Before integrating any binary blob into a BSP image, verify
its redistribution terms. Treat blobs as NDA-restricted until proven
otherwise.

**Sources and their typical terms:**

| Source | Typical terms | Action |
|---|---|---|
| `linux-firmware.git` | Various open licences | Check individual file licence |
| SoC vendor SDK (tz, uefi, xbl) | Proprietary, redistribution restricted | Requires vendor authorisation |
| Open source project release | Project licence | Verify matches expectation |
| Unknown origin | Unknown | Treat as NDA-restricted until confirmed |

---

### E5 — Firmware Version Traceability

**Rule:** Every firmware binary in the BSP must have a documented source,
version, and hash. The build must fail if a required binary is missing.
The mapping from binary filename to vendor package version must be recorded
in the project context document.

**Pattern in recipe:**
```bitbake
do_compile() {
    for f in ${REQUIRED_BINARIES}; do
        if [ ! -f "${PREBUILTS}/${f}" ]; then
            bbfatal "Missing required binary: ${f}
                     Source: ${VENDOR_PACKAGE} version ${VENDOR_VERSION}
                     Download: ${DOWNLOAD_INSTRUCTIONS}"
        fi
    done
}
```

---

### E6 — Never Redirect S to a Live Binary Source Directory

**Rule:** When bypassing the Yocto fetch/unpack cycle to read from a
vendor prebuilts directory, never set `S = ${PREBUILT_DIR}` or any
variable that points to a live, persistent source directory. Upstream
`.inc` files assume `S` points to a mutable temporary copy created by
the unpack task. Redirecting `S` to a live directory exposes it to any
destructive operation (`find -delete`, `rm`, `mv`) in the inherited `.inc`.

**Origin:** Confirmed data loss in a BSP bring-up project session. Setting
`S = ${VENDOR_PREBUILT_DIR}` caused `firmware-qcom-boot-common.inc`'s
`do_deploy` to run `find ${S} -name 'gpt_*.bin' -delete` against the
live prebuilts directory, permanently deleting 13 files required for
board recovery. Files were only recoverable from the original vendor SDK
download.

**Anti-pattern:**
```bitbake
require recipes-bsp/firmware-boot/firmware-qcom-boot-common.inc
# WRONG — .inc's do_deploy will -delete from the live prebuilts dir
S = "${VENDOR_PREBUILT_DIR}"
```

**Correct pattern:**
```bitbake
require recipes-bsp/firmware-boot/firmware-qcom-boot-common.inc

# Safe — points to a Yocto-managed temp dir, never a live source
S = "${UNPACKDIR}"

# Override do_deploy completely — reads from prebuilts path explicitly.
# The .inc's do_deploy (which runs -delete on S) never executes.
# Document the bypass and its removal condition.
do_deploy() {
    install -d ${DEPLOYDIR}/${DEPLOY_SUBDIR}
    for f in ${REQUIRED_BINARIES}; do
        if [ ! -f "${VENDOR_PREBUILT_DIR}/${f}" ]; then
            bbfatal "Missing: ${f} — check VENDOR_PREBUILT_DIR"
        fi
        install -m 0644 "${VENDOR_PREBUILT_DIR}/${f}" \
            "${DEPLOYDIR}/${DEPLOY_SUBDIR}/"
    done
    # Remove this override when proper SRC_URI fetch is available.
}
```

**Verification — mandatory before adopting any `.inc` that will be
required by a recipe with a non-standard source path:**
```bash
# Audit for destructive operations in the .inc
grep -n "\-delete\|rm \|rm -\|mv " \
    meta-upstream/recipes-bsp/firmware-boot/firmware-*-common.inc

# For each hit: confirm the path operated on is always UNPACKDIR
# or WORKDIR — never a variable the .bb author controls
```

**Scope:** Applies to any recipe that uses `require` to adopt an `.inc`
while also bypassing the normal fetch/unpack cycle for any reason
(artifact server not yet available, local developer path, CI path bypass).

---

## Domain F — AI-Assisted Generation Constraints

*Specific constraints that apply when using an AI to generate Yocto
recipes, configuration, or build infrastructure.*

### F1 — Upstream Layer Content Must Be in Context Before Generation

**Rule:** Before generating any Yocto recipe or configuration file,
the relevant upstream layer files must be read and placed in the AI's
context window.

**Why training data is insufficient:** AI general Yocto knowledge covers
common patterns from the training corpus. It does not contain the specific
capability surface of your upstream layer at its current commit. Without
the actual files in context, the AI will generate plausible but potentially
redundant code — passing the functional test (does it build) while failing
the architectural test (does it reinvent existing infrastructure).

**Files to read before any recipe generation session:**
```bash
# Read the relevant .inc files for your domain
cat meta-upstream/recipes-bsp/firmware-boot/firmware-*-common.inc
cat meta-upstream/recipes-kernel/linux/linux-*.inc

# Read the relevant bbclasses
cat meta-upstream/classes-recipe/image_types_*.bbclass

# Survey what exists
ls meta-upstream/recipes-bsp/
ls meta-upstream/classes-recipe/
```

Place the output of these commands in the chat context before asking
the AI to generate anything.

---

### F2 — Generated Code Requires Two Separate Reviews

**Rule:** A generated recipe that builds and boots satisfies the
functional acceptance criterion but not the architectural criterion.
Two reviews are required and must not be conflated:

| Review type | Question | When |
|---|---|---|
| Functional | Does it build, boot, produce correct output? | CI / immediate testing |
| Architectural | Does it follow layer conventions? Does it reinvent existing infrastructure? Is it at the correct abstraction layer? | Human review before merge |

Functional review does not substitute for architectural review.
A recipe can pass all tests and still violate every principle in this
document.

---

### F3 — Apply Domain Checklist Before Generating Any File

**Rule:** Before generating any Yocto artefact, verify against each domain:

```
Domain A — Layer architecture:
□ Have I read the upstream layer's .inc, .bbclass, PACKAGECONFIG options?
□ Is this logic at the correct abstraction layer?
□ Is this bbappend expressing an override or reimplementing a feature?

Domain B — Configuration:
□ Is every configuration surface explicitly owned in a file I control?
□ Am I depending on an upstream CI file or implementation detail?
□ Is there a custom distro conf?

Domain C — Reproducibility:
□ Is every SRCREV pinned? No AUTOREV?
□ Does kas manage all layer checkout? No manual git clone steps?
□ Do all downloaded files have hash verification?

Domain D — Patches:
□ Does a forked kernel repo with topic branch structure exist?
  If no and kernel patches are needed — STOP. Do not generate
  any kernel bbappend patch content. Create the fork first.
□ Is SRC_URI pointing at the integration branch (not a patch list)?
  If no — fix before adding any patch.
□ Is every patch in its own named topic branch?
□ Is every patch classified (UPSTREAM / BOARD / WORKAROUND)?
□ Are DTS changes split into per-subsystem fragments?

Domain E — Binaries:
□ Is every binary classified by provenance and licence?
□ Are any binaries being committed to git? (must be zero)
□ Is firmware version and source documented?
□ If requiring an upstream .inc: grep it for -delete/rm/mv and
  confirm no destructive operation runs against a path the .bb
  author controls (E6, F6)
```

If any answer is "I don't know" — read the upstream layer before
proceeding.

---

### F4 — Plausible Is Not Correct

**Rule:** AI-generated Yocto code that follows common patterns may be
structurally wrong for your specific layer stack. Training data contains
large amounts of expedient code written under delivery pressure that does
not follow best practices. Every generated file must be evaluated against
this document, not just against whether it produces correct build output.

**The pattern-matching trap:** An AI that has seen many firmware recipes
that hand-roll copy operations will generate a firmware recipe that
hand-rolls copy operations. This is the most common pattern in the
training data — not the best practice. Explicitly constraining the AI
with the upstream layer content (F1) is the mechanism that breaks
this pattern.

---

### F5 — Assume Unknown Upstream State Triggers Verification

**Rule:** If the AI does not have the current content of an upstream
file in its context, it must not assume content based on training data.
The correct response is to request the file content rather than generate
code based on what the file probably contains.

**The correct AI behaviour:**
> "Before I generate this recipe, I need to see the content of
> `meta-upstream/recipes-bsp/firmware-boot/firmware-common.inc`
> to avoid reimplementing what it already provides. Please paste
> or share that file."

**The incorrect AI behaviour:**
> "I'll generate a firmware recipe using standard Yocto patterns..."
> [generates recipe without reading the upstream layer]

---

### F6 — Audit Destructive Operations Before Adopting Any .inc

**Rule:** Before adopting any `.inc` via `require`, grep it for
destructive shell operations (`-delete`, `rm`, `mv`, `chmod -R`).
For each destructive operation found, verify the path it operates on
is always a Yocto-managed temporary directory (`UNPACKDIR`, `WORKDIR`,
`DEPLOYDIR`). If the path derives from `S` or any variable the `.bb`
author controls, confirm that variable cannot be redirected to a live
source directory before proceeding.

**Why this matters:** Reading an `.inc` file to identify the variable
mismatch and task structure is necessary (F1) but not sufficient. A
destructive operation that appears safe under normal fetch/unpack flow
becomes a data loss event when `S` is redirected to a persistent
directory as part of a fetch bypass.

**The audit command — run against every `.inc` before adopting it:**
```bash
grep -n "\-delete\|rm \|rm -rf\|mv \|chmod -R" \
    path/to/upstream.inc
```

**For each hit, ask three questions:**
1. What variable defines the path this operation runs against?
2. Can the `.bb` author redirect that variable to a non-temporary path?
3. If yes — is `do_deploy` (or the containing task) being overridden
   completely in the `.bb`? If not, override it before proceeding.

**Anti-pattern — tagging a destructive operation as low-risk without
verifying path ownership:**
```
# WRONG analysis:
# "The .inc deletes gpt_*.bin from S before deploy — same result
#  as our explicit list, no regression."
# (Missed: S = ${VENDOR_PREBUILT_DIR} makes this delete live files)
```

**Correct analysis:**
```
# RIGHT analysis:
# The .inc runs: find "${S}" -name 'gpt_*.bin' -delete
# S is set by: S = "${UNPACKDIR}/${BOOTBINARIES}" in the .inc
# Our recipe sets: S = "${VENDOR_PREBUILT_DIR}" (bypass pattern)
# CONCLUSION: -delete would run against live prebuilts — STOP.
# Fix: override do_deploy completely, set S = "${UNPACKDIR}"
```

**Relationship to E6:** F6 is the AI analysis discipline that prevents
the failure mode documented in E6. E6 states what must never happen;
F6 states the verification step that prevents it.

---

## Document History

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial release — 6 domains, 25 principles. Derived from Qualcomm SoC BSP bring-up code review findings. |
| 1.1 | May 2026 | Added E6 (never redirect S to live binary directory) and F6 (audit destructive ops before adopting .inc). Derived from a BSP bring-up session data loss incident — `S = ${VENDOR_PREBUILT_DIR}` caused `.inc` find -delete to run against live prebuilts. Also added F6 as the preventive analysis discipline paired with E6. Also extended F3 Domain E checklist with destructive op audit item. |
| 1.2 | May 2026 | Strengthened D2 — changed trigger from "before patch count exceeds three" to "before the first patch"; added explicit pre-implementation gate (fork must exist before any bbappend patch content is generated; TODO-migrate pattern forbidden); added reviewer quote as rationale. Strengthened F3 Domain D checklist with hard STOP if kernel fork does not exist. Derived from a BSP bring-up project session reviewer comment. |

---

## Contributing

This document is maintained as a living reference. To contribute a new
principle:

1. Open an issue or PR on the repository
2. Follow the five-step self-extension process in the Overview section
3. Include: the specific symptom that motivated the principle, the
   generalised rule, at least one anti-pattern, and a verification method
4. Update KNOWLEDGE_REGISTRY.md domain activation keywords if new trigger signals are needed

---

*This work is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).*
*© 2026 Lei Zhou*

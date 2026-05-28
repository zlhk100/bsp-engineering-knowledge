<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Security Firmware Porting Architecture Principles

**Version:** 1.1
**Date:** May 2026
**Author:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/zlhk100/bsp-engineering-knowledge

---

## Unifying Meta-Principle

A security firmware port is an interface contract fulfilment exercise,
not a coding exercise. The implementer's job is transcription of a
complete specification. The architect's job is to make that specification
complete before any implementation starts.

The cost of a specification gap is not the cost of fixing it in a design
session (minutes). It is the cost of discovering and debugging the
consequence in an implementation session where the output compiles,
passes Phase 1, and fails at Phase 2 or Phase 3 (hours to days).

The web interface design session is not overhead. It is the work.
Claude Code is the transcription.

---

## Domain A — Certification Scope and Standards

### A1 — Certification standard is a prerequisite for all other decisions

**Rule:** Decide and document the target certification standard before
writing any platform port specification section. The standard determines
which partitions are required, which isolation level is mandatory, which
profile to use, and which requirements are out of scope.

**Rationale:** Starting implementation before resolving certification
scope creates inconsistency that is expensive to fix mid-project. A
wrong profile selection can disable partitions already implemented,
require additional isolation files not in the build, or include
requirements that are out of scope for the target level.

**Anti-pattern:**
Targeting profile_medium_arotless (which maps to JSADEN019 — ARoT-less
L2) while writing PPS sections that implement PS as a standard L2
partition (which maps to JSADEN012). These two certification paths have
incompatible partition scope. The inconsistency surfaces when the
evaluator reviews the security target.

**Correct pattern:**
Decide first: JSADEN019 (ARoT-less L2, no PS, SFN, IL1) or JSADEN012
(standard L2, PS as ARoT service, IPC, IL2). Document the decision with
rationale. All subsequent PPS sections derive from this decision.

**Verification:**
Before writing any PPS section, confirm: which JSADEN standard? Which
profile .conf? Does the profile's default partition list match the
implementation plan?

---

### A2 — PSA certification level is a normative property

**Rule:** Never accept an AI characterisation of what is "required" or
"recommended" for a PSA certification level without verifying against
the applicable protection profile document directly.

**Rationale:** AI systems conflate PSA-L1, L2, and L3 requirements
because they are adjacent concepts in training data. A stated PSA-L2
requirement may in fact be a PSA-L3 requirement (physical attack
mitigation, FIH, DPA resistance). Implementing L3 requirements in an
L2 port adds unnecessary complexity and can cause compilation errors
when the L3 feature is OFF by default in the L2 profile.

**Anti-pattern:**
AI states: "FIH (Fault Injection Hardening) is highly recommended for
PSA-L2 certification." The TF-M physical attack mitigation design
document states: "Mitigation against physical attacks is a mandatory
requirement for PSA Level 3 certification." profile_medium_arotless
sets TFM_FIH_PROFILE=OFF by default. Adding FIH macros when FIH profile
is OFF causes compilation errors (undefined symbols).

**Correct pattern:**
For any security requirement an AI asserts, find the normative source
(protection profile, certification standard, TF-M design doc) and
verify which level it applies to. Check the applicable protection
profile .rst document directly.

**Verification:**
grep -r "FIH\|fih\|physical_attack" docs/design_docs/
Check profile .conf for TFM_FIH_PROFILE default value.

---

### A3 — CRA compliance is necessary but not sufficient for principled
trust isolation architecture

**Rule:** When porting TF-M to a product targeting the EU market, treat
CRA compliance (EN 304 623 + EN 304 626) and PSA certification as
complementary obligations, not substitutes. Verify the boot-to-OS
handoff interface is explicitly designed — neither standard defines it.

**Rationale:** EN 304 626 TR-MISO specifies hardware-enforced memory
access control without defining principals, trust levels, or the
identity × resource → policy structure that makes isolation meaningful.
A product can pass TR-MISO assessment while having an unprincipled
NS-to-S boundary that leaks key material, if it has no user accounts
(the applicability condition). EN 304 623 defines boot manager
obligations up to handoff, but neither standard specifies a joint
requirement that the OS verifies it is running on a compliant boot
manager before relying on properties the boot manager is supposed to
have provided. PSA certification closes this gap at the RoT layer.

**Anti-pattern:**
Assuming that because a product complies with EN 304 623 and EN 304 626,
its trust isolation architecture is sound. Presenting CRA compliance
to a security-focused customer as equivalent to PSA certification.

**Correct pattern:**
CRA compliance addresses the market regulator audience (CE marking,
essential requirements). PSA certification addresses the security
architect audience (verifiable RoT, defined isolation levels, normative
security claims). Both are needed. The handoff interface between the
boot manager and the OS requires explicit design attention beyond what
either standard mandates.

**Verification:**
For any product targeting the EU market, confirm: (1) CRA compliance
pathway identified (EN 304 623 for boot manager, EN 304 626 for OS
layer), (2) PSA certification pathway identified (evaluation lab,
protection profile), (3) boot-to-OS handoff interface explicitly
designed with attestation evidence chain.

---

## Domain B — Framework Interface Contracts

### B1 — Function signatures must come from actual headers, not AI training data

**Rule:** Every function signature in the platform port specification
must be verified from the actual header file in the specific TF-M
version in use. Never accept an AI-described function signature without
source verification.

**Rationale:** AI training data reflects older TF-M versions. Interfaces
change between versions without backwards compatibility. A function that
AI describes may not exist in your version, may have a different
signature, or may have been replaced by a different pattern entirely.

**Anti-pattern:**
AI describes `tfm_plat_get_huk(uint8_t *key, uint32_t size)` as the
HUK interface for PLATFORM_DEFAULT_CRYPTO_KEYS=OFF. Source reading of
`tfm_plat_crypto_keys.h` in TF-M v2.3 reveals the interface is a
descriptor table pattern — the function AI described does not exist.
Claude Code implements a function that satisfies no declaration.
Result: linker error that appears to be a cmake problem.

**Correct pattern:**
grep -n "tfm_plat.*huk\|huk\|builtin_key" \
  platform/include/tfm_plat_crypto_keys.h
Read the actual header. Use the exact signature found there.

**Verification:**
Before writing any PPS section, confirm: which header declares this
interface? What is the exact signature in that header at the version
in use? Is the return type FIH-wrapped or a plain enum?

---

### B2 — Three mandatory greps before writing any port file specification

**Rule:** Before writing the specification for any TF-M platform port
file, run all three mandatory greps against the framework source.

**Rationale:** TF-M common files use compile-time enforcement (`#error`,
`__WEAK`) to express requirements that no design document explicitly
lists. Skipping these greps produces cryptic errors in files the
implementer did not write, with no obvious connection to the missing
definition in the platform file.

**Grep 1 — #error grep (mandatory define discovery)**
```bash
grep -rn "#error\|#ifndef" platform/ext/common/ | grep -v "^Binary"
```
Finds defines your platform header MUST provide. TF-M common files use
`#error` to enforce platform header contracts at compile time. Missing
these produces errors in files you did not write.

**Grep 2 — __WEAK grep (optional override discovery)**
```bash
grep -rn "__WEAK" platform/ext/common/ platform/include/
```
Finds framework functions your platform CAN or MUST override. An
un-overridden weak function silently links the default stub. Some stubs
return 0 (harmless). Others return error codes that cascade into boot
failures. Some stubs are certification gaps.

**Grep 3 — Caller grep (execution context discovery)**
```bash
grep -rn "tfm_hal_platform_init\|<function_name>" \
  secure_fw/ platform/ext/common/
```
Finds who calls your function and what happens immediately before/after.
The framework's behaviour immediately before and after calling a platform
function defines its execution context and ordering constraints.

**Anti-pattern:**
Writing PPS-9 (platform init sequence) based on architecture documents
alone. Gemini described `platform_init` running before
`set_up_static_boundaries`. Source reading of `main.c` revealed the
opposite. The platform security lock register (SSC on Cortex-M
platforms using this pattern) placed in the wrong function based on
the wrong call ordering would have caused HardFault on first boot.

**Correct pattern:**
Run all three greps for every port file before writing its PPS section.
Source reading of `main.c` lines 40-51 confirmed the actual boot
sequence before any security lock placement was specified.

**Verification:**
Before any PPS section: confirm all three greps have been run and their
output has been reviewed for the file being specified.

---

### B3 — Boot sequence must be verified from main.c source

**Rule:** Verify the boot sequence for any TF-M framework version from
actual `secure_fw/spm/core/main.c` source before specifying any
security initialisation ordering. Never infer from architecture
documents alone.

**Rationale:** The ordering of security initialisation calls (SAU before
MPU, hardware locks before partition init, etc.) determines correctness.
Architecture documents describe the intent. Source code reflects the
actual call order for your specific version. These can diverge.

**Anti-pattern:**
SSC lock placement (the post-configuration security lock register)
specified based on AI description of boot sequence. Source reading of
main.c revealed set_up_static_boundaries() is called BEFORE
platform_init(), not after. Lock placed in wrong function would have
caused HardFault on first boot — not visible until Phase 2.

**Correct pattern:**
```bash
grep -n "tfm_hal\|platform_init\|static_boundaries\|partition_init" \
  secure_fw/spm/core/main.c
```
Read the actual call sequence with line numbers. Specify lock placement
relative to confirmed call sites.

**Verification:**
PPS section for any security lock or initialisation ordering must cite
main.c with line numbers.

---

### B4 — __WEAK functions require explicit decision before implementation

**Rule:** For every `__WEAK` function discovered in framework common
files, make an explicit decision: use the default stub (document why
it is sufficient), or override with platform-specific implementation
(specify what the override must do). Never leave this implicit.

**Rationale:** Un-overridden weak functions silently link the default
stub. The default stub may return a value that appears correct in Phase 1
but represents a certification gap. The ITS encryption HAL
(`tfm_hal_its_encryption.c`) is an example — the default stub returns
`TFM_HAL_ERROR_NOT_SUPPORTED`, which is the correct Phase 1 behaviour
but must be replaced in Phase 3 for PSA-L2 storage security compliance.

**Anti-pattern:**
Designing a custom crypto insertion layer between ITS filesystem and the
CMSIS driver. Source reading revealed TF-M v2.3 has an official platform
encryption HAL (`tfm_hal_its_encryption.c`) — the supported insertion
point. A custom modification to TF-M internals requires additional
justification in the security target documentation.

**Correct pattern:**
Run the `__WEAK` grep. For each weak function: document the decision
(stub or override). If overriding, write the specification before
handing to Claude Code.

**Verification:**
PPS file manifest must include an entry for every weak function that
requires a platform-specific override, with explicit Phase annotation
(Phase 1 stub vs Phase N real implementation).

---

## Domain C — Platform Architecture Constraints

### C1 — Closed-source SBL leaves hardware in unknown state

**Rule:** When any closed-source or undocumented boot stage runs before
the ported firmware, treat the hardware state at firmware entry as
unknown. Explicitly establish all required hardware state in the startup
file. Never assume clean reset state.

**Rationale:** A vendor SBL may leave D-Cache enabled, SAU regions
configured for SBL use, MPU regions active, pending interrupts, and
stack limits set. Any of these can corrupt firmware behaviour in ways
that are intermittent and hard to diagnose. D-Cache coherency failure
on byte-alterable NVM (such as MRAM) is particularly dangerous —
writes succeed at the hardware level but reads return stale cache
lines, producing data corruption that appears only under specific
write patterns.

**Anti-pattern:**
Startup file specified without explicit hardware sanitisation, assuming
post-reset state. SBL-configured SAU regions remain active when TF-M
configures its own regions, causing unexpected security partitioning.
SBL-enabled D-Cache causes NVM read-after-write to return stale data
in ITS — corruption is intermittent, depends on cache line alignment.
On MRAM-based platforms this is especially likely because MRAM is
byte-alterable and does not require an erase cycle, making partial
write/read races cache-line dependent.

**Correct pattern:**
Mandatory sanitisation block in Reset_Handler before any security setup:
```c
ARM_MPU_Disable();
SCB_DisableDCache();
SCB_InvalidateDCache();
for (uint32_t i = 0; i < (MAX_IRQn + 31U) / 32U; i++) {
    NVIC->ICPR[i] = 0xFFFFFFFFU;  /* clear all pending IRQs */
}
TZ_SAU_Disable();
/* zero all SAU regions */
__set_MSPLIM(0);
__set_PSPLIM(0);
```

**Verification:**
startup_<target>_s.c must contain explicit ARM_MPU_Disable(),
SCB_DisableDCache(), NVIC pending clear, SAU reset, and stack limit
clear before any security initialisation begins.

---

### C2 — D-Cache coherency after NVM writes

**Rule:** Every NVM write function (Driver_Flash.c ProgramData()) must
call cache maintenance after the write completes when the platform has
a data cache.

**Rationale:** Cortex-M55 D-Cache introduces a read-after-write hazard
for byte-alterable NVM (MRAM, eFlash). A write via the vendor HAL
(e.g. am_hal_mram_main_program() or equivalent) populates the NVM
physically, but a subsequent read may return a stale cache line. For
TF-M ITS this causes data corruption — written records read back as
old content. The defect is intermittent and cache-line-alignment
dependent, making it one of the hardest classes of bug to reproduce
in Phase 1 static code testing.

**Anti-pattern:**
Driver_Flash.c ProgramData() calls the vendor NVM write API and
returns immediately. No cache maintenance follows. ITS records are
written and read back correctly most of the time. Under specific
access patterns a read returns the previous value. The defect does
not appear in Phase 1 static code testing.

**Correct pattern:**
```c
/* After every NVM write — required for Cortex-M55 and any
   platform with D-Cache over byte-alterable NVM: */
SCB_CleanInvalidateDCache_by_Addr((uint32_t *)dst, (int32_t)len);
```
This applies whether or not the SBL enabled the cache — correct
regardless because TF-M itself may enable D-Cache for performance.

**Verification:**
Driver_Flash.c ProgramData() must contain
SCB_CleanInvalidateDCache_by_Addr after every vendor NVM write call.
Verify present before Phase 2.

---

### C3 — Peripheral security attribution with interleaved address maps

**Rule:** When a platform's peripheral address space is interleaved
(secure and NS peripherals scattered throughout the same range with no
clean S/NS separation), do not rely on SAU alone for peripheral
security. Use SAU + NVIC interrupt targeting as complementary mechanisms,
and verify each peripheral's protection model from the datasheet.

**Rationale:** With limited SAU regions (typically 8), carving out each
secure peripheral individually from an interleaved peripheral map is
not feasible. The correct approach is to mark the peripheral space NS
at the SAU level (initiator side attribution) and use NVIC interrupt
targeting to make secure peripheral interrupts Secure (preventing NS
from handling them). Peripheral write protection then relies on
hardware security features of the peripheral itself.

**Anti-pattern:**
Assuming that marking a peripheral range as SAU Secure provides
complete protection for all peripherals in that range. On platforms
using ARM CryptoCell-312 (or similar dual-domain crypto IP), the
crypto accelerator is explicitly designed for both S and NS access
and must NOT be marked SAU Secure — doing so breaks NS crypto
operations. The root key (e.g. RKEK/KDR) is protected by CryptoCell
internal key selection logic, not by SAU address attribution.

**Correct pattern:**
For each peripheral: (1) check the datasheet for its security model
(dual-domain vs Secure-only vs NS-accessible), (2) assign SAU
attribution accordingly, (3) add NVIC targeting for its interrupts,
(4) verify any internal protection mechanisms (power gates, lock
registers) complement the SAU attribution.

**Verification:**
For each peripheral in the port:
- Datasheet section consulted for security model?
- SAU region assignment justified by datasheet?
- NVIC interrupt target configured for Secure peripherals?
- Internal protection mechanism documented?

---

## Domain D — Security Design Correctness

### D1 — ITS encryption insertion point is tfm_hal_its_encryption.c

**Rule:** For any TF-M port implementing ITS cryptographic enforcement,
use `tfm_hal_its_encryption.c` as the insertion point. Never design a
custom insertion layer between ITS filesystem and the CMSIS driver.

**Rationale:** TF-M v2.3 provides an official platform encryption HAL
with three functions: `tfm_hal_its_aead_generate_nonce`,
`tfm_hal_its_aead_encrypt`, `tfm_hal_its_aead_decrypt`. ITS calls these
via `its_crypto_interface.c`. This is what PSA-L2 evaluators expect.
A custom modification to TF-M internals passes Phase 1 but requires
additional justification in the security target documentation.

**Anti-pattern:**
Inserting custom crypto logic between ITS filesystem layer and the
CMSIS flash driver. This works functionally but is not the supported
interface. Evaluators reviewing the port will flag the deviation from
the standard HAL insertion point.

**Correct pattern:**
Phase 1: `tfm_hal_its_encryption.c` returns TFM_HAL_ERROR_NOT_SUPPORTED
(ITS_ENCRYPTION=OFF in config.cmake).
Phase 3: implement CM1 (psa_key_derivation from HUK) and CM2
(psa_aead_encrypt with AES-GCM-SIV) in the three HAL functions.
The deriv_label is file_id = client_id || uid, provided by ITS
filesystem in ctx->deriv_label. Do not add partition_id — it is
applied automatically by tfm_builtin_key_loader.

**Verification:**
grep -n "its_aead\|tfm_hal_its" \
  platform/include/tfm_hal_its_encryption.h \
  trusted-firmware-m/secure_fw/partitions/internal_trusted_storage/

---

### D2 — Framework automatic behaviours are invisible in source code

**Rule:** Read the governing TF-M design document for each component
before specifying any PPS section involving that component's security
behaviour. Framework automatic behaviours that affect key derivation,
partition isolation, and attestation are documented in design docs, not
source code.

**Rationale:** `tfm_builtin_key_loader` automatically derives
per-partition keys from the HUK using partition_id as KDF input.
This happens transparently before any platform encryption code runs.
A PPS that specifies CM1 to include partition_id in the ITS derivation
label would double-apply the partition binding. Source code reading
alone does not reveal this — only the design document explains it.

**Anti-pattern:**
Writing PPS-22 (CM1 key derivation specification) based on source code
reading alone. Specifying deriv_label = uid || partition_id || version.
At Phase 3 execution, partition_id binding is applied twice — once by
tfm_builtin_key_loader automatically, once by the platform CM1
derivation. Keys are non-standard, ITS records are unreadable.

**Correct pattern:**
Read `docs/design_docs/crypto/tfm_builtin_keys.rst` before specifying
any key derivation. The design document reveals that partition binding
is applied automatically. CM1 deriv_label = file_id = client_id || uid
(confirmed from tfm_its_service.rst). Do not add partition_id.

**Verification:**
For every PPS section involving crypto, attestation, or partition
isolation: which TF-M design document governs this component?
Has that document been read?

---

### D3 — Rollback protection requires hardware-backed counter storage

**Rule:** Anti-rollback NV counters must be stored in OTP or equivalent
hardware-backed, write-once storage. Software-only counters and
RAM-backed counters are not acceptable for PSA-L2.

**Rationale:** PSA-L2 firmware anti-rollback requires that an attacker
with NS code execution cannot reset the counter to enable downgrade
attacks. OTP storage that survives reset and requires hardware key
access to write satisfies this. RAM-backed counters are cleared on
reset. Software-incrementable counters in writable flash are reversible.

**Anti-pattern:**
Storing NV counters in writable flash with only software access control.
An attacker with flash write access can restore the counter to a
previous value and replay the old firmware image.

**Correct pattern:**
Use OTP-backed storage for NV counters. Verify the OTP write API
requires hardware privilege or key access that NS code cannot obtain.
Add pre-flight FORBID/lock checks before every write: if the OTP
controller is permanently locked, return TFM_PLAT_ERR_SYSTEM_ERR
rather than silently succeeding.

**Verification:**
For each NV counter write path:
- Is counter stored in OTP or equivalent write-once storage?
- Does the write API require hardware privilege NS code cannot obtain?
- Is OTP lock status checked before write attempts?

---

## Domain E — Phase-Gated Implementation

### E1 — Phase 1 is compile-clean with 100% stubs

**Rule:** Phase 1 goal is cmake configure + cmake build with zero errors
and zero warnings. Every function returns a safe hardcoded value.
No real hardware access in Phase 1.

**Rationale:** Phase 1 validates the build system, linker scripts, flash
layouts, and include paths — all of which must be correct before any
hardware-dependent code is meaningful. Mixing real hardware access into
Phase 1 produces boot failures that appear to be hardware problems but
are actually specification gaps in build configuration.

**Anti-pattern:**
Implementing real NVM write operations (e.g. MRAM via am_hal_mram_main_program()
or equivalent vendor HAL) in Driver_Flash.c during Phase 1 while the
linker script and flash layout are still being calibrated. A flash
layout error causes the real NVM write to corrupt firmware at the
wrong address, producing a boot failure that looks like hardware
malfunction.

**Correct pattern:**
Phase 1 stubs return: TFM_HAL_SUCCESS for init functions,
TFM_HAL_ERROR_NOT_SUPPORTED for unimplemented crypto,
hardcoded test data for HUK derivation.
Mark every stub: `/* PHASE 1 STUB — replace in Phase N per PPS-X */`

**Verification:**
cmake --build 2>&1 | grep -E "error:|warning:"
Zero errors required. Zero warnings required (treat as errors).
No real hardware API calls outside UART and clock in Phase 1.

---

### E2 — Category 1 files require cold review before placement

**Rule:** Every file written by the design AI (web chat) that requires
deep analysis of TF-M design documents, PSA specs, or hardware
documentation must be reviewed by an independent AI instance (cold
review) before being placed in the repository.

**Rationale:** Bugs that are invisible to the author survive into the
build. Cold review finds bugs that source verification does not: dead
code, format violations, logic errors visible only at the whole-file
level. In a TF-M porting project, cold review found a null terminator
included in a CBOR text string (sizeof instead of sizeof-1) in
attest_hal.c, and a dead function (nv_counter_increment) in otp.c that
would have caused a -Werror -Wunused-function build failure.

**Anti-pattern:**
Placing attest_hal.c directly from web chat session into the repository
without cold review. The null terminator error passes all unit tests
but fails at the relying party verifier — undetectable without
end-to-end token verification. Detected only after Phase 3 testing.

**Correct pattern:**
Send the file only to the cold reviewer. Do not explain what it does.
Do not describe the design intent. The cold reviewer reads the file
the same way a compiler reads it — as a standalone artifact.

Category 1 files: otp.c, crypto_keys.c, target_cfg.c, tfm_hal_platform.c,
sau_config.h, Driver_Flash.c, tfm_hal_its_encryption.c, attest_hal.c.

**Verification:**
Every Category 1 file has a cold review record before its first commit.

---

### E3 — Cold review findings are hypotheses requiring source verification

**Rule:** Treat every cold review finding (from Gemini or any second AI)
as a hypothesis. Verify each finding against actual source before
applying any fix.

**Rationale:** Observed false-positive rate in TF-M porting project cold
review sessions: approximately 60%. Three of five Gemini findings in one
session were fabricated from training data: a non-existent struct field,
a function not called by the relevant partition, and a wrong function
signature. If applied without verification, these three false findings
would have introduced new bugs that compile cleanly and fail at runtime.

**Anti-pattern:**
Applying all five Gemini cold review findings without source
verification. Three fabricated findings introduce: wrong struct
initialisation (corrupts key policy at runtime), unnecessary linker
symbol (dead code), type mismatch causing undefined behaviour at call
site (not a compiler error in C without -Wpedantic).

**Correct pattern:**
For each cold review finding:
```bash
# Verify struct field exists
grep -n "field_name" platform/include/tfm_plat_crypto_keys.h
# Verify function is called
grep -rn "function_name" secure_fw/partitions/
# Verify function signature
grep -n "function_name" platform/include/relevant_header.h
```
The grep takes 30 seconds. Apply the fix only after source confirms
the finding.

**Verification:**
Each cold review finding has a source verification grep result recorded
before the fix is applied. Rejections are documented.

---

## Domain F — AI Generation Constraints

**These gates are mandatory before generating any platform port file.**

### F1 — Read the five knowledge source tiers in order before writing any PPS

Before writing any PPS section, the following tiers must be read in
order. Do not start writing until the corresponding tier is complete
for the component being specified.

```
Tier 1 — Standards and certification scope (read FIRST)
  Target certification standard (JSADEN019, JSADEN012)
  PSA Functional API specifications (Storage, Crypto, Attestation)
  Profile specification document

Tier 2 — Framework design documents (read SECOND)
  For TF-M: tfm_builtin_keys.rst, tfm_its_service.rst,
  tfm_crypto_design.rst, secure_partition_manager.rst,
  tfm_isolation.rst, tfm_fih.rst, tfm_attestation_design.rst
  config_base.cmake, profile .conf files

Tier 3 — Source code verification (read THIRD)
  Framework main/init (boot sequence and call ordering)
  Common platform files (what they call, what they require)
  Reference platform implementations (patterns, barriers)
  Interface headers in your specific version

Tier 4 — Vendor SDK documentation (read FOURTH)
  Security User's Guide (OTP layout, key management, hardware RoT)
  HAL API documentation (init sequences, power domains, timing)
  Hardware datasheet (memory map, peripheral addresses, granularity)

Tier 5 — Write PPS (write LAST)
  All 5 behavioural dimensions per function
  Every contract source-verified before handing to implementer
```

**Never write a PPS section for a framework component without first
reading the design document for that component.**

---

### F2 — Apply the pre-implementation checklist before generating any file

Before generating any platform port file implementation, confirm all
items in METHODOLOGY_RULES.md §Pre-implementation checklist are satisfied.

Hard stops — do not generate if any of these are unknown:
- Exact function signature verified from actual header in version in use
- Boot sequence call ordering verified from main.c with line numbers
- All `#error`-guarded defines confirmed present in platform headers
- Execution context (thread vs ISR, blocking allowed or not) confirmed
- Temporal ordering constraints (what must happen before this function)
  confirmed from source, not inferred from architecture documents

---

### F3 — CRA and PSA are separate compliance obligations, handle distinctly

When a session involves both CRA compliance analysis and TF-M
implementation work, treat them as distinct contexts. Do not allow
CRA mitigation terminology (TR-MISO, MI-MMAC, etc.) to appear in
TF-M platform port specifications, and do not allow TF-M PSA
certification requirements to be used as CRA compliance justification
without explicit mapping.

When the session goal involves CRA compliance for a product that
includes a TF-M-based RoT:
- CRA compliance covers the OS/boot manager layer (EN 304 626, EN 304 623)
- PSA certification covers the RoT layer (JSADEN019, JSADEN012)
- The handoff interface between the two layers requires explicit design
  — neither standard fully specifies it

---

## Maintenance Protocol Reference

See KNOWLEDGE_REGISTRY.md — Maintenance Protocol.
All principle additions follow Protocol 3.

New principles discovered during TF-M porting projects are added here
via the five-step self-extension process. The generalisation test is the
gate: would this principle prevent the same mistake on a different TF-M
port to a different Cortex-M platform?

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial version. Derived from Cortex-M55 SoC TF-M PSA-L2 porting project. Domains A–F covering certification scope, interface contracts, platform architecture, security design correctness, phase-gated implementation, and AI generation constraints. Anti-patterns 1–12 from project experience. CRA/PSA boundary principles from EN 304 623/626 joint analysis. |
| 1.1 | May 2026 | Updated F2 cross-reference from METHODOLOGY.md to METHODOLOGY_RULES.md following methodology file split. |

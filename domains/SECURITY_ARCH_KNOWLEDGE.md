<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Embedded Security Architecture Knowledge

**Version:** 0.2
**Date:** May 2026
**Author:** Lei Zhou + Claude (Anthropic)
**Licence:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Repository:** https://github.com/zlhk100/bsp-engineering-knowledge

---

## Purpose

This document captures reusable embedded security architecture patterns
derived from project experience. It is distinct from process constraints
(which belong in domain architecture documents like FIRMWARE_PORT_ARCH.md)
and from project-specific design decisions (which belong in
PROJECT_CONTEXT.md).

**Scope:** Cross-project architectural patterns that are stable, not
accurately covered by LLM training data alone, and not retrievable
on demand from normative sources. Each pattern carries an explicit
confidence tier: VERIFIED (validated by hardware testing or evaluation
lab), INFERRED (logically sound, pending hardware validation), or
ASSUMED (derived from analysis, not yet tested).

**Not in scope:** Normative standard text (retrieve at query time via
web search), project-specific design decisions, AI generation
constraints (belong in FIRMWARE_PORT_ARCH.md Domain F), and general
embedded systems knowledge covered by LLM training.

---

## Pattern 1 — Non-securable NVM storage security

### Statement

Cryptographic enforcement can substitute for hardware write protection
when hardware NVM access control is unavailable, achieving PSA-L2
storage security on platforms where the NVM technology is not
securable by design.

### Architectural implication

When hardware NVM write protection is absent, the security model shifts:
the evaluator's question changes from "can the attacker write to
flash?" to "can the attacker do anything useful with what they read
or write?" Cryptographic enforcement answers this question through
confidentiality (authenticated encryption per record), integrity
(authentication tag covering both data and metadata), and anti-replay
(hardware-backed epoch counter anchored in OTP). The NVM being
readable or writable by an adversary is an accepted condition, not a
failure of the design.

This pattern has three necessary components — none is sufficient alone:

**Authenticated encryption per record** binds each record to its
identity (owner, uid, version). An attacker who copies a record to a
different slot, replays an old version, or substitutes one record for
another will fail authentication. AES-GCM or equivalent AEAD provides
this.

**Nonce-misuse resistance** is necessary (not merely sufficient) for
correctness under realistic IoT power-loss conditions. Standard AEAD
(AES-GCM) requires a unique nonce per encryption. Under power loss
during a write, a device may reboot and re-encrypt the same plaintext
with a different nonce but the same key — or worse, with a reused
nonce if the nonce counter was not atomically committed. AES-GCM-SIV
(RFC 8452) bounds the damage of nonce reuse to per-message
confidentiality loss rather than full key compromise. This makes
nonce-misuse resistance a correctness requirement for IoT deployments,
not a conservative choice.

**Hardware-backed anti-replay** anchors the epoch counter in OTP or
equivalent write-once storage. A software-only counter is clearable
by reset. An SRAM-backed counter survives only as long as power is
maintained — an adversary who controls power controls the counter.
OTP storage that is inaccessible to NS software without hardware
privilege provides the necessary ground truth.

### Confidence

INFERRED — the pattern is logically sound and grounded in PSA-L2
normative requirements. Validated architecturally on a Cortex-M33
development board with deliberate removal of hardware NVM protection.
Full promotion to VERIFIED requires Phase 4 adversarial testing:
power-loss cycling under active write workload, replay attack
attempts against the anti-rollback mechanism, and substitution
attacks across record slots.

### Relationship to PSA-L2

PSA-L2 does not require hardware NVM write protection. It requires
that ITS confidentiality and integrity hold against the defined
attacker model (software attacker with NS code execution and physical
read access to NVM). Cryptographic enforcement satisfies this
requirement when correctly implemented. Platforms where MRAM, eFlash,
or other byte-alterable NVM is not securable by design are not
disqualified from PSA-L2 certification — they require stronger
cryptographic enforcement to compensate.

---

## Pattern 2 — Trust boundary definition precedes mechanism selection

### Statement

Security isolation mechanisms (SAU, MPU, MPC, PPC, TGU, SMMU, etc.)
must be selected and configured after the trust model is defined, not
before. Isolation without a trust model produces mechanism compliance
without security property guarantees.

### Architectural implication

A trust model has three components:

**Principals** — every entity that initiates access to a resource:
software processes, hardware DMA masters, execution domains (Secure,
Non-Secure), privilege levels (privileged, unprivileged).

**Resources** — every asset that requires protection: key material,
NVM regions, peripheral registers, SRAM partitions, interrupt lines.

**Policy** — for each (principal, resource) pair: which access types
are permitted (read, write, execute, configure) and under what
conditions.

The enforcement mechanism is selected last, as the implementation of
the policy for the available hardware. This ordering matters because
the same mechanism (for example, SAU) can express many different
policies, and without a prior policy definition there is no basis for
evaluating whether a given mechanism configuration is correct or
merely plausible.

The failure mode of mechanism-first design is security that passes
assessment but fails under adversarial conditions that were not
anticipated in the mechanism configuration. A classic example: marking
a peripheral address range as SAU Secure provides initiator-side
access control, but if the peripheral has a hardware mode that allows
dual-domain access by design (as some crypto accelerators do), the
SAU attribution is architecturally wrong regardless of whether it
passes a functional test.

This pattern directly addresses the structural gap in EN 304 626
TR-MISO (CRA harmonised standard for OS memory isolation), which
specifies hardware-enforced memory access control mechanisms without
defining the trust model those mechanisms are supposed to enforce.
A product can pass TR-MISO assessment while having an unprincipled
isolation boundary that leaks sensitive assets, if the applicability
conditions are met on a technicality. The pattern above is the
architectural lens that TR-MISO does not provide.

### Confidence

VERIFIED — normatively grounded in PSA architecture (which explicitly
defines isolation levels in terms of trust relationships between
partitions, not in terms of hardware mechanisms). Corroborated by
analysis of EN 304 626 TR-MISO structural limitations.

---

## Pattern 3 — CRA compliance and PSA certification are complementary,
not substitutes

### Statement

EU Cyber Resilience Act compliance (via EN 304 623 for boot managers
and EN 304 626 for operating systems) and PSA certification address
different evaluation audiences and leave a gap at their interface.
Neither substitutes for the other. Both are needed for products
targeting the EU market where principled trust architecture is required.

### Architectural implication

**CRA compliance** (EN 304 623 + EN 304 626) establishes the
regulatory floor: CE marking, essential requirements, vulnerability
handling obligations, secure update mechanisms, and OS/boot manager
security properties. The evaluation audience is the market regulator.
CRA compliance is legally mandatory for covered products sold in the
EU from September 2026.

**PSA certification** establishes principled trust architecture at the
RoT layer: a verifiable Root of Trust with defined isolation levels,
normative security claims against a specified attacker model, and
evaluator-verified implementation. The evaluation audience is the
security architect and customer. PSA certification is not legally
mandated but provides the trust evidence that CRA compliance alone
does not.

**The gap at the handoff interface:** EN 304 623 specifies what a
boot manager must do before handing off to the OS. EN 304 626
specifies what properties the OS must have. Neither standard specifies
a joint requirement that the OS verifies it is executing on a
compliant boot manager before relying on properties the boot manager
is supposed to have established. This gap means a product can comply
with both standards individually while having an integration that
undermines the security properties both standards are designed to
protect. Explicit design of the boot-to-OS handoff interface —
including attestation evidence chain and measurement verification —
is required to close this gap.

For products incorporating a TF-M-based Root of Trust: PSA
certification of the RoT layer plus CRA compliance for the boot
manager and OS layers together produce a stronger and more defensible
claim than either alone. The PSA attestation token provides the
evidence chain that bridges the EN 304 623 / EN 304 626 handoff gap.

**Structural explanation of the gap (sharpened by PQC migration
analysis):** The handoff gap is not a drafting oversight — it is a
structural consequence of decomposing standards by layer while
security-relevant bindings run orthogonally across layers. The online
session protocol SPDM demonstrates that binding capability-discovery
to state-attestation is a solved problem when a live two-party
transcript with a fresh nonce is available. The boot-chain handoff
is offline and single-party by construction; the transcript-binding
mechanism is therefore structurally unavailable. The gap requires an
alternative binding — such as measuring the boot manager's capability
surface (e.g., ECIT) into a TPM PCR alongside the code measurement
— that neither EN 304 623, EN 304 626, nor the UEFI PQC whitepaper
currently mandates. The UEFI whitepaper explicitly defers ECIT
measurement to TCG ("Defining measurement requirements (deferred to
TCG)"), and no TCG measurement profile has yet claimed that ownership.
This is an open, sourced, unowned gap as of May 2026.

### Confidence

VERIFIED — derived from joint normative analysis of EN 304 623 v0.1.1
(May 2026 draft) and EN 304 626 v0.2.0 (April 2026 draft) against
PSA-L2 protection profile requirements. The handoff gap is a
structural property of the two standards as drafted, not an
interpretation. The structural explanation (precondition-absence) is
INFERRED from SPDM protocol design and corroborated by the UEFI PQC
whitepaper deferral statement (direct quote, VERIFIED source).

---

## Pattern 4 — Crypto-capability becomes discoverable under migration

### Statement

A security property that is uniform across a population is an
invariant and can be assumed. The moment it becomes heterogeneous
across that population, it must become explicitly discoverable — and
the discovery mechanism must not depend on the property itself.

Cryptographic algorithm support is the canonical instance: in a
mono-algorithmic world it is an invariant and disappears from
protocols as an unstated global assumption. During a crypto migration,
it becomes a variable and must be discovered before any security
decision that depends on it.

### Architectural implication

**The bootstrap circularity:** You cannot, in general, use signed
attestation to attest verification capability, because verifying that
attestation requires the capability being negotiated. Capability
discovery is therefore logically prior to state attestation and must
be established by a mechanism that does not presuppose the property
being discovered.

This explains a design choice that appears to be a weakness but is
architecturally correct: the UEFI EFI Crypto Indicator Table (ECIT)
is published as an unsigned plaintext configuration/ACPI table rather
than a signed token. Signing it would require the consumer to already
possess the verification capability being advertised — the circularity
it was designed to avoid. The unsigned advertisement is the correct
resolution, not a security deficiency.

**The migration-period downgrade surface:** Dual-signing during a
crypto transition deliberately ensures a weak option is present for
backward compatibility. The transition window is therefore precisely
when the downgrade surface is maximal — every platform carries a
quantum-vulnerable signature alongside its quantum-safe one by design.
An attacker who can influence capability discovery (e.g., by tampering
with an unsigned capability table) can steer a relying party toward
the weakest algorithm both parties share. Defence requires a policy
floor applied independently of the capability advertisement: the
relying party must refuse algorithms below its security minimum even
if the platform offers them. Capability discovery tells you what is
possible; policy tells you what is acceptable; and these must be
evaluated separately.

**Cross-domain applicability:** The same invariant-to-variable
transition occurs in any security property that becomes heterogeneous:
key sizes, hash algorithm support, isolation level, attestation format.
Any system undergoing migration must ask: which properties have
transitioned from invariant to variable, and is each one now
discoverable without the circularity?

### Confidence

INFERRED — derived from first-principles analysis of the UEFI PQC
migration (ECIT design rationale) and corroborated by the observed
unsigned-table design choice in the UEFI PQC whitepaper (April 2026).
The invariant-to-variable law and bootstrap circularity are general
reasoning results, not empirically tested. The downgrade-surface
maximum is an INFERRED consequence of dual-signing being mandatory
during transition (CNSA 2.0 requirement, VERIFIED source).

### Relationship to other patterns

Pattern 4 is the generalisation of the specific crypto-migration
context. Pattern 3's handoff gap is an instance: the OS's assumption
that the boot manager supports the expected capability set is an
invariant that the PQC migration converts into a variable — creating
the need for capability discovery at the handoff layer.

---

## Pattern 5 — Trust decisions require Capability, State, and Policy
to be separately established and bound

### Statement

A trust decision — whether to rely on a platform, component, or
communication partner — requires three independently-established
properties, and the seams between them are where security failures
occur during migration:

**Capability** — what cryptographic algorithms and security properties
the platform can process. Must be discovered before relying on any
evidence produced in a specific algorithm.

**State** — what the platform did and what it currently is. Established
by signed attestation evidence (PSA token, TPM quote, SPDM
measurement). Presupposes that the verifier can process the signature
algorithm used.

**Policy** — which algorithms and properties are acceptable to the
relying party. Must be applied independently of capability discovery;
the intersection of capabilities is not automatically acceptable.

### Architectural implication

In steady state (mono-algorithmic, uniform fleet), all three properties
collapse into an assumed invariant. The protocol sees no capability
negotiation, no policy floor enforcement — they are invisible because
they are constants.

Migration forces them apart. Each becomes a distinct, load-bearing
concern. Security failures occur at the unbound seams:

- Capability discovered but not bound to state: an attacker who
  tampers with the capability advertisement (e.g., unsigned ECIT)
  steers algorithm selection before the state evidence is produced,
  making the state evidence unverifiable by the relying party.

- State established but policy not enforced: the relying party accepts
  a valid attestation produced in an algorithm below its security
  floor because nothing in the protocol mandates a policy check
  separate from the capability intersection.

- Capability and state bound but the binding not covering the full
  surface: SPDM's transcript hash covers the negotiated algorithms
  but (per published formal analysis) does not prescribe a selection
  policy — the responder offers, the requester selects, and no
  normative requirement forces the strongest available choice.

**The cross-layer ownership gap:** Standards decomposed by layer
(boot manager / OS / RoT) do not naturally own the cross-layer
bindings between Capability, State, and Policy. Each standard governs
its layer; the bindings run orthogonally and fall between standards.
This is the structural root cause of the EN 304 623 / EN 304 626
handoff gap (Pattern 3) viewed through the migration lens.

**Verification question for any migration-era security design:**
For each trust decision the system makes: which of the three axes
(Capability, State, Policy) is established by what mechanism, and
is each binding explicitly designed and owned, or implicitly assumed?

### Confidence

INFERRED — derived from joint analysis of SPDM protocol design,
the UEFI PQC whitepaper, and EN 304 623/626 normative structure.
The three-axis decomposition is a reasoning result, not an empirically
tested theorem. The SPDM policy-gap observation is corroborated by
published formal analysis of SPDM 1.2 (Dax et al., 2022), which
states: "intuitively the strongest available cryptographic algorithms
should be chosen, but the standard does not prescribe a selection
mechanism" — VERIFIED secondary source.

The claim that no standard currently owns the full Capability × State
× Policy binding in the boot-chain handoff context is INFERRED from
the observed state of EN 304 623/626 drafts and the UEFI whitepaper
TCG deferral, both as of May 2026. Subject to revision if TCG
publishes ECIT measurement requirements or CRA harmonised standards
are revised.

### Relationship to other patterns

Pattern 5 is the most general statement. Patterns 3 and 4 are
instances. Pattern 2's trust-model-first principle is the Pattern-2-
layer instantiation of the same insight: define what must be true
(policy) before selecting the mechanism to enforce it.

The Capability × State × Policy structure mirrors the Identity ×
Resource × Policy structure of the unified isolation framework
(Pattern 2) — same architectural shape, different security domain.
This resonance is useful for consulting: both can be explained from
one meta-principle ("define the relation before selecting the
mechanism") and then specialised to the domain at hand.

---

## Maintenance Protocol Reference

See KNOWLEDGE_REGISTRY.md — Maintenance Protocol.

New patterns are added via Protocol 3 (five-step self-extension
process). The generalisation test is the gate: the pattern must be
applicable across projects and platforms, not specific to one
product design.

Confidence tiers are updated as evidence accumulates:
- Phase 4 adversarial testing results promote Pattern 1 to VERIFIED
- Evaluation lab findings may refine any pattern
- Published standard revisions (EN 304 623/626 v1.x) may require
  Pattern 3 to be revisited
- TCG ECIT measurement profile (if published) may promote the
  INFERRED elements of Patterns 3, 4, 5 to VERIFIED or require
  revision of the gap claim

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 0.1 | May 2026 | Initial stub. Three patterns: non-securable NVM storage security (INFERRED), trust boundary definition precedes mechanism selection (VERIFIED), CRA/PSA complementarity (VERIFIED). Derived from Cortex-M55 SoC TF-M PSA-L2 porting project and EN 304 623/626 joint analysis. |
| 0.2 | May 2026 | Added Pattern 4 (crypto-capability becomes discoverable under migration), Pattern 5 (Capability × State × Policy decomposition). Extended Pattern 3 with structural explanation of the handoff gap (transcript-binding precondition-absence, ECIT measurement deferral sourced from UEFI PQC whitepaper April 2026). Source: UEFI PQC migration analysis session, including SPDM protocol comparison and web-verified CNSA 2.0 firmware-signing constraints. |

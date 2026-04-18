# Decision Log — Remote CSR Proxy

This file records the current architectural decisions reached so far. These are the working baseline for future refinement.

---

## D-001 — Architecture direction
**Decision:** Use a **Remote CSR Proxy** architecture rather than direct request/response remote register access as the primary control model.

**Why:**
- reduces MSS-side blocking/response-wait burden
- preserves a more local-MMIO-like software model
- generalizes beyond ADRV9004/Navassa
- supports reuse for future remote register-based IP blocks

**Implication:**
The PF Fabric is not a trivial transport bridge. It becomes a policy-driven proxy with shadow/state management behavior.

---

## D-002 — Scope is CSR virtualization, not full remote AXI emulation
**Decision:** The design target is **CSR/register-space virtualization**, not transparent universal remote AXI emulation.

**Why:**
- full AXI semantic preservation across a remote custom link is significantly broader and riskier
- the current use case is control/status space, not arbitrary data transactions
- CSR-focused behavior is sufficient for current control-plane needs

**Implication:**
The design should be described as a **Remote CSR Proxy**, not a generic "Remote AXI Controller" unless the architecture is later expanded and justified.

---

## D-003 — MSS reads come from local shadow/proxy state
**Decision:** MSS-side reads should be satisfied from the PF Fabric local CSR shadow/proxy bank.

**Why:**
- preserves fast local AXI reads
- avoids request/response read stalls in the common case
- increases software portability from local-MMIO-oriented drivers

**Implication:**
Read values may be stale if synchronization is delayed or broken.
This requires explicit stale/error/reporting policy.

---

## D-004 — Kintex-to-PF updates carry address + new value
**Decision:** When Kintex-originated register changes are reported back, the event payload must contain at least:
- register address
- new value

**Why:**
- allows PF Fabric to update the local shadow directly
- avoids an additional remote read-back cycle
- simplifies interrupt-driven synchronization

**Implication:**
The remote control protocol must support update events with value payloads.

---

## D-005 — Register ownership is per-register, not per-bit-field
**Decision:** Ownership is currently defined per register.

**Why:**
- avoids merge/partial-coherency complexity
- keeps synchronization and policy rules simpler
- reduces ambiguity in write authority

**Implication:**
If future use cases require split ownership inside a register, this must be treated as a substantial architecture change.

---

## D-006 — Resync only during boot / recovery / idle
**Decision:** Full/block resynchronization is allowed only during:
- initial system bring-up
- error recovery
- explicitly idle conditions

**Why:**
- avoids coherency conflicts during active operation
- simplifies synchronization guarantees
- makes recovery behavior easier to reason about

**Implication:**
Operational flows must not depend on live full-resync while traffic/actions are in progress.

---

## D-007 — Command/pulse-like registers are modeled as WO doorbells
**Decision:** Command registers that produce pulse-like remote effects should be treated as **write-only doorbell-like registers**.

**Why:**
- avoids forcing normal RW semantics onto inherently command-oriented behavior
- simplifies the shadow model
- aligns with the stated intent that these values are not meaningfully read back

**Implication:**
These registers require an explicit descriptor/semantic type.

---

## D-008 — Descriptor-driven behavior is required
**Decision:** PF Fabric behavior must be parameterized/configured by a descriptor table rather than hard-coded per-register logic.

**Why:**
- keeps the architecture generic
- allows MSS/software-driven configuration
- avoids baking remote-IP-specific knowledge directly into PF Fabric RTL policy logic

**Implication:**
A descriptor schema is now a first-class architectural artifact and must be specified.

---

## D-009 — Error state must be visible to software
**Decision:** Link failure / synchronization failure must not be invisible. Software must be able to observe error state through interrupt and/or status indication.

**Why:**
- shadow-read model can otherwise hide stale or invalid state
- recovery flows depend on explicit fault visibility

**Implication:**
Status bits such as `VALID`, `STALE`, `REMOTE_ERR`, `RESYNC_REQUIRED` are strong candidates, though exact encoding is still open.

---

## D-010 — PF Fabric is a policy-driven proxy, not a dumb bridge
**Decision:** PF Fabric should be treated as a **policy-driven proxy endpoint** rather than a simple pass-through interconnect.

**Why:**
- it maintains shadow state
- it applies synchronization policy
- it enforces register semantics through descriptor metadata

**Implication:**
Documentation should avoid calling PF Fabric a passive or purely transport-only bridge.


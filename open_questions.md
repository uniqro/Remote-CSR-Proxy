# Open Questions — Remote CSR Proxy

This file tracks unresolved topics that still need analysis before implementation details are frozen.

---

## OQ-001 — Write acknowledgment policy by ownership type
For MSS-visible AXI writes, when should the write be considered complete?

### Current direction
- MSS-owned register: local shadow update may be enough for immediate completion
- remote/Kintex-owned register: stricter completion semantics may be needed

### Still open
- exact `BRESP` timing policy
- whether a pending/commit distinction is exposed to software
- whether descriptor policy should select strict vs optimistic commit mode

---

## OQ-002 — Shadow validity / staleness exposure
Reads come from shadow state, but what exact visibility model should software get?

### Candidates
- per-register status bits
- per-block status bits
- sticky error flags with interrupt
- validity epoch/version counters

### Still open
- granularity of stale/error reporting
- whether stale reads are always allowed or can be blocked/faulted for selected regions

---

## OQ-003 — Descriptor table schema
The architecture now depends on a descriptor table, but its schema is not yet frozen.

### Likely fields
- register address / group range
- owner
- access type
- semantic type (`normal`, `status`, `doorbell_wo`, etc.)
- synchronization policy
- event enable
- resync group id
- error/reporting behavior

### Still open
- minimum required field set
- whether descriptor table lives purely in PF Fabric or is software-programmed at boot
- update model for descriptor contents

---

## OQ-004 — Register semantic taxonomy
The design needs a formal class system for registers.

### Already discussed
- RO
- WO
- RW
- doorbell/pulse-like command registers

### Still open
- whether to explicitly support/forbid additional side-effect classes
- how unsupported classes are declared and rejected

---

## OQ-005 — Kintex-to-PF event protocol
Kintex-originated updates carry address + new value, but event framing is still unspecified.

### Still open
- sequence / versioning field
- optional timestamp or cause code
- event batching vs single-event messages
- retry and deduplication behavior

---

## OQ-006 — Recovery model after link failure
The high-level rule is clear: resync during recovery/idle only.

### Still open
- who initiates recovery
- how software learns which ranges require resync
- whether recovery is global, per-block, or per-register-group
- how interrupts are masked/gated during resync

---

## OQ-007 — Existing driver compatibility boundaries
The architecture aims for maximum MMIO illusion, but exact compatibility limits are still unclear.

### Still open
- what classes of existing Zynq/PL drivers can be reused as-is
- what assumptions in existing drivers break under shadowed proxy semantics
- whether a thin portability layer is still advisable

---

## OQ-008 — Concurrency and ordering model
The high-level design reduces MSS-side race pressure, but system-level ordering guarantees are still underdefined.

### Still open
- ordering between local writes and remote update notifications
- interrupt timing relative to shadow commit
- whether barriers are required in software for selected operations

---

## OQ-009 — Interface contract between PF Fabric and Kintex
The protocol is conceptually defined, but the wire-level and transaction-level contract is not yet documented.

### Still open
- packet format
- ack/error semantics
- initialization handshake
- version/capability negotiation

---

## OQ-010 — State diagrams and lifecycle specification
The architecture needs explicit state machines.

### Required diagrams
- MSS-owned write lifecycle
- Kintex-owned update lifecycle
- doorbell WO lifecycle
- resync entry/exit lifecycle
- link error/recovery lifecycle

---

## OQ-011 - Implicit assumptions requiring confirmation
The interface-contract refinement exposes several assumptions that are useful
for implementation planning but are not yet decision-log entries.

### Conservative assumptions
- Descriptor contents are programmed by MSS during boot/recovery and are not
  modified for an active register group without quiescing that group.
- Descriptor lookup and access validation complete before a Shadow CSR aperture
  access is accepted as architecturally meaningful.
- Shadow entries start invalid or explicitly reset-valued until boot-time
  descriptor setup and initial resync/event initialization complete.
- PF Fabric applies a Kintex-originated address/value update to the Shadow CSR
  Bank before raising the corresponding MSS interrupt.
- Group-level validity/stale/error visibility is the minimum required status
  granularity; per-register status is an implementation enhancement.
- AXI write completion and remote-side commit are distinct concepts unless a
  strict descriptor-selected completion policy explicitly ties them together.
- For Kintex-owned registers, a confirmed Kintex update is the authoritative
  shadow commit source. Any local pending overlay must be reported as pending,
  not silently presented as committed state.
- Doorbell WO shadow content is debug/diagnostic state only and must not be
  treated as architectural readback state.
- Resync requires quiescing or blocking conflicting traffic for the affected
  resync group.
- Registers with clear-on-read, FIFO-pop, or other read-side effects are
  unsupported unless a future decision defines a special semantic class.

### Still open
- Which of these assumptions should become binding decisions?
- Which assumptions need descriptor-selectable alternatives for driver
  compatibility?
- Whether the canonical documentation should be moved under
  `docs/remote-csr-proxy/` to match `AGENTS.md`, since the current workspace
  has the canonical files at the repository root.

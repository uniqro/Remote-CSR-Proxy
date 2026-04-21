# Open Questions - Remote CSR Proxy

This file tracks unresolved topics that still need analysis before implementation details are frozen.

---

## OQ-001 - Write acknowledgment policy by ownership type
For MSS-visible AXI writes, when should the write be considered complete?

### Current direction
- MSS-owned register: local shadow update may be enough for immediate completion
- remote/Kintex-owned register: stricter completion semantics may be needed

### Still open
- exact `BRESP` timing policy
- whether a pending/commit distinction is exposed to software
- whether descriptor policy should select strict vs optimistic commit mode

---

## OQ-002 - Shadow validity / staleness exposure
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

## OQ-003 - Descriptor table schema
The architecture depends on a descriptor table, but the v1 proposal now favors a resource-optimized bitmap schema instead of a rich per-register descriptor.

### Current v1 proposal
- 256 fixed 32-bit CSR registers
- dense index-based address map: `csr_addr = csr_base + index * 4`
- `read_source_bitmap[255:0]`
  - `0 = LOCAL_SHADOW`
  - `1 = DIRECT_REMOTE`
- `notify_on_change_bitmap[255:0]`
  - `1 = raise notification status/interrupt when a CSR_NOTIFY updates that index`
- descriptor bitmap updates only during boot, recovery, or explicit idle/quiesced operation

### Deferred from v1
- owner field
- RO / WO / RW access field
- semantic type field
- write commit policy field
- per-register resync group
- per-register error policy
- per-register status

### Still open
- whether the v1 bitmap schema is sufficient for the first RTL implementation
- exact AXI-visible programming layout for the two 256-bit bitmaps
- whether a later revision needs rich descriptors for mixed-width, access-checked, or per-register recovery behavior

---

## OQ-004 - Register semantic taxonomy
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

## OQ-005 - Kintex-to-PF event protocol
Kintex-originated updates carry address + new value, but event framing is still unspecified.

### Still open
- sequence / versioning field
- optional timestamp or cause code
- event batching vs single-event messages
- retry and deduplication behavior

---

## OQ-006 - Recovery model after link failure
The high-level rule is clear: resync during recovery/idle only.

### Still open
- who initiates recovery
- how software learns which ranges require resync
- whether recovery is global, per-block, or per-register-group
- how interrupts are masked/gated during resync

---

## OQ-007 - Existing driver compatibility boundaries
The architecture aims for maximum MMIO illusion, but exact compatibility limits are still unclear.

### Still open
- what classes of existing Zynq/PL drivers can be reused as-is
- what assumptions in existing drivers break under shadowed proxy semantics
- whether a thin portability layer is still advisable

---

## OQ-008 - Concurrency and ordering model
The high-level design reduces MSS-side race pressure, but system-level ordering guarantees are still underdefined.

### Still open
- ordering between local writes and remote update notifications
- interrupt timing relative to shadow commit
- whether barriers are required in software for selected operations

---

## OQ-009 - Interface contract between PF Fabric and Kintex
The protocol is conceptually defined, but the wire-level and transaction-level contract is not yet documented.

### Still open
- packet format
- ack/error semantics
- initialization handshake
- version/capability negotiation

---

## OQ-010 - State diagrams and lifecycle specification
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
- Descriptor bitmap lookup completes before a Shadow CSR aperture read selects LOCAL_SHADOW or DIRECT_REMOTE behavior.
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
- Registers with clear-on-read, FIFO-pop, or other read-side effects require software-selected DIRECT_REMOTE reads in the v1 proposal; hardware does not enforce this as a semantic class.

### Still open
- Which of these assumptions should become binding decisions?
- Which assumptions need descriptor-selectable alternatives for driver
  compatibility?
- Whether the canonical documentation should be moved under
  `docs/remote-csr-proxy/` to match `AGENTS.md`, since the current workspace
  has the canonical files at the repository root.

---

## OQ-012 - Resource-optimized v1 selectable direct reads
The v1 proposal adds a selected exception to the current shadow-read baseline: MSS can configure individual registers to use direct remote reads over commLVDS.

### Current v1 proposal
- normal reads default to `LOCAL_SHADOW`
- side-effect-sensitive or freshness-critical reads may be configured as `DIRECT_REMOTE`
- only one direct remote read may be outstanding
- direct remote read blocks AXI `RVALID` until `CSR_READ_RESP`, timeout, or link/protocol error
- direct remote read timeout/error returns AXI error and sets sticky status
- writes are not direct remote operations in v1
- all writes update shadow and queue `CSR_WRITE(addr, value32)` to Kintex
- AXI write response returns after local shadow update and commLVDS write queue acceptance
- Kintex notifications use `CSR_NOTIFY(addr, value32)` and update shadow through an 8-entry FIFO

### Resource optimization intent
- RAM-based 256 x 32-bit Shadow CSR Bank
- two 256-bit descriptor bitmaps instead of rich descriptors
- one shared commLVDS TX request path
- one direct-read FSM and one timeout counter
- no per-register FSMs
- no direct remote write engine

### Still open
- whether this selectable direct-read proposal should become a new decision-log entry that revises D-003
- exact direct-read timeout value
- exact AXI error response encoding for direct-read failures
- whether `CSR_READ_RESP` should carry address only or an explicit transaction id even though only one read is outstanding
- write queue depth and backpressure behavior details
- notification FIFO overflow recovery details
- whether software-only side-effect classification is sufficient, or whether a later descriptor bit should warn/fault on unsafe local reads

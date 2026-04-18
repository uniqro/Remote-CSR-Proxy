# AGENTS.md

## Project purpose
This project defines a generic **Remote CSR Proxy** architecture that allows a PolarFire SoC MSS to control register-based IP blocks located in a remote Kintex UltraScale+ FPGA while preserving a near-local MMIO programming model for software.

The current target use case is ADRV9004-related control, but the architecture is intentionally **generic** and must not be specialized to Navassa/ADRV-only behavior unless explicitly documented.

## Canonical context files
Treat the following files as the canonical source of truth, in this order:

1. `docs/remote-csr-proxy/decision_log.md`
2. `docs/remote-csr-proxy/architecture_overview.md`
3. `docs/remote-csr-proxy/open_questions.md`
4. `docs/remote-csr-proxy/interface_contract.md`
5. `docs/remote-csr-proxy/codex_kickoff_prompt.md`

If two files appear to conflict, prefer the higher item in this list and note the conflict explicitly.

## Working assumptions
- PolarFire SoC MSS is the software/control-plane origin.
- PF Fabric hosts the Remote CSR Proxy logic.
- Kintex hosts the remote IP and the authoritative hardware-side CSR endpoints.
- MSS must interact with PF Fabric over local AXI.
- PF Fabric to Kintex communication uses a custom remote control link.
- The current goal is **CSR/register-space virtualization**, not universal remote AXI emulation.

## What has already been decided
- The architecture direction is **Remote CSR Proxy**, not pure request/response remote register access.
- The PF Fabric block contains:
  - a **Shadow CSR Bank**
  - a **CSR Synchronization Engine**
  - a **descriptor-driven policy layer**
- Reads from MSS should come from the local shadow/proxy view.
- Kintex-to-PF updates must carry **address + new value**.
- Ownership is currently defined **per register**, not per bit-field.
- Resynchronization is allowed only during boot / recovery / idle conditions.
- Command-style pulse-trigger registers are currently modeled as **WO / doorbell-like** semantics.

## What should not be done casually
Do **not** silently change any of these without updating `decision_log.md`:
- ownership model
- shadow-read policy
- resync policy
- command register semantics
- descriptor-table role
- distinction between generic CSR proxying vs full remote AXI emulation

## Desired work style
When refining the design:
1. Preserve the generic architecture unless a use case proves it insufficient.
2. Separate **architecture decisions** from **open design questions**.
3. Call out risks explicitly, especially around:
   - stale shadow data
   - write acknowledgment semantics
   - side-effect registers
   - ordering / coherency assumptions
   - error and recovery behavior
4. Prefer structured tables and state diagrams over loose prose.
5. When proposing a change, compare it against the current baseline and state:
   - what improves
   - what becomes more complex
   - what existing assumption it breaks

## Terminology
Use these names consistently unless changed by a documented decision:
- **Remote CSR Proxy**: top-level PF Fabric control block
- **Shadow CSR Bank**: MSS-visible local CSR image
- **CSR Synchronization Engine**: logic that propagates and reconciles updates
- **Descriptor Table**: policy/configuration metadata for each register or register group
- **Doorbell WO register**: write-only command/pulse register abstraction

## Deliverable preference
Prefer producing or updating:
- markdown design docs
- decision logs
- interface specs
- state diagrams
- register semantic tables

Before making substantial architectural changes, first update `open_questions.md` or add a new ADR-style entry to `decision_log.md`.

# Codex Kickoff Prompt

Use `AGENTS.md` and the files under `docs/remote-csr-proxy/` as the canonical project context.

## Context handling rules
- Do not replace the current architecture direction unless you explicitly compare against the baseline in `decision_log.md`.
- Treat unresolved items in `open_questions.md` as genuinely open; do not silently freeze them.
- Preserve the distinction between:
  - generic CSR/register-space virtualization
  - full remote AXI emulation

## Immediate task suggestion
Refine the Remote CSR Proxy architecture into an implementable specification by doing the following:

1. Propose a concrete **descriptor table schema**.
2. Define a **register semantic taxonomy**.
3. Produce state diagrams for:
   - MSS-owned write
   - Kintex-owned update
   - doorbell WO write
   - resync lifecycle
4. Identify which parts of the current design are sufficient for ADRV9004-related control and which parts are generic infrastructure.
5. Update `interface_contract.md` instead of creating a disconnected alternative spec.

## Output style
- prefer markdown
- use tables where possible
- separate **decision**, **rationale**, and **risk**
- if proposing a new rule, say whether it should become a new entry in `decision_log.md`


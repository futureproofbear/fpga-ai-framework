# Rule: an HLS tool's report is a behavioural model, not ground truth

Constrain an HLS kernel's inputs explicitly (pin bit widths, II, memory architecture, port count —
never rely on the tool inferring the same thing twice), gate its outputs with a check that fails
on a MISSING or STALE result (never one that only checks for the absence of a bad line), and keep a
ledger of what the tool's report predicted versus what silicon/board-free testing actually measured
(DRAM latency, bus backpressure, cache coherency, cross-IP handshake effects the tool cannot model).

**Two classes of intricacy:**
- Class A (the tool CAN respect if constrained): bit widths, II, memory architecture, interface
  bursts, clock period — fix by pinning them explicitly and gating on the achieved value.
- Class B (the tool STRUCTURALLY cannot see): DRAM latency, arbitration/backpressure, coherency,
  cross-IP handshakes, silicon errata — don't make the tool model these; restructure so they're off
  the critical path, and measure them into the ledger.

**Why:** an HLS tool's cosim/report can pass while the actual silicon-measured effective II, or the
actual synthesized RTL for a specific cast/shift pattern, is silently wrong or far slower — this has
been observed on more than one HLS toolchain, not a single vendor's quirk.

Full gate sequence + ledger format: `.claude/skills/hls-output-distrust/`.

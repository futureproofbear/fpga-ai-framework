# AMD/Xilinx timing constraint discipline (Vivado)

## What must be in the constraints fileset, and when

- Physical constraints (`set_property PACKAGE_PIN`/`IOSTANDARD` per port) and timing constraints
  (`create_clock`, `set_input_delay`/`set_output_delay`, `set_false_path`, `set_max_delay`) both need
  to be in the active constraints fileset BEFORE the run (synthesis and/or implementation) they are
  meant to affect. A constraint edited in after a run has already placed/routed the design does not
  retroactively re-time it — the affected run must be re-launched.
- `create_clock` on a primary clock port is not automatically propagated through every downstream
  generated clock (an MMCM/PLL output, a clock divider in RTL) — Vivado infers most generated clocks
  automatically from the netlist, but a clock produced by non-standard logic (a clock built from
  combinational logic, or gated in a way the tool can't trace) may need an explicit
  `create_generated_clock`, or it will simply not be timed at all (silently, not as an error).

## Clock-domain crossings need an explicit constraint — a PASS without one proves nothing

- A path crossing between two clock domains with no `set_false_path`/`set_max_delay
  -datapath_only` is, by default, evaluated as if both clocks needed to close synchronously against
  each other's worst-case relationship. Depending on the actual clock ratio, this can produce EITHER
  a spurious timing failure on a path that was never meant to be synchronous, OR — if the ratio
  happens to leave enough margin — a clean timing PASS on a crossing that has no real synchronizer
  logic behind it at all. **Timing MET is only a claim about the constrained paths; it says nothing
  about CDC correctness on an unconstrained or mis-constrained crossing.** A CDC-specific check
  (`report_cdc`, or an equivalent design-rule pass) is a separate, necessary step.

## Multicycle paths

- `set_multicycle_path` changes the number of clock edges a path is allowed to use — it must be
  applied consistently to BOTH the setup (`-setup`) and hold (`-hold`) checks for a path, or the
  default hold relationship (relative to the unmodified setup edge) can silently produce an
  overly-pessimistic (or, in rarer cases, insufficiently strict) hold constraint that doesn't match
  the actual intended multicycle relationship.

## False paths vs. max-delay

- `set_false_path` removes a path from timing analysis entirely — appropriate only for a path that
  is genuinely asynchronous with a proper synchronizer (e.g. a CDC path already synchronized by a
  double-flop or a handshake), never as a way to silence a real timing failure on a path that
  actually needs to close.
- `set_max_delay -datapath_only` constrains a path's absolute delay without requiring clock-relationship
  analysis — the appropriate middle ground for a genuinely asynchronous data path that still needs a
  bounded propagation delay, as opposed to `set_false_path`'s "don't check this at all."

## Timing report is the source of truth, not the build log's own summary line

- Parse `report_timing_summary`'s WNS/WHS numbers directly rather than trusting a build script's own
  printed "SUCCESS"/"PASS" line — a wrapper's own pass/fail determination can be wrong (see
  `rules/oem/amd_vivado/toolchain_gotchas.md` on stale-report and missing-report gate failures).

See also: `.claude/agents/vivado-build.md` (the full build + timing-gate flow), `rules/oem/
amd_vivado/toolchain_gotchas.md` (broader Vivado toolchain facts).

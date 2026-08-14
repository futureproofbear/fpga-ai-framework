# Timing closure basics (vendor-agnostic)

## The gate: a build is not done until timing is verified MET

Most FPGA toolchains will produce and let you program a **timing-failing bitstream** without
refusing. "Build completed", "bitstream written", and "device programmed" are not evidence the
design is correct. The build script itself must parse the timing report and refuse on a violation.

**Verify timing MET before treating any on-hardware misbehaviour as a logic/firmware bug.** A timing
violation mimics a functional bug perfectly — intermittent wrong data, a state machine that
occasionally takes an impossible transition, a counter that skips. Debugging the logic while the
real cause is a negative-slack path wastes the entire debug cycle.

Check BOTH:
- **Setup** (max-delay / WNS): the path is too slow for the clock period.
- **Hold** (min-delay / WHS): the path is too fast relative to clock skew. Hold violations are not
  fixable by lowering the clock frequency, and are often ignored by a gate that only reads a setup
  summary.

## Gates fail silently in two specific ways — design against both

1. **A missing report reads as zero violations.** A gate that greps a report file for violation
   lines returns "clean" when the file doesn't exist because the run died early. Require the report
   to EXIST and be fresh (mtime at/after this run's start), not merely to lack a bad line.
2. **A stale report from a previous successful run gets re-validated.** Delete timing artifacts at
   the top of the build script, so the gate can only ever grade this run's own output.

Generalized: **a gate must require positive evidence, never the absence of a match.** Require a
minimum parsed-row count, an explicit clean marker, and a fresh timestamp.

## Constraints are the contract — an unconstrained path is an ungraded path

- Every clock must be declared (`create_clock`), and every derived clock either auto-inferred by the
  tool from a recognized clock-generator primitive or explicitly declared (`create_generated_clock`).
  A clock the tool doesn't know about is not timed at all — silently, not as an error.
- I/O paths need `set_input_delay`/`set_output_delay` against the real board timing, or the tool is
  grading against an assumption rather than reality.
- Asynchronous crossings need explicit exceptions — see `cdc-guidelines.md`. Note that
  `set_false_path` on a path that genuinely needs to close is not a fix; it is hiding the failure.
- Multicycle paths (`set_multicycle_path`) must be applied consistently to BOTH setup and hold, or
  the default hold relationship relative to the unmodified setup edge produces a constraint that
  doesn't match the intent.

## Reading a failing path before changing anything

Before restructuring RTL, look at what the report actually says the path is:
- **Logic levels** — many levels of combinational logic between registers means the fix is
  pipelining or restructuring the expression, not a placement/routing hint.
- **Net delay dominant** — routing/congestion or high fanout, addressed by physical measures
  (fanout replication, placement constraints, floorplanning) rather than logic changes.
- **Clock skew / uncertainty dominant** — often a clocking-architecture issue (crossing clock
  regions, an unbalanced clock tree) rather than a datapath issue.
- **The path crosses a domain boundary** — the fix is a proper synchronizer plus the right
  exception, not more pipelining.

Fixing a routing-bound path by adding pipeline stages (or a logic-bound path by tweaking placement)
wastes a full build cycle. The report tells you which one it is.

## Common structural fixes, roughly in order of cost

1. **Pipeline the critical path** (register the intermediate result). Cheapest and most effective
   for logic-bound paths; costs latency, which the surrounding design must tolerate.
2. **Reduce fanout** — replicate a high-fanout driver so each copy drives a smaller, more local set.
3. **Retiming / register balancing** — let the tool move registers across logic (a synthesis
   option in most toolchains) when the logic is unevenly distributed between stages.
4. **Restructure the computation** — e.g. a balanced adder tree instead of a linear chain; move a
   wide multiply into a dedicated DSP primitive rather than fabric logic.
5. **Floorplan / placement constraints** — last resort; brittle across tool versions and design
   changes, and a maintenance burden. Reach for it only when the design is otherwise final.

## Measure before you spend a build

A synthesis-through-implementation run is expensive (tens of minutes to hours). Do not spend one on
a bottleneck you inferred rather than measured — read the actual critical path from the report, and
when a number and a mechanism disagree, chase the disagreement rather than picking one.

Corollary: do not report a timing margin hand-parsed from an intermediate file mid-build. Wait for
the tool's own final timing analysis; intermediate numbers have, in practice, been off by orders of
magnitude from the real result.

See also: `hdl-style.md` (register outputs, pipeline discipline), `cdc-guidelines.md` (crossings and
their exceptions), and your OEM tier's `timing_constraints.md` for vendor-specific constraint syntax
and gate mechanics.

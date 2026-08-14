# HDL style and structure (vendor-agnostic)

Rules for writing synthesizable RTL that behaves the same in simulation and on silicon. These are
about correctness and reviewability, not cosmetics — each one exists because violating it produces a
class of bug that survives simulation.

## Synthesis/simulation agreement

- **One clock per process/always block, and one reset.** A synchronous process is
  `always_ff @(posedge clk)` (SystemVerilog) / `process(clk)` (VHDL) with the reset handled inside.
  Mixing two clocks in one sequential block does not describe hardware that exists.
- **Never mix blocking and non-blocking assignments to the same signal** in Verilog/SystemVerilog.
  Sequential logic uses non-blocking (`<=`); combinational logic uses blocking (`=`). Mixing them
  produces a simulation result that depends on scheduler ordering and does not match synthesis.
- **A combinational process must be complete**: every input read appears in the sensitivity list
  (use `always_comb` / `process(all)` rather than a hand-written list), and every output is assigned
  on every path. An incompletely-assigned combinational output infers a LATCH — which synthesizes,
  passes a casual simulation, and then fails timing or behaves non-deterministically on hardware.
  Treat any latch-inference warning from synthesis as an error to fix, never as noise to filter.
- **Do not read a signal you drive from two processes.** Multiple drivers on one signal is either a
  synthesis error or (worse) resolves differently between simulator and tool.

## Reset discipline

- **Pick one reset style per design and state it**: synchronous or asynchronous-assert/
  synchronous-deassert. Most modern FPGA fabrics prefer synchronous reset (or no reset at all on
  datapath registers, letting them settle) because a global asynchronous reset net is a
  high-fanout timing burden and can itself need a reset synchronizer.
- **A reset crossing into a clock domain must be synchronized to that domain** — see
  `cdc-guidelines.md`. An asynchronous reset released at an arbitrary point relative to a domain's
  clock edge can put registers in that domain into inconsistent states (some see the release before
  the edge, some after), which is a real, intermittent, hard-to-reproduce silicon bug.
- **Do not reset what does not need it.** A datapath pipeline register that will be flushed by valid
  data before its output is used does not need a reset; resetting it only adds fanout. Control/state
  registers (state machine state, counters, valid flags) DO need it.

## State machines

- Encode state as a named enumerated type, not raw integers/parameters scattered through the code.
- **Every state machine needs a defined default/recovery path.** A `case` with no `default` (or a
  `case` over an enum that can hold an unencoded value after an SEU or a reset glitch) can lock the
  machine in an unreachable state with no way out.
- Separate next-state logic (combinational) from state registration (sequential) unless the design
  deliberately uses a one-process style — and if so, use it consistently across the whole design.

## Parameterization

- **A parameter that sizes a structure is load-bearing** — a buffer depth, an address width, a burst
  length, a table size. Changing one silently changes what the module IS.
- **A testbench must instantiate the SAME parameter values synthesis builds.** A bench passing at a
  small parameterization proves nothing about the built module — this has, in practice, produced a
  fully green test suite alongside a completely wrong hardware result. Gate this mechanically (a
  script that parses module defaults, synthesis wrapper overrides, and the TB instantiation and
  fails on divergence), because a comment documenting the divergence is not a gate. See
  `.claude/agents/generic-tb-specialist.md`.
- Scale is part of the parameterization: a bench running at N=64 against silicon at N=8192 cannot
  manifest anything that only appears past a few hundred elements.

## Interfaces and handshakes

- **Use a standard streaming handshake (AXI4-Stream-style `valid`/`ready`) rather than inventing
  one**, and honor its rules exactly: `valid` must not depend combinationally on `ready` (that
  creates a combinational loop between two modules), and once `valid` is asserted the data must
  remain stable until the transfer completes.
- **A module must not deadlock when its downstream backpressures.** Every buffer/FIFO needs enough
  slack to absorb whatever is already in flight when it de-asserts `ready` — reserve the pipeline
  latency's worth of slots, or in-flight beats arrive at a full buffer and are silently dropped.
- Register module outputs by default. An unregistered output pushes the receiving module's timing
  problem across the hierarchy boundary and makes timing closure a cross-module negotiation.

## Readability that affects correctness

- Name signals for what they carry, and suffix by direction/domain where it matters (`_i`/`_o`,
  `_clkA`/`_clkB`) — a domain suffix makes an unsynchronized crossing visible in review.
- Keep the module's clock domain(s) obvious from its port list. A module taking two clocks is
  a CDC module and should be reviewed as one.
- Prefer explicit width on every literal and every port connection; rely on neither implicit
  extension nor tool-default truncation, both of which are silent.

See also: `cdc-guidelines.md`, `timing-closure-basics.md`,
`.claude/agents/generic-rtl-architect.md` (the persona that applies these).

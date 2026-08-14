# Clock-domain crossing (CDC) guidelines (vendor-agnostic)

CDC bugs are the highest-value class to prevent rather than debug: they are intermittent,
temperature/voltage/silicon-lot dependent, frequently invisible in RTL simulation (which has no
notion of metastability unless explicitly modeled), and routinely survive a clean timing report
because the crossing was never constrained as a timed path in the first place.

**A timing PASS is not a CDC-correctness proof.** Timing analysis only grades the paths it was told
to grade. An unconstrained crossing is either not analyzed at all, or analyzed under a wrong
assumption — neither of which says anything about whether real synchronizer logic exists.

## The four crossing types and the correct structure for each

1. **Single-bit level/control signal** → two-flop (or three-flop, for very high ratios)
   synchronizer in the destination domain. Only valid if the source signal is stable long enough for
   the destination to sample it (see pulse rule below).
2. **Single-bit pulse, fast → slow domain** → a two-flop synchronizer is NOT sufficient: a pulse
   narrower than the destination clock period can be missed entirely. Convert to a level (toggle
   flop at source, edge-detect after synchronizing at destination), or use a proper handshake.
3. **Multi-bit data** → never synchronize the bits independently. Independent two-flop
   synchronizers on each bit will, on any cycle where more than one bit changes, let bits land on
   different destination cycles and produce a value that never existed at the source. Use either:
   - an **asynchronous FIFO** (dual-clock, Gray-coded pointers) for streaming data, or
   - a **handshake/MCP (multi-cycle path) structure**: hold the data bus stable, synchronize a
     single-bit request across, and only capture the bus in the destination once the synchronized
     request has arrived.
4. **Gray-coded counter** (the pointer mechanism inside an async FIFO) is the one multi-bit case
   safe to synchronize bit-by-bit — precisely because only one bit changes per increment. This
   property is what makes it safe; it does not generalize to arbitrary multi-bit values.

## Reset crossings are CDC too

A reset that is asserted asynchronously and released asynchronously relative to a domain's clock can
release at different effective cycles for different registers in that domain. Use a **reset
synchronizer** per domain (assert asynchronously, de-assert synchronously to that domain's clock).
This is an easy one to omit because the design "comes up fine" in most power-up sequences and then
fails on some fraction of boards or after some fraction of resets.

## Constrain what you build

Every crossing needs an explicit timing exception, matched to the structure used:
- `set_false_path` — for a genuinely asynchronous crossing already protected by a proper
  synchronizer. Never as a way to silence a real timing failure on a path that actually needs to
  close.
- `set_max_delay -datapath_only` — for an MCP/handshake data bus that needs a bounded skew (so the
  bus's bits arrive within one destination cycle of each other) but should not be timed against a
  clock relationship. This is the correct choice more often than a bare false path for multi-bit
  MCP structures, because a false path lets the tool route the bus bits arbitrarily far apart.

An unconstrained crossing left to default synchronous analysis will, depending on the clock ratio,
either produce a spurious failure (wasting effort "fixing" a non-problem) or silently pass — the
second being the dangerous one.

## Review discipline

- **Make crossings visible in the code.** A module taking two clocks is a CDC module; name it and
  its signals so a reviewer can see the domain of each port (`_clkA`/`_clkB` suffixes, or a naming
  convention stated once in the project).
- **Concentrate crossings into a small number of dedicated, reviewed modules** (an async FIFO
  instance, a handshake synchronizer instance) rather than scattering ad-hoc two-flop chains
  through the design. Scattered synchronizers are where the multi-bit mistake gets made.
- Run the toolchain's own CDC report (Vivado `report_cdc`, or the equivalent) and read it — it
  catches the structural cases (unsynchronized crossing, multi-bit fanning into independent
  synchronizers) that neither RTL simulation nor a timing report will.

## What simulation will and will not catch

- Plain RTL simulation does NOT model metastability and will happily show a design working that
  fails on hardware.
- Gate-level simulation with SDF back-annotation catches some, not all.
- A CDC linting/structural-analysis tool is the only thing that reliably catches the *structural*
  error (missing or wrong synchronizer). Treat its findings as design bugs, not advisories.

See also: `.claude/skills/cdc-checker/` (the audit workflow), `hdl-style.md` (reset discipline),
`timing-closure-basics.md` (constraint mechanics), and your OEM tier's constraint rules for
vendor-specific syntax.

---
name: vivado-iso-test-harness
description: >-
  The AMD-Xilinx-toolchain HOW for running a JTAG/Hardware-Manager silicon iso-test: hw_server +
  Vivado Hardware Manager Tcl API command patterns (VIO stimulus, ILA capture), or XSDB for an
  embedded-processor target, plus the fabric-only-device caveat (no hard/soft processor => no
  mailbox pattern). Complements the vendor-agnostic kernel-isolation-testing rule (what an iso-test
  is and why) with the concrete AMD-Xilinx mechanics (how to actually run one). Triggers: "run the
  iso-test on Vivado", "hw_server VIO/ILA test", "poke a fabric register over JTAG on Xilinx",
  "xsct mrd/mwr".
---

# vivado-iso-test-harness

This skill is the AMD-Xilinx-tool-specific counterpart to the vendor-agnostic
`rules/generic/kernel-isolation-testing.md`. That rule covers the *concept*; this skill covers the
*mechanics* of doing it over `hw_server` + Vivado Hardware Manager, or XSDB, specifically.

## Prerequisites (confirm first)

- Board powered ON; the design under test is the one actually programmed (see
  `vivado-ila-vio-probe` for why the debug-probes file must match the programmed bitstream exactly).
- No stale `hw_server` process already running and holding the cable.
- Decide the control path BEFORE picking a mechanism: a fabric-only device (no hard/soft processor
  system) has no mailbox/hart to halt — "poke and read" here means driving a VIO output register and
  reading an ILA capture or another VIO input register, not a gdb/JTAG-halt style interaction. Only
  reach for XSDB when the design actually has an embedded processor (MicroBlaze, or a Zynq/Zynq
  UltraScale+ hard processor system) to attach to.

## Command patterns — fabric-only device (VIO + ILA)

Headless Tcl, no GUI:
```tcl
open_hw_manager
connect_hw_server
open_hw_target
current_hw_device [lindex [get_hw_devices] 0]
refresh_hw_device [current_hw_device]

set vio [get_hw_vios -of_objects [current_hw_device]]
set probe_out [get_hw_probes <vio_probe_name> -of_objects $vio]
set_property OUTPUT_VALUE 1 $probe_out
commit_hw_vio $probe_out

set ila [get_hw_ilas -of_objects [current_hw_device]]
run_hw_ila $ila
wait_on_hw_ila $ila
set data [upload_hw_ila_data $ila]
display_hw_ila_data $data
```

## Command patterns — embedded-processor device (XSDB)

```tcl
connect
targets
target <n>
mrd <addr>                 ;# memory read
mwr <addr> <value>          ;# memory write
```
Some `xsct` builds' `mrd` output format has been observed to be terse/easy to mis-parse in a script
— confirm the exact output shape against your installed version before parsing it programmatically.

## Procedure

1. Pick the smallest decisive case (`kernel-isolation-testing`'s own guidance) — for a JTAG-driven
   test this specifically also protects the debug link itself: a large multi-transaction run is more
   likely to leave something mid-flight than a single minimal case is.
2. Run the test IN THE BACKGROUND if it involves any polling/waiting, so it can self-terminate
   rather than being killed externally (see `vivado-jtag-recovery` for why an external kill is risky
   here specifically).
3. Capture the decisive pass/fail signals (a VIO/ILA snapshot, a register readback) BEFORE any
   cleanup step that is itself known to sometimes hang — so the run still produces usable data even
   if that step hangs.
4. Report per test: probed signal states, output sample values, comparison against expected/golden
   values for that isolated stage, and PASS/FAIL.

## Hard rules (see `vivado-jtag-recovery` for the full recovery procedure)

- Only one Hardware Manager/`hw_server` session can own the cable at a time.
- Never force-kill `hw_server`/`xsct` mid-operation. Clean stop = `close_hw_target` +
  `disconnect_hw_server`, or the client's own clean `disconnect`.
- If forced to kill: the Tcl/GUI client first, then `hw_server`. A wedged cable may need a USB
  replug — a board power-cycle alone does not always clear a cable-level wedge.
- Always leave the toolchain disconnected cleanly; confirm no orphaned `hw_server`/`xsct` processes
  remain before starting the next run.

## After

If a new gotcha or reusable command pattern comes out of a session, write it into the project's own
runbook the same session, and consider whether it's generic enough to feed back into this framework
repo (see this repo's own `CLAUDE.md` for the tier-placement discipline).

See also: `rules/generic/kernel-isolation-testing.md` (the underlying methodology),
`vivado-jtag-recovery` (what to do when this goes wrong), `vivado-ila-vio-probe` (follow-up when a
simple register poke isn't enough internal visibility), `rules/generic/value-level-verification.md`
(what to check for once output is readable).

---
name: vivado-jtag-recovery
description: >-
  Safely tear down a wedged or stuck JTAG/Hardware Manager session on an AMD/Xilinx FPGA board
  (hw_server + Vivado Hardware Manager, or XSDB for an embedded-processor target) and decide the
  correct recovery, without wedging the cable further. Use when a Hardware Manager connect hangs,
  hw_server is orphaned/unresponsive, or the board won't program/connect. Triggers: "hw_server
  hung", "unable to open target", "vivado hardware manager stuck", "clean up the jtag cable",
  "xsct stuck".
---

# vivado-jtag-recovery

Safe teardown + recovery for a stuck JTAG toolchain (`hw_server` + Vivado Hardware Manager, or
`xsct`) on an AMD/Xilinx FPGA board. The point is to NOT compound the problem: force-killing
`hw_server` mid-operation can leave the USB-JTAG cable in a state that needs a physical replug to
clear, and a board power-cycle alone does not always clear that.

## Diagnose first (don't kill blindly)

- Check for another Hardware Manager session (a second Vivado GUI, `vivado_lab`, or a CI job) or an
  orphaned `hw_server` process already holding the cable — this is the single most common cause of
  "unable to open target" and needs no recovery beyond closing the other session.
- Read the `hw_server` log/console. If it stopped progressing right after a `open_hw_target`/tap
  scan and never reached device identification, the connect itself wedged — usually a marginal
  cable/driver state, not the design.
- If the log shows a `program_hw_devices` or `run_hw_ila` in progress then froze, it wedged
  MID-OPERATION — e.g. a capture waiting on a trigger that never fires because the design side is
  stuck, which can look identical to a cable wedge but isn't one.

## Ordered teardown (least-invasive first)

1. From the Tcl client: `close_hw_target`, then `disconnect_hw_server` — the clean path.
2. If the client itself is unresponsive, close/kill the Vivado GUI or Tcl console FIRST — this does
   NOT touch the cable directly.
3. Only if `hw_server` is still orphaned AND idle (no operation actually in flight — check its log
   isn't actively growing), terminate the `hw_server` process directly. Killing an IDLE `hw_server`
   is far lower risk than killing one mid-operation.
4. Confirm no `hw_server`/`xsct`/orphaned Vivado batch processes remain.

## Recovery decision

- Wedged at CONNECT after a fresh `hw_server` restart, or you had to force-kill `hw_server`:
  **unplug and replug the USB-JTAG cable, THEN power-cycle the board** if the target still doesn't
  enumerate. The cable is commonly USB-powered and electrically independent of the target board — a
  board power-cycle alone does not always clear a USB-level cable wedge.
- Wedged MID-OPERATION on a design-side transaction (e.g. an ILA trigger that never fires because
  the design itself is stuck): power-cycle the board to clear the stuck design state, then re-run
  with the smallest possible test case rather than the full run that triggered the wedge.
- On an XSDB/embedded-processor target: a debug-module operation left in flight on a halted core can
  prevent a clean `disconnect` — if `xsct`/gdb won't disconnect cleanly, close the client first,
  then the target connection, then consider a board power-cycle before re-attaching.

## Never

- Never force-kill `hw_server` while a `program_hw_devices`/`run_hw_ila` operation is genuinely in
  flight.
- Never leave orphaned `hw_server`/`xsct`/Vivado batch processes for the user to clean up — confirm
  the process list is clear before handing control back.

See also: `vivado-iso-test-harness` (the normal, non-wedged run this skill recovers FROM),
`rules/generic/kernel-isolation-testing.md` (the underlying debug-session-hygiene discipline this
skill is the AMD-Xilinx-specific instance of).

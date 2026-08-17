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

## "No matching targets found on connected servers: localhost"

This message means hw_server ran and Vivado connected to it — it does NOT mean hw_server is
missing, blocked, or crashed. It means the server has no *claimable* cable. Diagnose in this order
before concluding anything about hw_server itself:

1. **Prove hw_server actually runs** — launch it standalone (`hw_server -s tcp::3121`). If it prints
   "hw_server application started", the executable is fine and the problem is downstream. Skipping
   this step is how a driver problem gets misdiagnosed as an endpoint-security block.
2. **Check the cable is enumerated at all.** On Windows the USB registry subtree is authoritative
   and readable without admin:
   `reg query HKLM\SYSTEM\CurrentControlSet\Enum\USB /s` — AMD/Xilinx cables are `VID_03FD`
   (PID_0008 = Platform Cable USB, PID_0013 = Platform Cable USB II); Digilent/FTDI-based cables are
   `VID_0403`. (`&` in a device path is a cmd command separator — query the parent subtree and grep,
   rather than fighting the escaping.)
3. **Check a driver is actually BOUND to it.** This is the one that catches people: a cable can be
   physically present and fully enumerated while having no driver. In the device's registry key look
   for a `Service`/`Driver` value and at `ConfigFlags` — **`ConfigFlags = 0x40` is
   `CONFIGFLAG_FAILEDINSTALL`**, i.e. Windows tried to install a driver for this device and failed.
   No `Service` value + `ConfigFlags 0x40` = the cable will never appear as a target no matter what
   you do to hw_server.
4. **Fix:** run the cable-driver installer that ships with the tool
   (`<install>/data/xicom/cable_drivers/nt64/install_drivers.cmd`) **as Administrator**. In a
   locked-down corporate environment this is frequently the real blocker — driver installation
   requires admin, which is a different permission from "may I run this executable."
5. **If admin is genuinely unavailable**, the driver only has to exist on the machine physically
   holding the cable: run `hw_server` on a machine where you do have admin and connect remotely with
   `connect_hw_server -url <host>:3121`. A network JTAG module (SmartLynq-class) avoids host USB
   drivers entirely. Third-party programmers (openFPGALoader, xc3sprog) do NOT avoid this — they
   still need a USB driver bound (usually WinUSB via Zadig), so they hit the same admin wall.

Note the Digilent DLL warnings hw_server prints on startup (`cannot open library dpcomm.dll` /
`djtg.dll`, error 0x7e) are about *Digilent* cable support only — they are expected and harmless
when using an AMD/Xilinx-branded cable, and are not the cause of a missing target.

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

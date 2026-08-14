---
name: vivado-ila-vio-probe
description: >-
  Produce a Vivado ILA/VIO debug-core plan (exact net names from the IMPLEMENTED design's own
  netlist) and a decode table mapping captured values to a verdict, on an AMD/Xilinx FPGA design.
  Use when a fabric kernel/IP stalls and JTAG register reads (or no debug core yet) aren't enough
  internal visibility. Triggers: "probe the fabric on Xilinx", "add an ILA", "mark_debug", "what
  nets should I instrument", "internal signal visibility Vivado".
---

# vivado-ila-vio-probe

Plans and interprets Vivado ILA (capture)/VIO (stimulus) debug sessions for an AMD/Xilinx FPGA
fabric design. You cannot drive the Hardware Manager GUI yourself — produce an exact
`mark_debug`/probe list, then decode what comes back.

## THE critical rule

**The `.ltx` debug-probes file must correspond exactly to the specific implementation run that
produced the currently-programmed `.bit`.** A project that has accumulated multiple implementation
runs over its history (an RTL revision, a different constraint set, an experimental branch) will
have netlists that differ — a probe resolved against an older run's placed netlist will point at the
wrong resource (or none) in a currently-programmed different run. Loading a mismatched `.ltx`
against a `.bit` gives plausible-looking GARBAGE (a real, DC-stable value from some resource — just
not the one the probe name suggests). So:

1. Determine which implementation run is programmed right now (ask, or infer from the project's
   own record of what was last built and programmed).
2. Resolve every probe/net name from THAT run's own placed design — either its already-instrumented
   `.ltx`, or (if not yet instrumented) by adding `mark_debug` and rebuilding.
3. Confirm Hardware Manager shows no "probes file does not match" warning before trusting any
   capture.

## Procedure

1. **A signal not already behind an ILA requires a rebuild** — adding `(* mark_debug = "true" *)`
   in RTL, or `set_property mark_debug true [get_nets ...]` post-synthesis followed by re-running
   synthesis + implementation. This is not read-only on an already-built bitstream; say so.
2. Build the probe plan: map the symptom to a MINIMAL set of decisive, preferably REGISTERED nets
   (a purely combinational net may be optimized away, or forcing it to survive via `KEEP`/
   `DONT_TOUCH` can itself perturb timing). For each: the RTL signal path, its clock domain (the
   ILA samples on ONE clock), and value -> meaning.
3. Give a decode TABLE (trigger condition -> captured pattern -> verdict -> next action) that
   bifurcates the candidates in as few captures/rebuilds as possible — each NEW probe not already
   present costs a full synthesis+implementation cycle, unlike a same-session repeated capture on
   already-instrumented signals, which is cheap.
4. When values come back (waveform, or a headless `upload_hw_ila_data`/`display_hw_ila_data` dump),
   decode against the table, cross-check the probes came from the run actually programmed, and state
   the verdict + the next probe.

## Notes

- VIO can force a stimulus register directly, independent of any host-side control path — useful to
  reach a state that would otherwise require the full system to be up.
- ILA capture depth is finite (set at instrumentation time) — a stall that doesn't trip the trigger
  within the captured window needs either a tighter trigger condition or a deeper/rebuilt core, not
  a longer wait on the same capture.
- Treat any decode table from a previous debugging session as provisional — if the design changed
  (a net renamed, an IP swapped, a different run programmed), an old decode table can describe
  signals that no longer exist or no longer mean the same thing.

See also: `vivado-iso-test-harness` (getting a stall into a state a probe can see),
`rules/oem/amd_vivado/toolchain_gotchas.md` (other Vivado toolchain facts worth ruling out before
trusting a probe result as a design bug).

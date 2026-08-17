# AMD/Xilinx Vivado toolchain gotchas (UltraScale/UltraScale+ family)

Hard-won Vivado TOOLCHAIN peculiarities that generalize across the UltraScale/UltraScale+ family
(any part) using this toolchain. Excludes anything specific to one exact chip family's architecture
— that belongs in `rules/target/amd/<family>/`. If a symptom looks impossible, check here before
assuming RTL/firmware is wrong.

## Bitstream vs. timing

- **`write_bitstream` does NOT refuse a timing-failing design by default.** Unlike a toolchain that
  blocks export outright, Vivado will happily write a bitstream with negative WNS/WHS unless the
  build script itself parses `report_timing_summary` and gates on it. "write_bitstream completed" is
  not evidence of a correct design. See `.claude/agents/vivado-build.md` for the full gate.
- A clean timing report does not imply a clean design: also check `report_drc` — unplaced/unrouted
  logic or an unconstrained clock can coexist with a timing summary that has nothing to report
  (because it never saw those paths at all).

## Constraints ordering and CDC

- A constraint (physical or timing) added to the project after the implementation run it was meant
  to affect does not retroactively apply — re-run implementation.
- A domain-crossing path with no `set_false_path`/`set_max_delay -datapath_only` is, by Vivado's
  default treatment, evaluated as if it needed to close synchronously — this can produce either a
  spurious failure on a path that never needed to be timed together, or (if it happens to have
  enough margin) a silent PASS on a crossing that has no real synchronizer at all. Timing MET only
  proves the constrained paths closed; it is not a CDC-correctness proof.

## IP / synthesis caching

- An XCI-managed IP, or a source file swapped outside Vivado's own change tracking, can be reused
  across `launch_runs` without re-synthesizing. If an edit doesn't seem to reach the netlist,
  `reset_run` the affected run rather than assuming synthesis is broken.
- Out-of-context (OOC) IP synthesis runs independently of the top-level run — an OOC IP's own
  synthesis settings/constraints can silently diverge from what the top-level expects it to be if
  the IP is regenerated with different parameters and the OOC run isn't re-triggered.

## Debug cores (ILA/VIO)

- Inserting `mark_debug`/an ILA/VIO requires a full re-synthesis + re-implementation — it is not a
  post-bitstream, read-only operation. The resulting `.ltx` probes file is tied to that ONE
  implementation run; pairing it with a `.bit` from a different run gives plausible-looking garbage.
  See `.claude/skills/vivado-ila-vio-probe/`.
- Debug cores consume fabric resources and can perturb timing (or, rarely, whether a race condition
  reproduces at all) — note explicitly if a symptom changes once debug cores are added.

## JTAG / Hardware Manager

- Only one `hw_server`/Hardware Manager session (or `vivado_lab`) can own a given JTAG cable/target
  at a time; a held cable commonly surfaces as "unable to open target" rather than a clear
  "already in use." See `.claude/skills/vivado-jtag-recovery/`.

## Batch-mode / Tcl console output

- Vivado's own console output in batch mode can be lost on abnormal process exit. A build script
  that needs a durable log should append progress markers to a file with explicit flushing, rather
  than relying solely on captured stdout.
- **Run every build in the BACKGROUND, redirected to a log — never as a blocking foreground call.**
  Even a trivial design on a large part (a handful of LUTs on an XCKU115) takes many minutes: Vivado
  spends most of that loading the device database and running the full opt/place/route flow, so
  build time tracks the PART's size far more than the DESIGN's. An agent/tool call with a typical
  1–2 minute timeout will be killed mid-synthesis. Launch it detached, then poll the log for the
  script's own progress markers. (Killing a synthesis/implementation run is safe — no board or JTAG
  cable is involved. Killing a `program_hw_devices` is NOT; see `vivado-jtag-recovery`.)

See also: `.claude/agents/vivado-build.md` (the timing-gate discipline), `.claude/skills/
vitis-hls-kernel-authoring/` (Vitis HLS pragma reference), `.claude/skills/vivado-ila-vio-probe/`
and `.claude/skills/vivado-iso-test-harness/` (the debug workflows these quirks most often bite),
`.claude/skills/vivado-jtag-recovery/` (cable recovery). Chip-family-specific architecture facts
(transceiver types, DSP/BRAM/UltraRAM presence) are intentionally NOT covered here — see
`rules/target/amd/<family>/`.

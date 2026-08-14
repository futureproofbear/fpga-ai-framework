---
name: vivado-build-tcl
description: >-
  Generate automated Vivado build Tcl (non-project "batch" mode, or project mode) for AMD/Xilinx
  FPGAs, with the timing gate, DRC check, and stale/missing-report protections built in rather than
  bolted on. Trigger when asked to write/build/synthesize a Vivado flow script, set up a
  reproducible build, or convert a GUI-driven project into a scripted one. Triggers: "vivado tcl",
  "build script", "non-project mode", "synth_design / route_design", "write_bitstream", "automate
  the vivado build", "checkpoint / dcp flow".
---

# Vivado build Tcl generator

Generates the build script. The `vivado-build` AGENT runs and gates builds; this SKILL is about
authoring the script it runs. Read `rules/oem/amd_vivado/toolchain_gotchas.md` and
`rules/oem/amd_vivado/timing_constraints.md` first — the gate requirements below exist because of
specific, documented failure modes there.

## Non-project mode vs. project mode — pick deliberately

- **Non-project (batch) mode** — you call `read_verilog`/`read_vhdl`/`read_xdc`/`synth_design`/
  `opt_design`/`place_design`/`route_design`/`write_bitstream` directly, and manage checkpoints
  (`write_checkpoint`) yourself. Fully reproducible from source, diff-able, no `.xpr` state to
  drift. **Preferred for CI and for a design under version control.**
- **Project mode** — `create_project`, `add_files`, `launch_runs synth_1/impl_1`. Better GUI
  interop and IP-catalog/IP-integrator management; carries `.xpr` state that can drift from what's
  in git.

Do not mix idioms in one script. If the project already has a `.xpr` the team uses interactively,
generating a non-project script that ignores it creates two sources of truth — say so and let the
user choose rather than silently picking.

## Skeleton (non-project mode) — the gate is not optional

```tcl
set PART        xcku115-flvb2104-2-e
set TOP         <top_module>
set OUT         ./build
set RUN_START   [clock seconds]

file mkdir $OUT
# Delete timing artifacts FIRST so the gate can only ever grade THIS run's output.
foreach f [glob -nocomplain $OUT/timing_summary.rpt $OUT/drc.rpt] { file delete -force $f }

read_verilog  [glob ./src/rtl/*.v]
# read_vhdl   [glob ./src/rtl/*.vhd]
read_xdc      [glob ./constraints/*.xdc]

synth_design -top $TOP -part $PART
write_checkpoint -force $OUT/post_synth.dcp

opt_design
place_design
phys_opt_design
route_design
write_checkpoint -force $OUT/post_route.dcp

report_timing_summary -file $OUT/timing_summary.rpt
report_drc            -file $OUT/drc.rpt
report_utilization    -file $OUT/utilization.rpt

# ---- GATE ----
set wns [get_property SLACK [get_timing_paths -delay_type max]]
set whs [get_property SLACK [get_timing_paths -delay_type min]]
puts "WNS=$wns WHS=$whs"

if {![file exists $OUT/timing_summary.rpt]
    || [file mtime $OUT/timing_summary.rpt] < $RUN_START} {
    error "TIMING GATE: report missing or stale -- refusing to export"
}
if {$wns < 0 || $whs < 0} {
    error "TIMING NOT MET (WNS=$wns WHS=$whs) -- refusing to export"
}
write_bitstream -force $OUT/$TOP.bit
puts "TIMING_MET -- bitstream written to $OUT/$TOP.bit"
```

## Gate requirements — include all of these, every time

1. **Both WNS and WHS.** A gate reading only setup silently ignores hold violations, which are not
   fixable by slowing the clock and which corrupt data in ways that look exactly like logic bugs.
2. **Freshness check.** Require each report to exist AND have an mtime at/after the run's start.
   Deleting the artifacts up front plus this check is what stops a previous run's clean report from
   being re-validated when this run dies early.
3. **Never a gate that concludes "clean" from the absence of a match.** A missing report, an empty
   report, or a report of the wrong kind must FAIL, not pass. Require positive evidence.
4. **`report_drc` as a separate check** — unplaced/unrouted logic can coexist with a timing summary
   that has nothing to report because it never saw those paths.
5. **`error` (not `puts`) on failure**, so `vivado -mode batch` returns a non-zero exit status and a
   calling CI job/script actually fails.

## Other script-authoring notes

- Invoke as `vivado -mode batch -source build.tcl -journal $OUT/build.jou -log $OUT/build.log`.
  Vivado's console output can be lost on abnormal process exit — append progress markers to a file
  with explicit flushing rather than relying solely on captured stdout.
- **Debug cores**: `mark_debug`/ILA insertion requires re-running synthesis AND implementation, and
  the resulting `.ltx` is tied to that one implementation run. Write the `.ltx` next to the `.bit`
  in the same output directory and never mix them across runs — see
  `.claude/skills/vivado-ila-vio-probe/`.
- **Constraints must be read before the step they affect.** A physical/timing constraint added after
  implementation does not retroactively re-time placed logic.
- For an IP-based design, `generate_target`/`synth_ip` the IP (or read its `.dcp`) before the
  top-level synth; and if a source edit doesn't reach the netlist, `reset_run` rather than assuming
  synthesis is broken — see `rules/oem/amd_vivado/ip_catalog_rules.md`.
- Parameterize `PART`, `TOP`, and the source globs at the top of the script; never hard-code an
  absolute machine-specific path into a script that goes in git.

## Programming is a separate, user-authorized step

Do NOT put `program_hw_devices` in the build script. Programming a board is a distinct action
requiring the user's authorization and an available JTAG cable — and only one Hardware Manager/
`hw_server` session can own that cable at a time. Keep it in its own script.

See also: `.claude/agents/vivado-build.md` (runs and gates the build), `rules/oem/amd_vivado/*`
(the toolchain facts these gates defend against), `rules/generic/timing-closure-basics.md` (why the
gate is structured this way).

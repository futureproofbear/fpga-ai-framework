# Microchip toolchain gotchas (PolarFire SoC family)

Hard-won Microchip TOOLCHAIN and IP peculiarities that generalize across the PolarFire SoC family
(any die: MPFS250T, MPFS095T, etc.) using this toolchain. Excludes anything specific to one exact
die's engineering-sample silicon errata — that belongs in `rules/target/microchip/<die>/`. If a
symptom looks impossible, check here before assuming RTL/firmware is wrong; a good fraction of
"impossible" toolchain behaviour is a known quirk, not a design bug.

## Libero SoC

- **Libero will silently PROGRAM a timing-failing bitstream.** It does not refuse to export or
  program a design that failed setup or hold timing — the build gate has to check and refuse
  explicitly. "Build completed" and "device programmed" are not evidence of "design is correct."
  See `.claude/agents/libero-build.md` for the full timing-gate discipline.
- **Libero project HDL-core caching**: a hand-registered HDL core (`create_hdl_core` +
  `hdl_core_add_bif`) can be cached by the project and NOT pick up a source-file edit unless the
  core is explicitly re-registered/refreshed. If a Verilog-source change doesn't seem to reach the
  synthesized netlist, check whether the HDL core needs re-registering rather than assuming
  synthesis itself is broken.

## SmartDebug

- **The SmartDebug design database must match the bitstream actually programmed on the board.**
  Probing from a different (even a very similar, previously-built) project's netlist returns
  plausible-looking garbage rather than an error. See `.claude/skills/smartdebug-active-probe/`.

## FlashPro6

- **FlashPro6 has USB-HID-level wedge behaviour** independent of the target board: a board
  power-cycle alone does not clear a wedge caused by force-killing OpenOCD mid-operation; the
  FlashPro6 itself needs a USB replug. See `.claude/skills/flashpro6-jtag-recovery/`.
- Only one tool can own the FlashPro6/JTAG link at a time — a Libero `PROGRAMDEVICE` operation and a
  live OpenOCD session (or a SmartDebug session) cannot run concurrently against the same probe.

## SmartHLS

- **Pure memory-to-stream / stream-to-memory HLS kernels have been observed to synthesize to dead
  RTL** on this toolchain; a kernel that must stream data directly to/from an AXI-initiator
  interface (as opposed to a mem-to-mem read-compute-write shape) may need to be hand-written in RTL
  instead. See `.claude/skills/smarthls-kernel-authoring/` for the full mis-synthesis-class list and
  the pragma reference.

## Boot mode / JTAG halt interaction (MSS architecture)

- On PolarFire SoC's MSS, whether a hart can be halted over JTAG depends on the boot-mode image
  currently resident, not just on the fabric being programmed correctly. A Hart Software Services
  (HSS)-only image commonly refuses a JTAG halt outright. A cooperating application image (built to
  yield/WFI or otherwise cooperate with debug halt requests) is generally required for a JTAG debug
  session to reliably halt a hart.
- Firmware-only changes (no fabric rebuild needed) are typically rebuilt with the SoftConsole
  toolchain (`make all`) and reprogrammed via Microchip's boot-mode programmer utility
  (`mpfsBootmodeProgrammer` or equivalent) — much faster than a full Libero rebuild when only the
  MSS application needs to change, not the fabric.

## SoftConsole / GDB

- **The RISC-V gdb bundled with SoftConsole (riscv64, toolchain build 8.3.0) crashes on an inferior
  `call <fn>` (a `find_inferior_pid` assertion) if the target hart is mid-execution.** This is a bug
  in that specific gdb build, not a JTAG/hardware fault, and reproduces on any PolarFire SoC board
  using the same SoftConsole version. Guard every inferior `call` behind a completion-flag check.

## FIC / interconnect architecture (general MSS characteristics)

- PolarFire SoC's Fabric Interface Controllers (FICs) connecting the MSS to fabric are commonly
  documented as **non-coherent** with the MSS's own cache hierarchy on at least some paths — treat
  non-coherence as the default expectation and confirm which paths are coherent from the MSS
  configuration rather than assuming.
- FIC AXI-ID width has been observed to be narrower than a full AXI ID field on at least some FIC
  interfaces — check the actual ID width before assuming a source IP's full ID width survives the
  crossing.

## CoreFFT (Microchip hard IP)

- CoreFFT's in-place mode has a documented slow-clock ratio ceiling relative to its main clock
  (check the CoreFFT User Guide for the exact ratio) — driving the slow clock faster is a protocol
  violation, not just a performance choice.
- CoreFFT's in-place memory buffer has an overwrite hazard if the handshake isn't followed exactly
  as documented — read the handshake section, not just the block diagram, before wiring an adapter.
- **The output-valid/output-ready latency trap:** `DATAO_VALID` trails `READ_OUTP` by a fixed
  pipeline latency. An adapter that gates capture directly on `READ_OUTP` rather than
  `DATAO_VALID` drops in-flight beats under backpressure. Capture on `DATAO_VALID`.

See also: `.claude/agents/libero-build.md` (timing-gate discipline), `.claude/skills/
smarthls-kernel-authoring/` (pragma reference + mis-synthesis classes), `.claude/skills/
smartdebug-active-probe/` and `.claude/skills/microchip-iso-test-harness/` (the debug workflows
these quirks most often bite), `.claude/skills/flashpro6-jtag-recovery/` (recovery procedure).
Die-specific engineering-sample silicon errata is intentionally NOT covered here — see
`rules/target/microchip/<die>/`.

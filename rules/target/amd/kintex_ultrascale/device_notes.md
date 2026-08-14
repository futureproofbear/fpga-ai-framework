# Kintex UltraScale (first-generation, non-"+") device facts

Facts about the Kintex UltraScale FAMILY (any part: XCKU025 through XCKU115), verified where noted
against the exact part in use on this project (`xcku115-flvb2104-2-e`, Vivado 2023.2). Distinct from
Kintex UltraScale**+**, which differs in several of the ways below — do not assume a UltraScale+
fact (or a UltraScale+-targeted IP/example design) applies here without checking.

## Verified on XCKU115 (via Vivado `get_sites`/`get_property` — see `transceiver_rules.md` for method)
- DSP48E2 slices: **5,520**
- Block RAM: **2,160** RAMB36-equivalent blocks (~75.9 Mb total)
- **UltraRAM: zero.** UltraRAM was introduced with UltraScale+; it does not exist anywhere in the
  Kintex UltraScale (non-Plus) line. A design/IP that assumes UltraRAM availability (some memory
  controller reference designs, some AXI BRAM/UltraRAM controller IP configurations) will not build
  as-is against this family — check for a BRAM-only configuration option instead.
- GTH transceivers: **80** sites present on the part; **zero GTY** (see `transceiver_rules.md`).
- CLB slices: **82,920**.
- This project's exact part, `xcku115-flvb2104-2-e`, corresponds to the schematic's "104 HR, 520 HP,
  48 GTH" bank/transceiver description (HR = high-range I/O bank, HP = high-performance I/O bank).

## DSP48E2 (shared with UltraScale+, NOT shared with Versal's DSP58)

- Same primitive across UltraScale and UltraScale+ — a DSP48E2-targeted netlist/IP is portable
  between the two families at the DSP-primitive level (subject to everything else in this file).
  Versal's DSP58 is a different primitive with a wider native multiplier and additional native
  functions (e.g. floating-point) — do NOT assume DSP48E2-era pragma/attribute guidance (packing,
  cascade) carries over to a Versal target.

## Speed grades and packages

- This part is available in this package across `-1`/`-1L`/`-1LV`/`-2`/`-2E`/`-2I`/`-3E` speed-grade
  suffixes (confirmed via `get_parts -filter {NAME =~ *xcku115-flvb2104*}` in Vivado 2023.2); a board
  populated with one speed grade cannot be silently retargeted to another without re-running
  implementation and re-checking timing closure — a faster speed grade is not simply "the same
  bitstream, closes easier," and a slower one is not simply "add margin."

## Where family-general facts stop and board-specific facts start

This file (and `transceiver_rules.md`) covers the CHIP FAMILY, independent of which board it's
soldered to. A specific board's connector wiring, clock-generator topology, and which of a
multi-populate board's schematic nets are actually live on the populated part are BOARD facts, not
family facts — those belong in the application repo's own project-local skill/notes, not here (see
this framework repo's own `CLAUDE.md` for why a board is deliberately not a tier).

See also: `transceiver_rules.md` (GTH/GTY architecture + the pin-compatible-board verification
method used to produce every verified fact in this file).

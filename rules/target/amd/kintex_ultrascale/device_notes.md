# Kintex UltraScale (first-generation, non-"+") device facts

Facts about the Kintex UltraScale FAMILY (any part: XCKU025 through XCKU115), verified where noted
against the exact part in use on this project (`xcku115-flvb2104-2-e`, Vivado 2023.2). Distinct from
Kintex UltraScale**+**, which differs in several of the ways below — do not assume a UltraScale+
fact (or a UltraScale+-targeted IP/example design) applies here without checking.

## Verified on XCKU115 (Vivado 2023.2, `get_property <prop> [get_parts xcku115-flvb2104-2-e]` — no design open; see `rules/oem/amd_vivado/toolchain_gotchas.md` for why this beats `get_sites`)
- DSP48E2 slices: **5,520** (`DSP`)
- Block RAM: **2,160** RAMB36-equivalent blocks (`BLOCK_RAMS`; ~75.9 Mb total)
- **UltraRAM: zero.** UltraRAM was introduced with UltraScale+; it does not exist anywhere in the
  Kintex UltraScale (non-Plus) line. A design/IP that assumes UltraRAM availability (some memory
  controller reference designs, some AXI BRAM/UltraRAM controller IP configurations) will not build
  as-is against this family — check for a BRAM-only configuration option instead.
- GTH transceivers: **64** (`GTHE3_TRANSCEIVERS` = `GB_TRANSCEIVERS` = 64); **zero GTY** — the part
  has no `GTY*` property at all (see `transceiver_rules.md`). Note the raw site count is 80: 64
  `GTHE3_CHANNEL` + 16 `GTHE3_COMMON` (one per quad). Sixty-four is the transceiver number.
- CLB slices: **82,920** (`SLICES`); flip-flops **1,326,720**; LUT elements **663,360**; MMCM **24**;
  bonded IOBs available in this package **702** (`AVAILABLE_IOBS`); `IDCODE` **0x0390d093**.
- **SSI device: `SLRS` = 2.** XCKU115 is a stacked-silicon-interconnect part built from two super
  logic regions, not a monolithic die — unlike the smaller Kintex UltraScale parts. Any path that
  crosses between SLRs is a fundamentally more expensive path than an intra-SLR one and needs
  explicit pipelining/floorplanning attention; read UG949's SSI section before laying out a design
  that spans the device, and don't carry timing intuition from a monolithic part over to this one.
- **Do not take the FPGA description printed in a multi-populate board's schematic title block as a
  description of YOUR part.** On the HTG-830 the title block reads "FPGA, Virtex UltraScale, 52 HR,
  650 HP, 36 GTY, 40 GTH / XCVU160-FFVB2104" on every sheet — that is the Virtex populate option's
  device, not the XCKU115 actually fitted, and its GTY/GTH counts do not describe this part at all.
  (An earlier revision of this file quoted a "104 HR, 520 HP, 48 GTH" description for this part;
  that string appears nowhere in the schematic or the board UG and its GTH figure is contradicted by
  the part property above — it has been removed rather than left as an unverifiable claim.)

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
method used to produce every verified fact in this file), and
`rules/oem/amd_vivado/toolchain_gotchas.md` ("Asking what a part actually contains") for the
part-property queries these counts come from and the site-count wildcard trap they avoid.

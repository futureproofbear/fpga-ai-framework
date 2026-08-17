# Kintex UltraScale (first-generation, non-"+") transceiver architecture

**Kintex UltraScale has GTH transceivers only — zero GTY.** Verified against `xcku115-flvb2104-2-e`
in Vivado 2023.2: `get_property GTHE3_TRANSCEIVERS [get_parts xcku115-flvb2104-2-e]` = **64**, the
part carries no `GTY*` property at all, and after `link_design` `get_sites -filter {SITE_TYPE =~
*GTY*}` returns 0. (The `*GTH*` wildcard returns 80 sites — that is 64 `GTHE3_CHANNEL` plus 16
`GTHE3_COMMON`, one COMMON per quad; the transceiver count is 64, not 80. See
`rules/oem/amd_vivado/toolchain_gotchas.md`.) GTY was introduced in UltraScale+ (and in specific high-end Virtex
UltraScale (non-Plus) parts, e.g. XCVU160/XCVU190) — it is not present anywhere in the Kintex
UltraScale line. GTH on this family tops out well below GTY's line rate; do not plan a design around
GTY-class signaling (e.g. 25G-class serial links) on a Kintex UltraScale part.

## The pin-compatible multi-populate board trap

Several HiTech-Global-style (and other vendors') FPGA carrier boards share ONE footprint/PCB across
multiple populate options in the same package family (e.g. FFVB2104/FLVB2104), typically Virtex
UltraScale (XCVU160/XCVU190, which DO have GTY) alongside a Kintex UltraScale option (e.g. XCKU115,
which does NOT). The board's schematic is commonly drawn against ONE populate option and its net
names (e.g. `GTY_*`, a "Z-RAY" or similar GTY-only high-speed connector) do not change per populate
option — so a schematic net labeled `GTY_...` is not proof that pin is a live GTY connection on
every populated part.

**Confirmed on this project (HTG-830 / HTG-VKUS-PCIE board, populated with XCKU115-FLVB2104-2-E):**
of 84 pins wired to the board's Z-RAY connector + "free" GTY reference-clock nets, 54 report
`PIN_FUNC == NC` (No Connect) when queried against the actual populated part via
`get_package_pins`/`get_property PIN_FUNC` — including 3 of the Z-RAY connector's 4 lanes (B, C, D)
entirely, and 4 of 8 labeled "GTY" reference clocks. The 30 that DO connect resolve to real GTH
sites (e.g. `MGTHRXN0_127`), not GTY — the schematic's `GTY_*` net names on this populate option are
simply inherited from the Virtex UltraScale populate option's wiring and repurposed onto GTH where
the pins happen to still connect.

**Rule: before trusting ANY schematic net name that implies a specific transceiver type (GTH/GTY/
GTX/GTP) on a pin-compatible multi-populate board, verify against the ACTUAL target part:**
```tcl
# 1. which parts share this footprint at all (seconds, no design open)
get_property COMPATIBLE_PARTS [get_parts <exact_part>]
# 2. which transceiver types this part actually has (absence of a GTY* property = it has none)
list_property [get_parts <exact_part>]
# 3. what a specific ball is on THIS part (needs the device database loaded — minutes)
link_design -part <exact_part> -quiet
get_property PIN_FUNC [get_package_pins <ball>]
```
Step 1 is the cheap way to establish that you ARE on a multi-populate footprint before anything
else: on `xcku115-flvb2104-2-e`, `COMPATIBLE_PARTS` lists `xcvu095ffvb2104`, `xcvu125flvb2104`,
`xcvu160flgb2104`, `xcvu190flgb2104`, several UltraScale+ parts, and more — i.e. Vivado itself will
tell you which populate options a board vendor's one schematic may have been drawn against. Do not
take that list from the board vendor's marketing copy.
An `NC` result means the schematic's net name for that ball is not usable at all on this populate
option — do not write a constraint for it, and flag the connector/interface it belongs to as
partially or fully non-functional on this specific chip choice.

See also: `rules/oem/amd_vivado/toolchain_gotchas.md` (general Vivado facts), `device_notes.md`
(this family's other resource counts, verified the same way).

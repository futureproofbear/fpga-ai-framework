# Rule: verify against the reference before committing to a design or fix

Before integrating or debugging any hard IP block (a DMA engine, an FFT/DSP core, a CCC/PLL, an
interconnect), or before treating a silicon symptom as a design bug: pin the ACTUAL built
configuration (grep the generated parameter file / instantiation, not comments or memory), extract
the protocol from the vendor User Guide with exact quotes, diff the integration against the vendor's
own golden testbench, and only then rank root causes — explicitly marking which candidates were
RULED OUT and why.

**Why:** the most expensive bugs on FPGA/SoC projects are handshake/config mismatches against an IP
whose actual behavior nobody re-checked against its documented contract, and the golden testbench
usually reveals exactly which paths (re-arm, backpressure, 2nd transaction) it does NOT exercise —
which is exactly where silicon-only bugs live.

Full procedure + fan-out pattern: `.claude/skills/reference-first-verification/`. Agent that runs
this as a read-only gate: `.claude/agents/fpga-ref-verifier.md`.

## The extracted copy of a reference is not the reference

Machine-extracting a vendor PDF (`pdftotext`, a PDF-to-markdown converter, a doc-portal scrape) is
usually the only practical way to read a 300-page User Guide or a 100-sheet schematic. The
extraction is lossy in ways that are SILENT, and it produces output that reads as authoritative.

- **A multi-column table can come out with its row LABELS misaligned against its VALUES.** Text
  extractors group glyphs by baseline; when a table's label column and its numeric cells sit on
  slightly different baselines (common when the header row's part names share a baseline with the
  first row's label), every value lands against a neighbouring label. Nothing warns you — the output
  is a well-formed table of true numbers against wrong labels.
  Worked example (HTG-830 UG Rev2.0 Table (1), `pdftotext -layout`, verified 2026-08): the extracted
  rows read `GTH 16 Gb/s Transceivers ... 0` and `GTY 30.5 Gb/s Transceivers ... 64` in the XCKU115
  column. That part in fact has **64 GTH and zero GTY** (`get_property GTHE3_TRANSCEIVERS [get_parts
  xcku115-flvb2104-2-e]` = 64; the part carries no GTY property at all). Every value in the
  extraction was shifted up by exactly one row, so a naive read INVERTS GTH and GTY — and would have
  "corroborated" the board schematic's misleading `GTY_*` net names.
- **The offset is detectable, and it is uniform.** Check any two rows against a known-good source:
  in the same extraction the `DSP Slices` label carried 75.9 (the block-RAM megabit figure) while
  the part reports DSP = 5,520 — the value on the row below. A trailing orphan row with values but
  no label (here `1.0 - 3.3V`) is a strong tell that the whole column is off by one. Once one row is
  proven shifted, treat EVERY row of that table as corrupted, not just the one you caught.
- **Derived vendor exports drop characters as readily as PDFs misplace them.** A board pinout `.txt`
  used to generate constraints in one project truncated every differential clock net name to its
  base, dropping the trailing `_N`/`_P` — recoverable only by inferring the file's own pin-ordering
  convention, which stays a guess until it is spot-checked against the schematic.
- **A PDF text layer carries no geometry.** You can extract every net name and reference designator
  on a schematic sheet and still not know what connects to what: which end of an LED's series
  resistor goes to the rail — i.e. active-high vs active-low — is not recoverable from text at all.
  Settle that empirically on hardware (a blink/toggle design) or from a human read of the drawing;
  do not assert it from an extraction.

**Rule: before quoting a machine-extracted table or name list as fact, cross-check it against a
ground truth the TOOL produces** — the device database (`get_property` on a part/pin object, or the
vendor equivalent), the vendor's own datasheet, or hardware. When the extraction and the tool
disagree, the tool wins. This is the same discipline as the rest of this rule: the document is the
reference, and a lossy copy of it is not.

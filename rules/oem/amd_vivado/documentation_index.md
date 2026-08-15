# AMD/Xilinx documentation index — which document answers which question

The reference-first rule (`rules/generic/reference-first-verification.md`) is only actionable if you
know WHICH document to open. This is that map, plus the access mechanics, which are non-obvious
because AMD migrated most documentation to a JavaScript portal that cannot be scraped.

## Access mechanics (verified 2026-08)

| Method | Works? | Notes |
|---|---|---|
| `docs.amd.com/r/en-US/<doc-slug>` (HTML) | ✅ | **Readable by WebFetch** — converts to markdown. The reliable path for reading a UG/DS. Always current. |
| `docs.amd.com/v/u/en-US/<doc-slug>` (viewer) | ⚠️ | JS shell (~2.5 KB), no content. Fine for a human browser, useless programmatically. |
| `docs.amd.com/api/khub/documents/<opaque-id>/content` | ✅ | Direct PDF. **But the ID is an opaque hash with no resolver** — you can only discover it from a web-search result that happens to expose it. Not derivable from the doc number. |
| `xilinx.com/content/dam/.../ip_documentation/<ip>/<version>/<pg>.pdf` | ✅ | Direct PDF for **versioned IP product guides**. Still live. |
| `xilinx.com/content/dam/.../user_guides/<ug>.pdf` | ❌ | Redirects to the JS shell. The old unversioned UG/DS PDF paths are gone. |
| `amd.com/content/dam/xilinx/support/documents/user_guides/<ug>.pdf` | ❌ | 404, despite still appearing in search results. |

**Practical rule: to READ an architecture UG or a datasheet, WebFetch its `docs.amd.com/r/en-US/`
URL.** Download a local PDF only for an IP product guide (versioned path works) or when a search
result happens to hand you a khub ID.

## Layer 3 — architecture (which silicon resources exist and how they behave)

| Doc | Covers | Read when |
|---|---|---|
| **UG571** SelectIO Resources | I/O standards, VCCO/VREF, HP vs HR bank capabilities, DCI/termination, ODELAY/IDELAY | Setting `IOSTANDARD`, choosing a bank voltage, any single-ended/differential I/O decision |
| **UG572** Clocking Resources | MMCM/PLL, global/regional clock buffers and their routing, clock regions | Building a clock tree, deciding buffer types, diagnosing a clocking-architecture timing problem |
| **UG573** Memory Resources | Block RAM, built-in FIFO, ECC, UltraRAM | Sizing/inferring on-chip memory. **Note: UltraRAM is UltraScale+ only — see the chip-family tier before assuming it exists** |
| **UG574** CLB Architecture | LUTs, flip-flops, carry chain, distributed RAM/SRL | Understanding what a logic-bound critical path is actually built from |
| **UG579** DSP Slice (DSP48E2) | Multiplier/accumulator, pipeline registers, cascade paths, pre-adder | Any fixed-point arithmetic; register-packing for timing |
| **UG576** GTH Transceivers | **GTHE3 (UltraScale) / GTHE4 (UltraScale+)**, 500 Mb/s–16.375 Gb/s | Any serial link on a GTH-equipped part |
| **UG578** GTY Transceivers | GTY only | **UltraScale+ and select high-end Virtex UltraScale only.** Confirm your part HAS GTY before reading this — several families have none |
| **UG570** Configuration | Bitstream loading, config modes/pins, encryption | Boot/config bring-up, config-pin constraints |

**GTH vs GTY is a real trap**, not a naming detail: they are different primitives in different
documents, and a board schematic may label a net `GTY_*` on a part that has no GTY at all. Verify
against the part (`get_sites -filter {SITE_TYPE =~ *GTY*}`) before choosing which UG to read — see
`rules/target/amd/kintex_ultrascale/transceiver_rules.md` for a confirmed case of exactly this.

## Datasheets (the numbers, per family)

- **DS890** — UltraScale Architecture and Product Data Sheet: Overview. Family-wide resource
  counts; the fastest way to answer "does this family have UltraRAM / GTY / how many DSPs."
- **DS892** — Kintex UltraScale FPGAs Data Sheet: DC and AC Switching Characteristics. Speed
  grades (-3/-2/-1/-1L), VCCINT options, temperature ranges, switching characteristics.
  (Equivalents exist per family: DS893 Virtex UltraScale, DS922 Kintex UltraScale+, etc.)

## Layer 2 — Vivado tool documentation

| Doc | Covers | Read when |
|---|---|---|
| **UG949** UltraFast Design Methodology | The end-to-end recommended methodology: RTL structure, constraints strategy, timing closure order | Starting a design; deciding methodology, not syntax. The single highest-value tool doc |
| **UG903** Using Constraints | XDC semantics, ordering, scoping, exceptions (false path / multicycle / max delay) | Writing or debugging any constraint |
| **UG906** Design Analysis and Closure Techniques | Reading timing reports, congestion/utilization analysis, closure strategies | A path won't close and you need to know why before changing RTL |
| **UG908** Programming and Debugging | ILA/VIO insertion, Hardware Manager, `.ltx` probes, programming flows | Any on-hardware debug session |
| **UG835** Tcl Command Reference | Every Tcl command and its options | Writing build/debug scripts — do not guess an option |
| **UG894** Using Tcl Scripting | Tcl scripting concepts and the object model (`get_*`/`set_property` semantics) | Same, for the conceptual model rather than the command list |
| **UG901** Synthesis | Synthesis attributes/directives, inference rules (RAM/DSP/FSM) | An inference didn't happen (BRAM became LUTs, a DSP wasn't used) |
| **UG900** Logic Simulation | Simulator setup, mixed-language, IP simulation models | Bench infrastructure problems |
| **UG994** Designing IP Subsystems (IP Integrator) | Block design, IP packaging, board flow | Working in IP Integrator rather than pure RTL |
| **UG1399** Vitis HLS | HLS pragmas, interfaces, dataflow | Any HLS kernel — see also `.claude/skills/vitis-hls-kernel-authoring/` |

## IP product guides (direct PDF still available, versioned paths)

| Doc | IP | Direct PDF path (under `xilinx.com/content/dam/xilinx/support/documents/ip_documentation/`) |
|---|---|---|
| **PG150** | UltraScale Memory IP (DDR4/DDR3) | `ultrascale_memory_ip/v1_4/pg150-ultrascale-memory-ip.pdf` |
| **PG156** | UltraScale Gen3 Integrated Block for PCIe | `pcie3_ultrascale/v4_4/pg156-ultrascale-pcie-gen3.pdf` |
| **PG182** | UltraScale FPGAs Transceivers Wizard | `gtwizard_ultrascale/v1_7/pg182-gtwizard-ultrascale.pdf` |
| PG213 | UltraScale+ Integrated Block for PCIe | `pcie4_uscale_plus/v1_3/pg213-pcie4-ultrascale-plus.pdf` (UltraScale+ only) |

Version segments in these paths change as IP revs; if a path 404s, search for the doc number and
take the newest versioned path from the result.

## Discipline

- **Quote the version.** These documents revise (DS892 was at v1.20 in Aug 2025; UG573 at rev 1.14
  in Nov 2025). A protocol/parameter fact cited without its document revision is not verifiable
  later — record doc number, revision, and section.
- **Read the variant that is actually built.** UGs cover multiple generations in one document
  (UG576 covers both GTHE3 and GTHE4); quoting the wrong generation's timing or feature set is a
  common and expensive error.
- **A product guide's own golden testbench and example design ship with the generated IP**, not in
  the PDF — read both (see `rules/generic/reference-first-verification.md`).

See also: `toolchain_gotchas.md`, `timing_constraints.md`, `ip_catalog_rules.md`, and the chip
family tier under `rules/target/amd/<family>/` for which of the above actually apply to your part.

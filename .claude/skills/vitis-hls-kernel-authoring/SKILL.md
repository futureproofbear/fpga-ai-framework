---
name: vitis-hls-kernel-authoring
description: >-
  Vitis HLS-specific technical reference for authoring AMD/Xilinx FPGA fabric kernels: pragma
  syntax (INTERFACE, PIPELINE, UNROLL, DATAFLOW, ARRAY_PARTITION), the AXI interface options
  (m_axi/s_axilite/axis), the pin-don't-infer discipline as applied to Vitis HLS, and where to find
  the authoritative docs for the installed version. Load BEFORE writing or changing any Vitis HLS
  kernel or pragma. Complements the vendor-agnostic hls-output-distrust methodology
  (`rules/generic/hls-output-distrust.md`) with the AMD-tool specifics. Triggers: "Vitis HLS",
  "vitis_hls", "which HLS pragma", "m_axi / s_axilite", "HLS memory partition", "HLS mis-synthesis",
  "dataflow / task-level pipelining".
---

# Vitis HLS kernel authoring

This skill is the Vitis-HLS-tool-specific counterpart to the vendor-agnostic `hls-output-distrust`
rule. Load that for the *why* and the gate sequence; this one is the *how* for AMD's Vitis HLS
compiler specifically — pragma syntax and the AXI interface options.

## Rule: read the Vitis HLS reference before writing or asserting a pragma exists

- **TRIGGER**: about to add, change, or reason about any `#pragma HLS` / `hls::` construct, or to
  claim Vitis HLS can/cannot do something.
- **ACTION**: check the AMD UG1399 (Vitis HLS user guide) for the installed release before assuming
  a pragma's exact syntax or semantics — pragma behavior and defaults have changed across Vitis
  releases (e.g. default `INTERFACE` mode, `DATAFLOW` canonical-form requirements).
- **HALT**: if you cannot cite the option in the installed version's own documentation, do not
  assert it exists and do not plan a kernel architecture around it.

## Pragma quick reference (syntax — verify against the installed version's own manual)

```
#pragma HLS INTERFACE mode=m_axi port=<arg> bundle=<name> depth=<n> offset=slave
#pragma HLS INTERFACE mode=s_axilite port=<arg> bundle=control
#pragma HLS INTERFACE mode=axis port=<arg>
#pragma HLS PIPELINE II=<n>
#pragma HLS UNROLL factor=<n>
#pragma HLS DATAFLOW
#pragma HLS ARRAY_PARTITION variable=<v> type=cyclic|block|complete factor=<n> dim=<d>
#pragma HLS ARRAY_RESHAPE variable=<v> type=cyclic|block|complete factor=<n> dim=<d>
#pragma HLS STREAM variable=<v> depth=<n>
```

## Pin Class-A behaviour with an explicit pragma, never rely on inference

- **TRIGGER**: a kernel depends on a specific memory architecture, achieved II, or port count for
  correctness or performance.
- **ACTION**: state it as an explicit `ARRAY_PARTITION`/`ARRAY_RESHAPE` and a `PIPELINE II=<n>`
  target, then check the achieved II in the synthesis report rather than assuming the requested II
  was met — Vitis HLS reports both, and a silent II degradation (e.g. from an unresolved memory
  dependence) is exactly the kind of Class-A regression `hls-output-distrust`'s report-gate is for.

## `m_axi` burst/outstanding-transaction tuning before restructuring

- **TRIGGER**: a kernel's silicon/co-sim time exceeds its scheduled cycle count and the memory
  interface is not bandwidth-saturated (compute achieved MB/s before assuming bandwidth is the
  limit).
- **ACTION**: tune `num_read_outstanding` / `num_write_outstanding` / `max_read_burst_length` /
  `max_write_burst_length` on the `m_axi` INTERFACE pragma before restructuring the kernel's
  algorithm — these are documented, cheap to try, and are frequently the actual lever for a
  latency-bound kernel rather than a compute-bound one.

## `DATAFLOW` canonical-form requirement

`#pragma HLS DATAFLOW` requires the region's data to move strictly forward through function
calls/loops with no feedback, no conditional bypass of a stage, and no shared single-producer/
single-consumer violation — Vitis HLS's canonical-form checker will refuse or silently serialize a
region that violates this rather than always giving a clear error. If a `DATAFLOW` region does not
achieve the expected task-level overlap, check the canonical-form warnings in the synthesis log
before assuming the pragma itself failed to apply.

## `s_axilite` vs `m_axi` — pick deliberately

- **`s_axilite`** — a control/status register interface (scalar arguments, start/done/idle bits).
  Not for bulk data movement.
- **`m_axi`** — a full AXI4 master interface for bulk DDR/memory-mapped access, burst-inferred from
  the array-access pattern (or explicitly driven via `hls::burst_maxi` for finer control). Multiple
  arguments can share one `bundle` (serializing their traffic onto one port) or be split across
  bundles for concurrent access — decide based on whether the kernel needs genuine concurrent
  read/write, the same tradeoff other vendors' pointer-based-vs-explicit AXI initiator APIs make.

## No project-specific mis-synthesis catalog yet

Unlike the Microchip SmartHLS skill (which carries specific, hardware-confirmed mis-synthesis
classes from a real project's history — e.g. a signed-narrowing-cast-after-shift bug), this skill
has no equivalent catalog for Vitis HLS yet. Add entries here, with the exact pattern and how it was
confirmed (silicon value-check, not just cosim), as they are found on real hardware — do not invent
plausible-sounding "known issues" in the meantime.

See also: `rules/generic/hls-output-distrust.md` (the gate sequence and Class-A/Class-B framing
this skill's pragmas plug into), `rules/generic/value-level-verification.md` (the board-free value
gate).

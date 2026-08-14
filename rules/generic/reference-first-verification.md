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

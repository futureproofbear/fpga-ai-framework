# MPFS250T_ES engineering-sample errata

Facts specific to the **exact die/package** PolarFire SoC **MPFS250T, FCVG484EES, ES revision 1**
(an engineering sample, not production silicon). Everything here is scoped to ER0219 (the
Microchip/Microsemi "PolarFire SoC FPGA Engineering Samples" errata document) or to an ES-only
operating limit. This is narrower than `rules/oem/microchip_libero/toolchain_gotchas.md`
(family-general PolarFire SoC toolchain/IP behavior) — check that file first for anything not
listed here; it deliberately does not duplicate die-specific errata.

## How to use
1. Check whether the symptom's domain appears below (SmartDebug read, MPU, eNVM programming, MSS
   clock, SPI-flash boot, DDR/transceiver bring-up).
2. If it does, the workaround below is a known ES erratum, not a design bug.
3. If it doesn't, this is not necessarily ruled out — it may be family-general (check
   `toolchain_gotchas.md`) or a genuine design bug.
4. **Porting note**: every workaround below is ES-specific and ER0219 states it is "fixed in
   production silicon." A design depending on one of these workarounds is not guaranteed to port
   cleanly to a production MPFS250T — retarget the die, regenerate the MSS configurator + bitstream,
   and re-verify.

Source: Microchip/Microsemi "PolarFire SoC FPGA Engineering Samples" errata, document ER0219 v1.

## ES-only operating limits (ER0219 §2.1/2.2)
- Junction temperature **20 °C to 50 °C** for both programming/erase and operation.
- Core **VDD = 1.0 V ± 0.03 V**; **1.05 V core is NOT supported on ES silicon** (VDDA supports both).
- Device marking: "ES" appears in the temperature-grade field of the part marking.

## Errata relevant to fabric/DDR/JTAG bring-up

### ER0219 §3.8 — Fabric APB DRI write corrupts a concurrent SmartDebug JTAG/SPI read (read returns ZERO)
A fabric DRI write to a PCIESS APB config block can corrupt a concurrent SmartDebug JTAG/SPI read;
the read comes back as **zero**, not an error. Documented workaround: **redo the SmartDebug read
until the expected data is received.** This is a DIE-SPECIFIC cause of bad SmartDebug reads,
distinct from (and in addition to) the family-general "design database must match the programmed
bitstream" trap in `toolchain_gotchas.md` — rule out both before trusting a probe.

### ER0219 §3.2 — AXI Switch Memory Protection Unit (MPU) is not operational
ES silicon has an AXI-bus bug triggered when the MPU rejects an illegal message, so the MPU is
disabled by startup firmware and gives no access warnings or interrupts. Keep every fabric AXI
master's address range strictly in-bounds by construction — illegal accesses will not be caught.

### ER0219 §3.11 — Auto-program / auto-update of eNVM must NOT be used
Boot-initiated auto-program/auto-update of eNVM **fails** on ES silicon. Program eNVM via an
explicit tool flow instead (e.g. Microchip's boot-mode programmer utility, boot mode 1).

### ER0219 §3.4 — MSS CPU frequency ceiling: 625 MHz (600 MHz if eMMC/SD is used)
If the design uses eMMC or SD, the ceiling drops to 600 MHz, and only a specific tabulated set of
frequencies is permitted for eMMC/SD or CAN use.

### ER0219 §3.1 — MSS cannot access the System Controller's external SPI flash
The DRI-to-SPI RXFIFO path returns bad data on this die.

### ER0219 §4 — Transceivers + DDR interfaces are not fully validated on ES
Treat occasional DDR-training flakiness after a reflash as consistent with this not-fully-validated
status (a clean power-cycle is a reasonable first recovery step) rather than assuming a design bug.

## Other ES errata in ER0219 (cataloged for completeness)
- §3.3 MSS I2C requires core revision ≥ 2.0.108.
- §3.5 / §3.6 DRI interrupt line, and DRI Error+Fault maintenance-interrupt, issues.
- §3.7 MSS GPIO must be reset via the CPU, not the fabric (`soft_reset_select` must not be 0).
- §3.9 System Controller suspend mode is unsupported on ES.
- §3.10 GEM (1 Gbps) half-duplex undersize-frame counter issue.
- §3.12 Auto-update SPI master/slave contention (only relevant if using eNVM auto-update).

Each is a genuine ES-only erratum per ER0219; consult the errata document directly for the exact
mechanism if a design touches that peripheral.

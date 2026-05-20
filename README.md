# MicroBlaze + DDR3 RAM on Arty A7-100T
**Platform:** Digilent Arty A7-100T (Xilinx Artix-7 FPGA)  
**Tools:** Vivado 2025.1 · Vitis Unified IDE 2025.1  
**Language:** Verilog (Block Design) · C (Application)

---

## Overview

This project implements a fully functional **MicroBlaze soft-core processor** connected to **DDR3 external RAM** on the Digilent Arty A7-100T FPGA development board. The design was built from scratch in Vivado using IP Integrator (Block Design), without following any tutorial blindly — each design decision was understood before implementation.

The project includes a **DDR3 memory stress test application** that validates data integrity and measures memory read throughput — running entirely from DDR3 at 325 MHz clock speed.

---

## Block Design Architecture

```
CLK100MHZ (E3, LVCMOS33)
    │
    ▼
Utility Buffer (BUFG)
    │
    ├──► MIG sys_clk_i (100MHz)
    └──► Clocking Wizard (clk_in1)
              ├──► clk_out1 (200MHz) ──► MicroBlaze, AXI Interconnects, GPIO, UART
              └──► clk_out2 (200MHz) ──► MIG clk_ref_i (IDELAY calibration)

MIG ui_clk (81.25MHz) ──► ram_interconnect M00, rst_mig_7series_0_81M

Reset Chain:
ck_rst (C2, active-low) ──► Clocking Wizard resetn + MIG sys_rst
rst_clk_wiz_0_200M/peripheral_aresetn   ──► 200MHz domain resets
rst_mig_7series_0_81M/peripheral_aresetn ──► MIG aresetn + ram_interconnect M00_ARESETN
```

---

## IP Components Used

| IP | Role | Clock Domain |
|---|---|---|
| MIG 7 Series | DDR3 memory controller | 100MHz in, 325MHz DDR3, 81.25MHz ui_clk out |
| Clocking Wizard | 100MHz → 200MHz (×2 outputs) | 100MHz in |
| Utility Buffer (BUFG) | Clock buffering, bank isolation | — |
| MicroBlaze (Classic) | Soft-core processor | 200MHz |
| AXI UART Lite | Serial communication | 200MHz |
| AXI GPIO | Digital output (ck_a0) | 200MHz |
| AXI Interconnect (ram) | MicroBlaze ↔ DDR3, 2S/1M | 200MHz slave / 81.25MHz master |
| AXI Interconnect (perif) | MicroBlaze ↔ peripherals, 1S/2M | 200MHz |
| Processor System Reset ×2 | Synchronized reset generation | 200MHz and 81.25MHz |
| MDM | JTAG debug | — |

---

## Key Design Decisions & Why

### 1. BUFG on System Clock
Connecting `CLK100MHZ` directly to MIG `sys_clk_i` causes a **DRC BIVC-1** error — bank 35 voltage conflict between the 3.3V oscillator pin (E3) and MIG's expected 2.5V. Adding a BUFG re-routes the clock internally, resolving the conflict.

### 2. Two 200MHz Clocking Wizard Outputs
A single 200MHz clock shared between MIG reference and MicroBlaze caused cache failures in testing. Two separate 200MHz outputs — one for MIG reference (`clk_out2`), one for the processor domain (`clk_out1`) — ensures stable operation with caches enabled.

### 3. DDR3 Clock at 325MHz (3077ps)
With a 100MHz input clock, only specific PLL multiplication ratios are supported by MIG. The closest valid DDR3 frequency is **325MHz** — slightly below the chip's 333MHz maximum, but with negligible performance difference (<0.5% measured).

### 4. AXI Interconnect over AXI SmartConnect
AXI SmartConnect (Vivado's default automation choice) introduces ~10-14% memory throughput penalty in this design. Classic AXI Interconnect is used manually for better DDR3 read performance.

### 5. MIG aresetn Must Come from ui_clk Domain
The MIG's `aresetn` input must be synchronized to `ui_clk` (81.25MHz). A dedicated `rst_mig_7series_0_81M` Processor System Reset block provides this — connecting `aresetn_0` (asynchronous bare port) directly causes CDC violations and metastability.

---

## MicroBlaze Cache Configuration

| Cache | Size | Line Length | Victims |
|---|---|---|---|
| Instruction | 16KB | 8 | 8 |
| Data | 32KB | 8 | 8 |

Cache dramatically improves DDR3 access speed — a 30KB array reads in ~88 microseconds from cache vs ~7.24 milliseconds without.

---

## Pin Assignments (XDC)

| Signal | Pin | Standard | Description |
|---|---|---|---|
| CLK100MHZ | E3 | LVCMOS33 | Onboard 100MHz oscillator |
| ck_rst | C2 | LVCMOS33 | ChipKit reset (active-low) |
| ck_a0 | D9 | LVCMOS33 | GPIO output |
| init_calib_complete | H5 | LVCMOS33 | MIG calibration indicator (LED0) |

---

## DDR3 Memory Test Application

The C application (`src/main.c`) performs a full DDR3 read/write verification:

1. **Pattern A write** — fills 16MB DDR3 region with `0xA5A50000 ^ (i * 0x10203041)`
2. **Cache flush + invalidate** — forces data to physical DDR3 memory
3. **Pattern A verify** — reads back and checks every word
4. **Pattern B write + verify** — repeats with alternate pattern
5. **Read sweep** — XOR accumulates all words, outputs checksum
6. Reports `DDR3 test completed successfully` on UART

UART output visible via PuTTY — 9600 baud, COM port from Device Manager.

---

## Repository Structure

```
├── src/
│   └── main.c
├── constraints/
│   └── microblaze_ddr_constraint.xdc
├── pictures/
│   ├── final_block_design.png
│   ├── clocking_chain.png
│   ├── mig_config.png
│   └── putty_output.png
└── README.md
```

---

## How to Reproduce

1. Open Vivado 2025.1 → Create RTL Project → Select **Arty A7-100T** board
2. Create Block Design named `system`
3. Add and configure IPs as described above
4. Add XDC constraints file
5. Generate Bitstream → Export Hardware (include bitstream)
6. Open Vitis 2025.1 → Create Platform Component from `.xsa`
7. Create Embedded Application → Add `src/main.c`
8. Build → Run
9. Open PuTTY at 9600 baud to see output

> **Note:** Keep `ck_rst` HIGH (SW0 ON) before running — active-low reset.

---

## References

- [Viktor Nikolov's MicroBlaze DDR3 Tutorial](https://github.com/viktor-nikolov/MicroBlaze-DDR3-tutorial)
- [UG586 — 7 Series MIS User Guide](https://docs.amd.com/r/en-US/ug586_7Series_MIS)
- [UG984 — MicroBlaze Processor Reference Guide](https://docs.amd.com/r/en-US/ug984-vivado-microblaze-ref)
- [Arty A7 Reference Manual](https://digilent.com/reference/programmable-logic/arty-a7/reference-manual)

---

*Built as part of FPGA/VLSI learning path targeting embedded hardware design at the system level.*

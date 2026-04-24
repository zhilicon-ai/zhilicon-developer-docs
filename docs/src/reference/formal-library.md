# Formal-property library

Source:
[`src/tests/rtl/formal/`](../../../src/tests/rtl/formal/) —
bind-target SystemVerilog assertions for every shared-IP block and every
chip top-level module.

## Shared-IP properties

| File                                                                         | DUT                          | Property count |
| ---------------------------------------------------------------------------- | ---------------------------- | -------------- |
| [`zhi_async_fifo_props.sv`](../../../src/tests/rtl/formal/zhi_async_fifo_props.sv)       | `zhi_async_fifo`             | 11             |
| [`zhi_cdc_sync_props.sv`](../../../src/tests/rtl/formal/zhi_cdc_sync_props.sv)           | `zhi_cdc_sync`               | 5              |
| [`zhi_icg_props.sv`](../../../src/tests/rtl/formal/zhi_icg_props.sv)                     | `zhi_icg`                    | 5              |
| [`zhi_pmu_counter_props.sv`](../../../src/tests/rtl/formal/zhi_pmu_counter_props.sv)     | `zhi_pmu_counter`            | 8              |
| [`zhi_interrupt_ctrl_props.sv`](../../../src/tests/rtl/formal/zhi_interrupt_ctrl_props.sv) | `zhi_interrupt_ctrl`         | 6 × N          |
| [`zhi_watchdog_props.sv`](../../../src/tests/rtl/formal/zhi_watchdog_props.sv)           | `zhi_watchdog`               | 7              |

## Per-chip top-level properties

| File                                                                     | DUT                          | Property count | Covers |
| ------------------------------------------------------------------------ | ---------------------------- | -------------- | ------ |
| [`d1_top_props.sv`](../../../src/tests/rtl/formal/d1_top_props.sv)       | `d1_top`                     | 5              | Boot, thermal throttle, IRQ latching |
| [`h1_top_props.sv`](../../../src/tests/rtl/formal/h1_top_props.sv)       | `h1_top`                     | 7              | SEU FSM, SAFE_MODE, TMR-mismatch→SEU |
| [`n1_top_props.sv`](../../../src/tests/rtl/formal/n1_top_props.sv)       | `n1_top`                     | 4              | RF chain ordering, high-QAM/DPD gate |
| [`p1_top_props.sv`](../../../src/tests/rtl/formal/p1_top_props.sv)       | `p1_chiplet_top`             | 6              | Chiplet reset / linking / ZLink / HBM |
| [`s1_top_props.sv`](../../../src/tests/rtl/formal/s1_top_props.sv)       | `s1_top`                     | 6              | FIPS boundary, TEE mutex, TRNG, key lifecycle |

## Running the library

A common TCL driver in
[`src/tests/rtl/formal/tcl/jasper-common.tcl`](../../../src/tests/rtl/formal/tcl/jasper-common.tcl)
orchestrates every property module. Wrapper drivers per DUT:

```bash
jg -fpv src/tests/rtl/formal/tcl/jasper-zhi_async_fifo.tcl
jg -fpv src/tests/rtl/formal/tcl/jasper-d1_top.tcl
# …one per module
```

Reports land in `reports/formal/<block>/`. The driver exits non-zero if
any assertion is unproven.

## CI

On hosted runners the
[`formal-regression` workflow](../../../.github/workflows/formal-regression.yml)
runs a structural-only sanity pass (Verilator `--lint-only` on every
property file + TCL-driver reference-path check). The full JasperGold
regression runs on Zhilicon-managed self-hosted runners with commercial
licences.

## Adding a new property module

1. Create `<prefix>_props.sv` under `src/tests/rtl/formal/` following the
   naming convention `<PREFIX>_<CATEGORY>_<ID>` for each property.
2. Add a wrapper driver under `tcl/jasper-<dut>.tcl` that sources
   `jasper-common.tcl`.
3. Register the module and its property count in the README table and in
   this page.
4. Add the module identifier to the `jasper` matrix in the
   [formal-regression workflow](../../../.github/workflows/formal-regression.yml).

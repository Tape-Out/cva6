# cva6

OpenHW's CVA6, the application-class RISC-V core, taken as a black box.

![maturity](https://img.shields.io/badge/maturity-planned-lightgrey) ![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0%20OR%20MulanPSL--2.0-blue) ![upstream](https://img.shields.io/badge/upstream-SHL--0.51-lightgrey)

Part of the [Tape-Out](https://github.com/Tape-Out) IP library, wired up by
[`xirang`](https://github.com/Tape-Out/xirang). The core is a submodule at
`third_party/cva6`; nothing in it is modified. The file list is upstream's own
`core/Flist.cva6`, nested `-F` and `+incdir+` included.

## What this repository adds

CVA6 is not configured by a handful of scalar parameters. Its whole configuration is one
struct, `config_pkg::cva6_cfg_t`, 17270 bits wide, and you pick a variant by choosing which
`core/include/<variant>_config_pkg.sv` gets compiled — upstream calls that `TARGET_CFG`.

So the knob here selects a source file rather than a parameter value:

```yaml
params:
  cfg:
    type: choice
    values: [cv32a60x, cv32a65x, …, cv64a6_imafdch_sv39_wb]
    default: cv64a6_imafdc_sv39
```

and the manifest gates the twelve config packages on it:

```yaml
rtl:
- path: third_party/cva6/core/include/cv32a60x_config_pkg.sv
  when: {cfg: cv32a60x}
```

Twelve variants, each elaborated on its own: 32-bit and 64-bit, with and without FPU,
hypervisor, write-back cache, HPDcache, and the OpenPiton L1.5 adapter.

## Testing

No upstream test runs here. CVA6's own regressions need a RISC-V toolchain, Verilator and
Spike built from source; none of that fits a per-commit gate. What does run is the elaboration
of every variant and the receipt that compares the declared ports against the elaborated ones.

## Limits

The vector variant `cv64a6_imafdcv_sv39` does not compile in upstream's own tree — its
`acc_mmu_resp_t` is `logic` outside the V configuration while `load_store_unit.sv` accesses
its members — so it is not in the knob's domain.

Memory goes out as one packed struct (`noc_req_o` / `noc_resp_i`), not as AXI wires; the
co-processor interface (`cvxif_req_o` / `cvxif_resp_i`) and the RVFI probes are structs too.
Turning those into wires belongs outside this repository.

`AccCfg` is an `acc_cfg_t` struct and stays at `'0`.

Baking to plain Verilog does not work yet: sv2v 0.0.13 stops on a nested struct assignment in
`acc_dispatcher.sv`. Elaboration and the receipt are unaffected.

## License

This repository: 任选其一 [MIT](LICENSE-MIT) · [Apache 2.0](LICENSE-APACHE) ·
[木兰宽松许可证 第2版](LICENSE-MULAN). `third_party/cva6` stays **SHL-0.51**, which is not an
OSI-approved license.

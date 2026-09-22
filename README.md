# cva6

OpenHW's CVA6, the application-class RISC-V core, taken as a black box.

![maturity](https://img.shields.io/badge/maturity-planned-lightgrey) ![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0%20OR%20MulanPSL--2.0-blue) ![upstream](https://img.shields.io/badge/upstream-SHL--0.51-lightgrey)

Part of the [Tape-Out](https://github.com/Tape-Out) IP library, wired up by
[`xirang`](https://github.com/Tape-Out/xirang). The core is a submodule at
`third_party/cva6`; nothing in it is modified. The file list is upstream's own
`core/Flist.cva6`, nested `-F` and `+incdir+` included.

## What this repository adds

CVA6 is not configured by scalar parameters. Its whole configuration is one struct,
`config_pkg::cva6_cfg_t`, built from 49 `localparam`s in a per-variant package; you pick a
variant by choosing which `core/include/<variant>_config_pkg.sv` gets compiled — upstream
calls that `TARGET_CFG`. There is **no macro anywhere** in those thirteen files, so `-D`
cannot reach any of it.

So there are two kinds of knob here. The first selects a source file:

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

The second kind is a **fourteenth variant that xirang writes**. `cfg: xirang` takes one of
upstream's packages as a template and rewrites only the fields declared in the manifest —
everything else keeps the upstream value, so a field added upstream is never silently
dropped:

```yaml
generate:
- out: gen/cva6_config_pkg.sv
  from: third_party/cva6/core/include/cv64a6_imafdc_sv39_config_pkg.sv
  when: {cfg: xirang, xlen: 64}
  set: {dcacheByteSize: CVA6ConfigDcacheByteSize, btbEntries: CVA6ConfigBTBEntries, …}
```

That turns 27 of those `localparam`s into real knobs: cache sizes, associativity and line
width for both caches, AXI widths, BTB/BHT/RAS depths, scoreboard and load-buffer entries,
PMP entries, and the extension switches. `xlen` picks which template is used, because word
length runs through the whole configuration and is not a field you can flip on its own.

Every name in `set:` must match exactly once in the template. That check earns its keep: the
32-bit package has no `CVA6ConfigBExtEn`, and the mismatch was caught the first time it ran
instead of quietly changing nothing.

The matrix is **63 points** and every one of them elaborates.

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

Baking to plain Verilog does not work. sv2v 0.0.13 blows up on this design: it reaches 6 GB of
memory in four minutes and produces a 500 MB intermediate, whichever way the simulation-only
regions and the `VERILATOR` / `XSIM` guards are arranged. The same path bakes Ibex and
CV32E40P byte-for-byte reproducibly, so this is specific to CVA6 and not yet diagnosed.
Elaboration of all twelve variants and the declaration receipt are unaffected.

## License

This repository: 任选其一 [MIT](LICENSE-MIT) · [Apache 2.0](LICENSE-APACHE) ·
[木兰宽松许可证 第2版](LICENSE-MULAN). `third_party/cva6` stays **SHL-0.51**, which is not an
OSI-approved license.

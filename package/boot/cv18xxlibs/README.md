# cv18xxlibs — CV18xx FSBL/BL2 (built from source)

This package builds the **First Stage Boot Loader (FSBL / BL2)** for the Milk-V
Duo family from source, for both CV18xx chip families, and stages the resulting
`bl2.bin` blobs so `target/linux/cv18x0/image/Makefile` can pack them into each
board's `fip.bin` with `fiptool`.

## What the FSBL is

The FSBL is **CVITEK/Sophgo's fork of ARM Trusted Firmware (TF-A)** — its README
states it "acts as ATF BL2", and the source carries the TF-A layout
(`plat/`, `make_helpers/`, `include/bl_common.h`, ARM Limited copyright headers in
the shared code). The SoC mask ROM loads it out of the FIP; it brings up DDR
(using the compiled-in DDR config) and hands off to OpenSBI → U-Boot.

## Origin / provenance

| | |
|---|---|
| Source tree | `duo-buildroot-sdk-v2/fsbl` |
| Repo | https://github.com/milkv-duo/duo-buildroot-sdk-v2 |
| Commit | `6f8962c394dd0a05729abb089f0feb7d5cc4aa5e` |
| Upstream lineage | ARM Trusted Firmware (BSD-3-Clause) + CVITEK/Sophgo CV18xx additions |
| Toolchain | OpenWrt `riscv64-openwrt-linux-musl-gcc` (no vendor T-Head GCC) |

We fetch the whole `duo-buildroot-sdk-v2` monorepo but build **only** its
`fsbl/` subtree. This SDK is the only tree that carries the full CV181x FSBL
source (DDR/eMMC/UART/security) plus first-class board configs for the SG2000
(Duo S) and SG2002 (Duo 256M); the older `milkv-duo-buildroot-libraries` repo the
package previously used is frozen, has no Duo 256M / Duo S board, and ships the
CV181x DDR bring-up only as prebuilt object blobs.

## What is built

Two BL2 blobs, one per chip family:

| Chip family | Boards | DDR_CFG | Staged as |
|---|---|---|---|
| `cv180x` | Milk-V Duo (CV1800B, 64 MiB DDR2) | `ddr2_1333_x16` | `bl2_cv1800b_milkv_duo_sd.bin` |
| `cv181x` | Duo S (SG2000) + Duo 256M (SG2002), DDR3 | `ddr3_1866_x16` | `bl2_cv181x_milkv_sd.bin` |

The CV181x BL2 is shared by the Duo S and Duo 256M: the FSBL auto-detects the
DDR3 density (256 vs 512 MiB) from package strapping, so one build serves both.

Each board's memory map is generated into `fsbl/plat/<chip>/include/cvi_board_memmap.h`
with the SDK's own `build/scripts/mmap_conv.py` immediately before the BL2
compile (see `Build/BuildBL2` in the Makefile).

## Local patch

`patches/100-cv18xx-fsbl-portability.patch` makes the vendor FSBL build with a
**mainline rv64 GCC** instead of the vendor's T-Head toolchain. It:

- drops the T-Head vendor arch (`-march=…vxthead` → `rv64imafdc_zicsr_zifencei`),
- replaces T-Head custom CSR mnemonics (`mxstatus`/`mhcr`/`mcor`/`mhint`) with
  their numeric encodings (`0x7c0`/`0x7c1`/`0x7c2`/`0x7c5`), and
- replaces T-Head cache ops (`icache.iall`, `sync.i`) with raw `.long` opcodes.

It only touches `fsbl/lib/cpu/riscv/*` (shared C906 code), so it covers both the
cv180x and cv181x builds.

## Known issue: CV1800B (Duo) BL2 hangs at the OpenSBI handoff

**Status (2026-07-20): the Duo (CV1800B, cv180x) does not boot with this
package's from-source BL2. The Duo S (SG2000) and Duo 256M (SG2002), which use
the cv181x BL2, boot to the Linux console.**

Symptom: on real CV1800B hardware the board runs the FSBL through DDR init
(`DDR2-512M`, `DDR BIST PASS`) and loads OpenSBI + U-Boot into DRAM, then
freezes immediately at the FSBL → OpenSBI transfer — the last line is
`Jump to monitor at 0x80000000. OPENSBI: next_addr=0x80200000 arg1=...` and no
OpenSBI banner ever prints.

What has been ruled out (so the fault is isolated to the cv180x BL2 build):

- **OpenSBI** — the built `fw_dynamic-generic.bin` region inside the Duo fip is
  **byte-identical** to the known-good prebuilt firmware, and the same OpenSBI
  binary boots the Duo S. OpenSBI uses its **embedded** FDT (it ignores the
  FSBL's `arg1`), so the per-board `OPENSBI_FDT_ADDR` mismatch is a red herring.
- **The embedded FDT** — declares `memory@0x80000000` size `0x3f40000`
  (**63.75 MiB** = 64 MiB minus the 768 KiB FreeRTOS reserve), which is correct
  for the Duo. Nothing over-claims RAM (`DDR2-512M` is the DDR device density,
  512 **megabit** = 64 MByte).
- **DDR** — `DDR_CFG=ddr2_1333_x16` is the value the sdk-v2 board `config.json`
  lists for this board, and DDR BIST passes.

The Duo's previously-working firmware used a cv180x BL2 built from the older
`milkv-duo-buildroot-libraries` source (~44 KB, needs no portability patch); the
sdk-v2 cv180x BL2 (~50 KB) is a different, larger build that mis-hands-off on
CV1800B silicon. Root cause inside the sdk-v2 cv180x FSBL is not yet pinned down
(would need serial-level debugging on the board).

Candidate fixes (not yet applied — work kept as-is pending review):

1. Build the **cv180x** BL2 from `milkv-duo-buildroot-libraries` (proven) and
   keep the **cv181x** BL2 from sdk-v2 — a two-source hybrid.
2. Debug the sdk-v2 cv180x BL2 handoff on hardware (DDR_CFG variants,
   cache-flush before the monitor jump, plat/cv180x specifics).

## Notes

- The FSBL embeds a build timestamp + git hash in its version string, so a
  rebuild is **functionally equivalent** but not byte-identical to a prior build.
- `PKG_MIRROR_HASH` is currently `skip`: the sdk-v2 monorepo is multi-GB and has
  not been pinned to the OpenWrt download mirror. Run
  `make package/boot/cv18xxlibs/download` once and replace `skip` with the printed
  hash to lock the source down for reproducible clones.

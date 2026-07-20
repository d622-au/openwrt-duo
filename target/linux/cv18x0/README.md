# cv18x0 boot chain — how `fip.bin` is built

This target (Milk-V Duo family: CV1800B / SG2000 / SG2002) boots through Sophgo's
multi-stage firmware image package (**FIP**). Everything in the FIP is built from
source in-tree; a fresh `git clone` + `make` produces a bootable image with no
external blobs.

> **Boot status (2026-07-20):** Duo S (SG2000) and Duo 256M (SG2002) boot to the
> Linux console from these from-source fips. **The Duo (CV1800B) does not boot
> yet** — its sdk-v2 cv180x BL2 hangs at the FSBL → OpenSBI handoff. See
> `package/boot/cv18xxlibs/README.md` ("Known issue: CV1800B") for the diagnosis.

## Boot flow (per SG200X TRM §"Boot and Upgrade")

```
BOOTROM  (on-chip mask ROM)
   │  samples EMMC_DAT3 / EMMC_DAT0 straps to pick the boot medium
   │  (SD / SPI-NOR / SPI-NAND / eMMC), then loads the FIP from it
   ▼
FSBL / BL2   (Sophgo fork of ARM TF-A; package/boot/cv18xxlibs)
   │  applies CHIP_CONF, brings up DDR (compiled-in DDR config + DDR_PARAM)
   ▼
OpenSBI (MONITOR)   (package/boot/opensbi, generic, fw_dynamic; M-mode SBI)
   │  needs an embedded FDT (FW_FDT_PATH) or it faults before console init
   ▼
U-Boot (LOADER_2ND / BL33)   (package/boot/uboot-cv18x0)
   │  fatload mmc 0:1 sd.boot ; bootm  -> loads the kernel FIT
   ▼
Linux 6.12 (kernel FIT "sd.boot", per-board dtb) -> mounts rootfs on mmcblk0p2
```

## FIP container

The FIP is Sophgo's own format (magic `CVBL01` / `CVLD02`), **not** ARM TF-A's
FIP. It is assembled by `fiptool` (tools/fiptool, from `sophgo/fiptool`) in
`image/Makefile`'s `Build/riscv-sdcard`, and placed as `fip.bin` in the FAT32
boot partition. Approximate component layout (offsets vary with payload sizes):

| Component | Source |
|---|---|
| header + CHIP_CONF | fiptool / FSBL build |
| FSBL / BL2 | package/boot/cv18xxlibs (per chip family) |
| DDR_PARAM | cv18xxlibs (generic CV181x `ddr_param.bin`) |
| BLCP / RTOS | none (fiptool `--rtos /dev/null`) |
| OpenSBI (MONITOR) | package/boot/opensbi |
| U-Boot (LOADER_2ND) | package/boot/uboot-cv18x0 |

## Per-board FSBL parameters

| Board | SoC | chip family | DDR_CFG | BL2 blob |
|---|---|---|---|---|
| Milk-V Duo | CV1800B | cv180x | ddr2_1333_x16 | bl2_cv1800b_milkv_duo_sd.bin |
| Milk-V Duo 256M | SG2002 | cv181x | ddr3_1866_x16 | bl2_cv181x_milkv_sd.bin |
| Milk-V Duo S | SG2000 | cv181x | ddr3_1866_x16 | bl2_cv181x_milkv_sd.bin |

The one CV181x BL2 serves both SG2000 and SG2002 — the FSBL auto-detects the DDR3
density from package strapping; the kernel gets its real memory size from the
per-board FIT dtb.

## Inspecting a built FIP

```sh
# MBR p1 (FAT boot) starts at LBA 2048 = byte offset 1048576
gunzip -c <image>.img.gz > img
mcopy -n -i img@@1048576 ::fip.bin fip.bin      # the firmware package
mcopy -n -i img@@1048576 ::sd.boot sd.boot      # the kernel FIT (per-board dtb)
```

A healthy OpenSBI region inside the FIP contains the flattened-DT magic
`d0 0d fe ed`; its absence means the OpenSBI FW_FDT guard did not fire (see
`package/boot/opensbi/Makefile`) and the board will reset-loop right after
"Jump to monitor".

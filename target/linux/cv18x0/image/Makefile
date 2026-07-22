# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2024 Toco Technologies <info@toco.ae>
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

FAT32_BLOCK_SIZE=1024
FAT32_BLOCKS=$(shell echo $$(($(CONFIG_CV18X0_SD_BOOT_PARTSIZE)*1024*1024/$(FAT32_BLOCK_SIZE))))

KERNEL_LOADADDR:=0x80200000

# The FIP (FSBL/BL2 + DDR params + OpenSBI + U-Boot) placed in the FAT boot
# partition is assembled here by fiptool from the in-tree components: the
# per-chip BL2 built from source by package/boot/cv18xxlibs, plus OpenSBI and
# U-Boot. It is per-device because the BL2/FSBL inside it is chip-family
# specific: a cv180x (CV1800B) BL2 panics on cv181x (SG2000/SG2002) silicon
# and vice versa -- the mask ROM log ends in "E:RESET:panic:-1".
DEVICE_VARS += BL2 FIP DDRPARAM UBOOT

define Build/riscv-sdcard
	rm -f $@.boot #$(KDIR_TMP)/$(IMG_PREFIX)-$(PROFILE)-boot.img
	mkfs.fat $@.boot  -C $(FAT32_BLOCKS)

	$(STAGING_DIR_HOST)/bin/fiptool --fsbl $(STAGING_DIR_IMAGE)/$(BL2) \
		--opensbi $(STAGING_DIR_IMAGE)/fw_dynamic-generic.bin \
		--uboot $(STAGING_DIR_IMAGE)/$(UBOOT)-u-boot.bin \
		--ddr_param $(STAGING_DIR_IMAGE)/$(DDRPARAM) \
		--rtos /dev/null \
		$(STAGING_DIR_IMAGE)/$(FIP)

	mcopy -i $@.boot $(STAGING_DIR_IMAGE)/$(FIP) ::fip.bin
	mcopy -i $@.boot $(IMAGE_KERNEL) ::sd.boot

	./gen_cv18x0_sdcard_img.sh \
		$@ \
		$@.boot \
		$(IMAGE_ROOTFS) \
		$(CONFIG_CV18X0_SD_BOOT_PARTSIZE) \
		$(CONFIG_TARGET_ROOTFS_PARTSIZE)
endef

define Device/Default
  PROFILES := Default
  KERNEL_NAME := Image
  DTS_DIR := $(DTS_DIR)/sophgo
  DEVICE_DTS := $$(SOC)-$(subst _,-,$(1))
  IMAGES := sdcard.img.gz
  IMAGE/sdcard.img.gz := riscv-sdcard | append-metadata | gzip
endef

define Device/FitImageLzma
	KERNEL_SUFFIX := -fit-uImage.itb
	KERNEL = kernel-bin | lzma | fit lzma $$(DTS_DIR)/$$(DEVICE_DTS).dtb
	KERNEL_NAME := Image
endef

# KNOWN ISSUE (CV1800B / Duo): the cv180x BL2 built from duo-buildroot-sdk-v2
# hangs at the FSBL->OpenSBI handoff on real hardware (freezes right after
# "Jump to monitor"; no OpenSBI banner). The Duo S / Duo 256M (cv181x, same
# sdk-v2 FSBL codebase + the same OpenSBI/U-Boot) boot to the Linux console;
# OpenSBI and its embedded 63.75 MiB FDT were verified byte-identical to the
# known-good firmware, so the fault is isolated to the sdk-v2 cv180x BL2 build.
# See package/boot/cv18xxlibs/README.md ("Known issue: CV1800B") for details.
define Device/milkv_duo
  $(call Device/FitImageLzma)
  DEVICE_VENDOR := MilkV
  DEVICE_MODEL := Duo
  SOC := cv1800b
  DEVICE_DTS := $$(SOC)-milkv-duo
  DEVICE_PACKAGES := opensbi_generic
  UBOOT := milkv_duo
  BL2 := bl2_cv1800b_milkv_duo_sd.bin
  DDRPARAM := cv181x_ddr_param.bin
  FIP := fip_milkv_duo.bin
endef
TARGET_DEVICES += milkv_duo

define Device/milkv_duo256m
  $(call Device/FitImageLzma)
  DEVICE_VENDOR := MilkV
  DEVICE_MODEL := Duo 256M
  SOC := sg2002
  DEVICE_DTS := $$(SOC)-milkv-duo256m
  DEVICE_PACKAGES := opensbi_generic
  UBOOT := milkv_duo
  BL2 := bl2_cv181x_milkv_sd.bin
  DDRPARAM := cv181x_ddr_param.bin
  FIP := fip_milkv_duo256m.bin
endef
TARGET_DEVICES += milkv_duo256m

define Device/milkv_duos
  $(call Device/FitImageLzma)
  DEVICE_VENDOR := MilkV
  DEVICE_MODEL := Duo S
  SOC := sg2000
  DEVICE_DTS := $$(SOC)-milkv-duos
  DEVICE_PACKAGES := opensbi_generic
  UBOOT := milkv_duo
  # The SG2000 (Duo S) shares the SG2002 256M's boot chain: the CV181x
  # FSBL build is identical (cv181x / ddr3_1866_x16) and auto-detects
  # the 512 MiB DDR3 density from package strapping, so the same CV181x
  # BL2 is reused; only the output FIP name differs per device.
  BL2 := bl2_cv181x_milkv_sd.bin
  DDRPARAM := cv181x_ddr_param.bin
  FIP := fip_milkv_duos.bin
endef
TARGET_DEVICES += milkv_duos

define Device/milkv_duo_module_01_evb
  $(call Device/FitImageLzma)
  DEVICE_VENDOR := MilkV
  DEVICE_MODEL := Duo Module 01 EVB
  SOC := sg2000
  DEVICE_DTS := $$(SOC)-milkv-duo-module-01-evb
  DEVICE_PACKAGES := opensbi_generic
  UBOOT := milkv_duo
  # The Duo Module 01 carries the same SG2000 as the Duo S, so the whole
  # Duo S boot chain (CV181x BL2, DDR params, U-Boot) is reused unchanged;
  # only the output FIP name and the DTB differ.
  BL2 := bl2_cv181x_milkv_sd.bin
  DDRPARAM := cv181x_ddr_param.bin
  FIP := fip_milkv_duo_module_01_evb.bin
endef
TARGET_DEVICES += milkv_duo_module_01_evb

$(eval $(call BuildImage))

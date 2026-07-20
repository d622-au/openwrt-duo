# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2024 OpenWrt.org
#
# Builds the CV18xx First Stage Boot Loader (FSBL / BL2) from source for the
# Milk-V Duo family. The FSBL is CVITEK/Sophgo's fork of ARM Trusted Firmware
# (it "acts as ATF BL2"): the SoC mask ROM loads it out of the FIP, it brings up
# DDR and hands off to OpenSBI -> U-Boot.
#
# Source: the Milk-V "duo-buildroot-sdk-v2" BSP monorepo -- the only tree that
# carries the full CV181x FSBL source (DDR/eMMC/UART/security) plus first-class
# board configs for the SG2000 (Duo S) and SG2002 (Duo 256M). We fetch the whole
# repo (see PKG_MIRROR_HASH note) but build only its fsbl/ subtree, for both chip
# families:
#   cv180x -> Milk-V Duo   (CV1800B, 64 MiB DDR2)
#   cv181x -> Milk-V Duo S (SG2000) and Duo 256M (SG2002), DDR3
# The board-specific memory map is generated into the FSBL include dir with the
# SDK's own mmap_conv.py before each build; the DDR config is compiled in via
# DDR_CFG. See ./patches/100-cv18xx-fsbl-portability.patch for the mainline-GCC
# portability changes (the vendor source assumes a T-Head toolchain).
#

include $(TOPDIR)/rules.mk

PKG_NAME:=cv18xxlibs

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL=https://github.com/milkv-duo/duo-buildroot-sdk-v2
PKG_SOURCE_DATE:=2025-11-04
PKG_SOURCE_VERSION:=6f8962c394dd0a05729abb089f0feb7d5cc4aa5e
# TODO: replace with the real mirror hash once the source has been fetched once
# (run `make package/boot/cv18xxlibs/download` and copy the printed hash here).
# `skip` is used for now because the sdk-v2 monorepo is multi-GB and has not been
# pinned to the OpenWrt download mirror yet.
PKG_MIRROR_HASH:=skip

PKG_TARGETS:=bin
PKG_FLAGS:=nonshared

# The FSBL is CVITEK/Sophgo's fork of ARM Trusted Firmware (BSD-3-Clause);
# the vendor ships no LICENSE file in the fsbl/ tree, so PKG_LICENSE_FILES is
# omitted. Provenance is documented in ./README.md.
PKG_LICENSE:=BSD-3-Clause
PKG_MAINTAINER:=Zoltan HERPAI <wigyori@uid0.hu>

include $(INCLUDE_DIR)/package.mk

define Package/cv18xxlibs
    SECTION:=boot
    CATEGORY:=Boot Loaders
    DEPENDS:=@(TARGET_cv18x0)
    URL:=https://github.com/milkv-duo/duo-buildroot-sdk-v2
    TITLE:=CV18xx FSBL/BL2 (built from source)
endef

export GCC_HONOUR_COPTS=s

FSBL_DIR:=$(PKG_BUILD_DIR)/fsbl
MMAP_CONV:=$(PKG_BUILD_DIR)/build/scripts/mmap_conv.py
BOARDS_DIR:=$(PKG_BUILD_DIR)/build/boards

# $(1)=chip_arch (cv180x/cv181x)  $(2)=board project  $(3)=DDR_CFG
define Build/BuildBL2
	python3 $(MMAP_CONV) --type h \
		$(BOARDS_DIR)/$(1)/$(2)/memmap.py \
		$(FSBL_DIR)/plat/$(1)/include/cvi_board_memmap.h
	$(MAKE) -C $(FSBL_DIR) \
		CHIP_ARCH=$(1) ARCH=riscv BOOT_CPU=riscv DDR_CFG=$(3) \
		CROSS_COMPILE=$(TARGET_CROSS) \
		LDFLAGS=--no-warn-rwx-segments \
		bl2
endef

define Build/Compile
	# Milk-V Duo (CV1800B, 64 MiB DDR2)
	$(call Build/BuildBL2,cv180x,cv1800b_milkv_duo_musl_riscv64_sd,ddr2_1333_x16)
	# Milk-V Duo S (SG2000) / Duo 256M (SG2002), DDR3 -- one CV181x BL2, shared
	# by both boards (the FSBL auto-detects DDR3 density from package strapping).
	$(call Build/BuildBL2,cv181x,sg2002_milkv_duo256m_musl_riscv64_sd,ddr3_1866_x16)
endef

define Build/InstallDev
	$(INSTALL_DIR) $(STAGING_DIR_IMAGE)
	$(CP) $(FSBL_DIR)/build/cv180x/bl2.bin $(STAGING_DIR_IMAGE)/bl2_cv1800b_milkv_duo_sd.bin
	$(CP) $(FSBL_DIR)/build/cv181x/bl2.bin $(STAGING_DIR_IMAGE)/bl2_cv181x_milkv_sd.bin
	$(CP) $(FSBL_DIR)/test/cv181x/ddr_param.bin $(STAGING_DIR_IMAGE)/cv181x_ddr_param.bin
endef

$(eval $(call BuildPackage,cv18xxlibs))

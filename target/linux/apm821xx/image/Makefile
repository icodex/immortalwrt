#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

DEVICE_VARS += DTB_SIZE

define Build/boot-img
	$(RM) -rf $@.bootdir
	mkdir -p $@.bootdir/boot

	$(CP) $@.scr $@.bootdir/boot/boot.scr
	$(CP) $(IMAGE_KERNEL).dtb $@.bootdir/boot/$(DEVICE_DTB)
	$(CP) $(IMAGE_KERNEL) $@.bootdir/boot/uImage

	genext2fs --block-size $(BLOCKSIZE:%k=%Ki) \
		--size-in-blocks $$((1024 * $(CONFIG_TARGET_KERNEL_PARTSIZE))) \
		--root $@.bootdir $@.boot

	# convert it to revision 1 - needed for u-boot ext2load
	$(STAGING_DIR_HOST)/bin/tune2fs -O filetype $@.boot
	$(STAGING_DIR_HOST)/bin/e2fsck -pDf $@.boot > /dev/null
endef

define Build/boot-script
	$(STAGING_DIR_HOST)/bin/mkimage -A powerpc -T script -C none -n "$(PROFILE) Boot Script" \
		-d mbl_boot.scr \
		$@.scr
endef

define Build/copy-file
	cat "$(1)" > "$@"
endef

define Build/create-uImage-dtb
	# flat_dt target expect FIT image - which WNDR4700's uboot doesn't support
	-$(STAGING_DIR_HOST)/bin/mkimage -A $(LINUX_KARCH) \
		-O linux -T kernel -C none \
		-n '$(call toupper,$(LINUX_KARCH)) $(VERSION_DIST) Linux-$(LINUX_VERSION)' \
		-d "$@.dtb" "$@.dtb.uimage"
endef

define Build/dtb
	$(call Image/BuildDTB,../dts/$(DEVICE_DTS).dts,$@.dtb,,--space $(DTB_SIZE))
endef

define Build/export-dtb
	cp $(IMAGE_KERNEL).dtb $@
endef

define Build/hdd-img
	./mbl_gen_hdd_img.sh $@ $@.boot $(IMAGE_ROOTFS) $(CONFIG_TARGET_KERNEL_PARTSIZE) $(CONFIG_TARGET_ROOTFS_PARTSIZE)
endef

define Build/MerakiAdd-dtb
	$(call Image/BuildDTB,../dts/$(DEVICE_DTS).dts,$@.dtb)
	( \
		dd if=$@.dtb bs=$(DTB_SIZE) conv=sync; \
		dd if=$@ bs=$(BLOCKSIZE) conv=sync; \
	) > $@.new
	@mv $@.new $@
endef

define Build/MerakiNAND
	-$(STAGING_DIR_HOST)/bin/mkmerakifw \
		-B $(BOARD_NAME) -s \
		-i $@ \
		-o $@.new
	@cp $@.new $@
endef

define Build/MuImage-initramfs
	rm -rf $@.fakerd $@.new

	dd if=/dev/zero of=$@.fakerd bs=32 count=1 conv=sync

	# Netgear used an old uboot that doesn't have FIT support.
	# So we are stuck with either a full ext2/4 fs in a initrd.
	# ... or we try to make the "multi" image approach to work
	# for us.
	#
	# Sadly, the "multi" image has to consists of three
	# "fixed" parts in the following "fixed" order:
	# 1. The kernel which is in $@
	# 2. The (fake) initrd which is in $@.fakerd
	# 3. The device tree binary which is in $@.dtb
	#
	# Now, given that we use the function for the kernel which
	# already has a initramfs image inside, we still have to
	# add a "fake" initrd (which a mkimage header) in the second
	# part of the legacy multi image. Since we need to put the
	# device tree stuff into part 3.

	-$(STAGING_DIR_HOST)/bin/mkimage -A $(LINUX_KARCH) -O linux -T multi \
		-C $(1) -a $(KERNEL_LOADADDR) -e $(KERNEL_ENTRY) \
		-n '$(BOARD_NAME) initramfs' -d $@:$@.fakerd:$@.dtb $@.new
	mv $@.new $@
	rm -rf $@.fakerd
endef

define Build/prepend-dtb
	cat "$@.dtb.uimage" "$@" > "$@.new"
	mv "$@.new" "$@"
endef

define Image/cpiogz
	( cd $(TARGET_DIR); find . | cpio -o -H newc | gzip -9n >$(KDIR_TMP)/$(IMG_PREFIX)-rootfs.cpio.gz )
endef

define Device/Default
  PROFILES := Default
  KERNEL_DEPENDS = $$(wildcard ../dts/$$(DEVICE_DTS).dts)
  DEVICE_DTS :=
  KERNEL_ENTRY := 0x00000000
  KERNEL_LOADADDR := 0x00000000
  DEVICE_DTS_DIR := ../dts
endef

ifeq ($(SUBTARGET),nand)

define Device/meraki_mr24
  DEVICE_TITLE := Cisco Meraki MR24
  DEVICE_PACKAGES := kmod-spi-gpio -swconfig
  BOARD_NAME := mr24
  DEVICE_DTS := meraki-mr24
  BLOCKSIZE := 63k
  IMAGES := sysupgrade.bin
  DTB_SIZE := 64512
  IMAGE_SIZE := 8191k
  KERNEL := kernel-bin | lzma | uImage lzma | MerakiAdd-dtb | MerakiNAND
  KERNEL_INITRAMFS := kernel-bin | lzma | dtb | MuImage-initramfs lzma
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
  UBINIZE_OPTS := -E 5
  SUPPORTED_DEVICES += mr24
endef
TARGET_DEVICES += meraki_mr24

define Device/meraki_mx60
  DEVICE_TITLE := Cisco Meraki MX60/MX60W
  DEVICE_PACKAGES := kmod-spi-gpio kmod-usb-ledtrig-usbport kmod-usb-dwc2 \
		     kmod-usb-storage block-mount
  BOARD_NAME := mx60
  DEVICE_DTS := meraki-mx60
  BLOCKSIZE := 63k
  IMAGES := sysupgrade.bin
  DTB_SIZE := 64512
  IMAGE_SIZE := 1021m
  KERNEL_SIZE := 4031k
  KERNEL := kernel-bin | gzip | uImage gzip | MerakiAdd-dtb | MerakiNAND
  KERNEL_INITRAMFS := kernel-bin | gzip | dtb | MuImage-initramfs gzip
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
  UBINIZE_OPTS := -E 5
  SUPPORTED_DEVICES += mx60
endef
TARGET_DEVICES += meraki_mx60

define Device/netgear_wndap6x0
  DEVICE_PACKAGES := kmod-eeprom-at24
  SUBPAGESIZE := 256
  PAGESIZE := 512
  BLOCKSIZE := 16k
  DTB_SIZE := 32768
  IMAGE_SIZE := 27392k
  IMAGES := sysupgrade.bin factory.img
  KERNEL_SIZE := 4032k
  KERNEL := dtb | kernel-bin | gzip | MuImage-initramfs gzip
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
  IMAGE/factory.img := append-kernel | pad-to $$$$(KERNEL_SIZE) | append-ubi
  UBINIZE_OPTS := -E 5
endef

define Device/netgear_wndap620
  $(Device/netgear_wndap6x0)
  DEVICE_TITLE := Netgear WNDAP620 - Premium Wireless-N
  DEVICE_DTS := netgear-wndap620
endef
TARGET_DEVICES += netgear_wndap620

define Device/netgear_wndap660
  $(Device/netgear_wndap6x0)
  DEVICE_TITLE := Netgear WNDAP660 - Dual Radio Dual Band Wireless-N
  DEVICE_DTS := netgear-wndap660
endef
TARGET_DEVICES += netgear_wndap660

define Device/netgear_wndr4700
  DEVICE_TITLE := Netgear Centria N900 WNDR4700/WNDR4720
  DEVICE_PACKAGES := badblocks block-mount e2fsprogs \
	kmod-dm kmod-fs-ext4 kmod-fs-vfat kmod-usb-ledtrig-usbport \
	kmod-md-mod kmod-nls-cp437 kmod-nls-iso8859-1 kmod-nls-iso8859-15 \
	kmod-nls-utf8 kmod-usb3 kmod-usb-dwc2 kmod-usb-storage \
	partx-utils
  BOARD_NAME := wndr4700
  DEVICE_DTS := netgear-wndr4700
  PAGESIZE := 2048
  SUBPAGESIZE := 512
  BLOCKSIZE := 128k
  DTB_SIZE := 131008
  IMAGE_SIZE := 24960k
  IMAGES := factory.img sysupgrade.bin
  ARTIFACTS := device-tree.dtb
  KERNEL_SIZE := 3584k
  # append a fake/empty rootfs to fool netgear's uboot
  # CHECK_DNI_FIRMWARE_ROOTFS_INTEGRITY in do_chk_dniimg()
  KERNEL := kernel-bin | lzma | uImage lzma | pad-offset $$(BLOCKSIZE) 64 | \
	    append-uImage-fakehdr filesystem | dtb | create-uImage-dtb | prepend-dtb
  KERNEL_INITRAMFS := kernel-bin | gzip | dtb | MuImage-initramfs gzip
  IMAGE/factory.img := append-kernel | pad-to $$$$(KERNEL_SIZE) | append-ubi | \
		       netgear-dni | check-size
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
  ARTIFACT/device-tree.dtb := export-dtb | uImage none
  NETGEAR_BOARD_ID := WNDR4700
  NETGEAR_HW_ID := 29763875+128+256
  UBINIZE_OPTS := -E 5
  SUPPORTED_DEVICES += wndr4700
endef
TARGET_DEVICES += netgear_wndr4700

endif

ifeq ($(SUBTARGET),sata)

define Device/wd_mybooklive
  DEVICE_TITLE := Western Digital My Book Live Series (Single + Duo)
  DEVICE_PACKAGES := kmod-usb-dwc2 kmod-usb-ledtrig-usbport kmod-usb-storage kmod-fs-vfat wpad-basic-openssl
  DEVICE_TYPE := nas
  DEVICE_DTS := wd-mybooklive
  SUPPORTED_DEVICES += mbl wd,mybooklive-duo
  BLOCKSIZE := 1k
  DTB_SIZE := 16384
  KERNEL := kernel-bin | dtb | gzip | uImage gzip
  KERNEL_INITRAMFS := kernel-bin | gzip | dtb | MuImage-initramfs gzip
  IMAGES := factory.img.gz kernel.dtb sysupgrade.img.gz
  ARTIFACTS := apollo3g.dtb
  DEVICE_DTB := apollo3g.dtb
  FILESYSTEMS := ext4 squashfs
  IMAGE/factory.img.gz := boot-script | boot-img | hdd-img | gzip
  IMAGE/sysupgrade.img.gz := boot-script | boot-img | hdd-img | gzip | append-metadata
  ARTIFACT/apollo3g.dtb := export-dtb
endef

TARGET_DEVICES += wd_mybooklive

endif

$(eval $(call BuildImage))

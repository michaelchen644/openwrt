#
# Copyright (C) 2006-2015 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/version.mk
include $(INCLUDE_DIR)/feeds.mk

override MAKEFLAGS=

IB_NAME:=$(VERSION_DIST_SANITIZED)-imagebuilder-$(if $(CONFIG_VERSION_FILENAMES),$(VERSION_NUMBER)-)$(BOARD)-$(SUBTARGET).$(HOST_OS)-$(HOST_ARCH)
PKG_BUILD_DIR:=$(BUILD_DIR)/$(IB_NAME)
IB_KDIR:=$(patsubst $(TOPDIR)/%,$(PKG_BUILD_DIR)/%,$(KERNEL_BUILD_DIR))
IB_LDIR:=$(patsubst $(TOPDIR)/%,$(PKG_BUILD_DIR)/%,$(LINUX_DIR))
IB_DTSDIR:=$(patsubst $(TOPDIR)/%,$(PKG_BUILD_DIR)/%,$(LINUX_DIR))/arch/$(LINUX_KARCH)/boot/dts/
IB_IDIR:=$(patsubst $(TOPDIR)/%,$(PKG_BUILD_DIR)/%,$(STAGING_DIR_IMAGE))

BUNDLER_PATH := $(subst $(space),:,$(filter-out $(TOPDIR)/%,$(subst :,$(space),$(PATH))))
BUNDLER_COMMAND := PATH=$(BUNDLER_PATH) $(XARGS) $(SCRIPT_DIR)/bundle-libraries.sh $(PKG_BUILD_DIR)/staging_dir/host

PACKAGE_SUFFIX:=$(if $(CONFIG_USE_APK),apk,ipk)
PACKAGE_VERSION_SEPARATOR:=$(if $(CONFIG_USE_APK),-,_)

all: compile

$(BIN_DIR)/$(IB_NAME).tar.zst: clean
	rm -rf $(PKG_BUILD_DIR)
	mkdir -p $(IB_KDIR) $(IB_LDIR) $(PKG_BUILD_DIR)/staging_dir/host/lib \
		$(PKG_BUILD_DIR)/target/linux $(PKG_BUILD_DIR)/scripts $(IB_DTSDIR)
	-cp $(TOPDIR)/.config $(PKG_BUILD_DIR)/.config
	$(SED) 's/^CONFIG_BINARY_FOLDER=.*/# CONFIG_BINARY_FOLDER is not set/' $(PKG_BUILD_DIR)/.config
	$(SED) 's/^CONFIG_DOWNLOAD_FOLDER=.*/# CONFIG_DOWNLOAD_FOLDER is not set/' $(PKG_BUILD_DIR)/.config
	$(CP) -L \
		$(INCLUDE_DIR) $(SCRIPT_DIR) \
		$(TOPDIR)/rules.mk \
		./files/Makefile \
		$(TMP_DIR)/.targetinfo \
		$(TMP_DIR)/.packageinfo \
		$(PKG_BUILD_DIR)/

	$(INSTALL_DIR) $(PKG_BUILD_DIR)/packages

ifneq ($(CONFIG_USE_APK),)
  ifeq ($(CONFIG_IB_STANDALONE),)
	$(call FeedSourcesAppendAPK,$(PKG_BUILD_DIR)/repositories)
	$(VERSION_SED_SCRIPT) $(PKG_BUILD_DIR)/repositories
  endif

	$(INSTALL_DATA) ./files/README.apk.md $(PKG_BUILD_DIR)/packages/README.md
else
  ifeq ($(CONFIG_IB_STANDALONE),)
	echo '## Remote package repositories' >> $(PKG_BUILD_DIR)/repositories.conf
	$(call FeedSourcesAppendOPKG,$(PKG_BUILD_DIR)/repositories.conf)
	$(VERSION_SED_SCRIPT) $(PKG_BUILD_DIR)/repositories.conf

  endif

	# create an empty package index so `opkg` doesn't report an error
	touch $(PKG_BUILD_DIR)/packages/Packages
	$(INSTALL_DATA) ./files/README.opkg.md $(PKG_BUILD_DIR)/packages/README.md

	echo ''                                                        >> $(PKG_BUILD_DIR)/repositories.conf
	echo '## This is the local package repository, do not remove!' >> $(PKG_BUILD_DIR)/repositories.conf
	echo 'src imagebuilder file:packages'                          >> $(PKG_BUILD_DIR)/repositories.conf
endif

ifeq ($(CONFIG_IB_STANDALONE),)
	$(FIND) $(call FeedPackageDir,libc) -type f \
	  \( \
		-name 'base-files$(PACKAGE_VERSION_SEPARATOR)*.$(PACKAGE_SUFFIX)' -or \
		-name 'libc$(PACKAGE_VERSION_SEPARATOR)*.$(PACKAGE_SUFFIX)' -or \
		-name 'kernel$(PACKAGE_VERSION_SEPARATOR)*.$(PACKAGE_SUFFIX)' \) \
	  -exec $(CP) -t $(PKG_BUILD_DIR)/packages {} +
else
	$(FIND) $(wildcard $(PACKAGE_SUBDIRS)) -type f -name '*.$(PACKAGE_SUFFIX)' \
		-exec $(CP) -t $(PKG_BUILD_DIR)/packages/ {} +
endif

ifneq ($(CONFIG_SIGNATURE_CHECK),)
ifneq ($(CONFIG_USE_APK),y)
	echo ''                                                        >> $(PKG_BUILD_DIR)/repositories.conf
	echo 'option check_signature'                                  >> $(PKG_BUILD_DIR)/repositories.conf
	$(INSTALL_DIR) $(PKG_BUILD_DIR)/keys
	$(CP) -L $(STAGING_DIR_ROOT)/etc/opkg/keys/ $(PKG_BUILD_DIR)/
	$(CP) -L $(STAGING_DIR_ROOT)/usr/sbin/opkg-key $(PKG_BUILD_DIR)/scripts/
else
	$(INSTALL_DIR) $(PKG_BUILD_DIR)/keys
	$(CP) -L $(STAGING_DIR_ROOT)/etc/apk/keys/ $(PKG_BUILD_DIR)/
endif
endif

	$(CP) -L $(TOPDIR)/target/linux/Makefile $(PKG_BUILD_DIR)/target/linux
	$(CP) -L $(TOPDIR)/target/linux/generic $(PKG_BUILD_DIR)/target/linux
	# check if board is installed from a feeds subdirectory
	if [ -d $(TOPDIR)/target/linux/feeds/$(BOARD) ]; then \
		mkdir -p $(PKG_BUILD_DIR)/target/linux/feeds; \
		$(CP) -L $(TOPDIR)/target/linux/feeds/$(BOARD) $(PKG_BUILD_DIR)/target/linux/feeds; \
	else \
		$(CP) -L $(TOPDIR)/target/linux/$(BOARD) $(PKG_BUILD_DIR)/target/linux; \
	fi
	if [ -d $(TOPDIR)/staging_dir/host/lib/grub ]; then \
		$(CP) $(TOPDIR)/staging_dir/host/lib/grub/ $(PKG_BUILD_DIR)/staging_dir/host/lib; \
	fi
	rm -rf \
		$(PKG_BUILD_DIR)/target/linux/*/files{,-*} \
		$(PKG_BUILD_DIR)/target/linux/*/patches{,-*} \
		$(PKG_BUILD_DIR)/target/linux/generic/{pending,backport,hack}{,-*}
	-cp $(KERNEL_BUILD_DIR)/* $(IB_KDIR)/ # don't copy subdirectories here
	-cp $(LINUX_DIR)/.config $(IB_LDIR)/
	rm -f $(IB_KDIR)/root.*
	rm -f $(IB_KDIR)/vmlinux.debug
	# remove any file for initramfs and Per Device Rootfs initramfs files
	rm -f $(IB_KDIR)/vmlinux-initramfs*
	rm -f $(IB_KDIR)/vmlinuz-initramfs*
	rm -f $(IB_KDIR)/Image-initramfs*
	if [ -x $(LINUX_DIR)/scripts/dtc/dtc ]; then \
		$(INSTALL_DIR) $(IB_LDIR)/scripts/dtc; \
		$(INSTALL_BIN) $(LINUX_DIR)/scripts/dtc/dtc $(IB_LDIR)/scripts/dtc/dtc; \
	fi
	if [ -d $(LINUX_DIR)/arch/$(LINUX_KARCH)/boot/dts ]; then \
		$(CP) -L $(LINUX_DIR)/arch/$(LINUX_KARCH)/boot/dts/* $(IB_DTSDIR); \
	fi
	$(SED) 's,^# REVISION:=.*,REVISION:=$(REVISION),g' $(PKG_BUILD_DIR)/include/version.mk
	$(SED) 's,^# SOURCE_DATE_EPOCH:=.*,SOURCE_DATE_EPOCH:=$(SOURCE_DATE_EPOCH),g' $(PKG_BUILD_DIR)/include/version.mk
	$(SED) 's,^# BASE_FILES_VERSION:=.*,BASE_FILES_VERSION:=$(shell cat $(TMP_DIR)/base-files.version),g' $(PKG_BUILD_DIR)/include/version.mk
	$(SED) 's,^# LIBC_VERSION:=.*,LIBC_VERSION:=$(shell cat $(TMP_DIR)/libc.version),g' $(PKG_BUILD_DIR)/include/version.mk
	$(SED) 's,^# KERNEL_VERSION:=.*,KERNEL_VERSION:=$(shell cat $(TMP_DIR)/kernel.version),g' $(PKG_BUILD_DIR)/include/version.mk
	$(SED) '/LINUX_VERMAGIC:=/ { s,unknown,$(LINUX_VERMAGIC),g }' $(PKG_BUILD_DIR)/include/kernel.mk
	find $(PKG_BUILD_DIR) -name CVS -o -name .git -o -name .svn \
	  | $(XARGS) rm -rf
	$(INSTALL_DIR) $(IB_IDIR)
	-$(CP) $(STAGING_DIR_IMAGE)/* $(IB_IDIR)/
	$(INSTALL_DIR) $(PKG_BUILD_DIR)/staging_dir/host/bin
	$(CP) $(STAGING_DIR_HOST)/bin/* $(PKG_BUILD_DIR)/staging_dir/host/bin/
	(cd $(PKG_BUILD_DIR); find staging_dir/host/bin/ $(IB_LDIR)/scripts/dtc/ -type f | \
		$(BUNDLER_COMMAND))
	$(CP) $(TOPDIR)/staging_dir/host/lib/libfakeroot* $(PKG_BUILD_DIR)/staging_dir/host/lib
	STRIP=$(STAGING_DIR_HOST)/bin/sstrip $(SCRIPT_DIR)/rstrip.sh $(PKG_BUILD_DIR)/staging_dir/host/bin/

	(cd $(BUILD_DIR); \
		tar -I '$(STAGING_DIR_HOST)/bin/zstd -T0 --ultra -20' -cf $@ $(IB_NAME) \
		--mtime="$(shell date --date=@$(SOURCE_DATE_EPOCH))"; \
	)

download:
prepare:
compile: $(BIN_DIR)/$(IB_NAME).tar.zst
install: compile

clean: FORCE
	rm -rf $(PKG_BUILD_DIR) $(BIN_DIR)/$(IB_NAME).tar.zst

#
# Copyright (C) 2018-2019 Intel Corporation
#
# SPDX-License-Identifier: GPL-2.0-or-later
#

SHELL := /bin/bash

ifeq (, $(filter $(MAKECMDGOALS), clean distclean))
ifeq ("$(wildcard config.mk config.h)", "")
$(error Run ./configure before invoking make)
endif
include config.mk
endif

#
# Flags for compilation
#
CFLAGS += $(patsubst %,-I%,$(INCLUDES))
CFLAGS += $(patsubst %,-D%,$(DEFINES))

OBJDIR = .obj/
LIBOBJDIR = .libobj/
TARGET = sedcli
LIB = libsed
#
# Files to be compiled

LIBOBJS = sed.o
LIBOBJS += sed_util.o
ifndef CONFIG_OPAL_DRIVER
LIBOBJS += nvme_access.o
LIBOBJS += nvme_pt_ioctl.o
LIBOBJS += opal_parser.o
else
LIBOBJS += sed_ioctl.o
endif

OBJS = argp.o
OBJS += sedcli_main.o
OBJS += sedcli_util.o

ifdef CONFIG_KMIP
KMIP_OBJS = argp.o
KMIP_OBJS += metadata_serializer.o
KMIP_OBJS += config_file.o
KMIP_OBJS += crypto_lib.o
KMIP_OBJS += kmip_lib.o
KMIP_OBJS += sedcli_util.o
KMIP_OBJS += sedcli_kmip.o
endif

ALL_TARGETS = $(TARGET)-static $(TARGET)-dynamic
ifdef CONFIG_KMIP
ALL_TARGETS += $(TARGET)-kmip
endif

INSTALL_TARGETS = install-$(TARGET)
ifdef CONFIG_KMIP
INSTALL_TARGETS += install-$(TARGET)-kmip
endif

all: $(ALL_TARGETS)
	@ln -sf $(TARGET)-static $(TARGET)

$(TARGET)-static: $(TARGET).a $(LIB).a
	@echo "  LD " $@
	@$(CC) $(TARGET).a $(LDFLAGS) -Wl,-Bstatic -lsed -Wl,-Bdynamic -o $@

$(TARGET)-dynamic: $(TARGET).a $(LIB).so
	@echo "  LD " $@
	@$(CC) $(TARGET).a $(LDFLAGS) -lsed -o $@

$(TARGET).a: $(patsubst %,$(OBJDIR)%,$(OBJS))
	@echo "  AR " $@
	@ar rcs $@ $^

$(TARGET)-kmip.a: libkmip.a $(patsubst %,$(OBJDIR)%,$(KMIP_OBJS))
	@echo "  AR " $@
	@ar rcs $@ $(patsubst %,$(OBJDIR)%,$(KMIP_OBJS))

$(TARGET)-kmip: $(TARGET)-kmip.a $(LIB).a
	@echo "  LD " $@
	@$(CC) $(TARGET)-kmip.a $(LDFLAGS) $(LDFLAGS_KMIP) -Wl,-Bstatic -lsed -Wl,-Bdynamic -o $@

libkmip.a:
	@pushd . > /dev/null && cd libkmip && make $@ && popd > /dev/null

#
# Static library
#
$(LIB).a: $(patsubst %,$(LIBOBJDIR)%,$(LIBOBJS))
	@echo "  AR " $@
	@ar rcs $@ $^

#
# Shared library
#
$(LIB).so: $(LIB).a
	@echo "  LD " $@
	@$(CC) -shared -Wl,-soname,$@.1 -Wl,--whole-archive $(LIB).a -Wl,--no-whole-archive -o $@.1.0.1
	@ln -sf $@.1.0.1 $@.1
	@ln -sf $@.1.0.1 $@

#
# Main targets for compilation
#
$(OBJDIR)%.o: %.c
	@echo "  CC " $<
	@mkdir -p $(dir $@)
	@$(CC) -c $(CFLAGS) -MMD -MP -MF"$(@:%.o=%.d)" -MT"$(@:%.o=%.d)" -o "$@" "$<"

$(LIBOBJDIR)%.o: lib/%.c
	@echo "  CC " $<
	@mkdir -p $(dir $@)
	@$(CC) -c $(CFLAGS) -shared -fPIC -MMD -MP -MF"$(@:%.o=%.d)" -MT"$(@:%.o=%.d)" -o "$@" "$<"

clean:
	@echo "  CLEAN "
	@rm -f *.a $(TARGET)-kmip $(TARGET)-static $(TARGET)-dynamic $(TARGET) *.so*
	@rm -fr $(OBJDIR) $(LIBOBJDIR)
	@rm -f $(shell find -name \*.d) $(shell find -name \*.o)

distclean: clean
	@rm -f config.log config.mk config.h

install-$(TARGET):
	@echo " Installing $(TARGET)"
	install -m 755 $(TARGET)-dynamic /usr/sbin/$(TARGET)
	install -m 755 $(LIB).so.1.0.1 $(LIB_DIR)
	ln -sf $(LIB_DIR)/$(LIB).so.1.0.1 $(LIB_DIR)/$(LIB).so.1
	install -m 644 ../doc/$(TARGET).8 /usr/share/man/man8/$(TARGET).8

install-$(TARGET)-kmip: install-$(TARGET)
	@echo " Installing $(TARGET)-kmip"
	install -m 755 $(TARGET)-kmip /usr/sbin/$(TARGET)-kmip
	install -m 644 ../etc/udev/rules.d/63-sedcli.rules /etc/udev/rules.d/
	install -m 755 -d /etc/sedcli /etc/sedcli/certs
	install -m 644 ../etc/sedcli/sedcli.conf /etc/sedcli/
	install -m 644 ../doc/$(TARGET)-kmip.8 /usr/share/man/man8/$(TARGET)-kmip.8
	touch /etc/sedcli/sedcli_kmip && chmod 644 /etc/sedcli/sedcli_kmip

install-cert:
	install -m 644 ../certs/ca/ca_cert.pem /etc/sedcli/certs/
	install -m 644 ../certs/client/client_cert.pem /etc/sedcli/certs/
	install -m 644 ../certs/client/client_key.pem /etc/sedcli/certs/

install: $(INSTALL_TARGETS)

uninstall:
	@echo "  Removing $(TARGET)"
	-rm /usr/sbin/$(TARGET)
	-rm $(LIB_DIR)/$(LIB).so*
	-rm /usr/share/man/man8/$(TARGET).8
	-rm /usr/share/man/man8/$(TARGET)-kmip.8
	-rm /usr/sbin/$(TARGET)-kmip
	-rm /etc/udev/rules.d/63-sedcli.rules
	-rm /etc/sedcli/sedcli.conf

uninstall-cert:
	-rm /etc/sedcli/certs/ca_cert.pem
	-rm /etc/sedcli/certs/client_cert.pem
	-rm /etc/sedcli/certs/client_key.pem

.PHONY: clean all distclean install uninstall

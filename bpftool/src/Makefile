# SPDX-License-Identifier: (GPL-2.0-only OR BSD-2-Clause)
include Makefile.include

ifeq ($(srctree),)
srctree := $(patsubst %/,%,$(dir $(CURDIR)))
endif

ifeq ($(V),1)
  Q =
else
  Q = @
endif

BPF_DIR = $(srctree)/libbpf/src

ifneq ($(OUTPUT),)
  _OUTPUT := $(OUTPUT)
else
  _OUTPUT := $(CURDIR)/
endif
BOOTSTRAP_OUTPUT := $(_OUTPUT)bootstrap/

LIBBPF_OUTPUT := $(_OUTPUT)libbpf/
LIBBPF_DESTDIR := $(LIBBPF_OUTPUT)
LIBBPF_INCLUDE := $(LIBBPF_DESTDIR)include
LIBBPF_HDRS_DIR := $(LIBBPF_INCLUDE)/bpf
LIBBPF := $(LIBBPF_OUTPUT)libbpf.a

LIBBPF_BOOTSTRAP_OUTPUT := $(BOOTSTRAP_OUTPUT)libbpf/
LIBBPF_BOOTSTRAP_DESTDIR := $(LIBBPF_BOOTSTRAP_OUTPUT)
LIBBPF_BOOTSTRAP_INCLUDE := $(LIBBPF_BOOTSTRAP_DESTDIR)include
LIBBPF_BOOTSTRAP_HDRS_DIR := $(LIBBPF_BOOTSTRAP_INCLUDE)/bpf
LIBBPF_BOOTSTRAP := $(LIBBPF_BOOTSTRAP_OUTPUT)libbpf.a

# We need to copy hashmap.h, nlattr.h, relo_core.h and libbpf_internal.h
# which are not otherwise exported by libbpf, but still required by bpftool.
LIBBPF_INTERNAL_HDRS := $(addprefix $(LIBBPF_HDRS_DIR)/,hashmap.h nlattr.h relo_core.h libbpf_internal.h)
LIBBPF_BOOTSTRAP_INTERNAL_HDRS := $(addprefix $(LIBBPF_BOOTSTRAP_HDRS_DIR)/,hashmap.h relo_core.h libbpf_internal.h)

$(LIBBPF_OUTPUT) $(BOOTSTRAP_OUTPUT) $(LIBBPF_BOOTSTRAP_OUTPUT) $(LIBBPF_HDRS_DIR) $(LIBBPF_BOOTSTRAP_HDRS_DIR):
	$(QUIET_MKDIR)mkdir -p $@

$(LIBBPF): $(wildcard $(BPF_DIR)/*.[ch] $(BPF_DIR)/Makefile) | $(LIBBPF_OUTPUT)
	$(Q)$(MAKE) -C $(BPF_DIR) OBJDIR=$(patsubst %/,%,$(LIBBPF_OUTPUT)) \
		PREFIX=$(LIBBPF_DESTDIR:/=) $(LIBBPF) install_headers

$(LIBBPF_INTERNAL_HDRS): $(LIBBPF_HDRS_DIR)/%.h: $(BPF_DIR)/%.h | $(LIBBPF_HDRS_DIR)
	$(call QUIET_INSTALL, $@)
	$(Q)install -m 644 -t $(LIBBPF_HDRS_DIR) $<

$(LIBBPF_BOOTSTRAP): $(wildcard $(BPF_DIR)/*.[ch] $(BPF_DIR)/Makefile) | $(LIBBPF_BOOTSTRAP_OUTPUT)
	$(Q)$(MAKE) -C $(BPF_DIR) OBJDIR=$(patsubst %/,%,$(LIBBPF_BOOTSTRAP_OUTPUT)) \
		PREFIX=$(LIBBPF_BOOTSTRAP_DESTDIR:/=) \
		ARCH= CROSS_COMPILE= CC=$(HOSTCC) LD=$(HOSTLD) AR=$(HOSTAR) $@ install_headers

$(LIBBPF_BOOTSTRAP_INTERNAL_HDRS): $(LIBBPF_BOOTSTRAP_HDRS_DIR)/%.h: $(BPF_DIR)/%.h | $(LIBBPF_BOOTSTRAP_HDRS_DIR)
	$(call QUIET_INSTALL, $@)
	$(Q)install -m 644 -t $(LIBBPF_BOOTSTRAP_HDRS_DIR) $<

$(LIBBPF)-clean: FORCE | $(LIBBPF_OUTPUT)
	$(call QUIET_CLEAN, libbpf)
	$(Q)$(MAKE) -C $(BPF_DIR) OUTPUT=$(LIBBPF_OUTPUT) clean >/dev/null

$(LIBBPF_BOOTSTRAP)-clean: FORCE | $(LIBBPF_BOOTSTRAP_OUTPUT)
	$(call QUIET_CLEAN, libbpf-bootstrap)
	$(Q)$(MAKE) -C $(BPF_DIR) OUTPUT=$(LIBBPF_BOOTSTRAP_OUTPUT) clean >/dev/null

prefix ?= /usr/local
bash_compdir ?= /usr/share/bash-completion/completions

CFLAGS += -O2
CFLAGS += -W -Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers
CFLAGS += $(filter-out -Wswitch-enum -Wnested-externs,$(EXTRA_WARNINGS))
CFLAGS += -DPACKAGE='"bpftool"' -D__EXPORTED_HEADERS__ \
	-I$(or $(OUTPUT),.) \
	-I$(LIBBPF_INCLUDE) \
	-I$(srctree)/src/kernel/bpf/ \
	-I$(srctree)/include \
	-I$(srctree)/include/uapi
ifneq ($(BPFTOOL_VERSION),)
CFLAGS += -DBPFTOOL_VERSION='"$(BPFTOOL_VERSION)"'
endif
ifneq ($(EXTRA_CFLAGS),)
CFLAGS += $(EXTRA_CFLAGS)
endif
ifneq ($(EXTRA_LDFLAGS),)
LDFLAGS += $(EXTRA_LDFLAGS)
endif

INSTALL ?= install
RM ?= rm -f

FEATURE_USER = .bpftool
FEATURE_TESTS = libbfd libbfd-liberty libbfd-liberty-z \
	disassembler-four-args disassembler-init-styled libcap \
	clang-bpf-co-re
FEATURE_DISPLAY = libbfd libbfd-liberty libbfd-liberty-z \
	libcap clang-bpf-co-re

check_feat := 1
NON_CHECK_FEAT_TARGETS := clean uninstall doc doc-clean doc-install doc-uninstall
ifdef MAKECMDGOALS
ifeq ($(filter-out $(NON_CHECK_FEAT_TARGETS),$(MAKECMDGOALS)),)
  check_feat := 0
endif
endif

ifeq ($(check_feat),1)
include Makefile.feature
endif

ifeq ($(feature-disassembler-four-args), 1)
CFLAGS += -DDISASM_FOUR_ARGS_SIGNATURE
endif
ifeq ($(feature-disassembler-init-styled), 1)
    CFLAGS += -DDISASM_INIT_STYLED
endif

LIBS = $(LIBBPF) -lelf -lz
LIBS_BOOTSTRAP = $(LIBBPF_BOOTSTRAP) -lelf -lz
ifeq ($(feature-libcap), 1)
CFLAGS += -DUSE_LIBCAP
LIBS += -lcap
endif

include $(wildcard $(OUTPUT)*.d)

all: $(OUTPUT)bpftool

BFD_SRCS = jit_disasm.c

SRCS = $(filter-out $(BFD_SRCS),$(wildcard *.c))

ifeq ($(feature-libbfd),1)
  LIBS += -lbfd -ldl -lopcodes
else ifeq ($(feature-libbfd-liberty),1)
  LIBS += -lbfd -ldl -lopcodes -liberty
else ifeq ($(feature-libbfd-liberty-z),1)
  LIBS += -lbfd -ldl -lopcodes -liberty -lz
endif

ifneq ($(filter -lbfd,$(LIBS)),)
CFLAGS += -DHAVE_LIBBFD_SUPPORT
SRCS += $(BFD_SRCS)
endif

HOST_CFLAGS = $(subst -I$(LIBBPF_INCLUDE),-I$(LIBBPF_BOOTSTRAP_INCLUDE),\
		$(subst $(CLANG_CROSS_FLAGS),,$(CFLAGS)))

BPFTOOL_BOOTSTRAP := $(BOOTSTRAP_OUTPUT)bpftool

BOOTSTRAP_OBJS = $(addprefix $(BOOTSTRAP_OUTPUT),main.o common.o json_writer.o gen.o btf.o xlated_dumper.o btf_dumper.o disasm.o)
$(BOOTSTRAP_OBJS): $(LIBBPF_BOOTSTRAP)

OBJS = $(patsubst %.c,$(OUTPUT)%.o,$(SRCS)) $(OUTPUT)disasm.o
$(OBJS): $(LIBBPF) $(LIBBPF_INTERNAL_HDRS)

VMLINUX_BTF_PATHS ?= $(if $(O),$(O)/vmlinux)				\
		     $(if $(KBUILD_OUTPUT),$(KBUILD_OUTPUT)/vmlinux)	\
		     ../../../vmlinux					\
		     /sys/kernel/btf/vmlinux				\
		     /boot/vmlinux-$(shell uname -r)
VMLINUX_BTF ?= $(abspath $(firstword $(wildcard $(VMLINUX_BTF_PATHS))))

bootstrap: $(BPFTOOL_BOOTSTRAP)

ifneq ($(VMLINUX_BTF)$(VMLINUX_H),)
ifeq ($(feature-clang-bpf-co-re),1)

BUILD_BPF_SKELS := 1

$(OUTPUT)vmlinux.h: $(VMLINUX_BTF) $(BPFTOOL_BOOTSTRAP)
ifeq ($(VMLINUX_H),)
	$(QUIET_GEN)$(BPFTOOL_BOOTSTRAP) btf dump file $< format c > $@
else
	$(Q)cp "$(VMLINUX_H)" $@
endif

$(OUTPUT)%.bpf.o: skeleton/%.bpf.c $(OUTPUT)vmlinux.h $(LIBBPF_BOOTSTRAP)
	$(QUIET_CLANG)$(CLANG) \
		-I$(or $(OUTPUT),.) \
		-I$(srctree)/include/uapi/ \
		-I$(LIBBPF_BOOTSTRAP_INCLUDE) \
		-g -O2 -Wall -target bpf -c $< -o $@
	$(Q)$(LLVM_STRIP) -g $@

$(OUTPUT)%.skel.h: $(OUTPUT)%.bpf.o $(BPFTOOL_BOOTSTRAP)
	$(QUIET_GEN)$(BPFTOOL_BOOTSTRAP) gen skeleton $< > $@

$(OUTPUT)prog.o: $(OUTPUT)profiler.skel.h

$(OUTPUT)pids.o: $(OUTPUT)pid_iter.skel.h

endif
endif

CFLAGS += $(if $(BUILD_BPF_SKELS),,-DBPFTOOL_WITHOUT_SKELETONS)

$(BOOTSTRAP_OUTPUT)disasm.o: $(srctree)/src/kernel/bpf/disasm.c
	$(QUIET_CC)$(HOSTCC) $(HOST_CFLAGS) -c -MMD $< -o $@

$(OUTPUT)disasm.o: $(srctree)/src/kernel/bpf/disasm.c
	$(QUIET_CC)$(CC) $(CFLAGS) -c -MMD $< -o $@

$(BPFTOOL_BOOTSTRAP): $(BOOTSTRAP_OBJS) $(LIBBPF_BOOTSTRAP)
	$(QUIET_LINK)$(HOSTCC) $(HOST_CFLAGS) $(LDFLAGS) $(BOOTSTRAP_OBJS) $(LIBS_BOOTSTRAP) -o $@

$(OUTPUT)bpftool: $(OBJS) $(LIBBPF)
	$(QUIET_LINK)$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) $(LIBS) -o $@

$(BOOTSTRAP_OUTPUT)%.o: %.c $(LIBBPF_BOOTSTRAP_INTERNAL_HDRS) | $(BOOTSTRAP_OUTPUT)
	$(QUIET_CC)$(HOSTCC) $(HOST_CFLAGS) -c -MMD $< -o $@

$(OUTPUT)%.o: %.c
	$(QUIET_CC)$(CC) $(CFLAGS) -c -MMD $< -o $@

feature-detect-clean:
	$(call QUIET_CLEAN, feature-detect)
	$(Q)$(MAKE) -C $(srctree)/tools/build/feature/ clean >/dev/null

clean: $(LIBBPF)-clean $(LIBBPF_BOOTSTRAP)-clean
	$(call QUIET_CLEAN, bpftool)
	$(Q)$(RM) -- $(OUTPUT)bpftool $(OUTPUT)*.o $(OUTPUT)*.d
	$(Q)$(RM) -- $(OUTPUT)*.skel.h $(OUTPUT)vmlinux.h
	$(Q)$(RM) -r -- $(LIBBPF_OUTPUT) $(BOOTSTRAP_OUTPUT)
	$(call QUIET_CLEAN, core-gen)
	$(Q)$(RM) -- $(OUTPUT)FEATURE-DUMP.bpftool
	$(Q)$(RM) -r -- $(OUTPUT)feature/

install-bin: $(OUTPUT)bpftool
	$(call QUIET_INSTALL, bpftool)
	$(Q)$(INSTALL) -m 0755 -d $(DESTDIR)$(prefix)/sbin
	$(Q)$(INSTALL) $(OUTPUT)bpftool $(DESTDIR)$(prefix)/sbin/bpftool

install: install-bin
	$(Q)$(INSTALL) -m 0755 -d $(DESTDIR)$(bash_compdir)
	$(Q)$(INSTALL) -m 0644 $(srctree)/bash-completion/bpftool $(DESTDIR)$(bash_compdir)

uninstall:
	$(call QUIET_UNINST, bpftool)
	$(Q)$(RM) -- $(DESTDIR)$(prefix)/sbin/bpftool
	$(Q)$(RM) -- $(DESTDIR)$(bash_compdir)/bpftool

doc:
	$(call descend,$(srctree)/docs)

doc-clean:
	$(call descend,$(srctree)/docs,clean)

doc-install:
	$(call descend,$(srctree)/docs,install)

doc-uninstall:
	$(call descend,$(srctree)/docs,uninstall)

FORCE:

.SECONDARY:
.PHONY: all FORCE bootstrap clean install-bin install uninstall
.PHONY: doc doc-clean doc-install doc-uninstall
.DEFAULT_GOAL := all

---
title: DPDK的makefile学习
date: 2017-11-09 15:33:24
tags: DPDK源码解析
categories:
- Makefile
- DPDK
---

本文介绍一下DPDK的makefile架构。

<!-- more -->
# DPDK的makefile架构
DPDK采用GNUmakefile，并且在source_dir/mk/文件夹中有一系列预定义的文件帮助用户快捷的构建自己的DPDK app, lib等。

# DPDK的编译
## 编译DPDK的方法
```bash
#在DPDK的根目录执行：
make config T=x86_64-native-linuxapp-gcc
make
```

下面我们来看，运行make config 和make具体都干了些什么。

---

### 1.我们看根目录下的GNUmakefile  
```makefile
RTE_SDK := $(CURDIR)
export RTE_SDK

#
# directory list
#

ROOTDIRS-y := buildtools lib drivers app
ROOTDIRS-  := test

include $(RTE_SDK)/mk/rte.sdkroot.mk
```
首先将当前的根目录定义为RTE_SDK,并export;  
然后将需要编译的buildtools, lib, drivers, app目录定义为ROOTDIRS-y,  
将test目录定义为ROOTDIRS-。  

然后include `$(RTE_SDK)/mk/rte.sdkroot.mk`

### 2.在rte.sdkroot.mk中，定义并export  
  - Q='@'
  - RTE_SRCDIR=$(CURDIR)
  - BUILDING_RTE_SDK = 1
  - RTE_CONFIG_TEMPLATE = $(RTE_SRCDIR)/config/defconfig_$(T) 其中T为命令行传入
  - RTE_OUTPUT = $(O) 如果命令行有"O="则为命令行参数，否则为默认的$(RTE_SRCDIR/build)
  - BUILDDIR = $(RTE_OUTPUT)/build

```makefile
MAKEFLAGS += --no-print-directory

# define Q to '@' or not. $(Q) is used to prefix all shell commands to
# be executed silently.
Q=@
ifeq '$V' '0'
override V=
endif
ifdef V
ifeq ("$(origin V)", "command line")
Q=
endif
endif
export Q

ifeq ($(RTE_SDK),)
$(error RTE_SDK is not defined)
endif

RTE_SRCDIR = $(CURDIR)
export RTE_SRCDIR

BUILDING_RTE_SDK := 1
export BUILDING_RTE_SDK

#
# We can specify the configuration template when doing the "make
# config". For instance: make config T=x86_64-native-linuxapp-gcc
#
RTE_CONFIG_TEMPLATE :=
ifdef T
ifeq ("$(origin T)", "command line")
RTE_CONFIG_TEMPLATE := $(RTE_SRCDIR)/config/defconfig_$(T)
endif
endif
export RTE_CONFIG_TEMPLATE

#
# Default output is $(RTE_SRCDIR)/build
# output files wil go in a separate directory
#
ifdef O
ifeq ("$(origin O)", "command line")
RTE_OUTPUT := $(abspath $(O))
endif
endif
RTE_OUTPUT ?= $(RTE_SRCDIR)/build
export RTE_OUTPUT

# the directory where intermediate build files are stored, like *.o,
# *.d, *.cmd, ...
BUILDDIR = $(RTE_OUTPUT)/build
export BUILDDIR

export ROOTDIRS-y ROOTDIRS- ROOTDIRS-n
```

### 3.看`make config T=x86_64-native-linuxapp-gcc`的流程  
在上一步中，我们看到在rte.sdkroot.mk中对T的定义进行了处理，对RTE_CONFIG_TEMPLATE进行了赋值，然后进入了config对应的目标
```makefile
.PHONY: config defconfig showconfigs showversion showversionum
config defconfig showconfigs showversion showversionum:
	$(Q)$(MAKE) -f $(RTE_SDK)/mk/rte.sdkconfig.mk $@
```
即采用$(RTE_SDK)/mk/rte.sdkconfig.mk进行make config操作  

- `make config` in `rte.sdkconfig.mk`
```makefile
.PHONY: config
ifeq ($(RTE_CONFIG_TEMPLATE),)
config: notemplate
else
config: $(RTE_OUTPUT)/include/rte_config.h $(RTE_OUTPUT)/Makefile
	@echo "Configuration done using" \
		$(patsubst defconfig_%,%,$(notdir $(RTE_CONFIG_TEMPLATE)))
endif
```
 可以看到会执行，`$(RTE_OUTPUT)/include/rte_config.h`和`$(RTE_OUTPUT)/Makefile`


- ```makefile
# generate a Makefile for this build directory
# use a relative path so it will continue to work even if we move the directory
SDK_RELPATH=$(shell $(RTE_SDK)/buildtools/relpath.sh $(abspath $(RTE_SRCDIR)) \
				$(abspath $(RTE_OUTPUT)))
OUTPUT_RELPATH=$(shell $(RTE_SDK)/buildtools/relpath.sh $(abspath $(RTE_OUTPUT)) \
				$(abspath $(RTE_SRCDIR)))
$(RTE_OUTPUT)/Makefile: | $(RTE_OUTPUT)
	$(Q)$(RTE_SDK)/buildtools/gen-build-mk.sh $(SDK_RELPATH) $(OUTPUT_RELPATH) \
		> $(RTE_OUTPUT)/Makefile

# clean installed files, and generate a new config header file
# if NODOTCONF variable is defined, don't try to rebuild .config
$(RTE_OUTPUT)/include/rte_config.h: $(RTE_OUTPUT)/.config
	$(Q)rm -rf $(RTE_OUTPUT)/include $(RTE_OUTPUT)/app \
		$(RTE_OUTPUT)/lib \
		$(RTE_OUTPUT)/hostlib $(RTE_OUTPUT)/kmod $(RTE_OUTPUT)/build
	$(Q)mkdir -p $(RTE_OUTPUT)/include
	$(Q)$(RTE_SDK)/buildtools/gen-config-h.sh $(RTE_OUTPUT)/.config \
		> $(RTE_OUTPUT)/include/rte_config.h
```
 scripts目录下relpath.sh接受两个路径作为输入，然后获取两个路径间的相对路径。

- ```makefile
$(RTE_OUTPUT):
	$(Q)mkdir -p $@

ifdef NODOTCONF
$(RTE_OUTPUT)/.config: ;
else
# Generate config from template, if there are duplicates keep only the last.
# To do so the temp config is checked for duplicate keys with cut/sort/uniq
# Then for each of those identified duplicates as long as there are more than
# just one left the last match is removed.
$(RTE_OUTPUT)/.config: $(RTE_CONFIG_TEMPLATE) FORCE | $(RTE_OUTPUT)
	$(Q)if [ "$(RTE_CONFIG_TEMPLATE)" != "" -a -f "$(RTE_CONFIG_TEMPLATE)" ]; then \
		$(CPP) -undef -P -x assembler-with-cpp \
		-ffreestanding \
		-o $(RTE_OUTPUT)/.config_tmp $(RTE_CONFIG_TEMPLATE) ; \
		config=$$(cat $(RTE_OUTPUT)/.config_tmp) ; \
		echo "$$config" | awk -F '=' 'BEGIN {i=1} \
			/^#/ {pos[i++]=$$0} \
			!/^#/ {if (!s[$$1]) {pos[i]=$$0; s[$$1]=i++} \
				else {pos[s[$$1]]=$$0}} END \
			{for (j=1; j<i; j++) print pos[j]}' \
			> $(RTE_OUTPUT)/.config_tmp ; \
		if ! cmp -s $(RTE_OUTPUT)/.config_tmp $(RTE_OUTPUT)/.config; then \
			cp $(RTE_OUTPUT)/.config_tmp $(RTE_OUTPUT)/.config ; \
			cp $(RTE_OUTPUT)/.config_tmp $(RTE_OUTPUT)/.config.orig ; \
		fi ; \
		rm -f $(RTE_OUTPUT)/.config_tmp ; \
	else \
		$(MAKE) -rRf $(RTE_SDK)/mk/rte.sdkconfig.mk notemplate; \
	fi
endif
```
 CPP是Makefile内置变量，代表C语言预处理器，这里将会在RTE_OUTPUT目录下生成.config和.config.orig两个文件，最后生成了rte_config.h和Makefile文件。  
 生成完所有依赖文件后，执行make depdirs，这个目标在rte.sdkroot.mk中：

- ```makefile

```

### 3 make的流程
会执行rte.sdkroot.mk中的
```makefile
# all other build targets
%:
	$(Q)$(MAKE) -f $(RTE_SDK)/mk/rte.sdkconfig.mk checkconfig
	$(Q)$(MAKE) -f $(RTE_SDK)/mk/rte.sdkbuild.mk $@
```
首先会检查是否完成了.config的生成，然后对rte.sdkbuild.mk进行make。
```makefile
.PHONY: build
build: $(ROOTDIRS-y)
	@echo "Build complete [$(RTE_TARGET)]"

$(ROOTDIRS-y) $(ROOTDIRS-):
	@[ -d $(BUILDDIR)/$@ ] || mkdir -p $(BUILDDIR)/$@
	@echo "== Build $@"
	$(Q)$(MAKE) S=$@ -f $(RTE_SRCDIR)/$@/Makefile -C $(BUILDDIR)/$@ all
	@if [ $@ = drivers ]; then \
		$(MAKE) -f $(RTE_SDK)/mk/rte.combinedlib.mk; \
	fi
```

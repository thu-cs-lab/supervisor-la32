RM	:= rm -f
GCCPREFIX	?= loongarch32-unknown-linux-gnu-
QEMU := qemu-system-loongarch64
LD	:= $(GCCPREFIX)ld
CC	:= $(GCCPREFIX)gcc
MODULE	:= init
OBJDIR	:= obj
INCLUDE	:= include
SRCDIR	:= kern
LDSCRIPT	:= kern/kernel.ld
SRC	:= $(foreach sdir, $(SRCDIR), $(wildcard $(sdir)/*.S))
OBJ	:= $(patsubst $(SRCDIR)/%.S, $(OBJDIR)/%.o, $(SRC))
TARGET	:= kernel.elf
ASFLAG	:= -D__ASSEMBLY__ -g

override EN_INT ?= n
ifeq ($(EN_INT), y)
	ASFLAG += -DENABLE_INT
	EN_TLB = n
endif

override EN_TLB ?= n
ifeq ($(EN_TLB), y)
	ASFLAG += -DENABLE_TLB
endif

override EN_IBAR ?= n
ifeq ($(EN_IBAR), y)
	ASFLAG += -DENABLE_IBAR
endif

override EN_CACOP ?= n
ifeq ($(EN_CACOP), y)
	ASFLAG += -DENABLE_CACOP
endif

CONFIG_FILE := .config_$(EN_INT)_$(EN_TLB)_$(EN_IBAR)_$(EN_CACOP)

.PHONY: all clean checkdirs show-utest commits-num sim

all: $(TARGET) kernel.bin show-utest

$(CONFIG_FILE):
	@rm -f .config_*
	touch $@

$(TARGET): checkdirs $(OBJ) $(LDSCRIPT)
	$(LD) $(OBJ) -T$(LDSCRIPT)

$(OBJ): $(OBJDIR)/%.o : $(SRCDIR)/%.S $(CONFIG_FILE)
	$(CC) -g -o $@ $< -c -I$(INCLUDE) $(ASFLAG)


checkdirs: $(OBJDIR)

$(OBJDIR):
	test -d $@ || mkdir $@

kernel.bin: $(TARGET)
	@$(GCCPREFIX)objcopy -j .text -j .rodata -O binary -v $< kernel.bin

show-utest: $(TARGET)
	@$(GCCPREFIX)objdump -D $< | grep "^[0-9a-f]\{8\} <UTEST_" | grep -n --color=auto 'UTEST_'

sim: kernel.bin
	$(QEMU) -cpu la132-loongarch-cpu -m 1G -bios $< -nographic -monitor none -serial tcp::6666,server -s

debug: kernel.bin
	$(QEMU) -cpu la132-loongarch-cpu -m 1G -bios $< -nographic -monitor none -serial tcp::6666,server -S -s

inst: $(TARGET)
	@$(GCCPREFIX)objdump -S $(TARGET) -M no-aliases | awk '{print $$3}' | sort | uniq | grep -v section | grep -v format

commits-num:
	@git log --pretty=format:"commit %H" | grep -c 'commit'

clean:
	-$(RM) -r $(OBJDIR)

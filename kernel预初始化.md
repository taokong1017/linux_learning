# 内核预初始化分析

&emsp;&emsp;本文对ARM64 linux内核预初始化流程进行了分析，内容包括: 入口函数分析，主流程进入，主流程切换，以及内核初始化启动。

## 入口函数分析

&emsp;&emsp;ARM64的初始化入口函数是efi_signature_nop, 并在arch/arm64/kernel/head.S文件内使用，具体如下：

```C
__HEAD
/*
 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
 */
efi_signature_nop         // special NOP to identity as PE/COFF executable
b   primary_entry         // branch to kernel start, magic
.quad    0                // Image load offset from start of RAM, little-endian
le64sym _kernel_size_le   // Effective size of kernel image, little-endian
le64sym _kernel_flags_le  // Informative flags, little-endian
.quad    0                // reserved
.quad    0                // reserved
.quad    0                // reserved
.ascii ARM64_IMAGE_MAGIC  // Magic number
.long .Lpe_header_offset  // Offset to the PE header.
```

&emsp;&emsp;宏__HEAD定义在include/linux/init.h中, 具体如下：

```C
#define __HEAD  .section  ".head.text","ax"
``` 

&emsp;&emsp;linux ARM64使用的链接脚本是arch/arm64/boot/vmlinux.lds，包括初始化代码段".head.text"和代码段".text"，具体如下：

```C
. = ((((-(((1)) << (((48)) - 1)))) + (0x80000000)));
.head.text : {
    _text = .;
    KEEP(*(.head.text))
}
.text : ALIGN(0x00010000) {
    _stext = .;
    . = ALIGN(4); __irqentry_text_start = .; *(.irqentry.text) __irqentry_text_end = .;
    . = ALIGN(4); __softirqentry_text_start = .; *(.softirqentry.text) __softirqentry_text_end = .;
    . = ALIGN(4); __entry_text_start = .; *(.entry.text) __entry_text_end = .;
    . = ALIGN(4); *(.text.hot .text.hot.*) *(.text .text.fixup) *(.text.unlikely .text.unlikely.*) *(.text.unknown .text.unknown.*) . = ALIGN(4); __noinstr_text_start = .; *(.noinstr.text) __cpuidle_text_start = .; *(.cpuidle.text) __cpuidle_text_end = .; __noinstr_text_end = .; *(.ref.text) *(.text.asan.* .text.tsan.*) *(.meminit.text*)
    . = ALIGN(4); __sched_text_start = .; *(.sched.text) __sched_text_end = .;
    . = ALIGN(4); __lock_text_start = .; *(.spinlock.text) __lock_text_end = .;
    . = ALIGN(4); __kprobes_text_start = .; *(.kprobes.text) __kprobes_text_end = .;
    . = ALIGN((1 << 12)); __hyp_idmap_text_start = .; *(.hyp.idmap.text) __hyp_idmap_text_end = .; __hyp_text_start = .; *(.hyp.text) . = ALIGN(0x00000008); __start___kvm_ex_table = .; *(__kvm_ex_table) __stop___kvm_ex_table = .; . = ALIGN((1 << 12)); __hyp_text_end = .;
    *(.gnu.warning)
 }
```

## 主流程进入阶段

&emsp;&emsp;主流程进入流程，包含MMU状态记录、启动参数保存、获取早期内核栈、地址翻译表建立、dcache无效化、内核异常等级初始化、CPU建立。

### MMU状态记录

&emsp;&emsp;MMU状态记录函数功能，是根据异常级别，通过系统控制寄存器，设置小端模式，关闭Cache并禁止MMU，然后再将MMU状态记录到x19寄存器。

```C
SYM_CODE_START_LOCAL(record_mmu_state)
    mrs x19, CurrentEL
    cmp x19, #CurrentEL_EL2
    mrs x19, sctlr_el1
    b.ne 0f
    mrs x19, sctlr_el2
0:
    CPU_LE(tbnz x19, #SCTLR_ELx_EE_SHIFT, 1f)
    CPU_BE(tbz x19, #SCTLR_ELx_EE_SHIFT, 1f)
    tst x19, #SCTLR_ELx_C        // Z := (C == 0)
    and x19, x19, #SCTLR_ELx_M   // isolate M bit
    csel    x19, xzr, x19, eq    // clear x19 if Z
    ret

    /*
     * Set the correct endianness early so all memory accesses issued
     * before init_kernel_el() occur in the correct byte order. Note that
     * this means the MMU must be disabled, or the active ID map will end
     * up getting interpreted with the wrong byte order.
     */
1:  
    eor x19, x19, #SCTLR_ELx_EE
    bic x19, x19, #SCTLR_ELx_M
    b.ne 2f
    pre_disable_mmu_workaround
    msr sctlr_el2, x19
    b 3f
2:
    pre_disable_mmu_workaround
    msr sctlr_el1, x19
3:
    isb
    mov x19, xzr
    ret
SYM_CODE_END(record_mmu_state)
```

### 启动参数保存

&emsp;&emsp;启动参数保存函数的功能是，将bootloader的参数x0、x1、x2、x3以及x19保存到全局变量boot_args中。在这过程中，如果MMU已使能，则需要将boot_args地址空间的前32个字节进行数据无效化操作。

```C
SYM_CODE_START_LOCAL(preserve_boot_args)
    mov x21, x0             // x21=FDT

    adr_l x0, boot_args     // record the contents of
    stp x21, x1, [x0]       // x0 .. x3 at kernel entry
    stp x2, x3, [x0, #16]
    cbnz x19, 0f           // skip cache invalidation if MMU is on
    dmb sy                 // needed before dc ivac with MMU off
    add x1, x0, #0x20      // 4 x 8 bytes
    b dcache_inval_poc     // tail call
0:  str_l x19, mmu_enabled_at_boot, x0
    ret
SYM_CODE_END(preserve_boot_args)
```

### 早期内核栈获取

&emsp;&emsp;即获取内核早期运行的栈，由链接器指定首地址和大小。

```C
adrp x1, early_init_stack
mov sp, x1
```

### 地址翻译表建立

&emsp;&emsp;

## 主流程切换阶段

## 内核启动阶段

本节内容将放到内核初始化进行分析。

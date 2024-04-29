# 内核预初始化流程分析

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

&emsp;&emsp;主流程进入，包含MMU状态记录、启动参数保存、早期内核栈获取、地址翻译表建立、内核异常等级初始化和CPU建立。

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

&emsp;&emsp;启动参数保存函数功能，将bootloader的参数x0、x1、x2、x3以及x19保存到全局变量boot_args中。在这过程中，如果MMU已使能，则需要将boot_args地址空间的前32个字节进行数据无效化操作。

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

&emsp;&emsp;即获取早期运行的内核栈，且栈的起始地址和大小均由连接器指定。

```C
adrp x1, early_init_stack
mov sp, x1
```

### 地址翻译表建立

&emsp;&emsp;内核在使能MMU之前，须建立数据、代码对应的虚拟地址和物理地址的映射。
&emsp;&emsp;首先，获取linux页表项的首地址，方法如下：

```C
adrp x0, init_idmap_pg_dir      #x0 is the end of pte
mov x1, xzr
bl __pi_create_init_idmap
```

&emsp;&emsp;其次，分别对代码段和数据段进行1:1的内存页表映射，具体如下：

```C
asmlinkage u64 __init create_init_idmap(pgd_t *pg_dir, pteval_t clrmask)
{
    u64 ptep = (u64)pg_dir + PAGE_SIZE;
    pgprot_t text_prot = PAGE_KERNEL_ROX;
    pgprot_t data_prot = PAGE_KERNEL;

    pgprot_val(text_prot) &= ~clrmask;
    pgprot_val(data_prot) &= ~clrmask;

    map_range(&ptep, (u64)_stext, (u64)__initdata_begin, (u64)_stext, text_prot, IDMAP_ROOT_LEVEL, (pte_t *)pg_dir, false, 0);
    map_range(&ptep, (u64)__initdata_begin, (u64)_end, (u64)__initdata_begin, data_prot, IDMAP_ROOT_LEVEL, (pte_t *)pg_dir, false, 0);

    return ptep;
}
```

&emsp;&emsp;map_range的函数原型如下：

```c
void __init map_range(u64 *pte, u64 start, u64 end, u64 pa, pgprot_t prot, int level, pte_t *tbl, bool may_use_cont, u64 va_offset)
```

&emsp;&emsp;最后，在将映射的页表无效化，具体如下：

```C
cbnz x19, 0f
dmb sy
mov x1, x0                  // end of used region
adrp x0, init_idmap_pg_dir
adr_l x2, dcache_inval_poc
blr x2
b 1f
```

&emsp;&emsp;对于函数dcache_inval_poc的原型如下：

```c
SYM_FUNC_ALIAS(dcache_inval_poc, __pi_dcache_inval_poc)
SYM_FUNC_START(__pi_dcache_inval_poc)
   dcache_line_size x2, x3
   sub x3, x2, #1
   tst x1, x3            // end cache line aligned?
   bic x1, x1, x3
   b.eq 1f
   dc civac, x1          // clean & invalidate D / U line
1: tst x0, x3            // start cache line aligned?
   bic x0, x0, x3
   b.eq 2f
   dc civac, x0          // clean & invalidate D / U line
   b 3f
2: dc ivac, x0           // invalidate D / U line
3: add x0, x0, x2
   cmp x0, x1
   b.lo 2b
   dsb sy
   ret
SYM_FUNC_END(__pi_dcache_inval_poc)
```

### 内核异常等级初始化

&emsp;&emsp;CPU异常等级初始化，主要完成内核最高异常级别的CPU配置，对于EL1和EL2，它们的处理方式不同，具体详见函数init_kernel_el。

```C
1: mov x0, x19
   bl init_kernel_el    // w0=cpu_boot_mode
   mov x20, x0
```

&emsp;&emsp;函数init_kernel_el的模型如下：

```C
SYM_FUNC_START(init_kernel_el)
   mrs x1, CurrentEL
   cmp x1, #CurrentEL_EL2
   b.eq init_el2

SYM_INNER_LABEL(init_el1, SYM_L_LOCAL)
   mov_q x0, INIT_SCTLR_EL1_MMU_OFF  #设置小端序(LE)、支持未对齐的内存访问、关闭TLS(Thread Local Storage)寄存器的动态检测、使能异常终止、 非安全计数器和流水线终止
   pre_disable_mmu_workaround
   msr sctlr_el1, x0
   isb
   mov_q x0, INIT_PSTATE_EL1         #使能中断禁止位、异步终止位、IRQ中断位、FIQ中断位和EL1的64位模式
   msr spsr_el1, x0
   msr elr_el1, lr
   mov w0, #BOOT_CPU_MODE_EL1        #异常级别EL1、64位模式、不使用SIMD/浮点指令、使用安全扩展
   eret

SYM_INNER_LABEL(init_el2, SYM_L_LOCAL)
   msr elr_el2, lr

   cbz x0, 0f
   adrp x0, __hyp_idmap_text_start
   adr_l x1, __hyp_text_end
   adr_l x2, dcache_clean_poc
   blr x2

   mov_q x0, INIT_SCTLR_EL2_MMU_OFF
   pre_disable_mmu_workaround
   msr sctlr_el2, x0
   isb
0:
   mov_q x0, HCR_HOST_NVHE_FLAGS

   /*
    * Compliant CPUs advertise their VHE-onlyness with
    * ID_AA64MMFR4_EL1.E2H0 < 0. HCR_EL2.E2H can be
    * RES1 in that case. Publish the E2H bit early so that
    * it can be picked up by the init_el2_state macro.
    *
    * Fruity CPUs seem to have HCR_EL2.E2H set to RAO/WI, but
    * don't advertise it (they predate this relaxation).
    */
   mrs_s x1, SYS_ID_AA64MMFR4_EL1
   tbz x1, #(ID_AA64MMFR4_EL1_E2H0_SHIFT + ID_AA64MMFR4_EL1_E2H0_WIDTH - 1), 1f

   orr x0, x0, #HCR_E2H
1:
   msr hcr_el2, x0
   isb

   init_el2_state

   /* Hypervisor stub */
   adr_l x0, __hyp_stub_vectors
   msr vbar_el2, x0
   isb

   mov_q x1, INIT_SCTLR_EL1_MMU_OFF

   mrs x0, hcr_el2
   and x0, x0, #HCR_E2H
   cbz x0, 2f

   /* Set a sane SCTLR_EL1, the VHE way */
   msr_s SYS_SCTLR_EL12, x1
   mov x2, #BOOT_CPU_FLAG_E2H
   b 3f

2:
   msr sctlr_el1, x1
   mov x2, xzr
3:
   __init_el2_nvhe_prepare_eret

   mov w0, #BOOT_CPU_MODE_EL2
   orr x0, x0, x2
   eret
SYM_FUNC_END(init_kernel_el)
```

### CPU建立

&emsp;&emsp;CPU建立的主要功能是，初始化MMU和内存管理相关的关键寄存器，为启动内存管理功能做准备。

```c
SYM_FUNC_START(__cpu_setup)
   tlbi vmalle1                 // Invalidate local TLB
   dsb nsh

   msr cpacr_el1, xzr          // Reset cpacr_el1
   mov x1, #1 << 12            // Reset mdscr_el1 and disable
   msr mdscr_el1, x1           // access to the DCC from EL0
   isb                        // Unmask debug exceptions now,
   enable_dbg                 // since this is per-cpu
   reset_pmuserenr_el0 x1     // Disable PMU access from EL0
   reset_amuserenr_el0 x1     // Disable AMU access from EL0

   /*
    * Default values for VMSA control registers. These will be adjusted
    * below depending on detected CPU features.
    */
   mair .req x17
   tcr .req x16
   mov_q mair, MAIR_EL1_SET
   mov_q tcr, TCR_T0SZ(IDMAP_VA_BITS) | TCR_T1SZ(VA_BITS_MIN) | TCR_CACHE_FLAGS | \
     TCR_SMP_FLAGS | TCR_TG_FLAGS | TCR_KASLR_FLAGS | TCR_ASID16 | \
     TCR_TBI0 | TCR_A1 | TCR_KASAN_SW_FLAGS | TCR_MTE_FLAGS

   tcr_clear_errata_bits tcr, x9, x5

#ifdef CONFIG_ARM64_VA_BITS_52
   mov x9, #64 - VA_BITS
alternative_if ARM64_HAS_VA52
   tcr_set_t1sz tcr, x9
#ifdef CONFIG_ARM64_LPA2
   orr tcr, tcr, #TCR_DS
#endif
alternative_else_nop_endif
#endif

   /*
    * Set the IPS bits in TCR_EL1.
    */
   tcr_compute_pa_size tcr, #TCR_IPS_SHIFT, x5, x6
#ifdef CONFIG_ARM64_HW_AFDBM
   /*
    * Enable hardware update of the Access Flags bit.
    * Hardware dirty bit management is enabled later,
    * via capabilities.
     */
    mrs x9, ID_AA64MMFR1_EL1
    and x9, x9, ID_AA64MMFR1_EL1_HAFDBS_MASK
    cbz x9, 1f
    orr tcr, tcr, #TCR_HA             // hardware Access flag update
1:
#endif /* CONFIG_ARM64_HW_AFDBM */
   msr mair_el1, mair
   msr tcr_el1, tcr

   mrs_s x1, SYS_ID_AA64MMFR3_EL1
   ubfx x1, x1, #ID_AA64MMFR3_EL1_S1PIE_SHIFT, #4
   cbz x1, .Lskip_indirection

   /*
    * The PROT_* macros describing the various memory types may resolve to
    * C expressions if they include the PTE_MAYBE_* macros, and so they
    * can only be used from C code. The PIE_E* constants below are also
    * defined in terms of those macros, but will mask out those
    * PTE_MAYBE_* constants, whether they are set or not. So #define them
    * as 0x0 here so we can evaluate the PIE_E* constants in asm context.
    */

#define PTE_MAYBE_NG 0
#define PTE_MAYBE_SHARED 0

   mov_q x0, PIE_E0
   msr REG_PIRE0_EL1, x0
   mov_q x0, PIE_E1
   msr REG_PIR_EL1, x0

#undef PTE_MAYBE_NG
#undef PTE_MAYBE_SHARED

   mov x0, TCR2_EL1x_PIE
   msr REG_TCR2_EL1, x0

.Lskip_indirection:

   /*
   * Prepare SCTLR
   */
   mov_q x0, INIT_SCTLR_EL1_MMU_ON
   ret                     // return to head.S

   .unreq mair
   .unreq tcr
SYM_FUNC_END(__cpu_setup)
```

## 主流程切换阶段

&emsp;&emsp;主流程切换包括3部分类型，MMU使能、早期内核映射和kernel_init切换

### MMU使能

&emsp;&emsp;主流程发生切换环前，需要使能MMU，具体代码如下：

```c
adrp x1, reserved_pg_dir
adrp x2, init_idmap_pg_dir
bl __enable_mmu
```

```c
__enable_mmu:
#if defined(CONFIG_ALIGNMENT_TRAP) && __LINUX_ARM_ARCH__ < 6
   orr r0, r0, #CR_A
#else
   bic r0, r0, #CR_A
#endif
#ifdef CONFIG_CPU_DCACHE_DISABLE
   bic r0, r0, #CR_C
#endif
#ifdef CONFIG_CPU_BPREDICT_DISABLE
   bic r0, r0, #CR_Z
#endif
#ifdef CONFIG_CPU_ICACHE_DISABLE
   bic r0, r0, #CR_I
#endif
#ifdef CONFIG_ARM_LPAE
   mcrr p15, 0, r4, r5, c2           @ load TTBR0
#else
   mov r5, #DACR_INIT
   mcr p15, 0, r5, c3, c0, 0         @ load domain access register
   mcr p15, 0, r4, c2, c0, 0         @ load page table pointer
#endif
   b __turn_mmu_on
ENDPROC(__enable_mmu)

ENTRY(__turn_mmu_on)
   mov r0, r0
   instr_sync
   mcr p15, 0, r0, c1, c0, 0        @ write control reg
   mrc p15, 0, r3, c0, c0, 0        @ read id reg
   instr_sync
   mov r3, r3
   mov r3, r13
   ret r3
__turn_mmu_on_end:
ENDPROC(__turn_mmu_on)
```

### 早期内核映射

&emsp;&emsp;早期内核映射，即根据启动参数和CPU特性，计算内核映射的地址，并调用底层函数完成内核的虚拟地址空间初始化

```c
adrp x1, early_init_stack
mov sp, x1
mov x29, xzr
mov x0, x20                 // pass the full boot status
mov x1, x21                 // pass the FDT
bl __pi_early_map_kernel    // Map and relocate the kernel
```

&emsp;&emsp;函数early_map_kernel内部包含启动内核地址随机化功能。当dbg调试时，需要将该功能关闭，具体代码如下：

```c
asmlinkage void __init early_map_kernel(u64 boot_status, void *fdt)
{
    static char const chosen_str[] __initconst = "/chosen";
    u64 va_base, pa_base = (u64)&_text;
    u64 kaslr_offset = pa_base % MIN_KIMG_ALIGN;
    int root_level = 4 - CONFIG_PGTABLE_LEVELS;
    int va_bits = VA_BITS;
    int chosen;

    map_fdt((u64)fdt);

    /* Clear BSS and the initial page tables */
    memset(__bss_start, 0, (u64)init_pg_end - (u64)__bss_start);

    /* Parse the command line for CPU feature overrides */
    chosen = fdt_path_offset(fdt, chosen_str);
    init_feature_override(boot_status, fdt, chosen);

    if (IS_ENABLED(CONFIG_ARM64_64K_PAGES) && !cpu_has_lva()) {
       va_bits = VA_BITS_MIN;
    } else if (IS_ENABLED(CONFIG_ARM64_LPA2) && !cpu_has_lpa2()) {
       va_bits = VA_BITS_MIN;
       root_level++;
    }

    if (va_bits > VA_BITS_MIN)
      sysreg_clear_set(tcr_el1, TCR_T1SZ_MASK, TCR_T1SZ(va_bits));

    /*
     * The virtual KASLR displacement modulo 2MiB is decided by the
     * physical placement of the image, as otherwise, we might not be able
     * to create the early kernel mapping using 2 MiB block descriptors. So
    * take the low bits of the KASLR offset from the physical address, and
    * fill in the high bits from the seed.
    */
    if (IS_ENABLED(CONFIG_RANDOMIZE_BASE)) {
       u64 kaslr_seed = kaslr_early_init(fdt, chosen);

       if (kaslr_seed && kaslr_requires_kpti())
          arm64_use_ng_mappings = true;

       kaslr_offset |= kaslr_seed & ~(MIN_KIMG_ALIGN - 1);
    }

    if (IS_ENABLED(CONFIG_ARM64_LPA2) && va_bits > VA_BITS_MIN)
      remap_idmap_for_lpa2();

    va_base = KIMAGE_VADDR + kaslr_offset;
    map_kernel(kaslr_offset, va_base - pa_base, root_level);
}
```

### 内核初始化启动切换

&emsp;&emsp;内核初始化启动切换，即完成到start_kernel函数的跳转。

```c
#define KERNEL_START _stext

ldr x8, =__primary_switched
adrp x0, KERNEL_START                  // x0 = __pa(KERNEL_START)
br x8
```

```c
SYM_FUNC_START_LOCAL(__primary_switched)
   adr_l x4, init_task
   init_cpu_task x4, x5, x6

   adr_l x8, vectors                  // load VBAR_EL1 with virtual
   msr vbar_el1, x8                   // vector table address
   isb

   stp x29, x30, [sp, #-16]!
   mov x29, sp

   str_l x21, __fdt_pointer, x5       // Save FDT pointer

   adrp x4, _text                     // Save the offset between
   sub x4, x4, x0                     // the kernel virtual and
   str_l x4, kimage_voffset, x5       // physical mappings

   mov x0, x20
   bl set_cpu_boot_mode_flag

#if defined(CONFIG_KASAN_GENERIC) || defined(CONFIG_KASAN_SW_TAGS)
   bl kasan_early_init
#endif
   mov x0, x20
   bl finalise_el2                    // Prefer VHE if possible
   ldp x29, x30, [sp], #16
   bl start_kernel
   ASM_BUG()
SYM_FUNC_END(__primary_switched)
```

## 内核启动阶段

内核启动阶段，主要完成内核启动功能，其中内核启动入口函数start_kernel的声明如下：

```c
#define __init __section(".init.text")
asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector void start_kernel(void);
```

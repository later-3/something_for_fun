 这篇blog讨论QEMU TCG 引擎如何管理guest内存访问。
 由于 JIT 编译器支持将客户代码翻译成主机代码，因此出现了许多问题:
 - 如何将虚拟 PowerPC 用户地址转换并访问到主机中？
 - 如何将主机 CPU 指令限制为只能访问可用的 QEMU 虚拟机内存？
 - 有地址转换缓存吗？

# 环境
我们假设读者以前就了解现代体系结构和操作系统中的内存管理。许多资源是可以公开获得的:
-   [Memory Management Unit](https://wiki.osdev.org/Memory_Management_Unit)
-   [Memory Paging](https://wiki.osdev.org/Paging)
-   [TLBs](https://wiki.osdev.org/TLB)
像在博客系列中一样，我们将假设来宾机是 PowerPC 32位机器，而主机是 Intel x86 _ 64位机器。

我们不会详细介绍任何与主机中的地址转换相关的内容，因为 QEMU 是您的主机操作系统的一个简单用户进程。客户物理内存(RAM)只是分配给 QEMU 进程的一个缓冲区。因此，QEMU TCG 作业将任何访客内存访问重定向到地址 X (无论是虚拟的还是物理的)到该内存缓冲区。

显然，guest vCPU 类型和运行模式将对地址的转换方式产生影响。这就是所谓的 QEMU-softmmu 进入游戏的地方。在系统模式仿真中，与用户模式相反，对于某些架构，QEMU 引擎支持软件内存管理单元(soft-MMU)。该组件能够将guest虚拟地址转换为来宾物理地址。QEMU 还支持虚拟转换后备缓冲区(vTLB) ，以加快对以前转换过的地址的进一步访问。

guest物理地址总是被转换为guest RAM 的 QEMU 维护表示的偏移量，这通常是主机中的内存映射区域(通过 malloc ()或 mmap ()获得的缓冲区)。

# 一种 QEMU TCG 存储器操作分析
当 QEMU 必须翻译以下 PowerPC 指令时，会发生什么情况:
```asm
0xfff0017c:  90010004  stw      r0, 4(r1)
```

## 将 PowerPC stw 转换为 IR
与任何其他指令一样，opcode 表有一个特定的 stw 条目:
```c
static opcode_t opcodes[] = {
...
GEN_STS(stw, st32, 0x04, PPC_INTEGER)
...
};

#define GEN_ST(name, stop, opc, type)                        \
  GEN_HANDLER(name, opc, 0xFF, 0xFF, 0x00000000, type),

#define GEN_HANDLER(name, opc1, opc2, opc3, inval, type)     \
  GEN_OPCODE(name, opc1, opc2, opc3, inval, type, PPC_NONE)
```
我们故意省略了 GEN _ STS 到 GEN _ HANDLER 的扩展，因为它分解为所有有符号的、无符号的和扩展的存储操作变体，我们对此不感兴趣。

如果您还记得我们的 TCG 文章，GEN _ OPCODE 宏将扩展为 GEN _ stw 处理程序的声明。我们无法在 QEMU 源代码中找到它的直接定义，至于 TCG _helpers_，它是在编译时部分生成的:
```c
#define GEN_ST(name, stop, opc, type)                                         \
static void glue(gen_, name)(DisasContext *ctx)                               \
{                                                                             \
    TCGv EA;                                                                  \
    gen_set_access_type(ctx, ACCESS_INT);                                     \
    EA = tcg_temp_new();                                                      \
    gen_addr_imm_index(ctx, EA, 0);                                           \
    gen_qemu_##stop(ctx, cpu_gpr[rS(ctx->opcode)], EA);                       \
    tcg_temp_free(EA);                                                        \
}
```

除了一些有效的地址计算，有趣的代码行是 gen _ qemu _ # # stop 到 gen _ qemu _ st32的扩展，同样没有直接的定义，但是由于几个宏扩展而生成:
```c
#define GEN_QEMU_STORE_TL(stop, op)                                     \
static void glue(gen_qemu_, stop)(DisasContext *ctx,                    \
                                  TCGv val,                             \
                                  TCGv addr)                            \
{                                                                       \
    tcg_gen_qemu_st_tl(val, addr, ctx->mem_idx, op);                    \
}

GEN_QEMU_STORE_TL(st32, DEF_MEMOP(MO_UL))

/* from tcg/tcg-op.h */
#if TARGET_LONG_BITS == 32
...
#define tcg_gen_qemu_st_tl tcg_gen_qemu_st_i32
...
```

对于32位 PowerPC 客户机，初始的 stw 客户机指令被转换成 QEMU TCG 前端操作 tcg_ gen_ qemu_ st_i32:
```c
void tcg_gen_qemu_st_i32(TCGv_i32 val, TCGv addr, TCGArg idx, TCGMemOp memop)
{
...
    gen_ldst_i32(INDEX_op_qemu_st_i32, val, addr, memop, idx);
...
}
```

从这一点开始，我们有一个发出的 IR qemu_st_i32操作码。

## 将 IR qemu_st_i32转换为 Intel x86_64执行

我们不会再解释主机代码生成，请阅读专门的博客文章。一旦执行循环到达 TCG_ gen_code，更确切地说是 TCG_reg_alloc_op，QEMU 将为 qemu_ st_i32生成 TCG 后端操作。
```c
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
{
...
     tcg_out_op(s, op->opc, new_args, const_args);
...

}
```

在我们的情况下，tcg-target 是一台 Intel x86机器，因此我们将在 tcg/i386/tcg-target. inc.c 中找到合适的 tcg _ out _ op 定义
```c
static inline void tcg_out_op(TCGContext *s, TCGOpcode opc,
                              const TCGArg *args, const int *const_args)
{
...
    case INDEX_op_qemu_st_i32:
        tcg_out_qemu_st(s, args, 0);
...
}
```
我们到了。tcg_out_qemu_st 是一个非常有趣的函数。它包含 QEMU guest内存寻址的内部机制。

# 解析主机地址

博客系列并不打算成为 TCG 内部的完整指南。请记住，在这个级别上，函数是使用与 TCG 参数相关的一些约定开发的(即。TCGArg * args).
```c
static void tcg_out_qemu_st(TCGContext *s, const TCGArg *args, bool is64)
{
    TCGReg datalo, datahi, addrlo;
    TCGReg addrhi __attribute__((unused));
    TCGMemOpIdx oi;
    MemOp opc;
#if defined(CONFIG_SOFTMMU)
    int mem_index;
    tcg_insn_unit *label_ptr[2];
#endif

    datalo = *args++;
    datahi = (TCG_TARGET_REG_BITS == 32 && is64 ? *args++ : 0);
    addrlo = *args++;
    addrhi = (TARGET_LONG_BITS > TCG_TARGET_REG_BITS ? *args++ : 0);
    oi = *args++;
    opc = get_memop(oi);

#if defined(CONFIG_SOFTMMU)
    mem_index = get_mmuidx(oi);

    tcg_out_tlb_load(s, addrlo, addrhi, mem_index, opc,
                     label_ptr, offsetof(CPUTLBEntry, addr_write));

    /* TLB Hit.  */
    tcg_out_qemu_st_direct(s, datalo, datahi, TCG_REG_L1, -1, 0, 0, opc);

    /* Record the current context of a store into ldst label */
    add_qemu_ldst_label(s, false, is64, oi, datalo, datahi, addrlo, addrhi,
                        s->code_ptr, label_ptr);
#else
    tcg_out_qemu_st_direct(s, datalo, datahi, addrlo, x86_guest_base_index,
                           x86_guest_base_offset, x86_guest_base_seg, opc);
#endif
}
```

由于soft mmu 和虚拟 TLB 的支持，QEMU 在访问客户内存时提供了一个缓慢的路径和一个快速的路径。缓慢的路径可以被看作是 TLB 错过，并意味着随后调用 QEMU 内部实现的 PowerPC soft MMU，以将客户虚拟地址转换为客户物理地址。
如果有一个 TLB 命中，QEMU 已经在它的 vCPU 维护的 TLB 中保存了来宾物理地址，并且能够使用 Intel x86指令直接生成对来宾 RAM 的最终内存访问。看看 tcg_out_ qemu_st_direct。

背后的machine与以下三条线相连:
```c
/* try to find a filled TLB entry */
tcg_out_tlb_load(s, addrlo, addrhi, mem_index, opc,
                 label_ptr, offsetof(CPUTLBEntry, addr_write));

/* TLB Hit. So generate a physical guest memory access */
tcg_out_qemu_st_direct(s, datalo, datahi, TCG_REG_L1, -1, 0, 0, opc);

/* TLB Miss. Filled during tlb_load and redirect to soft-MMU */
add_qemu_ldst_label(s, false, is64, oi, datalo, datahi, addrlo, addrhi,
                    s->code_ptr, label_ptr);
```

### QEMU vCPU TLBs

首先，tcg_out_TLB_load 将生成主机指令来检查 TLB 条目。QEMU TLB 是体系结构的通用类型，在 cpu-defs.h中定义:
```c
typedef struct CPUTLBEntry {
    /* bit TARGET_LONG_BITS to TARGET_PAGE_BITS : virtual address
       bit TARGET_PAGE_BITS-1..4  : Nonzero for accesses that should not
                                    go directly to ram.
       bit 3                      : indicates that the entry is invalid
       bit 2..0                   : zero
    */
    union {
        struct {
            target_ulong addr_read;
            target_ulong addr_write;
            target_ulong addr_code;
            /* Addend to virtual address to get host address.  IO accesses
               use the corresponding iotlb value.  */
            uintptr_t addend;
        };
        /* padding to get a power of two size */
        uint8_t dummy[1 << CPU_TLB_ENTRY_BITS];
    };
} CPUTLBEntry;
```

由于已翻译的块只生成一次并执行多次(可能) ，所以生成主机代码很方便，这些代码将动态检查 QEMU 维护的 vCPU TLB。在已翻译块的生命周期内，给定的 TLB 条目可能会失效，然后再次填充。
tcg_out_tlb_ oad 代码读起来相当烦人，充满了 tcg-target 操作码生成器。由此产生的包含 tlb _ load 算法的 Intel x86 _ 64转换块如下所示:
```asm
tcg_out_tlb_load:
0x7ffff41888e9 <code_gen_buffer+22716>:	mov    %esp,%edi
0x7ffff41888eb <code_gen_buffer+22718>:	shr    $0x7,%edi
0x7ffff41888ee <code_gen_buffer+22721>:	and    0x338(%rbp),%edi
0x7ffff41888f4 <code_gen_buffer+22727>:	add    0x388(%rbp),%rdi
0x7ffff41888fb <code_gen_buffer+22734>:	lea    0x3(%r12),%esi
0x7ffff4188900 <code_gen_buffer+22739>:	and    $0xfffff000,%esi
0x7ffff4188906 <code_gen_buffer+22745>:	cmp    0x4(%rdi),%esi
0x7ffff4188909 <code_gen_buffer+22748>:	mov    %r12d,%esi
0x7ffff418890c <code_gen_buffer+22751>:	jne    0x7ffff418897f  ---> back to LDST labels
0x7ffff4188912 <code_gen_buffer+22757>:	add    0x10(%rdi),%rsi
```

QEMU 尝试读取给定来宾虚拟地址的 CPUTLBEntry。对于存储操作，将使用 addr _ write 字段对提取中的0x7ffff4188906进行比较。RDI 寄存器指向 CPUTLBEntry，ESI 保存来宾地址。如果比较失败，这是一个 TLB 错过，我们跳到 LDST 标签，我们将在后面解释。否则，由于 CPUTLBEntry.addend，RSI 寄存器被调整为最终的主机地址，并且可以进行内存访问:
```asm
tcg_out_qemu_st_direct:
0x7ffff4188925 <code_gen_buffer+22776>:	movbe  %ebx,(%rsi)
```

TLB 验证实现也可以在 QEMU cpu _ ld/st _ xxx API 函数中找到。正如文档中所定义的，它们对来宾虚拟地址进行操作，并可能导致来宾 CPU 异常。因此，他们确实会检查 TLB，并且可能会重定向到软件 MMU。它们是通过 cpu _ ldst _ template.h 中的宏实现的:
```c
/* generic store macro */

static inline void
glue(glue(glue(cpu_st, SUFFIX), MEMSUFFIX), _ra)(CPUArchState *env,
                                                 target_ulong ptr,
                                                 RES_TYPE v, uintptr_t retaddr)
{
...
    addr = ptr;
    mmu_idx = CPU_MMU_INDEX;
    entry = tlb_entry(env, mmu_idx, addr);
    if (unlikely(tlb_addr_write(entry) !=
                 (addr & (TARGET_PAGE_MASK | (DATA_SIZE - 1))))) {
        oi = make_memop_idx(SHIFT, mmu_idx);
        glue(glue(helper_ret_st, SUFFIX), MMUSUFFIX)(env, addr, v, oi,
                                                     retaddr);
    } else {
        uintptr_t hostaddr = addr + entry->addend;
        glue(glue(st, SUFFIX), _p)((uint8_t *)hostaddr, v);
    }
...
}
```
看起来很眼熟，不是吗？

### QEMU LDST labels

LDST 标签代表装载/存储标签。它是 QEMU 使用的机制，通过 TCG helper将 TLB 未命中重定向到对软件 MMU 的调用。
TCGContext 对象保存接收生成的主机程序集操作码的输出缓冲区。每当添加一条指令时，指向该缓冲区的 code_ptr 指针显然是递增的。
在 tcg_out_tlb_load 期间，当 QEMU 为 TLB 错过生成比较和跳转指令时，它还将位置记录到输出缓冲区，该缓冲区将保存 JNE 偏移量，以将执行重定向到。

```c
static inline void tcg_out_tlb_load(TCGContext *s, TCGReg addrlo, TCGReg addrhi,
                                    int mem_index, MemOp opc,
                                    tcg_insn_unit **label_ptr, int which)
{
...
    /* jne slow_path */
    tcg_out_opc(s, OPC_JCC_long + JCC_JNE, 0, 0, 0);
    label_ptr[0] = s->code_ptr;
    s->code_ptr += 4;
...
}
```

此外，当我们返回到 tcg_out_qemu_ st 时，add_qemu_LDST_label 调用将创建一个新的 LDST 标签并记录上下文细节，以便稍后准备对 softmmu 缓慢路径 TCG helper 的调用:

```c
static void add_qemu_ldst_label(TCGContext *s, bool is_ld, bool is_64,
                                TCGMemOpIdx oi,
                                TCGReg datalo, TCGReg datahi,
                                TCGReg addrlo, TCGReg addrhi,
                                tcg_insn_unit *raddr,
                                tcg_insn_unit **label_ptr)
{
    TCGLabelQemuLdst *label = new_ldst_label(s);

    label->is_ld = is_ld;
    label->oi = oi;
    label->type = is_64 ? TCG_TYPE_I64 : TCG_TYPE_I32;
    label->datalo_reg = datalo;
    label->datahi_reg = datahi;
    label->addrlo_reg = addrlo;
    label->addrhi_reg = addrhi;
    label->raddr = raddr;
    label->label_ptr[0] = label_ptr[0];
    if (TARGET_LONG_BITS > TCG_TARGET_REG_BITS) {
        label->label_ptr[1] = label_ptr[1];
    }
}
```

Tcg _ gen _ code 通过调用 tcg _ out _ ldst _ finalize 在翻译块后记生成期间插入对缓慢路径helper的调用。QEMU 检查是否存在 LDST 标签，并生成相应的 TCG 帮助器调用。对于我们的存储操作，tcg _ out _ ldst _ finalize 将发出一个 tcg _ out _ qemu _ st _ slow _ path:
```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb)
{
...
#ifdef TCG_TARGET_NEED_LDST_LABELS
    i = tcg_out_ldst_finalize(s);
    if (i < 0) {
        return i;
    }
#endif
...
}

static int tcg_out_ldst_finalize(TCGContext *s)
{
...
    /* qemu_ld/st slow paths */
    QSIMPLEQ_FOREACH(lb, &s->ldst_labels, next) {
        if (lb->is_ld
            ? !tcg_out_qemu_ld_slow_path(s, lb)
            : !tcg_out_qemu_st_slow_path(s, lb)) {
            return -2;
        }
...
}

static bool tcg_out_qemu_st_slow_path(TCGContext *s, TCGLabelQemuLdst *l)
{
...
    /* "Tail call" to the helper, with the return address back inline.  */
    tcg_out_push(s, retaddr);
    tcg_out_jmp(s, qemu_st_helpers[opc & (MO_BSWAP | MO_SIZE)]);
    return true;
}
```

准备缓慢路径helper调用意味着:
- 解决标签偏移，以前记录在 JNE 生成
- 参数/寄存器设置
- 调用适当的 qemu_st_helper
与其他内存访问函数一样，存在许多用于存储和加载操作的变体: 有符号、无符号、字节、 word、 long、 big 或 little endian。每一个都有一个特定的 TCG helper，它是 store _ helper 的包装器:
```c
/* helper signature: helper_ret_st_mmu(CPUState *env, target_ulong addr,
 *                                     uintxx_t val, int mmu_idx, uintptr_t ra)
 */
static void * const qemu_st_helpers[16] = {
    [MO_UB]   = helper_ret_stb_mmu,
    [MO_LEUW] = helper_le_stw_mmu,
    [MO_LEUL] = helper_le_stl_mmu,
    [MO_LEQ]  = helper_le_stq_mmu,
    [MO_BEUW] = helper_be_stw_mmu,
    [MO_BEUL] = helper_be_stl_mmu,
    [MO_BEQ]  = helper_be_stq_mmu,
};

void helper_be_stl_mmu(CPUArchState *env, target_ulong addr, uint32_t val,
                       TCGMemOpIdx oi, uintptr_t retaddr)
{
    store_helper(env, addr, val, oi, retaddr, MO_BEUL);
}

static inline void QEMU_ALWAYS_INLINE
store_helper(CPUArchState *env, target_ulong addr, uint64_t val,
             TCGMemOpIdx oi, uintptr_t retaddr, MemOp op)
{
...
        if (!tlb_hit_page(tlb_addr2, page2)) {
            if (!victim_tlb_hit(env, mmu_idx, index2, tlb_off, page2)) {
                tlb_fill(env_cpu(env), page2, size2, MMU_DATA_STORE,
                         mmu_idx, retaddr);
                index2 = tlb_index(env, mmu_idx, page2);
                entry2 = tlb_entry(env, mmu_idx, page2);
            }
            tlb_addr2 = tlb_addr_write(entry2);
        }
...
    haddr = (void *)((uintptr_t)addr + entry->addend);
    store_memop(haddr, val, op);
}
```

helper检查 TLB，如果有遗漏，则调用 tlb _ fill。在任何情况下，一旦解析了地址，它就使用 store _ memop 对主机内存进行最后的内存访问。

```c
static void tlb_fill(CPUState *cpu, target_ulong addr, int size,
                     MMUAccessType access_type, int mmu_idx, uintptr_t retaddr)
{
    CPUClass *cc = CPU_GET_CLASS(cpu);
    bool ok;

    ok = cc->tlb_fill(cpu, addr, size, access_type, mmu_idx, false, retaddr);
    assert(ok);
}
```

正如您所猜测的，填充 TLB 的过程是由softmmu MMU 完成的，其实现依赖于仿真架构。在 PowerPC CPU 初始化期间，cc-> tlb _ fill 被设置为 ppc _ CPU _ tlb _ fill。
```c
bool ppc_cpu_tlb_fill(CPUState *cs, vaddr addr, int size,
                      MMUAccessType access_type, int mmu_idx,
                      bool probe, uintptr_t retaddr)
{
    PowerPCCPU *cpu = POWERPC_CPU(cs);
    PowerPCCPUClass *pcc = POWERPC_CPU_GET_CLASS(cs);
    CPUPPCState *env = &cpu->env;
    int ret;

    if (pcc->handle_mmu_fault) {
        ret = pcc->handle_mmu_fault(cpu, addr, access_type, mmu_idx);
    } else {
        ret = cpu_ppc_handle_mmu_fault(env, addr, access_type, mmu_idx);
    }
    if (unlikely(ret != 0)) {
        if (probe) {
            return false;
        }
        raise_exception_err_ra(env, cs->exception_index, env->error_code,
                               retaddr);
    }
    return true;
}
```

根据您的 PowerPC CPU 系列，ppc-> handle _ mmu _ fault 最终可能设置为 ppc _ hash32 _ handle _ mmu _ fault。这就是 PowerPC 的软件 MMU 实现所在。







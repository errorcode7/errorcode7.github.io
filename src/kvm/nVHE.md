# nVHE的调用过程

## nVHE调用路径

nvhe的完整路径，host el1 -> hyp el2 ->host el1的过程。

```
[主机调用]: kvm_call_hyp_ret(__kvm_vcpu_run, vcpu) 
     ↓
[展开为]: arm_smccc_1_1_hvc(KVM_HOST_SMCCC_FUNC(__kvm_vcpu_run), vcpu, &res)
     ↓
[执行]: hvc #0 (x0=KVM功能号, x1=vcpu)
     ↓
[异常处理]: host_el1_sync_vect → __host_hvc → __host_exit
     ↓
[调用处理]: handle_trap → handle_host_hcall → host_hcall[KVM_FUNC_IDX] → handle___kvm_vcpu_run → __kvm_vcpu_run(vcpu)
     ↓
[返回]: __host_exit(vcpu) ->eret
     ↓
[返回]: kvm_call_hyp_ret
```

__kvm_vcpu_run内恢复host el1的 pc 寄存器，`__host_exit`最后ret，后续会讲到。

kvm_call_hyp_ret展开

```c
{
     struct arm_smccc_res res;
     
     // KVM_HOST_SMCCC_FUNC(__kvm_vcpu_run) 展开为 (ARM_SMCCC_OWNER_KVM << ARM_SMCCC_OWNER_SHIFT) |      __KVM_HOST_SMCCC_FUNC___kvm_vcpu_run
     // 假设 __KVM_HOST_SMCCC_FUNC___kvm_vcpu_run = 19
     typeof(vcpu) __a1 = vcpu;                           // arg1 = vcpu
     struct arm_smccc_res *___res = &res;                // ___res = &res
     register unsigned long arg0 asm("r0") = (u32)((ARM_SMCCC_OWNER_KVM << ARM_SMCCC_OWNER_SHIFT) | 19); // arg0=function ID
     register typeof(vcpu) arg1 asm("r1") = __a1;        // arg1=vcpu
     
     // 汇编部分
     register unsigned long r0 asm("r0");      // 用于接收返回值
     register unsigned long r1 asm("r1");      // 用于接收返回值
     register unsigned long r2 asm("r2");      // 用于接收返回值
     register unsigned long r3 asm("r3");      // 用于接收返回值
     
     asm volatile("hvc     #0\n :                // SMCCC_HVC_INST = "hvc  #0"
                  "=r" (r0), "=r" (r1), "=r" (r2), "=r" (r3)
                  : "r" (arg0), "r" (arg1)    // function ID and vcpu
                  : "memory");
     
     if (___res) {
         *___res = (typeof(*___res)){r0, r1, r2, r3};   // 将返回值存入res结构体
     }
     
     WARN_ON(res.a0 != SMCCC_RET_SUCCESS);    // 检查返回状态
     
     ret = res.a1;  // 返回函数执行结果
 }
```



## hyp异常向量表映射

  __kvm_hyp_host_vector的结构是这样的（每个异常类型对应一个64字节的槽）：

    __kvm_hyp_host_vector:  // 基地址 (VBAR_EL2)
    ├─ [0x000] Synchronous EL2t     → invalid_host_el2_vect
    ├─ [0x080] IRQ EL2t             → invalid_host_el2_vect  
    ├─ [0x100] FIQ EL2t             → invalid_host_el2_vect
    ├─ [0x180] Error EL2t           → invalid_host_el2_vect
    ├─ [0x200] Synchronous EL2h     → invalid_host_el2_vect
    ├─ [0x280] IRQ EL2h             → invalid_host_el2_vect
    ├─ [0x300] FIQ EL2h             → invalid_host_el2_vect
    ├─ [0x380] Error EL2h           → invalid_host_el2_vect
    ├─ [0x400] Synchronous 64-bit EL1/EL0 → host_el1_sync_vect
    ├─ [0x480] IRQ 64-bit EL1/EL0   → invalid_host_el1_vect
    ├─ [0x500] FIQ 64-bit EL1/EL0   → invalid_host_el1_vect
    ├─ [0x580] Error 64-bit EL1/EL0 → invalid_host_el1_vect
    ├─ [0x600] Synchronous 32-bit EL1/EL0 → host_el1_sync_vect
    ├─ [0x680] IRQ 32-bit EL1/EL0   → invalid_host_el1_vect
    ├─ [0x700] FIQ 32-bit EL1/EL0   → invalid_host_el1_vect
    ├─ [0x780] Error 32-bit EL1/EL0 → invalid_host_el1_vect

中断向量执行流程

```asm
EL1执行: hvc #0 (功能号在x0中)
↓
ARM处理器检测到HVC #0异常
↓
根据ESR_EL2确定异常来源为EL1/EL0
↓
查找向量表: VBAR_EL2 + 0x400 (Synchronous 64-bit EL1/EL0的偏移)
↓
跳转到: host_el1_sync_vect (定义在host.S中)
↓
host_el1_sync_vect宏执行:
         stp x0, x1, [sp, #-16]!    // 保存x0, x1到栈
         mrs x0, esr_el2            // 读取ESR
         ubfx x0, x0, #ESR_ELx_EC_SHIFT, #ESR_ELx_EC_WIDTH  // 提取异常类
         cmp x0, #ESR_ELx_EC_HVC64  // 检查是否为HVC64异常类
         b.eq __host_hvc            // 如果是HVC64，跳转到__host_hvc
         b __host_exit              // 否则，跳转到__host_exit
↓
执行: __host_hvc (在host.S中)
```
host_el1_sync_vect中断向量的核心是判断是否为HVC64异常。x0是功能号，x1是参数，如vpcu指针。

压栈是为了传递给__host_hvc。

## 异常处理分发

`__host_hvc`根据x0记录的kvm功能号是判断是hvc还是其他调用，x0=`__KVM_HOST_SMCCC_FUNC___kvm_vcpu_run`。

```asm
SYM_FUNC_START(__host_hvc)
	ldp	x0, x1, [sp]		// Don't fixup the stack yet

	/* No stub for you, sonny Jim */
alternative_if ARM64_KVM_PROTECTED_MODE//KVM_保护模式直接退出
	b	__host_exit
alternative_else_nop_endif

	/* Check for a stub HVC call */
	cmp	x0, #HVC_STUB_HCALL_NR //成立
	b.hs	__host_exit //普通kvm hvc调用。

	add	sp, sp, #16
	/*
	 * Compute the idmap address of __kvm_handle_stub_hvc and
	 * jump there.
	 *
	 * Preserve x0-x4, which may contain stub parameters.
	 */
	adr_l	x5, __kvm_handle_stub_hvc
	hyp_pa	x5, x6
	br	x5
SYM_FUNC_END(__host_hvc)
```

__host_exit的核心功能就是保存host上下文，从perCPU变量获取指针到x0寄存器，将当前的各种寄存器存这个上下文。在进入异常和中断向量之前，这些寄存器属于el1 host在用，而中断向量中间也只是少量的使用寄存器，且做到了用后恢复。

到然后进入handle_trap，执行具体的hvc功能函数调用，需要host的上下文作为参数提供功能号。

handle_trap

## 保存host上下文，进入到hyp上下文

`__host_exit`先保存host上下文，handle_trap执行hyp功能号对应的函数，返回后，继续执行到eret返回host el1。

```asm
SYM_FUNC_START(__host_exit)
	get_host_ctxt	x0, x1

	/* Store the host regs x2 and x3 */
	stp	x2, x3,   [x0, #CPU_XREG_OFFSET(2)]

	/* Retrieve the host regs x0-x1 from the stack */
	ldp	x2, x3, [sp], #16	// x0, x1

	/* Store the host regs x0-x1 and x4-x17 */
	stp	x2, x3,   [x0, #CPU_XREG_OFFSET(0)]
	stp	x4, x5,   [x0, #CPU_XREG_OFFSET(4)]
	stp	x6, x7,   [x0, #CPU_XREG_OFFSET(6)]
	stp	x8, x9,   [x0, #CPU_XREG_OFFSET(8)]
	stp	x10, x11, [x0, #CPU_XREG_OFFSET(10)]
	stp	x12, x13, [x0, #CPU_XREG_OFFSET(12)]
	stp	x14, x15, [x0, #CPU_XREG_OFFSET(14)]
	stp	x16, x17, [x0, #CPU_XREG_OFFSET(16)]

	/* Store the host regs x18-x29, lr */
	save_callee_saved_regs x0

	/* Save the host context pointer in x29 across the function call */
	mov	x29, x0

#ifdef CONFIG_ARM64_PTR_AUTH_KERNEL
alternative_if_not ARM64_HAS_ADDRESS_AUTH
b __skip_pauth_save
alternative_else_nop_endif

alternative_if ARM64_KVM_PROTECTED_MODE
	/* Save kernel ptrauth keys. */
	add x18, x29, #CPU_APIAKEYLO_EL1
	ptrauth_save_state x18, x19, x20

	/* Use hyp keys. */
	adr_this_cpu x18, kvm_hyp_ctxt, x19
	add x18, x18, #CPU_APIAKEYLO_EL1
	ptrauth_restore_state x18, x19, x20
	isb
alternative_else_nop_endif
__skip_pauth_save:
#endif /* CONFIG_ARM64_PTR_AUTH_KERNEL */

	bl	handle_trap

__host_enter_restore_full:
	/* Restore kernel keys. */
#ifdef CONFIG_ARM64_PTR_AUTH_KERNEL
alternative_if_not ARM64_HAS_ADDRESS_AUTH
b __skip_pauth_restore
alternative_else_nop_endif

alternative_if ARM64_KVM_PROTECTED_MODE
	add x18, x29, #CPU_APIAKEYLO_EL1
	ptrauth_restore_state x18, x19, x20
alternative_else_nop_endif
__skip_pauth_restore:
#endif /* CONFIG_ARM64_PTR_AUTH_KERNEL */

	/* Restore host regs x0-x17 */
	ldp	x0, x1,   [x29, #CPU_XREG_OFFSET(0)]
	ldp	x2, x3,   [x29, #CPU_XREG_OFFSET(2)]
	ldp	x4, x5,   [x29, #CPU_XREG_OFFSET(4)]
	ldp	x6, x7,   [x29, #CPU_XREG_OFFSET(6)]

	/* x0-7 are use for panic arguments */
__host_enter_for_panic:
	ldp	x8, x9,   [x29, #CPU_XREG_OFFSET(8)]
	ldp	x10, x11, [x29, #CPU_XREG_OFFSET(10)]
	ldp	x12, x13, [x29, #CPU_XREG_OFFSET(12)]
	ldp	x14, x15, [x29, #CPU_XREG_OFFSET(14)]
	ldp	x16, x17, [x29, #CPU_XREG_OFFSET(16)]

	/* Restore host regs x18-x29, lr */
	restore_callee_saved_regs x29

	/* Do not touch any register after this! */
__host_enter_without_restoring:
	eret  //使用eret指令返回到主机
	sb
SYM_FUNC_END(__host_exit)

```

在handle_trap里进入到handle_host_hcall分支。

```c
arch/arm64/kvm/hyp/nvhe/hyp-main.c
void handle_trap(struct kvm_cpu_context *host_ctxt)
{
	u64 esr = read_sysreg_el2(SYS_ESR);

	switch (ESR_ELx_EC(esr)) {
	case ESR_ELx_EC_HVC64:
		handle_host_hcall(host_ctxt);
		break;
	case ESR_ELx_EC_SMC64:
		handle_host_smc(host_ctxt);
		break;
	case ESR_ELx_EC_IABT_LOW:
	case ESR_ELx_EC_DABT_LOW:
		handle_host_mem_abort(host_ctxt);
		break;
	default:
		BUG();
	}
}

```

从host_ctxt取,HVC功能号,然后找到对应的调用函数。

```c

#define cpu_reg(ctxt, r)	(ctxt)->regs.regs[r]
#define DECLARE_REG(type, name, ctxt, reg)					\
				__always_unused int ___check_reg_ ## reg;	\
				type name = (type)cpu_reg(ctxt, (reg))


static void handle_host_hcall(struct kvm_cpu_context *host_ctxt)
{
   // 从host_ctxt的x0寄存器获取功能号放到id
   // cpu_reg(host_ctxt, 0) 本质上是从host_ctxt->regs.regs[0]获取值
   // 这个值就是HVC调用时x0寄存器中的功能号

	DECLARE_REG(unsigned long, id, host_ctxt, 0);
	unsigned long hcall_min = 0;
	hcall_t hfn;

	/*
	 * If pKVM has been initialised then reject any calls to the
	 * early "privileged" hypercalls. Note that we cannot reject
	 * calls to __pkvm_prot_finalize for two reasons: (1) The static
	 * key used to determine initialisation must be toggled prior to
	 * finalisation and (2) finalisation is performed on a per-CPU
	 * basis. This is all fine, however, since __pkvm_prot_finalize
	 * returns -EPERM after the first call for a given CPU.
	 */
	if (static_branch_unlikely(&kvm_protected_mode_initialized))
		hcall_min = __KVM_HOST_SMCCC_FUNC___pkvm_prot_finalize;

	id &= ~ARM_SMCCC_CALL_HINTS;
	id -= KVM_HOST_SMCCC_ID(0);

	if (unlikely(id < hcall_min || id >= ARRAY_SIZE(host_hcall)))
		goto inval;

	hfn = host_hcall[id];//先索引到函数
	if (unlikely(!hfn))
		goto inval;

	cpu_reg(host_ctxt, 0) = SMCCC_RET_SUCCESS;
	hfn(host_ctxt);//在跳转到函数

	return;
inval:
	cpu_reg(host_ctxt, 0) = SMCCC_RET_NOT_SUPPORTED;
}
```

对应的函数是`__kvm_vcpu_run`，展开是`handle___kvm_vcpu_run`,host_hcall数组存储函数指针，索引值`__KVM_HOST_SMCCC_FUNC_##x`拼接的调用号。

最终在`handle___kvm_vcpu_run`内调用`__kvm_vcpu_run`。

```c
#define HANDLE_FUNC(x)	[__KVM_HOST_SMCCC_FUNC_##x] = (hcall_t)handle_##x

static const hcall_t host_hcall[] = {
	/* ___kvm_hyp_init */
	HANDLE_FUNC(__pkvm_init),
	HANDLE_FUNC(__pkvm_create_private_mapping),
	HANDLE_FUNC(__pkvm_cpu_set_vector),
	HANDLE_FUNC(__kvm_enable_ssbs),
	HANDLE_FUNC(__vgic_v3_init_lrs),
	HANDLE_FUNC(__vgic_v3_get_gic_config),
	HANDLE_FUNC(__pkvm_prot_finalize),

	HANDLE_FUNC(__pkvm_host_share_hyp),
	HANDLE_FUNC(__pkvm_host_unshare_hyp),
	HANDLE_FUNC(__pkvm_host_share_guest),
	HANDLE_FUNC(__pkvm_host_unshare_guest),
	HANDLE_FUNC(__pkvm_host_relax_perms_guest),
	HANDLE_FUNC(__pkvm_host_wrprotect_guest),
	HANDLE_FUNC(__pkvm_host_test_clear_young_guest),
	HANDLE_FUNC(__pkvm_host_mkyoung_guest),
	HANDLE_FUNC(__kvm_adjust_pc),
	HANDLE_FUNC(__kvm_vcpu_run),
	HANDLE_FUNC(__kvm_flush_vm_context),
	HANDLE_FUNC(__kvm_tlb_flush_vmid_ipa),
	HANDLE_FUNC(__kvm_tlb_flush_vmid_ipa_nsh),
	HANDLE_FUNC(__kvm_tlb_flush_vmid),
	HANDLE_FUNC(__kvm_tlb_flush_vmid_range),
	HANDLE_FUNC(__kvm_flush_cpu_context),
	HANDLE_FUNC(__kvm_timer_set_cntvoff),
	HANDLE_FUNC(__vgic_v3_save_vmcr_aprs),
	HANDLE_FUNC(__vgic_v3_restore_vmcr_aprs),
	HANDLE_FUNC(__pkvm_reserve_vm),
	HANDLE_FUNC(__pkvm_unreserve_vm),
	HANDLE_FUNC(__pkvm_init_vm),
	HANDLE_FUNC(__pkvm_init_vcpu),
	HANDLE_FUNC(__pkvm_teardown_vm),
	HANDLE_FUNC(__pkvm_vcpu_load),
	HANDLE_FUNC(__pkvm_vcpu_put),
	HANDLE_FUNC(__pkvm_tlb_flush_vmid),
};

static void handle___kvm_vcpu_run(struct kvm_cpu_context *host_ctxt)
{
	DECLARE_REG(struct kvm_vcpu *, host_vcpu, host_ctxt, 1);
	int ret;

	if (unlikely(is_protected_kvm_enabled())) {
		struct pkvm_hyp_vcpu *hyp_vcpu = pkvm_get_loaded_hyp_vcpu();

		/*
		 * KVM (and pKVM) doesn't support SME guests for now, and
		 * ensures that SME features aren't enabled in pstate when
		 * loading a vcpu. Therefore, if SME features enabled the host
		 * is misbehaving.
		 */
		if (unlikely(system_supports_sme() && read_sysreg_s(SYS_SVCR))) {
			ret = -EINVAL;
			goto out;
		}

		if (!hyp_vcpu) {
			ret = -EINVAL;
			goto out;
		}

		flush_hyp_vcpu(hyp_vcpu);

		ret = __kvm_vcpu_run(&hyp_vcpu->vcpu);

		sync_hyp_vcpu(hyp_vcpu);
	} else {
		struct kvm_vcpu *vcpu = kern_hyp_va(host_vcpu);

		/* The host is fully trusted, run its vCPU directly. */
		fpsimd_lazy_switch_to_guest(vcpu);
		ret = __kvm_vcpu_run(vcpu);
		fpsimd_lazy_switch_to_host(vcpu);
	}
out:
	cpu_reg(host_ctxt, 1) =  ret;
}
```

`__kvm_vcpu_run`上下文切换前的准备，如果要切换到guest，则__sysreg_restore_state_nvhe(guest_ctxt);将guest的pc寄存器写elr寄存器，方便`__enter_guest` eret进入到guest代码执行。

当guest陷入异常后，`__enter_exit`只是ret返回当前代码仍然在el2级别，只有在退出`__kvm_vcpu_run` 前，调用`__sysreg_restore_state_nvhe(host_ctxt)`，将host 的pc寄存器写入到elr寄存，方便后续返回到host el1状态，这一点是nvhe独有的。

当`__kvm_vcpu_run`层层返回，直到`__host_exit`里的`handle_trap`返回，继续执行到eret,最终从__host_exit返回到host el1，页就是`kvm_call_hyp_ret`展开后hvc调用后的那个指令，宏观上可以认为是kvm_call_hyp_ret返回后，从el2回到host el1。

## 在hyp里切换host和guest状态

```c
/* Switch to the guest for legacy non-VHE systems */
int __kvm_vcpu_run(struct kvm_vcpu *vcpu)
{
	struct kvm_cpu_context *host_ctxt;
	struct kvm_cpu_context *guest_ctxt;
	struct kvm_s2_mmu *mmu;
	bool pmu_switch_needed;
	u64 exit_code;

	host_ctxt = host_data_ptr(host_ctxt);
	host_ctxt->__hyp_running_vcpu = vcpu;
	guest_ctxt = &vcpu->arch.ctxt;

	__sysreg_save_state_nvhe(host_ctxt);


	/*
	 * We must restore the 32-bit state before the sysregs, thanks
	 * to erratum #852523 (Cortex-A57) or #853709 (Cortex-A72).
	 *
	 * Also, and in order to be able to deal with erratum #1319537 (A57)
	 * and #1319367 (A72), we must ensure that all VM-related sysreg are
	 * restored before we enable S2 translation.
	 */
	__sysreg32_restore_state(vcpu);
	__sysreg_restore_state_nvhe(guest_ctxt);

	mmu = kern_hyp_va(vcpu->arch.hw_mmu);
	__load_stage2(mmu, kern_hyp_va(mmu->arch));
	__activate_traps(vcpu);

	do {
		/* Jump in the fire! */
		exit_code = __guest_enter(vcpu);

		/* And we're baaack! */
	} while (fixup_guest_exit(vcpu, &exit_code));

	__sysreg_save_state_nvhe(guest_ctxt);

	/*
	 * Same thing as before the guest run: we're about to switch
	 * the MMU context, so let's make sure we don't have any
	 * ongoing EL1&0 translations.
	 */

	__deactivate_traps(vcpu);
	__load_host_stage2();

	__sysreg_restore_state_nvhe(host_ctxt);

	return exit_code;
}
```

## 在hyp返回前恢复ELR

`__sysreg_restore_state_nvhe`将ctxt里记录的PC寄存器写入ELR寄存，将pstate写入SPSR寄存器，eret就回到了host el1。

```c
void __sysreg_restore_state_nvhe(struct kvm_cpu_context *ctxt)
{
	u64 midr = ctxt_midr_el1(ctxt);

	__sysreg_restore_el1_state(ctxt, midr, ctxt_sys_reg(ctxt, MPIDR_EL1));
	__sysreg_restore_common_state(ctxt);
	__sysreg_restore_user_state(ctxt);
	__sysreg_restore_el2_return_state(ctxt);
}


static inline void __sysreg_restore_el2_return_state(struct kvm_cpu_context *ctxt)
{
	u64 pstate = to_hw_pstate(ctxt);
	u64 mode = pstate & PSR_AA32_MODE_MASK;
	u64 vdisr;

	/*
	 * Safety check to ensure we're setting the CPU up to enter the guest
	 * in a less privileged mode.
	 *
	 * If we are attempting a return to EL2 or higher in AArch64 state,
	 * program SPSR_EL2 with M=EL2h and the IL bit set which ensures that
	 * we'll take an illegal exception state exception immediately after
	 * the ERET to the guest.  Attempts to return to AArch32 Hyp will
	 * result in an illegal exception return because EL2's execution state
	 * is determined by SCR_EL3.RW.
	 */
	if (!(mode & PSR_MODE32_BIT) && mode >= PSR_MODE_EL2t)
		pstate = PSR_MODE_EL2h | PSR_IL_BIT;

	write_sysreg_el2(ctxt->regs.pc,			SYS_ELR);
	write_sysreg_el2(pstate,			SYS_SPSR);

	if (!cpus_have_final_cap(ARM64_HAS_RAS_EXTN))
		return;

	if (!vserror_state_is_nested(ctxt_to_vcpu(ctxt)))
		vdisr = ctxt_sys_reg(ctxt, DISR_EL1);
	else if (ctxt_has_ras(ctxt))
		vdisr = ctxt_sys_reg(ctxt, VDISR_EL2);
	else
		vdisr = 0;

	write_sysreg_s(vdisr, SYS_VDISR_EL2);
}
```



本文是对[Shoot4U: Using VMM Assists to Optimize TLB Operations on Preempted vCPUs
](http://dl.acm.org/citation.cfm?id=2892245) / [patch](https://github.com/ouyangjn/shoot4u)的学习笔记。

该方法通过直接在VMM刷TLB而不是等到相应vCPU被调度时才刷，提高了性能。同时利用了Individual-address invalidation，可以只刷指定地址。

## 背景

__invvpid在kvm中已经有定义：

```c
static inline void __invvpid(int ext, u16 vpid, gva_t gva)
{
    struct {
    u64 vpid : 16;
    u64 rsvd : 48;
    u64 gva;
    } operand = { vpid, 0, gva };

    asm volatile (__ex(ASM_VMX_INVVPID)
          /* CF==1 or ZF==1 --> rc = -1 */
          "; ja 1f ; ud2 ; 1:"
          : : "a"(&operand), "c"(ext) : "cc", "memory");
}
```

调的是INVVPID指令，作用是根据VPID使TLB和page cache失效。接收两个参数，vpid和内存地址

有四种模式：

* Individual-address invalidation(type=0)

    针对tag为VPID且地址为指定地址(参数传入)的

* Single-context invalidation(type=1)

    针对tag为VPID的

* All-contexts invalidation(type=2)

    针对除了vpid为0000H(应该是VMM)的所有

* Single-context invalidation, retaining global translations(type=3)

    针对tag为VPID的TLB的，但保留global translations



原来已经用到了VMX_VPID_EXTENT_SINGLE_CONTEXT(1)和VMX_VPID_EXTENT_ALL_CONTEXT(2)


```c
static inline void vpid_sync_vcpu_single(struct vcpu_vmx *vmx)
{
        if (vmx->vpid == 0)
                return;

        if (cpu_has_vmx_invvpid_single())
                __invvpid(VMX_VPID_EXTENT_SINGLE_CONTEXT, vmx->vpid, 0);
}

static inline void vpid_sync_vcpu_global(void)
{
        if (cpu_has_vmx_invvpid_global())
                __invvpid(VMX_VPID_EXTENT_ALL_CONTEXT, 0, 0);
}
```

## HOST修改(KVM)

### arch/x86/include/asm/vmx.h

新增了0的操作。即Individual-address invalidation

```c
#define VMX_VPID_EXTENT_INDIVIDUAL_ADDR        0
#define VMX_VPID_EXTENT_INDIVIDUAL_ADDR_BIT      (1ull << 8) /* (40 - 32) */
```

### arch/x86/kvm/vmx.c


在tlb_flush后新增了三个操作

```diff
    .tlb_flush = vmx_flush_tlb,

+   .tlb_flush_vpid_single_ctx = vmx_flush_tlb_single_ctx,
+   .tlb_flush_vpid_single_addr = vmx_flush_tlb_single_addr,
+   .get_vpid = vmx_get_vpid,
```

在当前版本是存在vmx_x86_ops里面的，里面保存的都是vmx平台支持的操作，这个数组会在KVM初始化(kvm_init)时作为参数传入，存到kvm_x86_ops中。相当于注册了函数。


#### vmx_get_vpid

从vcpu_vmx结构中读取当前的vpid。vcpu_vmx包含了kvm_vcpu，表示vmx平台下的一个vcpu。


```c
static inline int vmx_get_vpid(struct kvm_vcpu *vcpu)
{
        struct vcpu_vmx *vmx = container_of(vcpu, struct vcpu_vmx, vcpu);
        return vmx->vpid;
}
```

### vmx_flush_tlb_single_ctx

老方法，single/all，刷掉全部

```c
static void vmx_flush_tlb_single_ctx(struct kvm_vcpu *vcpu)
{
    vpid_sync_context(to_vmx(vcpu));
}
```

### vmx_flush_tlb_single_addr

尝试刷掉单条地址。

vmx_capability保存的是从MSR读MSR_IA32_VMX_EPT_VPID_CAP出来的信息，其中vpid放在高32位，所以实际上是读vmx_capability.vpid的第8个bit


```c
static inline bool cpu_has_vmx_invvpid_addr(void)
{
    return vmx_capability.vpid & VMX_VPID_EXTENT_INDIVIDUAL_ADDR_BIT;
}


static inline void vpid_sync_addr(struct vcpu_vmx *vmx, unsigned long addr)
{
    if (vmx->vpid == 0)
        return;

    // 如果vcpu支持新特性，则单独让该地址失效
    if (cpu_has_vmx_invvpid_addr())
        __invvpid(VMX_VPID_EXTENT_INDIVIDUAL_ADDR, vmx->vpid, addr);
    else
    // 否则使用老办法，single/all
        vpid_sync_context(vmx);
}

static void vmx_flush_tlb_single_addr(struct kvm_vcpu *vcpu, unsigned long addr)
{
    // 判断一下
    vpid_sync_addr(to_vmx(vcpu), addr);
}

```




### include/uapi/linux/kvm_para.h

```c
#define KVM_HC_SHOOT4U         12
```


在kvm_emulate_hypercall里新增：

```diff
+   case KVM_HC_SHOOT4U:
+       kvm_pv_shoot4u_op(vcpu, a0, a1, a2);
+       ret = 0;
+       break;
```

注册了一种新的调用类型，当调用时通过kvm_pv_shoot4u_op进行处理。kvm_pv_shoot4u_op会根据定义的mode，去调用上文的vmx_flush_tlb_single_ctx/vmx_flush_tlb_single_addr。


### arch/x86/kvm/x86.c

```c
/* lapic timer advance (tscdeadline mode only) in nanoseconds */
#define SHOOT4U_MODE_DEFAULT   0
#define SHOOT4U_MODE_TEST1     1
#define SHOOT4U_MODE_TEST2     2
#define SHOOT4U_MODE_TEST3     3
unsigned int shoot4u_mode = SHOOT4U_MODE_DEFAULT;
module_param(shoot4u_mode, uint, S_IRUGO | S_IWUSR);
```

可以开启不同模式：

* 0  刷掉整个tlb
* 1  刷掉vpid的tlb
* 2  如果有结束地址，刷掉整个tlb，否则尝试刷单个地址
* 3  如果有结束地址，刷掉vpid的tlb，否则尝试刷单个地址

怀疑是当前不支持指定区域，只能单条刷?


```c
/*
 * kvm_pv_shoot4u_op:  Handle tlb shoot down hypercall
 *
 * @apicid - apicid of vcpu to be kicked.
 */

// 当前vcpu，VM包含的vcpu设置的bitmap，要失效的起始和结束地址
static void kvm_pv_shoot4u_op(struct kvm_vcpu *vcpu, unsigned long vcpu_bitmap,
        unsigned long start, unsigned long end)
{
    struct kvm_shoot4u_info info;
    struct kvm *kvm = vcpu->kvm;
    struct kvm_vcpu *v;
    int i;

    info.flush_start = start;
    info.flush_end = end;

    //printk("[shoot4u] inside hypercall handler\n");
    // construct phsical cpu mask from vcpu bitmap
    // 对于VM中除了自己以外的每个vcpu，查看它是否在bitmap中，如果是，调用flush_tlb_func_shoot4u刷掉自己在他之上的tlb
    kvm_for_each_vcpu(i, v, kvm) {
        if (v != vcpu && test_bit(v->vcpu_id, (void*)&vcpu_bitmap)) {
            info.vcpu = v;
            //printk("[shoot4u] before send IPI to vcpu %d at pcpu %d\n", v->vcpu_id, v->cpu);
            // it is fine if a vCPU migrates because migratation triggers tlb_flush automatically
            smp_call_function_single(v->cpu, flush_tlb_func_shoot4u, &info, 1);
        }
    }
}

struct kvm_shoot4u_info {
    struct kvm_vcpu *vcpu;
    unsigned long flush_start;
    unsigned long flush_end;
};


// 跨处理器操作
/* shoot4u host IPI handler with invvipd */
static void flush_tlb_func_shoot4u(void *info)
{
    struct kvm_shoot4u_info *f = info;

    //printk("[shoot4u] IPI handler at pCPU %d: invalidate vCPU %d\n", smp_processor_id(), f->vcpu->vcpu_id);
    if (shoot4u_mode == SHOOT4U_MODE_DEFAULT) {
        // all (linear + EPT mappings)
        kvm_x86_ops->tlb_flush(f->vcpu);
    } else if (shoot4u_mode == SHOOT4U_MODE_TEST1) {
        // all (linear mappings only)
        kvm_x86_ops->tlb_flush_vpid_single_ctx(f->vcpu);
    } else if (shoot4u_mode == SHOOT4U_MODE_TEST2) {
        // single or all (linear + EPT mappings)
        if (!f->flush_end)
            kvm_x86_ops->tlb_flush_vpid_single_addr(f->vcpu, f->flush_start);
        else {
            kvm_x86_ops->tlb_flush(f->vcpu);
        }
    } else if (shoot4u_mode == SHOOT4U_MODE_TEST3) {
        // seg fault
        // single or all (linear mappings only)
        if (!f->flush_end)
            kvm_x86_ops->tlb_flush_vpid_single_addr(f->vcpu, f->flush_start);
        else {
            kvm_x86_ops->tlb_flush_vpid_single_ctx(f->vcpu);
        }
    } else {
        ...
    }

    return;
}
```


## Guest

```diff
+#ifdef CONFIG_SHOOT4U
+        pv_mmu_ops.flush_tlb_others = shoot4u_flush_tlb_others;
+#endif
```

通过kvm_hypercall请求刷别人的tlb

暂不支持超过64个vcpu，因为它用long来存，只有64bit

```c
void shoot4u_flush_tlb_others(const struct cpumask *cpumask,
                struct mm_struct *mm, unsigned long start,
                unsigned long end)
{
    // shoot4u currently uses a 8 bytes bitmap to pass target cores
    // thus it supports up to 64 physical cores
    u64 cpu_bitmap = 0;
    int cpu;

    // 将要flush的vcpu设置到bitmap中
    for_each_cpu(cpu, cpumask) {
        if (cpu >= 64) {
            panic("[shoot4u] ERROR: do not support more than 64 cores\n");
        }
        set_bit(cpu, (void *)&cpu_bitmap);
    }

    //printk("[shoot4u] before KVM_HC_SHOOT4U hypercall, cpumask: %llx, start %lx, end %lx\n", cpu_bitmap, start, end);
    kvm_hypercall3(KVM_HC_SHOOT4U, cpu_bitmap, start, end);
}

```



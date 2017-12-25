
本文主要分析 [https://lkml.org/lkml/2017/12/8/116](https://lkml.org/lkml/2017/12/8/116) 中 pvtimer 的实现。

## 原始实现

Linux kernel 中会通过 lapic_next_deadline 设置定时器，即设置下一个超时的时间点。原来的设置很简单：

```c
static int lapic_next_deadline(unsigned long delta,
                   struct clock_event_device *evt)
{
    u64 tsc;

    tsc = rdtsc();
    wrmsrl(MSR_IA32_TSC_DEADLINE, tsc + (((u64) delta) * TSC_DIVISOR));
    return 0;
}
```

就是读取当前的 TSC ，加上要等待的时间 delta ，写入到 MSR_IA32_TSC_DEADLINE 中。

> TSC-Deadline Mode
> APIC Timer 的三种操作模式之一，为其写入一个非零64位值即可激活 Timer ，使得在 TSC 达到该值时触发一个中断。该中断只会触发一次，触发后 IA32_TSC_DEADLINE_MSR 就被重置为零。

如果 Linux 是跑在 KVM 之上的 Guest ，则触发 VMExit ，退回到 KVM ，执行 `kvm_set_lapic_tscdeadline_msr(vcpu, data)`

```c
void kvm_set_lapic_tscdeadline_msr(struct kvm_vcpu *vcpu, u64 data)
{
    struct kvm_lapic *apic = vcpu->arch.apic;

    if (!lapic_in_kernel(vcpu) || apic_lvtt_oneshot(apic) ||
            apic_lvtt_period(apic))
        return;

    hrtimer_cancel(&apic->lapic_timer.timer);
    apic->lapic_timer.tscdeadline = data;
    start_apic_timer(apic);
}
```

于是 KVM 就会取消当前 apic->lapic_timer.timer 上的定时，重新设置新的超时时间。注意，vCPU 设置的 timer 会被加入到物理 CPU 的 timer 红黑树中。


apic->lapic_timer.timer 的回调函数在 kvm_create_lapic 设置为 apic_timer_fn ：


```c
static enum hrtimer_restart apic_timer_fn(struct hrtimer *data)
{
    struct kvm_timer *ktimer = container_of(data, struct kvm_timer, timer);
    struct kvm_lapic *apic = container_of(ktimer, struct kvm_lapic, lapic_timer);

    apic_timer_expired(apic);

    if (lapic_is_periodic(apic)) {
        hrtimer_add_expires_ns(&ktimer->timer, ktimer->period);
        return HRTIMER_RESTART;
    } else
        return HRTIMER_NORESTART;
}
```


### Guest

每个 CPU 都维护有 pvtimer_vcpu_event_info 类型的 per-CPU 变量 pvtimer_shared_buf 。在 kvm_guest_cpu_init 时，会将其的地址填入 MSR_KVM_PV_TIMER_EN 中，供 KVM 填充:

```c
#define MSR_KVM_PV_TIMER_EN     0x4b564d05
```

存放 pvtimer_vcpu_event_info 的地址：

```c
struct pvtimer_vcpu_event_info {
    __u64 expire_tsc;
    __u64 next_sync_tsc;
} __attribute__((__packed__));
```

上文提过，原本 Linux kernel 在 lapic_next_deadline 把下一个超时的时间点写到 MSR_IA32_TSC_DEADLINE 中，会发生 VMExit 。而该 patch 的本质思想就是将其写到 pvtimer_vcpu_event_info 中，这样就避免了 VMExit 。

```c
static int lapic_next_deadline(unsigned long delta,
                   struct clock_event_device *evt)
{
    u64 tsc = rdtsc() + (((u64) delta) * TSC_DIVISOR);

    /* TODO: undisciplined function call */
    if (kvm_pv_timer_next_event(tsc, evt))
        return 0;

    wrmsrl(MSR_IA32_TSC_DEADLINE, tsc);
    return 0;
}

static DEFINE_PER_CPU(int, pvtimer_enabled);
static DEFINE_PER_CPU(struct pvtimer_vcpu_event_info,
             pvtimer_shared_buf) = {0};
#define PVTIMER_PADDING        25000

int kvm_pv_timer_next_event(unsigned long tsc,
        struct clock_event_device *evt)
{
    struct pvtimer_vcpu_event_info *src;
    u64 now;

    if (!this_cpu_read(pvtimer_enabled))
        return false;

    /* 将当前设置的超时时间写到 pvtimer_vcpu_event_info.expire_tsc 中
     * 取出上次设置的 expire_tsc ，如果它：
     *  1. 小于 pvtimer 下一次的 pv_sync_timer 超时的时间（pvtimer_vcpu_event_info.next_sync_tsc）
     *      表示在 KVM 主动去检查是否有 timer 超时之前，该 timer 已经超时，所以需要将该超时时间通过传统方式，
     *      即设置 MSR_IA32_TSC_DEADLINE 来让 KVM 立刻调用 kvm_apic_sync_pv_timer
     *  2. 小于当前时间，表示已经超时，需要尽快触发中断，只能通过传统方式设置，让 KVM 立刻调用 kvm_apic_sync_pv_timer
     *  3. 其他情况表示还未超时，无需进行处理
     */
    src = this_cpu_ptr(&pvtimer_shared_buf);
    xchg((u64 *)&src->expire_tsc, tsc);

    barrier();

    if (tsc < src->next_sync_tsc)
        return false;

    rdtscll(now);
    if (tsc < now || tsc - now < PVTIMER_PADDING)
        return false;

    return true;
}
```




### KVM

#### cache 初始化

当 Guest 设置 MSR_KVM_PV_TIMER_EN 时，会 VMExit 到 KVM ，调用 kvm_lapic_enable_pv_timer 进行初始化 vcpu->arch.pv_timer 结构。

```c
struct {
     u64 msr_val;
     struct gfn_to_hva_cache data;
} pv_timer;

int kvm_lapic_enable_pv_timer(struct kvm_vcpu *vcpu, u64 data)
{
    u64 addr = data & ~KVM_MSR_ENABLED;
    int ret;

    if (!lapic_in_kernel(vcpu))
        return 1;

    if (!IS_ALIGNED(addr, 4))
        return 1;

     // 保存 pvtimer_vcpu_event_info 的地址
    vcpu->arch.pv_timer.msr_val = data;
    if (!pv_timer_enabled(vcpu))
        return 0;

    // 建立 GPA 到 HVA 的 cache
    ret = kvm_gfn_to_hva_cache_init(vcpu->kvm, &vcpu->arch.pv_timer.data,
                    addr, sizeof(struct pvtimer_vcpu_event_info));

    return ret;
}
```





#### pvtimer 初始化

在 kvm_lapic_init 时，通过 kvm_pv_timer_init 初始化 pvtimer ：

```c
#define PVTIMER_SYNC_CPU   (NR_CPUS - 1) /* dedicated CPU */
#define PVTIMER_PERIOD_NS  250000L /* pvtimer default period */

static long pv_timer_period_ns = PVTIMER_PERIOD_NS;

static void kvm_pv_timer_init(void)
{
    ktime_t ktime = ktime_set(0, pv_timer_period_ns);

    hrtimer_init(&pv_sync_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_PINNED);
    pv_sync_timer.function = &pv_sync_timer_callback;

    /* kthread for pv_timer sync buffer */
    pv_timer_polling_worker = kthread_create(pv_timer_polling, NULL,
                        "pv_timer_polling_worker/%d",
                        PVTIMER_SYNC_CPU);
    if (IS_ERR(pv_timer_polling_worker)) {
        pr_warn_once("kvm: failed to create thread for pv_timer\n");
        pv_timer_polling_worker = NULL;
        hrtimer_cancel(&pv_sync_timer);

        return;
    }

    kthread_bind(pv_timer_polling_worker, PVTIMER_SYNC_CPU);
    wake_up_process(pv_timer_polling_worker);
    hrtimer_start(&pv_sync_timer, ktime, HRTIMER_MODE_REL);
}
```

创建了高精度定时器 pv_sync_timer ，使用 monotonic （单调）时间，在 pv_timer_period_ns （默认为 250000 ns）后调用一次 pv_sync_timer_callback

```c
static enum hrtimer_restart pv_sync_timer_callback(struct hrtimer *timer)
{
    // 将 timer 的超时时间推后 pv_timer_period_ns
    hrtimer_forward_now(timer, ns_to_ktime(pv_timer_period_ns));
    // 唤醒 pv_timer_polling_worker
    wake_up_process(pv_timer_polling_worker);

    // 返回 restart 表示 timer 会重新被激活
    return HRTIMER_RESTART;
}
```

kvm_pv_timer_init 同时创建了名为 pv_timer_polling_worker/x 的内核进程，其中 x 为最后一个 CPU 的编号，表示它将在该 CPU 上执行（kthread_bind）。该线程执行 pv_timer_polling 。

配合 pv_sync_timer，相当于每隔 pv_timer_period_ns 唤醒一次 pv_timer_polling_worker ，执行 pv_timer_polling。


```c
static int pv_timer_polling(void *arg)
{
    struct kvm *kvm;
    struct kvm_vcpu *vcpu;
    int i;
    mm_segment_t oldfs = get_fs();

    while (1) {
        // 设置为处于可中断睡眠状态
        set_current_state(TASK_INTERRUPTIBLE);

        if (kthread_should_stop()) {
            __set_current_state(TASK_RUNNING);
            break;
        }

        spin_lock(&kvm_lock);
        // 设置处于可运行状态，此时如果被调度器选中会立刻运行
        __set_current_state(TASK_RUNNING);
        list_for_each_entry(kvm, &vm_list, vm_list) {
            set_fs(USER_DS);
            use_mm(kvm->mm);
            kvm_for_each_vcpu(i, vcpu, kvm) {
                kvm_apic_sync_pv_timer(vcpu);
            }
            unuse_mm(kvm->mm);
            set_fs(oldfs);
        }

        spin_unlock(&kvm_lock);

        // 主动让出控制权给下一个进程
        schedule();
    }

    return 0;
}
```

该函数会遍历所有 VM ，对其中的每个 vCPU 调用 kvm_apic_sync_pv_timer 。调用完成后，pv_timer_polling 通过 schedule 将控制权让给下一个进程，等待 pv_sync_timer 的下一次超时唤醒。

```c
void kvm_apic_sync_pv_timer(void *data)
{
    struct kvm_vcpu *vcpu = data;
    struct kvm_lapic *apic = vcpu->arch.apic;
    unsigned long flags, this_tsc_khz = vcpu->arch.virtual_tsc_khz;
    u64 guest_tsc, expire_tsc;
    long rem_tsc;

    if (!lapic_in_kernel(vcpu) || !pv_timer_enabled(vcpu))
        return;

    local_irq_save(flags);
    // 获取 Guest 当前的实际 TSC 值
    guest_tsc = kvm_read_l1_tsc(vcpu, rdtsc());
    // 计算 pv_sync_timer 器还有多少 TSC 超时
    rem_tsc = ktime_to_ns(hrtimer_get_remaining(&pv_sync_timer))
            * this_tsc_khz;
    if (rem_tsc <= 0)
        rem_tsc += pv_timer_period_ns * this_tsc_khz;
    do_div(rem_tsc, 1000000L);

    /*
     * make sure guest_tsc and rem_tsc are assigned before to update
     * next_sync_tsc.
     */
    smp_wmb();
    // 将下一次 pv_sync_timer 超时的 TSC 写入到 pvtimer_vcpu_event_info.next_sync_tsc
    kvm_xchg_guest_cached(vcpu->kvm, &vcpu->arch.pv_timer.data,
        offsetof(struct pvtimer_vcpu_event_info, next_sync_tsc),
        guest_tsc + rem_tsc, 8);

    /* make sure next_sync_tsc is visible */
    smp_wmb();

    // 读取 pvtimer_vcpu_event_info.expire_tsc 并将其设置为 0
    // 此时 expire_tsc 存放的是 Guest （先前设置的）下一次超时时间点
    expire_tsc = kvm_xchg_guest_cached(vcpu->kvm, &vcpu->arch.pv_timer.data,
            offsetof(struct pvtimer_vcpu_event_info, expire_tsc),
            0UL, 8);

    /* make sure expire_tsc is visible */
    smp_wmb();

    // 如果当前 Guest timer 未超时，则把 expire_tsc 设置到 apic->lapic_timer.tscdeadline ，设置 timer
    //   等效于 Guest 写入 MSR_IA32_TSCDEADLINE 发现 VMExit 后 KVM 进行的操作
    // 如果已经超时，直接注入 APIC_LVTT 中断
    if (expire_tsc) {
        if (expire_tsc > guest_tsc)
            /*
             * As we bind this thread to a dedicated CPU through
             * IPI, the timer is registered on that dedicated
             * CPU here.
             */
            kvm_set_lapic_tscdeadline_msr(apic->vcpu, expire_tsc);
        else
            /* deliver immediately if expired */
            kvm_apic_local_deliver(apic, APIC_LVTT);
    }
    local_irq_restore(flags);
}
```

另外还修改了写 MSR_IA32_TSCDEADLINE 时的处理，由原来的直接 `kvm_set_lapic_tscdeadline_msr(vcpu, data);` 改为

```c
if (pv_timer_enabled(vcpu))
    smp_call_function_single(PVTIMER_SYNC_CPU,
            kvm_apic_sync_pv_timer, vcpu, 0);
else
    kvm_set_lapic_tscdeadline_msr(vcpu, data);
```

当然，如果在 kvm_apic_sync_pv_timer 中发现 Guest 设置的 timer 超时了，还是会调用 kvm_set_lapic_tscdeadline_msr 去设置 apic->lapic_timer.timer。

apic->lapic_timer.timer 超时时，会发送 APIC_LVTT 的中断，如果 timer 处于 TSC deadline mode ，按照规范需要将 deadline 设置为 0

```c
static enum hrtimer_restart apic_timer_fn(struct hrtimer *data) {
    struct kvm_timer *ktimer = container_of(data, struct kvm_timer, timer);
    struct kvm_lapic *apic = container_of(ktimer, struct kvm_lapic, lapic_timer);

    if (pv_timer_enabled(apic->vcpu)) {
        kvm_apic_local_deliver(apic, APIC_LVTT);
        if (apic_lvtt_tscdeadline(apic))
            apic->lapic_timer.tscdeadline = 0;
    } else
        apic_timer_expired(apic);
}
```


## 总结

原来设置 APIC timer 的方式：Guest 将超时的时间戳（tsc-deadline timestamp）写入到 MSR_IA32_TSC_DEADLINE ，触发 VMExit 。KVM 会设置在物理CPU上设置 timer 。

pvtimer 将本来要设置到 MSR_IA32_TSCDEADLINE 里面的超时时间戳设置到 pvtimer_vcpu_event_info （的 expire_tsc）中。

KVM 创建一个线程，定时检查 pvtimer_vcpu_event_info.expire_tsc ，发现超时了就直接注入超时中断，还没超时就调用 kvm_set_lapic_tscdeadline_msr 。原来是设置了 MSR 后发生 VMExit ，KVM 中对应的 handler 去调用，现在变成了定时检查，如被设置了，则调用。减少了 VMExit 。




[linux 上下文切换时对pthread 和 kthread区别对待——针对fpu](https://blog.csdn.net/qq_40392804/article/details/107713216)

> linux 5.10.8，本文以（上下文切换时，pthread和kthread对fpu的load/store的区别）为主线
>

# 和kthread/pthread区别对待相关的数据结构
&emsp;&emsp;linux中的TCB是`task_struct`，定义在文件`./include/linux/sched.h`中，和调度时和kthread和pthread区别对待相关的域如下：
```c
struct task_struct {
    /* ... */
    struct thread_info  thread_info;
    /* ... */
    /* Per task flags (PF_*), defined further below: */
    unsigned int   flags;
    /* ... */
    struct mm_struct  *mm;
    struct mm_struct  *active_mm;
    /* ... */
}

/*
 * Per process flags
 */
#define PF_IDLE   0x00000002 /* I am an IDLE thread */
#define PF_EXITING  0x00000004 /* Getting shut down */
#define PF_VCPU   0x00000010 /* I'm a virtual CPU */
#define PF_WQ_WORKER  0x00000020 /* I'm a workqueue worker */
#define PF_FORKNOEXEC  0x00000040 /* Forked but didn't exec */
#define PF_MCE_PROCESS  0x00000080      /* Process policy on mce errors */
#define PF_SUPERPRIV  0x00000100 /* Used super-user privileges */
#define PF_DUMPCORE  0x00000200 /* Dumped core */
#define PF_SIGNALED  0x00000400 /* Killed by a signal */
#define PF_MEMALLOC  0x00000800 /* Allocating memory */
#define PF_NPROC_EXCEEDED 0x00001000 /* set_user() noticed that RLIMIT_NPROC was exceeded */
#define PF_USED_MATH  0x00002000 /* If unset the fpu must be initialized before use */
#define PF_USED_ASYNC  0x00004000 /* Used async_schedule*(), used by module init */
#define PF_NOFREEZE  0x00008000 /* This thread should not be frozen */
#define PF_FROZEN  0x00010000 /* Frozen for system suspend */
#define PF_KSWAPD  0x00020000 /* I am kswapd */
#define PF_MEMALLOC_NOFS 0x00040000 /* All allocation requests will inherit GFP_NOFS */
#define PF_MEMALLOC_NOIO 0x00080000 /* All allocation requests will inherit GFP_NOIO */
#define PF_LESS_THROTTLE 0x00100000 /* Throttle me less: I clean memory */
#define PF_KTHREAD  0x00200000 /* I am a kernel thread */
#define PF_RANDOMIZE  0x00400000 /* Randomize virtual address space */
#define PF_SWAPWRITE  0x00800000 /* Allowed to write to swap */
#define PF_MEMSTALL  0x01000000 /* Stalled due to lack of memory */
#define PF_UMH   0x02000000 /* I'm an Usermodehelper process */
#define PF_NO_SETAFFINITY 0x04000000 /* Userland is not allowed to meddle with cpus_mask */
#define PF_MCE_EARLY  0x08000000      /* Early kill for mce process policy */
#define PF_MEMALLOC_NOCMA 0x10000000 /* All allocation request will have _GFP_MOVABLE cleared */
#define PF_IO_WORKER  0x20000000 /* Task is an IO worker */
#define PF_FREEZER_SKIP  0x40000000 /* Freezer should not count it as freezable */
#define PF_SUSPEND_TASK  0x80000000      /* This thread called freeze_processes() and should not be frozen */
```
&emsp;&emsp;`flag`字段是task的标志位，标志位所有的信息如上所示。其中`PF_KTHREAD`标志位置1表示task是一个kthread。

&emsp;&emsp;对于普通用户进程来说，`mm`指向虚拟地址空间的用户空间部分，而对于内核线程，`mm`为`NULL`。由于内核线程之前可能是任何用户层进程在执行，故用户空间部分的内容本质上是随机的，内核线程决不能修改其内容，故将`mm`设置为`NULL`，同时如果切换出去的是用户进程，内核将原来进程的mm存放在新内核线程的`active_mm`中，因为某些时候内核必须知道用户空间当前包含了什么。在linux的代码实现中，这个字段和fpu的load/store没有关系，只是和kthread和pthread的区别对待有关系。

&emsp;&emsp;`thread_info` 定义在`./arch/x86/include/asm/thread_info.h`的结构如下。它的`flag`字段设有32个标志位，如下所示。其中`TIF_NEED_FPU_LOAD`标志位置1，当task被调度到，在返回用户态之前，或者kernel需要用到fpu时，fpu会被加载到cpu上。
```c
struct thread_info {
 unsigned long  flags;  /* low level flags */
 u32   status;  /* thread synchronous flags */
};

/*
 * thread information flags
 * - these are process state flags that various assembly files
 *   may need to access
 */
#define TIF_SYSCALL_TRACE 0 /* syscall trace active */
#define TIF_NOTIFY_RESUME 1 /* callback before returning to user */
#define TIF_SIGPENDING  2 /* signal pending */
#define TIF_NEED_RESCHED 3 /* rescheduling necessary */
#define TIF_SINGLESTEP  4 /* reenable singlestep on user return*/
#define TIF_SSBD  5 /* Speculative store bypass disable */
#define TIF_SYSCALL_EMU  6 /* syscall emulation active */
#define TIF_SYSCALL_AUDIT 7 /* syscall auditing active */
#define TIF_SECCOMP  8 /* secure computing */
#define TIF_SPEC_IB  9 /* Indirect branch speculation mitigation */
#define TIF_SPEC_FORCE_UPDATE 10 /* Force speculation MSR update in context switch */
#define TIF_USER_RETURN_NOTIFY 11 /* notify kernel of userspace return */
#define TIF_UPROBE  12 /* breakpointed or singlestepping */
#define TIF_PATCH_PENDING 13 /* pending live patching update */
#define TIF_NEED_FPU_LOAD 14 /* load FPU on return to userspace */
#define TIF_NOCPUID  15 /* CPUID is not accessible in userland */
#define TIF_NOTSC  16 /* TSC is not accessible in userland */
#define TIF_IA32  17 /* IA32 compatibility process */
#define TIF_NOHZ  19 /* in adaptive nohz mode */
#define TIF_MEMDIE  20 /* is terminating due to OOM killer */
#define TIF_POLLING_NRFLAG 21 /* idle is polling for TIF_NEED_RESCHED */
#define TIF_IO_BITMAP  22 /* uses I/O bitmap */
#define TIF_FORCED_TF  24 /* true if TF in eflags artificially */
#define TIF_BLOCKSTEP  25 /* set when we want DEBUGCTLMSR_BTF */
#define TIF_LAZY_MMU_UPDATES 27 /* task is updating the mmu lazily */
#define TIF_SYSCALL_TRACEPOINT 28 /* syscall tracepoint instrumentation */
#define TIF_ADDR32  29 /* 32-bit address space on 64 bits */
#define TIF_X32   30 /* 32-bit native x86-64 binary */
#define TIF_FSCHECK  31 /* Check FS is USER_DS on return */
```
# 调用图
&emsp;&emsp;具体的linux代码实现在后文中介绍。这里是调度和返回用户态的过程中和fpu load/store相关的过程。图中，箭头线表示执行流(在前一个空心箭头的上级函数中），而空心箭头表示函数调用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200803114440223.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMzkyODA0,size_16,color_FFFFFF,t_70)


# fpu load/store 在调度中的处理
&emsp;&emsp;fpu在调度中的处理流程如下：
* `shedule` in `/kernel/sched/core.c`是调度主要的函数。
```c
asmlinkage __visible void __sched schedule(void){
    /* ... */
    do {
        preempt_disable();
        __schedule(false);
        sched_preempt_enable_no_resched();
    } while (need_resched());
    /* ... */
}
```
* `__schedule` in `./kernel/sched/core.c`。更新调度队列的当前task(rq->cur)(但是此时还换栈，下文中的`current`指的都是栈上的current task，也就是old task)，并调用`context_switch`进行上下文切换。
```c
static void __sched notrace __schedule(bool preempt){
    struct task_struct *prev, *next;
    unsigned long *switch_count;
    struct rq_flags rf;
    struct rq *rq;
    int cpu;
    
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;
    /* ... */
    next = pick_next_task(rq, prev, &rf);
    /* ... */
    if (likely(prev != next)) {
        /* ... */
        RCU_INIT_POINTER(rq->curr, next);
        /* ... */
        rq = context_switch(rq, prev, next, &rf);
    } else {
        /* ... */
    }
    /* ... */
}
```
* `context_switch` in `./kernel/sched/core.c`。
```c
/*
 * context_switch - switch to the new MM and the new thread's register state.
 */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
        struct task_struct *next, struct rq_flags *rf){
        
     /* ... */
     /* Here we just switch the register state and the stack. */
     switch_to(prev, next, prev);
     /* ... */
 }
```
* `switch_to` in `./arch/x86/include/asm/switch_to.h`。
```c
#define switch_to(prev, next, last)     \
do {         \
 prepare_switch_to(next);     \
         \
 ((last) = __switch_to_asm((prev), (next)));   \
} while (0)
```
* `__switch_to_asm` in `./arch/x86/kernel/process_64.c`。对fpu的store/load分两步进行：首先，它调用`test_thread_flag(TIF_NEEED_FPU_LOAD)` 判断是否需要store fpu(old thread)，然后调用`switch_fpu_prepare`  store fpu；其次，调用`switch_fpu_finish`把`TIF_NEED_FPU_LOAD`位置1，这样当新的thread返回用户态，或者需要用到fpu时，就会load fpu。
```c
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p){
    struct thread_struct *prev = &prev_p->thread;
    struct thread_struct *next = &next_p->thread;
    struct fpu *prev_fpu = &prev->fpu;
    struct fpu *next_fpu = &next->fpu;
    int cpu = smp_processor_id();

    /* ... */
    if (!test_thread_flag(TIF_NEED_FPU_LOAD))
        switch_fpu_prepare(prev_fpu, cpu);
    /* ... */
    switch_fpu_finish(next_fpu);
    /* ... */
}
```
* `test_thread_flag` in `./include/linux/thread_info.h`，检查对应的位是否置1，这个调用链中是检查`TIF_NEED_FPU_LOAD`。
```c
#define test_thread_flag(flag) \
    test_ti_thread_flag(current_thread_info(), flag)

static inline int test_ti_thread_flag(struct thread_info *ti, int flag){
    return test_bit(flag, (unsigned long *)&ti->flags);
}

#define current_thread_info() ((struct thread_info *)current)

/* in ./arch/x86/include/asm/current.h */
DECLARE_PER_CPU(struct task_struct *, current_task);
static __always_inline struct task_struct *get_current(void){
 return this_cpu_read_stable(current_task);
}
#define current get_current()

/* in ./include/asm-generic/bitops/non-atomic.h */
/**
 * test_bit - Determine whether a bit is set
 * @nr: bit number to test
 * @addr: Address to start counting from
 */
static inline int test_bit(int nr, const volatile unsigned long *addr)
{
 return 1UL & (addr[BIT_WORD(nr)] >> (nr & (BITS_PER_LONG-1)));
}
```
* `switch_fpu_prepare` in `./arch/x86/include/asm/fpu/internal.h`。调用`copy_fpregs_to_fpstate`进行fpu store。
```c
/*
 * FPU state switching for scheduling.
 *
 * This is a two-stage process:
 *
 *  - switch_fpu_prepare() saves the old state.
 *    This is done within the context of the old process.
 *
 *  - switch_fpu_finish() sets TIF_NEED_FPU_LOAD; the floating point state
 *    will get loaded on return to userspace, or when the kernel needs it.
 *
 * If TIF_NEED_FPU_LOAD is cleared then the CPU's FPU registers
 * are saved in the current thread's FPU register state.
 *
 * If TIF_NEED_FPU_LOAD is set then CPU's FPU registers may not
 * hold current()'s FPU registers. It is required to load the
 * registers before returning to userland or using the content
 * otherwise.
 *
 * The FPU context is only stored/restored for a user task and
 * PF_KTHREAD is used to distinguish between kernel and user threads.
 */
static inline void switch_fpu_prepare(struct fpu *old_fpu, int cpu){
    if (static_cpu_has(X86_FEATURE_FPU) && !(current->flags & PF_KTHREAD)) {
        if (!copy_fpregs_to_fpstate(old_fpu))
            old_fpu->last_cpu = -1;
        else
            old_fpu->last_cpu = cpu;
        /* But leave fpu_fpregs_owner_ctx! */
        trace_x86_fpu_regs_deactivated(old_fpu);
    }
}

// in ./arch/x86/include/asm/cpufeatures.h
/* Intel-defined CPU features, CPUID level 0x00000001 (EDX), word 0 */
#define X86_FEATURE_FPU   ( 0*32+ 0) /* Onboard FPU */
```
* `switch_fpu_finish` in `./arch/x86/include/asm/fpu/internal.h`。设置新 task的`TIF_NEED_FPU_LOAD`标志位(在返回用户态之前，会检查并根据结果load fpu)。
```c
/*
 * Load PKRU from the FPU context if available. Delay loading of the
 * complete FPU state until the return to userland.
 */
static inline void switch_fpu_finish(struct fpu *new_fpu){
   /* ... */
   if (!static_cpu_has(X86_FEATURE_FPU))
      return;
   set_thread_flag(TIF_NEED_FPU_LOAD);
   /* ... */
}
```
* `prepare_exit_to_usermode` in `./arch/x86/entry/common.c`。当调度结束返回用户态之前，会调用`prepare_exit_to_usermode`，它会调用`switch_fpu_return` 完成fpu load的操作。
```c
/* Called with IRQs disabled. */
__visible inline void prepare_exit_to_usermode(struct pt_regs *regs){
    struct thread_info *ti = current_thread_info();
    u32 cached_flags;
    /* ... */
    cached_flags = READ_ONCE(ti->flags);
    /* ... */
    if (unlikely(cached_flags & _TIF_NEED_FPU_LOAD))
        switch_fpu_return();
    /* ... */
}
```
* `switch_fpu_return` in `./arch/x86/kernel/fpu/core.c`。它调用`__fregs_load_activate` in `./arch/x86/include/asm/fpu/internal.h`进行fpu的加载。首先，如果当前线程是kthread，就不会load fpu；然后，load fpu；最后，清除`TIF_NEED_FPU_LOAD`标志位。清除操作是因为调度时，会根据这个标志位进行store fpu的操作。如果是kthread，这个标志位就不会被清除，所以kthread被调度出去的时候不会保存fpu的状态；kthread被调度进来的时候也不会恢复fpu的状态。如果是pthread，这个标志位会被清除，所以pthread被调度出去的时候会保存fpu的状态；pthread被调度进来的时候会在返回用户态之前恢复fpu的状态。
```c
/*
 * Load FPU context before returning to userspace.
 */
void switch_fpu_return(void){
    if (!static_cpu_has(X86_FEATURE_FPU))
        return;
    __fpregs_load_activate();
}
EXPORT_SYMBOL_GPL(switch_fpu_return);

/*
 * Internal helper, do not use directly. Use switch_fpu_return() instead.
 */
static inline void __fpregs_load_activate(void){
    struct fpu *fpu = &current->thread.fpu;
    int cpu = smp_processor_id();
    if (WARN_ON_ONCE(current->flags & PF_KTHREAD))
        return;
    if (!fpregs_state_valid(fpu, cpu)) {
        copy_kernel_to_fpregs(&fpu->state);
        fpregs_activate(fpu);
        fpu->last_cpu = cpu;
    }
    clear_thread_flag(TIF_NEED_FPU_LOAD);
}
```

# fpu load/store 在KVM中的处理
&emsp;&emsp;当vm-exit发生时，在内核中会执行当前线程(也就是vcpu对应的线程，是一个普通的线程，可以是kthread，也可以是pthread)。然后，vcpu线程和别的线程一样参与调度，而在调度的过程中对fpu load/store的处理和前文描述的一样。不同的是，kvm是要vm-entry到客户机执行，而非返回到用户态。所以需要和返回用户态一样，在返回客户机执行前执行load fpu的操作。
* `vcpu_enter_guest` in `./arch/x86/kvm/x86.c`
```c
/*
 * Returns 1 to let vcpu_run() continue the guest execution loop without
 * exiting to the userspace.  Otherwise, the value will be returned to the
 * userspace.
 */
static int vcpu_enter_guest(struct kvm_vcpu *vcpu){
    /* .... */
    if (test_thread_flag(TIF_NEED_FPU_LOAD))
        switch_fpu_return();
    /* ... */
    kvm_x86_ops->run(vcpu);
    /* ... */
}    
```
&emsp;&emsp;根据以上的描述。如果vcpu对应的thread时kthread，那么到那个kthread被调度出去时，os调度器不会执行store fpu的操作，这就需要在vm-exit手动的执行fpu store，以防被调度出去时，客户机需要的fpu state被修改。如果vcpu对应的thread选择为pthread，那么当pthread被调度出去时，os调度器会执行store fpu的操作；当要返回客户机之前，需要kvm执行load fpu的操作。

# 参考
[Linux内核线程kernel thread详解--Linux进程的管理与调度](https://blog.csdn.net/weixin_30399821/article/details/99665474)


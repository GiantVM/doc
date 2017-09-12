## 定义

### 传统(物理机)中断

中断从某个设备发出，送到IOAPIC。IOAPIC查PRT表找到对应的表项PTE，得知目标LAPIC。于是格式化出中断消息发送给LAPIC，通知置remote irr为1(level)。

LAPIC收到中断消息后，根据向量号设置IRR后，进行中断选取，取得取得优先级最高的中断后，清除IRR，设置ISR，提交CPU进行中断处理，CPU处理完中断后，写LAPIC的EOI，通知IOAPIC清除remote irr(level且deassert)。

### QEMU+KVM(虚拟机)中断

#### 中断退出
虚拟机发生中断时，主动使guest发生 VMEXIT ，这样接下来能够在 VMENTRY 前进行中断注入。

#### 中断注入
通过将中断写入 VMCS 的 VM-Entry interruption-infomation field ，实现向guest注入中断。

#### VMCS
在 SDM 3 24.6 的 VM-EXECUTION CONTROL FIELDS 中定义了：

* Pin-Based VM-Execution Controls
    负责控制 External-interrupt / NMI / Virtual NMIs 时是否发生VMExit，退回到KVM中。

    比如 External-interrupt exiting 设置为1表示所有的外部中断都会产生VMExit，否则由VM自己处理

* Secondary Processor-Based VM-Execution Controls - virtual-interrupt delivery
    设置为1则当 VM entry/TPR virtualization/EOI virtualization/self-IPI virtualization/posted-interrupt processing 时会触发evaluate pending中断

由于设置了 VMCS - Secondary Processor-Based VM-Execution Controls - virtualize APIC accesses 位，通过设置特定的VM-execution controls的位，使VM在访问APIC对应的页的时产生VMEXIT。

当该中断被recognized了，并且满足以下四个条件，就会触发该虚拟中断的delivery：

1. RFLAGS.IF = 1
2. 没有因为STI产生的blocking
3. 没有因为MOV SS或者POP SS产生的blocking
4. Primary Processor-Based VM-Execution Controls中的 interrupt-window exiting bit 为0。

虚拟中断的delivery会更新 guest interrupt status 中的RVI和SVI，并且在non-root环境下产生一个中断事件


#### 中断芯片

QEMU 和 KVM 都实现了对中断芯片的模拟，这是由于历史原因造成的。早在KVM诞生之前，QEMU就提供了一整套对设备的模拟，包括中断芯片。而KVM诞生之后，为了进一步提高中断性能，因此又在KVM中实现了一套中断芯片。我们可以通过QEMU的启动参数 kernel-irqchip 来决定使用谁的中断芯片(irq chip)。

* on： KVM 模拟全部
* split： QEMU模拟IOAPIC和PIC，KVM模拟LAPIC
* off： QEMU 模拟全部


#### GPIO(General-purpose input/output)
从定义上，GPIO是一种通用的PIN，可以在运行时控制其作为input或者output。input可读，output可读写。

在QEMU中，大量使用GPIO来表示设备和中断控制器的PIN。和硬件实现一样，它分为IN和OUT。以以下的数据结构描述：

```c
struct NamedGPIOList {
    char *name;
    qemu_irq *in;
    int num_in;
    int num_out;
    QLIST_ENTRY(NamedGPIOList) node;
};
```

PIN的单元为 qemu_irq 。因此该结构维护了input qemu_irq数组的首地址，可以访问到所有input qemu_irq。同时维护了input和output的数目。

每个设备都会维护一个至多个 NamedGPIOList ，以链表形式组织(成员node就是用来串起来的)，用 DeviceState.gpios 指向。比如 8259 有一个拥有1个out和8个in的名为NULL的 NamedGPIOList 。

qemu_irq 是 IRQState 的指针，定义如下：

```c
struct IRQState {
    Object parent_obj;

    qemu_irq_handler handler;
    void *opaque;
    int n;
};
```

通常来说，n为PIN号，opaque指向所属设备，而 handler 是该PIN的回调函数。

当有信号要发送给设备的某个PIN时，就调用对应input qemu_irq 的handler，表示设备收到了信号，于是handler设置设备的一些属性，表示其状态发生了改变。如果需要输出，则调用某个output qemu_irq 的handler，表示将信号发送出去。

output PIN在初始化时往往设置为 qemu_irq 指针，在其上游设备(接收该设备的输出)初始化时，会将该指针指向上游设备自己的 input qemu_irq。因此当设备进行输出时，调用的是上游设备对应input qemu_irq的handler，这样就模拟了一个信号从一个设备传递到另一个设备的过程。

根据q35的 qom-tree ，我们发现有GPIO的设备主要为 PIC 和 IOAPIC ：

```
/machine (pc-q35-2.8-machine)
    /device[9] (hpet)
      /unnamed-gpio-in[0] (irq)
      /unnamed-gpio-in[1] (irq)
    /device[7] (isa-i8259)
      /unnamed-gpio-in[3] (irq)
      /unnamed-gpio-in[4] (irq)
      /unnamed-gpio-in[5] (irq)
      /unnamed-gpio-in[6] (irq)
      /unnamed-gpio-in[2] (irq)
      /unnamed-gpio-in[7] (irq)
      /unnamed-gpio-in[0] (irq)
      /unnamed-gpio-in[1] (irq)
    /device[8] (isa-i8259)
      /unnamed-gpio-in[3] (irq)
      /unnamed-gpio-in[4] (irq)
      /unnamed-gpio-in[5] (irq)
      /unnamed-gpio-in[6] (irq)
      /unnamed-gpio-in[2] (irq)
      /unnamed-gpio-in[7] (irq)
      /unnamed-gpio-in[0] (irq)
      /unnamed-gpio-in[1] (irq)
 /q35 (q35-pcihost)
    /pcie.0 (PCIE)
    /ioapic (ioapic)
      /unnamed-gpio-in[17] (irq)
      /unnamed-gpio-in[9] (irq)
      /unnamed-gpio-in[20] (irq)
      /unnamed-gpio-in[19] (irq)
      /unnamed-gpio-in[22] (irq)
      /unnamed-gpio-in[0] (irq)
      /unnamed-gpio-in[10] (irq)
      /unnamed-gpio-in[2] (irq)
      /unnamed-gpio-in[12] (irq)
      /unnamed-gpio-in[4] (irq)
      /unnamed-gpio-in[14] (irq)
      /unnamed-gpio-in[6] (irq)
      /unnamed-gpio-in[16] (irq)
      /unnamed-gpio-in[8] (irq)
      /unnamed-gpio-in[18] (irq)
      /unnamed-gpio-in[21] (irq)
      /unnamed-gpio-in[23] (irq)
      /unnamed-gpio-in[1] (irq)
      /unnamed-gpio-in[11] (irq)
      /unnamed-gpio-in[3] (irq)
      /unnamed-gpio-in[13] (irq)
      /unnamed-gpio-in[5] (irq)
      /unnamed-gpio-in[15] (irq)
      /unnamed-gpio-in[7] (irq)
```



#### GSI(Global System Interrupt)

ACPI(Advanced Configuration and Power Interface)规范 为x86机器定义了统一的配置接口，中断也不例外。10年前，人们觉得计算机架构处于并将长期处于PIC和APIC混用的阶段，比如QEMU的经典架构q35，于是定义了GSI。

GSI为系统中每个中断控制器上的input pin都指定一个唯一的中断号：

* 对于 8259A ，GSI直接映射到ISA IRQ。比如 GSI 0 映射 IRQ 0。
* 对于 IOAPIC ，每一个IOAPIC都会被BIOS分配一个GSI base。映射时为base + pin。比如IOAPIC0的GSI base为0，有24个引脚，则它们对应的GSI为0-23。IOAPIC1的GSI base为24，有16个引脚，则范围为24-39，以此类推。


QEMU使用 GSIState 来描述GSI：

```c
typedef struct GSIState {
    qemu_irq i8259_irq[ISA_NUM_IRQS];
    qemu_irq ioapic_irq[IOAPIC_NUM_PINS];
} GSIState;
```

对于Q35，在machine初始化函数 pc_q35_init 中负责对GSI进行初始化，同时填充GPIO。





## 中断模拟

### KVM模拟芯片

#### PIC
KVM使用 kvm_pic 模拟8259A芯片。它的指针被保存在 kvm.arch.vpic 中。

```c
struct kvm_pic {
    spinlock_t lock;
    bool wakeup_needed;
    unsigned pending_acks;
    struct kvm *kvm;
    struct kvm_kpic_state pics[2]; /* 0 is master pic, 1 is slave pic */    // 维护了中断芯片上所有寄存器和状态
    int output;     /* intr from master PIC */
    struct kvm_io_device dev_master;                                        // 主片设备
    struct kvm_io_device dev_slave;                                         // 从片设备
    struct kvm_io_device dev_eclr;                                          // 控制中断触发模式的寄存器
    void (*ack_notifier)(void *opaque, int irq);
    unsigned long irq_states[PIC_NUM_PINS];
};

struct kvm_kpic_state {
    u8 last_irr;    /* edge detection */
    u8 irr;     /* interrupt request register */
    u8 imr;     /* interrupt mask register */
    u8 isr;     /* interrupt service register */
    u8 priority_add;    /* highest irq priority */
    u8 irq_base;
    u8 read_reg_select;
    u8 poll;
    u8 special_mask;
    u8 init_state;
    u8 auto_eoi;
    u8 rotate_on_auto_eoi;
    u8 special_fully_nested_mode;
    u8 init4;       /* true if 4 byte init */
    u8 elcr;        /* PIIX edge/trigger selection */
    u8 elcr_mask;
    u8 isr_ack; /* interrupt ack detection */
    struct kvm_pic *pics_state;
};
```

dev_master 、 dev_slave 、 dev_eclr 定义了设备对应的操作函数，同时通过 kvm_io_bus_register_dev 将它们注册到PIO总线(KVM_PIO_BUS)上。

当需要对设备进行读写时，会调用到以下函数：

```c
static const struct kvm_io_device_ops picdev_master_ops = {
    .read     = picdev_master_read,
    .write    = picdev_master_write,
};

static const struct kvm_io_device_ops picdev_slave_ops = {
    .read     = picdev_slave_read,
    .write    = picdev_slave_write,
};

static const struct kvm_io_device_ops picdev_eclr_ops = {
    .read     = picdev_eclr_read,
    .write    = picdev_eclr_write,
};
```

#### IOAPIC

KVM只模拟了一种IOAPIC，名为 kvm_ioapic 。它的指针被保存在 kvm.kvm_arch.vioapic 中。

```c
struct kvm_ioapic {
    u64 base_address;
    u32 ioregsel;
    u32 id;
    u32 irr;                                                        // IRR寄存器
    u32 pad;
    union kvm_ioapic_redirect_entry redirtbl[IOAPIC_NUM_PINS];      // PRT，每个表项代表一个引脚
    unsigned long irq_states[IOAPIC_NUM_PINS];
    struct kvm_io_device dev;
    struct kvm *kvm;
    void (*ack_notifier)(void *opaque, int irq);
    spinlock_t lock;
    struct rtc_status rtc_status;
    struct delayed_work eoi_inject;
    u32 irq_eoi[IOAPIC_NUM_PINS];
    u32 irr_delivered;
};

// RTE
union kvm_ioapic_redirect_entry {
    u64 bits;
    struct {
        u8 vector;                  // 中断向量(ISRV)号，指定中断对应的vector。优先级 = vector / 16，越大越高
        u8 delivery_mode:3;         // 传送模式，指定该中断以何种方式发送给目的LAPIC，有Fixed、Lowest Priority、SMI、NMI、INIT、ExtINT
        u8 dest_mode:1;             // 目的地模式，0为Physical Mode，1为Logical Mode
        u8 delivery_status:1;       // 传送状态，0为IDEL(没有中断)，1为Send Pending(已收到该中断但由于某种原因还未发送)
        u8 polarity:1;              // 管脚极性，指定该管脚的有效电平是高电平还是低电平，0为高，1为低
        u8 remote_irr:1;            // 远程IRR，(中断水平触发)当LAPIC收到该中断后设为1，LAPIC写入EOI时清0
        u8 trig_mode:1;             // 触发模式，1为水平，2为边缘
        u8 mask:1;                  // 中断屏蔽位，1时屏蔽该中断
        u8 reserve:7;               // 未用
        u8 reserved[4];             // 未用
        u8 dest_id;                 // 目标，Physical Mode下表示目标LAPIC的ID，Logical Mode下表示一组CPU?
    } fields;
};
```

在收到QEMU发来的 KVM_CREATE_IRQCHIP 后，调用 kvm_ioapic_init 进行初始化：调用 kvm_iodevice_init 绑定操作：

```c
static const struct kvm_io_device_ops ioapic_mmio_ops = {
    .read     = ioapic_mmio_read,
    .write    = ioapic_mmio_write,
};
```

并通过 kvm_io_bus_register_dev 将 dev 注册到MMIO总线(KVM_MMIO_BUS)上。

#### LAPIC

在KVM中，每个vCPU都有自己的LAPIC，名为 kvm_lapic 。它的指针被保存在 vcpu.arch.apic 中。

```c
struct kvm_lapic {
    unsigned long base_address;                 // 基地址(GPA)
    struct kvm_io_device dev;                   // 保存了LAPIC对应的操作
    struct kvm_timer lapic_timer;
    u32 divide_count;
    struct kvm_vcpu *vcpu;
    bool sw_enabled;
    bool irr_pending;
    bool lvt0_in_nmi_mode;
    /* Number of bits set in ISR. */
    s16 isr_count;
    /* The highest vector set in ISR; if -1 - invalid, must scan ISR. */
    int highest_isr_cache;
    /**
     * APIC register page.  The layout matches the register layout seen by
     * the guest 1:1, because it is accessed by the vmx microcode.
     * Note: Only one register, the TPR, is used by the microcode.
     */
    void *regs;                                 // 指向host的一个page，保存了LAPIC使用的所有虚拟寄存器，如IRR，ISR，LVT等
    gpa_t vapic_addr;
    struct gfn_to_hva_cache vapic_cache;
    unsigned long pending_events;
    unsigned int sipi_vector;
};
```

它在 vmx_create_vcpu => kvm_vcpu_init => kvm_arch_vcpu_init => kvm_create_lapic 中被创建和初始化。

当需要对设备进行MMIO读写时，会调用到以下函数：

```c
static const struct kvm_io_device_ops apic_mmio_ops = {
    .read     = apic_mmio_read,
    .write    = apic_mmio_write,
};
```

在读/写LAPIC的某个寄存器时，由于设置了 VMCS - Secondary Processor-Based VM-Execution Controls - virtualize APIC accesses 位为1，产生VMEXIT，回到KVM，将寄存器地址减去 base_address 得到offset，然后通过 kvm_lapic_reg_read / kvm_lapic_reg_write 对 LAPIC 这个struct进行操作。





#### 创建流程

QEMU在 kvm_init 中，如果是on或split，说明需要KVM来模拟中断芯片，因此进行初始化， on 的调用流程如下：

```
kvm_irqchip_create => kvm_arch_irqchip_create => kvm_vm_enable_cap(s, KVM_CAP_SPLIT_IRQCHIP, 0, 24) => kvm_vm_ioctl(s, KVM_ENABLE_CAP, &cap)
                   => kvm_vm_ioctl(s, KVM_CREATE_IRQCHIP)
```

因此这里的关键是通过 KVM_CREATE_IRQCHIP 创建芯片。

在 KVM 中，调用链如下：

```
kvm_arch_vm_ioctl => kvm_create_pic                                           创建PIC芯片
                  => kvm_ioapic_init                                          创建并初始化IOAPIC芯片
                  => kvm_setup_default_irq_routing => kvm_set_irq_routing     设置中断路由
```

在 on 下，KVM将直接以 default_routing 作为中断路由。即由KVM来初始化中断路由表 kvm->irq_routing 。


```c
static const struct kvm_irq_routing_entry default_routing[] = {
  ROUTING_ENTRY2(0), ROUTING_ENTRY2(1),
  ROUTING_ENTRY2(2), ROUTING_ENTRY2(3),
  ROUTING_ENTRY2(4), ROUTING_ENTRY2(5),
  ROUTING_ENTRY2(6), ROUTING_ENTRY2(7),
  ROUTING_ENTRY2(8), ROUTING_ENTRY2(9),
  ROUTING_ENTRY2(10), ROUTING_ENTRY2(11),
  ROUTING_ENTRY2(12), ROUTING_ENTRY2(13),
  ROUTING_ENTRY2(14), ROUTING_ENTRY2(15),
  ROUTING_ENTRY1(16), ROUTING_ENTRY1(17),
  ROUTING_ENTRY1(18), ROUTING_ENTRY1(19),
  ROUTING_ENTRY1(20), ROUTING_ENTRY1(21),
  ROUTING_ENTRY1(22), ROUTING_ENTRY1(23),
};

#define ROUTING_ENTRY2(irq) \
  IOAPIC_ROUTING_ENTRY(irq), PIC_ROUTING_ENTRY(irq)

#define ROUTING_ENTRY1(irq) IOAPIC_ROUTING_ENTRY(irq)

#define PIC_ROUTING_ENTRY(irq) \
  { .gsi = irq, .type = KVM_IRQ_ROUTING_IRQCHIP,  \
    .u.irqchip = { .irqchip = SELECT_PIC(irq), .pin = (irq) % 8 } }

#define IOAPIC_ROUTING_ENTRY(irq) \
  { .gsi = irq, .type = KVM_IRQ_ROUTING_IRQCHIP,  \
    .u.irqchip = { .irqchip = KVM_IRQCHIP_IOAPIC, .pin = (irq) } }
```

可以看到GSI的前16号(0-15)即有PIC又有IOAPIC。而16-23就只有IOAPIC了。

##### kvm_set_irq_routing

```
kvm_set_irq_routing => setup_routing_entry => kvm_set_routing_entry
                    => rcu_assign_pointer(kvm->irq_routing, new)
```

它会创建新的 kvm_irq_routing_table ，然后遍历新传入的entrys数组，对每一个entry一一调用 setup_routing_entry ，构造出 kvm_irq_routing_entry 并设置到新table中。最后将 kvm->irq_routing 指向新的table

中断路由表 kvm->irq_routing 的类型为 kvm_irq_routing_table 。表项存储在 map 中，在这里是 kvm_irq_routing_entry 组成的列表：

```c
struct kvm_irq_routing_table {
    int chip[KVM_NR_IRQCHIPS][KVM_IRQCHIP_NUM_PINS];            // 一级索引指对应的中断芯片，二级索引对应引脚，存放引脚对应的GSI号。目前已弃用
    u32 nr_rt_entries;
    /*
     * Array indexed by gsi. Each entry contains list of irq chips
     * the gsi is connected to.
     */
    struct hlist_head map[0];                                   // 指向项为list的哈希表，key为gsi，value为kvm_kernel_irq_routing_entry列表
};

struct kvm_kernel_irq_routing_entry {
    u32 gsi;                                                    // 引脚的GSI号
    u32 type;
    int (*set)(struct kvm_kernel_irq_routing_entry *e,          // 设置中断函数
           struct kvm *kvm, int irq_source_id, int level,       // kvm，中断资源ID，高低电平
           bool line_status);
    union {
        struct {
            unsigned irqchip;
            unsigned pin;
        } irqchip;
        struct {
            u32 address_lo;
            u32 address_hi;
            u32 data;
            u32 flags;
            u32 devid;
        } msi;
        struct kvm_s390_adapter_int adapter;
        struct kvm_hv_sint hv_sint;
    };
    struct hlist_node link;
};
```

具体的中断芯片(如PIC、IOAPIC)通过实现 kvm_irq_routing_entry 的 set 函数，实现了在中断注入时的对应行为。

##### setup_routing_entry

函数 setup_routing_entry 负责设置某个gsi的中断路由：

```c
static int setup_routing_entry(struct kvm *kvm,
                   struct kvm_irq_routing_table *rt,
                   struct kvm_kernel_irq_routing_entry *e,
                   const struct kvm_irq_routing_entry *ue)
{
    int r = -EINVAL;
    struct kvm_kernel_irq_routing_entry *ei;

    /*
     * Do not allow GSI to be mapped to the same irqchip more than once.
     * Allow only one to one mapping between GSI and non-irqchip routing.
     */
    hlist_for_each_entry(ei, &rt->map[ue->gsi], link)
        if (ei->type != KVM_IRQ_ROUTING_IRQCHIP ||
            ue->type != KVM_IRQ_ROUTING_IRQCHIP ||
            ue->u.irqchip.irqchip == ei->irqchip.irqchip)
            return r;

    e->gsi = ue->gsi;
    e->type = ue->type;
    r = kvm_set_routing_entry(kvm, e, ue);
    if (r)
        goto out;
    if (e->type == KVM_IRQ_ROUTING_IRQCHIP)
        rt->chip[e->irqchip.irqchip][e->irqchip.pin] = e->gsi;

    hlist_add_head(&e->link, &rt->map[e->gsi]);
    r = 0;
out:
    return r;
}
```

这里遍历了 kvm_irq_routing_table->map 中gsi对应的列表，如果发现其中存在目标irqchip的 kvm_kernel_irq_routing_entry ，表示已设置，则直接返回。因为一个中断控制器中所有引脚对应的GSI都应该不同。

否则对该entry进行设置，在填充了 gsi 和 type 后通过 kvm_set_routing_entry 进一步设置。

```c

int kvm_set_routing_entry(struct kvm *kvm,
              struct kvm_kernel_irq_routing_entry *e,
              const struct kvm_irq_routing_entry *ue)
{
    int r = -EINVAL;
    int delta;
    unsigned max_pin;

    switch (ue->type) {
    case KVM_IRQ_ROUTING_IRQCHIP:
        delta = 0;
        switch (ue->u.irqchip.irqchip) {
        case KVM_IRQCHIP_PIC_MASTER:
            e->set = kvm_set_pic_irq;
            max_pin = PIC_NUM_PINS;
            break;
        case KVM_IRQCHIP_PIC_SLAVE:
            e->set = kvm_set_pic_irq;
            max_pin = PIC_NUM_PINS;
            delta = 8;
            break;
        case KVM_IRQCHIP_IOAPIC:
            max_pin = KVM_IOAPIC_NUM_PINS;
            e->set = kvm_set_ioapic_irq;
            break;
        default:
            goto out;
        }
        e->irqchip.irqchip = ue->u.irqchip.irqchip;
        e->irqchip.pin = ue->u.irqchip.pin + delta;
        if (e->irqchip.pin >= max_pin)
            goto out;
        break;
    case KVM_IRQ_ROUTING_MSI:
        e->set = kvm_set_msi;
        e->msi.address_lo = ue->u.msi.address_lo;
        e->msi.address_hi = ue->u.msi.address_hi;
        e->msi.data = ue->u.msi.data;

        if (kvm_msi_route_invalid(kvm, e))
            goto out;
        break;
    case KVM_IRQ_ROUTING_HV_SINT:
        e->set = kvm_hv_set_sint;
        e->hv_sint.vcpu = ue->u.hv_sint.vcpu;
        e->hv_sint.sint = ue->u.hv_sint.sint;
        break;
    default:
        goto out;
    }

    r = 0;
out:
    return r;
}
```

这里就设置了上文所诉 kvm_kernel_irq_routing_entry 结构的 set 函数。对于普通中断(KVM_IRQ_ROUTING_IRQCHIP)，会根据不同的中断控制器(irqchip)设置不同的set函数：

* 对于PIC，由于type为 KVM_IRQ_ROUTING_IRQCHIP ，因此 set为 kvm_set_pic_irq
* 对于IOAPIC，由于type为 KVM_IRQCHIP_PIC_MASTER / KVM_IRQCHIP_PIC_SLAVE ，因此 set为 kvm_set_ioapic_irq

至此KVM中的中断路由初始化完毕。





### 中断注入
如果设备是在QEMU中模拟的，则产生中断时需要进行中断注入。

在QEMU对KVM加速器进行初始化的函数 kvm_init 中，有：

```c
    s->irq_set_ioctl = KVM_IRQ_LINE;
    if (kvm_check_extension(s, KVM_CAP_IRQ_INJECT_STATUS)) {
        s->irq_set_ioctl = KVM_IRQ_LINE_STATUS;
    }

#define KVM_IRQ_LINE              _IOW(KVMIO,  0x61, struct kvm_irq_level)
#define KVM_IRQ_LINE_STATUS       _IOWR(KVMIO, 0x67, struct kvm_irq_level)
```

如果KVM支持返回注入的结果，则设置 s->irq_set_ioctl = KVM_IRQ_LINE_STATUS ，否则为 KVM_IRQ_LINE

于是QEMU会在 kvm_set_irq 中通过ioctl向KVM注入中断：

```c
int kvm_set_irq(KVMState *s, int irq, int level)
{
    struct kvm_irq_level event;
    int ret;

    assert(kvm_async_interrupts_enabled());

    event.level = level;
    event.irq = irq;
    ret = kvm_vm_ioctl(s, s->irq_set_ioctl, &event);
    if (ret < 0) {
        perror("kvm_set_irq");
        abort();
    }

    return (s->irq_set_ioctl == KVM_IRQ_LINE) ? 1 : event.status;
}
```

在 KVM 中调用链为 kvm_vm_ioctl => kvm_vm_ioctl_irq_line => kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irq_event->irq, irq_event->level, line_status)


```c
/*
 * Return value:
 *  < 0   Interrupt was ignored (masked or not delivered for other reasons)
 *  = 0   Interrupt was coalesced (previous irq is still pending)
 *  > 0   Number of CPUs interrupt was delivered to
 */
int kvm_set_irq(struct kvm *kvm, int irq_source_id, u32 irq, int level,
        bool line_status)
{
    struct kvm_kernel_irq_routing_entry irq_set[KVM_NR_IRQCHIPS];
    int ret = -1, i, idx;

    trace_kvm_set_irq(irq, level, irq_source_id);

    /* Not possible to detect if the guest uses the PIC or the
     * IOAPIC.  So set the bit in both. The guest will ignore
     * writes to the unused one.
     */
    idx = srcu_read_lock(&kvm->irq_srcu);
    // 查询 kvm->irq_routing ，将对应中断号(pin?)的 kvm_kernel_irq_routing_entry 一一取出，设置到 irq_set 返回
    i = kvm_irq_map_gsi(kvm, irq_set, irq);
    srcu_read_unlock(&kvm->irq_srcu, idx);

    // 调用kvm_kernel_irq_routing_entry的set函数设置中断，如果芯片没实现，则set为空
    while (i--) {
        int r;
        r = irq_set[i].set(&irq_set[i], kvm, irq_source_id, level,
                   line_status);
        if (r < 0)
            continue;

        ret = r + ((ret < 0) ? 0 : ret);
    }

    return ret;
}
```

其中 irq_source_id 为中断源设备id， irq 为原始中断请求号(未转换成gsi)， level 表示中断的高低电平。

这里从中断路由表中找到对应的 entry ，调用中断路由初始化时设置的set函数。前面提到过：

* 对于PIC，set为 kvm_set_pic_irq
* 对于IOAPIC，set为 kvm_set_ioapic_irq

#### kvm_set_pic_irq

```
kvm_set_pic_irq => pic_irqchip                                              找到对应的中断芯片
                => kvm_pic_set_irq => pic_set_irq1                          设置irq对应的pin，设置irr(interrupt request register)
                                   => pic_update_irq => pic_irq_request     发送中断请求
```

其中：

```c
static void pic_irq_request(struct kvm *kvm, int level)
{
    struct kvm_pic *s = pic_irqchip(kvm);

    if (!s->output)
        s->wakeup_needed = true;
    s->output = level;
}
```

负责设置中断芯片 kvm_pic 中的 output 为对应电平。同时如果原来 output 为0，则设置 wakeup_needed 为true，于是在 pic_unlock 中会调用 `kvm_make_request(KVM_REQ_EVENT, found)` 设置请求然后通过 kvm_vcpu_kick 让目标vCPU退出来处理请求。


#### kvm_set_ioapic_irq

kvm_set_ioapic_irq => kvm_ioapic_set_irq => ioapic_set_irq => ioapic_service

```
ioapic_service
=> 创建并初始化中断消息 kvm_lapic_irq
=> kvm_irq_delivery_to_apic                 将中断消息发送到LAPIC
```

kvm_lapic_irq 为 IOAPIC格式化后的中断消息，定义如下：

```c
struct kvm_lapic_irq {
    u32 vector;
    u16 delivery_mode;
    u16 dest_mode;
    bool level;
    u16 trig_mode;
    u32 shorthand;
    u32 dest_id;
    bool msi_redir_hint;
};
```

将消息 kvm_lapic_irq 作为参数，调用 kvm_irq_delivery_to_apic

```c
int kvm_irq_delivery_to_apic(struct kvm *kvm, struct kvm_lapic *src,
        struct kvm_lapic_irq *irq, struct dest_map *dest_map)
{
    int i, r = -1;
    struct kvm_vcpu *vcpu, *lowest = NULL;
    unsigned long dest_vcpu_bitmap[BITS_TO_LONGS(KVM_MAX_VCPUS)];
    unsigned int dest_vcpus = 0;

    if (irq->dest_mode == 0 && irq->dest_id == 0xff &&
            kvm_lowest_prio_delivery(irq)) {
        printk(KERN_INFO "kvm: apic: phys broadcast and lowest prio\n");
        irq->delivery_mode = APIC_DM_FIXED;
    }

    if (kvm_irq_delivery_to_apic_fast(kvm, src, irq, &r, dest_map))
        return r;

    memset(dest_vcpu_bitmap, 0, sizeof(dest_vcpu_bitmap));

    kvm_for_each_vcpu(i, vcpu, kvm) {
        if (!kvm_apic_present(vcpu))
            continue;

        if (!kvm_apic_match_dest(vcpu, src, irq->shorthand,
                    irq->dest_id, irq->dest_mode))
            continue;

        if (!kvm_lowest_prio_delivery(irq)) {
            if (r < 0)
                r = 0;
            r += kvm_apic_set_irq(vcpu, irq, dest_map);
        } else if (kvm_lapic_enabled(vcpu)) {
            if (!kvm_vector_hashing_enabled()) {
                if (!lowest)
                    lowest = vcpu;
                else if (kvm_apic_compare_prio(vcpu, lowest) < 0)
                    lowest = vcpu;
            } else {
                __set_bit(i, dest_vcpu_bitmap);
                dest_vcpus++;
            }
        }
    }

    if (dest_vcpus != 0) {
        int idx = kvm_vector_to_index(irq->vector, dest_vcpus,
                    dest_vcpu_bitmap, KVM_MAX_VCPUS);

        lowest = kvm_get_vcpu(kvm, idx);
    }

    if (lowest)
        r = kvm_apic_set_irq(lowest, irq, dest_map);

    return r;
}
```

该函数除了可以处理外部中断(ioapic => lapic)，还可以处理IPI(lapic => lapic, 见 apic_send_ipi)。

它首先尝试从 kvm.arch.apic_map 中找到目标LAPIC。 kvm.arch.apic_map 定义如下：

```c
struct kvm_apic_map {
    struct rcu_head rcu;
    u8 mode;
    u32 max_apic_id;
    union {
        struct kvm_lapic *xapic_flat_map[8];
        struct kvm_lapic *xapic_cluster_map[16][4];
    };
    struct kvm_lapic *phys_map[];               // 维护了LAPIC ID到 kvm_lapic 指针的映射
};
```

于是 kvm_irq_delivery_to_apic_fast => kvm_apic_map_get_dest_lapic 中，对于不是广播和最低优先级的中断，可以直接根据 irq->dest_id 从 phys_map 中取出对应的 kvm_lapic 。然后直接 kvm_apic_set_irq 对目标vCPU设置中断。否则需要遍历所有的vCPU，逐一的和RTE的 irq->dest_id 进行匹配。对匹配的vcpu调用 kvm_apic_set_irq 。


kvm_apic_set_irq 实现为该vcpu的lapic设置中断：

```
=> __apic_accept_irq
    => 根据 delivery_mode 进行对应设置，如 APIC_DM_FIXED 为 kvm_lapic_set_vector + kvm_lapic_set_irr
    => kvm_make_request(event, vcpu) ，event 可取 KVM_REQ_EVENT / KVM_REQ_SMI / KVM_REQ_NMI
    => kvm_vcpu_kick(vcpu)              让目标vCPU退出来处理请求
```

kvm_make_request 本质上是设置 vcpu->requests 中请求对应的bit ，在下次 vcpu_enter_guest 时会对请求进行处理。



##### kvm_vcpu_kick

kvm_vcpu_kick => smp_send_reschedule (native_smp_send_reschedule) => apic->send_IPI(cpu, RESCHEDULE_VECTOR) (x2apic_send_IPI)

向目标vcpu产生一个中断，让其重新被调度，由于在VMCS中设置了外部中断会发生 VMExit，因此返回到 KVM ，从而能够实现在其重新 VMENTRY (vcpu_enter_guest) 之前注入中断

于是 kvm_x86_ops->run (vmx_vcpu_run) 返回到 vcpu_enter_guest 再到 vcpu_run 进入下一轮循环，于是又调用 vcpu_enter_guest ：

```
vcpu_enter_guest => inject_pending_event        run前检查请求，如果kvm_check_request(KVM_REQ_EVENT, vcpu)，在运行vcpu前进行中断注入
                 => kvm_x86_ops->run            VMLAUNCH/VMRESUME
                 => vmx->idt_vectoring_info = vmcs_read32(IDT_VECTORING_INFO_FIELD)
                 => vmx_complete_interrupts => __vmx_complete_interrupts    根据中断信息更新vcpu，该入队的入队
```

具体流程是：

```c
    if (kvm_check_request(KVM_REQ_EVENT, vcpu) || req_int_win) {
        kvm_apic_accept_events(vcpu);
        if (vcpu->arch.mp_state == KVM_MP_STATE_INIT_RECEIVED) {
            r = 1;
            goto out;
        }
        // 中断注入
        if (inject_pending_event(vcpu, req_int_win) != 0)
            req_immediate_exit = true;
        else {
            /* Enable NMI/IRQ window open exits if needed.
             *
             * SMIs have two cases: 1) they can be nested, and
             * then there is nothing to do here because RSM will
             * cause a vmexit anyway; 2) or the SMI can be pending
             * because inject_pending_event has completed the
             * injection of an IRQ or NMI from the previous vmexit,
             * and then we request an immediate exit to inject the SMI.
             */
            if (vcpu->arch.smi_pending && !is_smm(vcpu))
                req_immediate_exit = true;
            if (vcpu->arch.nmi_pending)
                kvm_x86_ops->enable_nmi_window(vcpu);
            if (kvm_cpu_has_injectable_intr(vcpu) || req_int_win)
                kvm_x86_ops->enable_irq_window(vcpu);
        }

        if (kvm_lapic_enabled(vcpu)) {
            update_cr8_intercept(vcpu);
            kvm_lapic_sync_to_vapic(vcpu);
        }
    }
```

发现有 KVM_REQ_EVENT ，于是调用 inject_pending_event

```
=> 如果有 pending的异常 ，调用 kvm_x86_ops->queue_exception (vmx_queue_exception) 重新排队
=> 如果 nmi_injected ，调用 kvm_x86_ops->set_nmi (vmx_inject_nmi)
=> 如果有 pending的中断 ，调用 kvm_x86_ops->set_irq (vmx_inject_irq)
=> 如果有 pending的不可屏蔽中断 ，调用 kvm_x86_ops->set_nmi (vmx_inject_nmi)
=> kvm_cpu_has_injectable_intr                                                          如果vCPU有可注入的中断
=> kvm_queue_interrupt(vcpu, kvm_cpu_get_interrupt(vcpu), false)                        将最高优先级的中断设置到 vcpu->arch.interrupt 中
=> kvm_x86_ops->set_irq (vmx_inject_irq)                                                将中断信息写入VMCS
    => vmcs_write32(VM_ENTRY_INSTRUCTION_LEN, vmx->vcpu.arch.event_exit_inst_len)       对于软中断，需要写指令长度
    => vmcs_write32(VM_ENTRY_INTR_INFO_FIELD, intr)                                     更新中断信息区域
```

##### kvm_cpu_has_injectable_intr
用于判断是否有可注入的中断。

=> lapic_in_kernel                      如果 LAPIC 不在KVM中，表示由QEMU负责模拟，于是 vcpu.arch.interrupt 早已被设置好，返回 interrupt.pending
=> kvm_cpu_has_extint                   如果有pending的外部(非non-APIC)中断，返回 true
=> kvm_vcpu_apicv_active                如果启用了virtual interrupt delivery，则APIC的中断会由硬件处理，无需软件干涉，返回 false
=> kvm_apic_has_interrupt               如果 LAPIC 在KVM中，找到优先级最高的中断号，如果其大于PPR，返回 true
    => apic_update_ppr                                                    更新PPR
    => apic_find_highest_irr => apic_search_irr => find_highest_vector    从IRR中找到优先级最高的中断号
    => 如果该中断号小于等于PPR，则返回-1


#### 小结

在重新 run 前，判断是否有中断请求，如果有，则检查LAPIC的中断队列，找到优先级最高的中断，如果其中断向量号大于PPR(Processor Priority Register)，则需要进行注入。

于是设置 vcpu->arch.interrupt (kvm_queued_interrupt)，其中 pending 设置为true

```c
struct kvm_queued_interrupt {
    bool pending;
    bool soft;      // 是否软中断
    u8 nr;          // 中断向量号
} interrupt;
```


在VMEXIT时，如果注入成功，会在 vmx_vcpu_run => vmx_complete_interrupts => __vmx_complete_interrupts => kvm_clear_interrupt_queue 将 pending 设置为 false。

如果注入失败，会在 __vmx_complete_interrupts 调用 requeue ，重新进行注入。









### QEMU模拟芯片
我们知道，QEMU中的设备都是通过 TypeInfo 定义，然后以 TypeImpl 进行注册。在创建设备时，调用 class_init 初始化类对象，然后调用 instance_init 初始化类实例对象，最后通过 realize 完成设备的构造。

#### PIC

```c
static const TypeInfo i8259_info = {
    .name       = TYPE_I8259,
    .instance_size = sizeof(PICCommonState),
    .parent     = TYPE_PIC_COMMON,
    .class_init = i8259_class_init,
    .class_size = sizeof(PICClass),
    .interfaces = (InterfaceInfo[]) {
        { TYPE_INTERRUPT_STATS_PROVIDER },
        { }
    },
};
```

在 pc_q35_init 中，有以下一段代码：

```c
i8259 = i8259_init(isa_bus, pc_allocate_cpu_irq());
for (i = 0; i < ISA_NUM_IRQS; i++) {
    gsi_state->i8259_irq[i] = i8259[i];
}
```

这里首先为 PIC 设备分配一个中断(parent_irq)，于是调用 pc_allocate_cpu_irq => qemu_allocate_irq(pic_irq_request, NULL, 0) ，创建了一个序号为0，handler为 pic_irq_request 的中断。作为上游中断。

然后初始化 PIC ：

```c
qemu_irq *i8259_init(ISABus *bus, qemu_irq parent_irq)
{
    qemu_irq *irq_set;
    DeviceState *dev;
    ISADevice *isadev;
    int i;

    irq_set = g_new0(qemu_irq, ISA_NUM_IRQS);

    // 创建PIC master device，挂到 isa_bus 上
    isadev = i8259_init_chip(TYPE_I8259, bus, true);
    dev = DEVICE(isadev);

    qdev_connect_gpio_out(dev, 0, parent_irq);
    for (i = 0 ; i < 8; i++) {
        irq_set[i] = qdev_get_gpio_in(dev, i);
    }

    isa_pic = dev;
    // 创建PIC slave device，挂到 isa_bus 上
    isadev = i8259_init_chip(TYPE_I8259, bus, false);
    dev = DEVICE(isadev);

    qdev_connect_gpio_out(dev, 0, irq_set[2]);
    for (i = 0 ; i < 8; i++) {
        irq_set[i + 8] = qdev_get_gpio_in(dev, i);
    }

    slave_pic = PIC_COMMON(dev);

    return irq_set;
}
```

其负责创建两个8259中断芯片的类实例对象。i8259_init_chip => qdev_init_nofail => ... => pic_realize

```c
static void pic_realize(DeviceState *dev, Error **errp)
{
    PICCommonState *s = PIC_COMMON(dev);
    PICClass *pc = PIC_GET_CLASS(dev);

    memory_region_init_io(&s->base_io, OBJECT(s), &pic_base_ioport_ops, s,
                          "pic", 2);
    memory_region_init_io(&s->elcr_io, OBJECT(s), &pic_elcr_ioport_ops, s,
                          "elcr", 1);

    qdev_init_gpio_out(dev, s->int_out, ARRAY_SIZE(s->int_out));
    qdev_init_gpio_in(dev, pic_set_irq, 8);

    pc->parent_realize(dev, errp);
}
```

##### qdev_init_gpio_out

```
=> qdev_init_gpio_out_named(dev, pins, NULL, n)
    => qdev_get_named_gpio_list         从设备实例(DeviceState)中取出gpio，遍历该链表找到对应名称的 NamedGPIOList ，如果找不到，创建一个，插到最前
    => 如果未传入名称，设置name为 "unnamed-gpio-out"
    => object_property_add_link         根据传入的 qemu_irq 数组和长度，将每个 qemu_irq *指针*以名称 "name[i]" 作为dev的link属性
    => NamedGPIOList.num_out +=n
```

##### qdev_init_gpio_in

```
=> qdev_init_gpio_in_named(dev, handler, NULL, n)
    => qdev_get_named_gpio_list         从设备实例(DeviceState)中取出gpio，遍历该链表找到对应名称的 NamedGPIOList ，如果找不到，创建一个，插到最前
    => NamedGPIOList.in = qemu_extend_irqs(gpio_list->in, gpio_list->num_in, handler, dev, n)  在原有基础上创建n个 qemu_irq ，handler为传入函数
    => 如果未传入名称，设置name为 "unnamed-gpio-in"
    => 根据传入的数目，将每个 qemu_irq 以名称 "name[i]" 作为dev的child属性
    => NamedGPIOList.num_in 增加n
```

于是每个 8259 会有1个out，8个in GPIO，存在名字为NULL的 DeviceState.gpios 链表中

其中out对应的"unnamed-gpio-out[0]"的值是一个 qemu_irq *指针*，指向 s->int_out ，存放的是该成员的地址，而其还没有设置。
而in对应的"unnamed-gpio-in[i]"的值是一个 qemu_irq 。其handler为 pic_set_irq ，opaque为 dev


#### PIC 连接

在创建了8259中断芯片的类实例对象后， i8259_init 对主片(master)调用 qdev_connect_gpio_out 进行连接 ：

qdev_connect_gpio_out(dev, 0, parent_irq) => qdev_connect_gpio_out_named(dev, NULL, n, pin)  *<-pin就是parent_irq*

##### qdev_connect_gpio_out_named

```
=> object_property_add_child    将上级中断(parent_irq) 以 "non-qdev-gpio[*]" 为名作为 "/machine/unattached" container 的child属性。
=> object_property_set_link     设置dev名为 "name[i]" 的link属性值为 parent_irq 。该属性也就是前面的 qdev_init_gpio_out 中创建的 "name[i]" ，值指向 s->int_out ，于是 s->int_out 就被设置为了 parent_irq ，out的坑被填上了。
```

如果上级中断已有路径，则 `child->parent != NULL`， object_property_add_child 返回。否则进行添加为 "/machine/unattached" container的child。这里有个细节是实际上它们的属性名不是"non-qdev-gpio[*]"， 因为在 object_property_add_child => object_property_add 中，会尝试从0开始替换掉`*`。比如这里的 parent_irq 被分到的属性名为 "non-qdev-gpio[24]" ，于是完整路径为 "/machine/unattached/non-qdev-gpio[24]" ，这点在 qom-tree 也有体现：

```
/machine (pc-q35-2.8-machine)
  /unattached (container)
    /non-qdev-gpio[24] (irq)
```

之所以要把上级中断(如 parent_irq )设置为 "/machine/unattached" container 的child属性，是因为在接下来的 object_property_set_link 中需要上级中断有自己的路径：

```c
void object_property_set_link(Object *obj, Object *value,
                              const char *name, Error **errp)
{
    if (value) {
        // 取出上级中断的路径
        gchar *path = object_get_canonical_path(value);
        object_property_set_str(obj, path, name, errp);
        g_free(path);
    } else {
        object_property_set_str(obj, "", name, errp);
    }
}
```

##### qdev_get_gpio_in => qdev_get_gpio_in_named

获取 qdev_init_gpio_in 中创建的存在 NamedGPIOList.in 中的8个 qemu_irq ，存到 irq_set 数组的 0-7 位置

从片(slave)也会通过类似的过程来初始化，只不过其上游端口不是parent_irq，而是主片的第三个qemu_irq，即 irq_set[2] ，模拟了硬件上从片 out 接到主片的 irq2 引脚的电路。最后获取 qdev_init_gpio_in 中创建的存在 NamedGPIOList.in 中的8个 qemu_irq ，存到 irq_set 数组的 8-15 位置。

PIC初始化完成后，将 irq_set 返回，被存到 gsi_state->i8259_irq 。



#### LAPIC

pc_new_cpu ==> apic_init(env,env->cpuid_apic_id) ==> qdev_create(NULL, "kvm-apic");

根据 x86_cpu_realizefn => x86_cpu_apic_create => apic_get_class ，此时LAPIC不放在KVM中，由QEMU负责对其进行模拟，于是 apic_type = "apic" ：

```c
static const TypeInfo apic_common_type = {
    .name = TYPE_APIC_COMMON,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(APICCommonState),
    .instance_init = apic_common_initfn,
    .class_size = sizeof(APICCommonClass),
    .class_init = apic_common_class_init,
    .abstract = true,
};
```

apic_common_realize => apic_realize

```c
static void apic_realize(DeviceState *dev, Error **errp)
{
    APICCommonState *s = APIC(dev);

    if (s->id >= MAX_APICS) {
        error_setg(errp, "%s initialization failed. APIC ID %d is invalid",
                   object_get_typename(OBJECT(dev)), s->id);
        return;
    }

    memory_region_init_io(&s->io_memory, OBJECT(s), &apic_io_ops, s, "apic-msi",
                          APIC_SPACE_SIZE);

    s->timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, apic_timer, s);
    local_apics[s->id] = s;

    msi_nonbroken = true;
}
```

可以看到它为MSI注册了对应的 MemoryRegion，当对该 MemoryRegion 进行操作时，执行以下操作：

```c
static const MemoryRegionOps apic_io_ops = {
    .old_mmio = {
        .read = { apic_mem_readb, apic_mem_readw, apic_mem_readl, },
        .write = { apic_mem_writeb, apic_mem_writew, apic_mem_writel, },
    },
    .endianness = DEVICE_NATIVE_ENDIAN,
};
```


#### IOAPIC

定义如下：

```c
static const TypeInfo ioapic_info = {
    .name          = "ioapic",
    .parent        = TYPE_IOAPIC_COMMON,
    .instance_size = sizeof(IOAPICCommonState),
    .class_init    = ioapic_class_init,
};
```

如果开启了PIC， pc_q35_init 会调用 `ioapic_init_gsi(gsi_state, "q35");` 初始化IOAPIC：

```c
void ioapic_init_gsi(GSIState *gsi_state, const char *parent_name)
{
    DeviceState *dev;
    SysBusDevice *d;
    unsigned int i;

    if (kvm_ioapic_in_kernel()) {
        dev = qdev_create(NULL, "kvm-ioapic");
    } else {
        dev = qdev_create(NULL, "ioapic");
    }
    if (parent_name) {
        object_property_add_child(object_resolve_path(parent_name, NULL),
                                  "ioapic", OBJECT(dev), NULL);
    }
    qdev_init_nofail(dev);
    d = SYS_BUS_DEVICE(dev);
    sysbus_mmio_map(d, 0, IO_APIC_DEFAULT_ADDRESS);

    for (i = 0; i < IOAPIC_NUM_PINS; i++) {
        gsi_state->ioapic_irq[i] = qdev_get_gpio_in(dev, i);
    }
}
```

此时IOAPIC不放在KVM中，由QEMU负责对其进行模拟。 于是 qdev_init_nofail => ... => ioapic_realize

```c
static void ioapic_realize(DeviceState *dev, Error **errp)
{
    IOAPICCommonState *s = IOAPIC_COMMON(dev);

    if (s->version != 0x11 && s->version != 0x20) {
        error_report("IOAPIC only supports version 0x11 or 0x20 "
                     "(default: 0x11).");
        exit(1);
    }

    memory_region_init_io(&s->io_memory, OBJECT(s), &ioapic_io_ops, s,
                          "ioapic", 0x1000);

    qdev_init_gpio_in(dev, ioapic_set_irq, IOAPIC_NUM_PINS);

    ioapics[ioapic_no] = s;
    s->machine_done.notify = ioapic_machine_done_notify;
    qemu_add_machine_init_done_notifier(&s->machine_done);
}
```

同样是通过 qdev_init_gpio_in 创建in的 qemu_irq ，一共创建24个，handler为 ioapic_set_irq

IOAPIC被存到全局数组 ioapics 中，index也由全局变量 ioapic_no 来维护。

于是IOAPIC会有24个in GPIO。区别于 PIC 将创建的 qemu_irq 返回并由 pc_q35_init 负责设置到 gsi_state 中，ioapic_init_gsi 直接传入 gsi_state 指针，在函数内对 gsi_state->ioapic_irq 进行设置。

#### GSI

至此， gsi_state 中 i8259_irq 和 ioapic_irq 都被填充完毕。而实际上在初始化 PIC 和 IOAPIC 前， pc_q35_init 会创建 GSI 的 qemu_irq ：

```c
    // 如果 ioapic 在kernel(KVM)中，即在开启kvm的情况下，指定参数 kernel-irqchip=on ，则
    if (kvm_ioapic_in_kernel()) {
        kvm_pc_setup_irq_routing(pcmc->pci_enabled);                        // 创建中断路由，并设置到KVM
        pcms->gsi = qemu_allocate_irqs(kvm_pc_gsi_handler, gsi_state,
                                       GSI_NUM_PINS);
    }
    // 否则 (off / split时ioapic在QEMU中)
    else {
        pcms->gsi = qemu_allocate_irqs(gsi_handler, gsi_state, GSI_NUM_PINS);
    }
```

会创建 GSI_NUM_PINS(24) 个 qemu_irq ，编号从0-23，opaque指向 gsi_state ，handler 为 kvm_pc_gsi_handler (IOAPIC由KVM模拟时) / gsi_handler (IOAPIC由QEMU模拟时) ，保存到 PCMachineState.gsi 中。

随后 pc_q35_init 会通过 pci_create_simple_multifunction 创建并初始化 ICH9-LPC ，调用其 realize 函数：

```
ich9_lpc_realize => isa_bus = isa_bus_new(...)                                              创建 ISABus
                 => lpc->isa_bus = isa_bus                                                  将 ISABus 设置为 ICH9LPCState 的成员
                 => qdev_init_gpio_out_named(dev, lpc->gsi, ICH9_GPIO_GSI, GSI_NUM_PINS)    创建 24 个out GPIO
                 => isa_bus_irqs(isa_bus, lpc->gsi)                                         将 ISABus.irqs 设置为 ICH9LPCState.gsi
```

通过 qdev_init_gpio_out_named 创建了24个out GPIO(qemu_irq)，存在设备父类 DeviceState 的 gpios 链表中，名为"gsi"。每个 qemu_irq *指针*以名称 "name[i]" 作为dev的link属性，指向 ICH9LPCState.gsi 数组成员的地址。

接下来的 isa_bus_irqs 将 ISABus.irqs 设置为 ICH9LPCState.gsi 。也就是说， ISABus.irqs 指向了 ICH9-LPC out GPIO(qemu_irq) 所指向的值 。

接下来 pc_q35_init 会将 ICH9LPCState 刚创建的 out GPIO 一一连接到 pcms->gsi ：

```c
    for (i = 0; i < GSI_NUM_PINS; i++) {
        qdev_connect_gpio_out_named(lpc_dev, ICH9_GPIO_GSI, i, pcms->gsi[i]);
    }
```

于是 ISABus.irqs 等同于 PCMachineState.gsi，即 ISABus.irqs[i] == PCMachineState.gsi[i] 。如图所示：

```
PCMachineState.gsi  (gsi_handler)
| | | | | | ... |
0 1 2 3 4 5     23
| | | | | | ... |  out
------------------
|    ICH9-LPC    |
------------------
   (ISABus.irqs)
```


可以用GDB进行验证：

```
(gdb) p *isa_bus->irqs
$15 = (qemu_irq) 0x55555694f8c0
(gdb) p *PC_MACHINE(qdev_get_machine())->gsi
$16 = (qemu_irq) 0x55555694f8c0
```


#### 中断注入

不同于KVM模拟的中断芯片(IOAPIC)通过查找获取目标LAPIC然后直接设置其变量来传递中断，QEMU通过GPIO。

一般来说，产生中断的设备的 irq 成员都会设置为 PCMachineState.gsi 。以串口(isa-serial)为例，在其 realize 函数 serial_isa_realizefn 中，调用了 `isa_init_irq(isadev, &s->irq, isa->isairq)` ，设置设备对象(SerialState)的 irq 成员，其中 isa->isairq 通过 isa_serial_irq[isa->index] 得到， index 是 serial_isa_realizefn 中的静态变量，每调用一次加一。

根据 isa_serial_irq 的定义，共有4个串口设备，对应的 isairq 分别为 4, 3, 4, 3 。对于 index 为 0 的串口设备，其 isairq 为 4，于是：

```c
void isa_init_irq(ISADevice *dev, qemu_irq *p, int isairq)
{
    assert(dev->nirqs < ARRAY_SIZE(dev->isairq));
    dev->isairq[dev->nirqs] = isairq;
    *p = isa_get_irq(dev, isairq);
    dev->nirqs++;
}
```

调用 isa_get_irq 从 isabus->irqs 中取出对应的 qemu_irq (isabus->irqs[4])，将其设置到串口设备的类实例对象，即 SerialState.irq 。

前面提到，ISABus.irqs 等同于 PCMachineState.gsi 。于是串口设备的 irq 实际上指向了 GSI qemu_irq 。这相当于每个设备都对应到GSI上。GSI qemu_irq 的 handler 为 gsi_handler ，n指定了其在GSIState 数组中的序号。

```
PCMachineState.gsi  (gsi_handler)
          |
          4
          | irq
-----------------------
|  isa-serial device  |
-----------------------
```


继续中断注入的分析。设备在发送中断时会调用以下两个函数设置电平：

* qemu_irq_lower => qemu_set_irq(irq, 0)       设为低电平
* qemu_irq_raise => qemu_set_irq(irq, 1)       设为高电平


##### qemu_set_irq

=> irq->handler(irq->opaque, irq->n, level) 负责取出 qemu_irq 中的 handler 进行调用。

由于属于 GSIState ，因此调用的是 gsi_handler ：

```c
void gsi_handler(void *opaque, int n, int level)
{
    GSIState *s = opaque;

    DPRINTF("pc: %s GSI %d\n", level ? "raising" : "lowering", n);
    if (n < ISA_NUM_IRQS) {
        qemu_set_irq(s->i8259_irq[n], level);
    }
    qemu_set_irq(s->ioapic_irq[n], level);
}
```

它根据序号(qemu_irq.n)，取出对应芯片的对应 qemu_irq 的 handler 进行调用。

#### PIC

```
parent_irq (pic_irq_request)
    | out
-----------------
|  8259 master  |
-----------------
| | | | | | | |
0 1 | 3 4 5 6 7     (pic_set_irq)
    |
    | out
-----------------
|  8259 slave   |
-----------------
| | | | | | | |
0 1 2 3 4 5 6 7     (pic_set_irq)
```

对于 PIC ，handler为 pic_set_irq ，属于PIC 的 in 。于是 pic_set_irq 设置PIC芯片(PICCommonState)的irr寄存器(变量)，然后调用 pic_update_irq ：

```c
static void pic_update_irq(PICCommonState *s)
{
    int irq;

    irq = pic_get_irq(s);
    if (irq >= 0) {
        DPRINTF("pic%d: imr=%x irr=%x padd=%d\n",
                s->master ? 0 : 1, s->imr, s->irr, s->priority_add);
        qemu_irq_raise(s->int_out[0]);
    } else {
        qemu_irq_lower(s->int_out[0]);
    }
}

```

通过 pic_get_irq 获取 irr 中没被imr屏蔽掉的优先级最高的中断，如果有，则设置 out (s->int_out[0]) 为高电平，否则设置为低电平。于是

```
qemu_set_irq => pic_irq_request => 如果CPU有LAPIC，调用 apic_deliver_pic_intr 设置到LAPIC
                                => 否则根据电平调用 cpu_interrupt / cpu_reset_interrupt
```

这里有一个有趣的地方：在SMP中，PIC的中断应该发送给哪个CPU呢？QEMU的实现简单粗暴，根据 pic_irq_request ，其选择的是第一个CPU(first_cpu)。

#### IOAPIC

```
------------------
|    IOAPIC      |
------------------
| | | | | | ... |
0 1 | 3 4 5     23  in (ioapic_set_irq)
```

对于 IOAPIC ， handler 为 ioapic_set_irq ，属于 IOAPIC 的 in。

```c
static void ioapic_set_irq(void *opaque, int vector, int level)
{
    IOAPICCommonState *s = opaque;

    /* ISA IRQs map to GSI 1-1 except for IRQ0 which maps
     * to GSI 2.  GSI maps to ioapic 1-1.  This is not
     * the cleanest way of doing it but it should work. */

    DPRINTF("%s: %s vec %x\n", __func__, level ? "raise" : "lower", vector);
    if (vector == 0) {
        vector = 2;
    }
    if (vector >= 0 && vector < IOAPIC_NUM_PINS) {
        uint32_t mask = 1 << vector;
        uint64_t entry = s->ioredtbl[vector];

        if (((entry >> IOAPIC_LVT_TRIGGER_MODE_SHIFT) & 1) ==
            IOAPIC_TRIGGER_LEVEL) {
            /* level triggered */
            if (level) {
                s->irr |= mask;
                if (!(entry & IOAPIC_LVT_REMOTE_IRR)) {
                    ioapic_service(s);
                }
            } else {
                s->irr &= ~mask;
            }
        } else {
            /* According to the 82093AA manual, we must ignore edge requests
             * if the input pin is masked. */
            if (level && !(entry & IOAPIC_LVT_MASKED)) {
                s->irr |= mask;
                ioapic_service(s);
            }
        }
    }
}
```

首先，它从 IOAPIC 的 I/O REDIRECTION TABLE 中找到中断向量号所对应的 entry 寄存器。其中包含 Interrupt Mask、Trigger Mode 、Remote IRR 等bit。如果 Trigger Mode bit 为1，表示水平触发，0表示边缘触发。

对于水平触发，在设置irr中对应的bit后，需要判断 Remote IRR bit ，如果为1，表示 LAPIC 已经收到 IOAPIC 发来的中断了，正在处理中；如果为0，表示 LAPIC 已经处理完中断，向 IOAPIC 发送 EOI 消息，表示可以继续接收中断。因此如果为0，则可以调用 ioapic_service 发送中断消息。

对于边缘触发，需要判断 Interrupt Mask bit ，如果为1，表示中断被屏蔽，无需设置irr；如果为0，表示可以发送中断，于是设置irr中对应的bit后，调用 ioapic_service 发送中断消息。

ioapic_service 会遍历 IOAPIC 上的所有pin，如果 irr 在对应的bit为1，则需要发送中断：

若LAPIC在KVM中(kernel-irq=split)，则通过 kvm_set_irq 设置到KVM中，否则将其转换成 *MSI* 。根据定义，设备可以直接构造MSI消息，其中标明了中断目标地址，然后由设备直接发送中断给LAPIC，绕过了IOAPIC。

由于我们讨论 LAPIC 由 QEMU 模拟的情况，因此其先用pin号查询 I/O REDIRECTION TABLE (IOAPICCommonState.ioredtbl)得到 entry ，然后通过 ioapic_entry_parse 得到相关信息 (ioapic_entry_info) ，最后通过 `stl_le_phys(ioapic_as, info.addr, info.data)` 修改 IOAPIC AddressSpace 。


如果开启了IR， IOAPIC AddressSpace 是一个虚拟机的地址空间 `vtd_host_dma_iommu(bus, s, Q35_PSEUDO_DEVFN_IOAPIC)` 。否则为 address_space_memory 。当对该 AddressSpace 进行写入时，类似MMIO一样最终调用到 MemoryRegion 绑定的 apic_io_ops ，前文提到过，它们在 apic_realize 时被绑定到LAPIC的 apic-msi MemoryRegion。

于是

```
stl_le_phys => address_space_stl_le => address_space_stl_internal => memory_region_dispatch_write => access_with_adjusted_size => memory_region_oldmmio_write_accessor => mr->ops->old_mmio.write[ctz32(size)] (apic_mem_writel)
```

apic_mem_writel 通过 cpu_get_current_apic 获取当前CPU的LAPIC(APICCommonState)，然后根据addr将data写入到其对应位置。

因此 IOAPIC 没有out，其通过MSI将中断送达LAPIC。










#### QEMU模拟PIC、IOAPIC芯片，KVM模拟LAPIC

比起前文所述的中断送达流程，spilt模式下在 ioapic_service 中就会将中断送入KVM中。KVM根据自己的 kvm->irq_routing 进行中断路由。


#### 中断芯片初始化

QEMU在 kvm_init 中，会对KVM进行中断芯片的初始化：

```
kvm_irqchip_create => kvm_arch_irqchip_create => kvm_vm_enable_cap(s, KVM_CAP_SPLIT_IRQCHIP, 0, 24) => kvm_vm_ioctl(s, KVM_ENABLE_CAP, &cap)
                   => kvm_init_irq_routing
```

对于split，只需要在KVM中创建LAPIC而无需 kvm_vm_ioctl(s, KVM_CREATE_IRQCHIP) 。它通过 KVM_ENABLE_CAP 尝试开启 split 能力，然后调用 kvm_init_irq_routing ，初始化IOAPIC所有pin的中断路由。


##### kvm_init_irq_routing

```
=> kvm_check_extension(s, KVM_CAP_IRQ_ROUTING)     获取KVM支持的gsi总数
=> 创建 used_gsi_bitmap ，分配 irq_routes 数组
=> kvm_arch_init_irq_routing => kvm_irqchip_add_msi_route => kvm_add_routing_entry  将entry添加到 KVMState.entries 数组中
                                                          => kvm_irqchip_commit_routes => kvm_vm_ioctl(KVM_SET_GSI_ROUTING) 将entries设置到KVM中
```

kvm_irqchip_add_msi_route 会被调用24次，依次将nr(entries数组的长度)为1到24时的 KVMState.entries 作为 kvm_irq_routing 设置到QEMU中，kvm_irq_routing 定义如下：

```c
struct kvm_irq_routing {
  __u32 nr;
  __u32 flags;
  struct kvm_irq_routing_entry entries[0];
};

struct kvm_irq_routing_entry {
  __u32 gsi;
  __u32 type;
  __u32 flags;
  __u32 pad;
  union {
    struct kvm_irq_routing_irqchip irqchip;
    struct kvm_irq_routing_msi msi;
    struct kvm_irq_routing_s390_adapter adapter;
    struct kvm_irq_routing_hv_sint hv_sint;
    __u32 pad[8];
  } u;
};
```

此时由于设备还未初始化，因此路由表项 kvm_irq_routing_entry 中的属性都为0。

之后直到在虚拟机启动之后，BIOS/OS对中断路由表进行更新时，触发VMExit，退回到QEMU中进行更新(因为IOAPIC在QEMU中模拟)：

```
address_space_rw => address_space_write => address_space_write_continue => memory_region_dispatch_write => access_with_adjusted_size => memory_region_write_accessor => ioapic_mem_write => ioapic_update_kvm_routes
=> ioapic_entry_parse(s->ioredtbl[i], &info)
=> msg.address = info.addr
=> msg.data = info.data
=> kvm_irqchip_update_msi_route(kvm_state, i, msg, NULL) => kvm_update_routing_entry 用entry更新 KVMState.entries 数组
=> kvm_irqchip_commit_routes => kvm_vm_ioctl(s, KVM_SET_GSI_ROUTING, s->irq_routes)     将新的 KVMState.entries 数组更新到KVM中
```

举个例子，e1000对应的gsi 22的entry内容如下：

```
(gdb) p kvm_state->irq_routes->entries[22]
$29 = {
  gsi = 22,
  type = 2,
  flags = 0,
  pad = 0,
  u = {
    irqchip = {
      irqchip = 4276092928,
      pin = 0
    },
    msi = {
      address_lo = 4276092928,
      address_hi = 0,
      data = 32865,
      {
        pad = 0,
        devid = 0
      }
    },
    adapter = {
      ind_addr = 4276092928,
      summary_addr = 32865,
      ind_offset = 0,
      summary_offset = 0,
      adapter_id = 0
    },
    hv_sint = {
      vcpu = 4276092928,
      sint = 0
    },
    pad = {[0] = 4276092928, [1] = 0, [2] = 32865, [3] = 0, [4] = 0, [5] = 0, [6] = 0, [7] = 0}
  }
}
```


#### KVM

在KVM中， kvm_vm_ioctl(s, KVM_ENABLE_CAP, &cap) 的调用链如下：

```
kvm_vm_ioctl_enable_cap => kvm_setup_empty_irq_routing => kvm_set_irq_routing(kvm, empty_routing, 0, 0)
                        => kvm->arch.irqchip_split = true;
```

不同于IOAPIC由KVM模拟时通过 kvm_set_irq_routing 将路由初始化成 default_routing ，在split模式下路由需要等待QEMU来进行设置，因此将其设置为空，即 empty_routing 。

同时设置 kvm->arch.irqchip_split = true ，此后KVM中用于判断是否为split模式的函数 irqchip_split 检查的就是这个变量。


kvm_vm_ioctl(s, KVM_SET_GSI_ROUTING, s->irq_routes) 的调用链如下：

```
kvm_vm_ioctl => kvm_set_irq_routing => setup_routing_entry => kvm_set_routing_entry
                                    => rcu_assign_pointer(kvm->irq_routing, new)
```

它会创建新的 kvm_irq_routing_table ，然后遍历新传入的entrys数组，对每一个entry一一调用 setup_routing_entry ，构造出 kvm_irq_routing_entry 并设置到新table中。最后将 kvm->irq_routing 指向新的table。

由于传入entry的type为 KVM_IRQ_ROUTING_MSI(2)， 因此在 kvm_set_routing_entry 中设置 set 为 kvm_set_msi




#### 中断注入

话题回到spilt模式下在 ioapic_service 中就会调用 kvm_set_irq => kvm_vm_ioctl(s, s->irq_set_ioctl, &event) 向KVM注入中断。 s->irq_set_ioctl 根据 KVM能力可能为 KVM_IRQ_LINE 或 KVM_IRQ_LINE_STATUS ，区别在于后者会返回状态。

于是进到 KVM 中， kvm_vm_ioctl => kvm_vm_ioctl_irq_line

```
=> kvm_irq_map_gsi    查询 kvm->irq_routing ，将对应gsi对应的 kvm_kernel_irq_routing_entry 一一取出
=> kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irq_event->irq, irq_event->level, line_status) => irq_set[i].set (kvm_set_msi)
```

##### kvm_set_msi

```
kvm_set_msi => kvm_set_msi_irq
            => kvm_irq_delivery_to_apic
```

负责将irq消息解析，构造 kvm_lapic_irq ，然后设置到对应vCPU的LAPIC中。

kvm_irq_delivery_to_apic => kvm_apic_set_irq => __apic_accept_irq 实现对目标LAPIC设置中断：

```
=> 根据 delivery_mode 进行对应设置，如 APIC_DM_FIXED 为 kvm_lapic_set_vector + kvm_lapic_set_irr
=> kvm_make_request(KVM_REQ_EVENT, vcpu)
=> kvm_vcpu_kick(vcpu)              让目标vCPU退出来处理请求
```

接下来在 vcpu_run => vcpu_enter_guest 中，由于 LAPIC 在KVM中，先通过 kvm_x86_ops->hwapic_irr_update (vmx_hwapic_irr_update) 更新 irr 中优先级最高的中断？

之后在KVM中检测到有 KVM_REQ_EVENT 请求，调用 inject_pending_event 进行中断注入：

```c
static int inject_pending_event(struct kvm_vcpu *vcpu, bool req_int_win)
{
  ...
  if (vcpu->arch.interrupt.pending) {
      kvm_x86_ops->set_irq(vcpu);
      return 0;
  }
  ...
}
```

最后由 vmx_inject_irq 将中断写入到VMCS中。






















### 中断的完整注入流程

以 e1000 收到包后的中断为例

需要设置中断的场景如下：

* MMIO
    address_space_rw => address_space_read => address_space_read_full => address_space_read_continue => memory_region_dispatch_read => memory_region_dispatch_read1 => access_with_adjusted_size => memory_region_read_accessor => e1000_mmio_read => mac_icr_read / ... => set_interrupt_cause => pci_set_irq

* QEMU收到e1000的包

    main_loop => main_loop_wait => qemu_clock_run_all_timers => qemu_clock_run_timers => timerlist_run_timers => ra_timer_handler => ndp_send_ra ip6_output => if_output => if_start => if_encap => slirp_output => qemu_send_packet => qemu_sendv_packet_async => qemu_net_queue_send_iov => qemu_net_queue_deliver_iov => qemu_deliver_packet_iov => e1000_receive_iov => set_ics => set_interrupt_cause => pci_set_irq

* Mitigation timer超时 (主线程触发)

    main_loop => main_loop_wait => qemu_clock_run_all_timers => qemu_clock_run_timers => timerlist_run_timers => e1000_mit_timer => set_interrupt_cause => pci_set_irq

最终都调用到 pci_set_irq 来设置中断。

```
pci_set_irq => pci_intx   获取 PCI 配置空间的 PCI_INTERRUPT_PIN
            => pci_irq_handler => pci_set_irq_state          设置设备的 irq_state
                               => pci_update_irq_status      为配置空间的 PCI_STATUS 加上 PCI_STATUS_INTERRUPT bit
                               => pci_irq_disabled           如果禁止中断，则直接返回
                               => pci_change_irq_level       否则发射中断
```

##### pci_change_irq_level

```c
static void pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
{
    PCIBus *bus;
    for (;;) {
        bus = pci_dev->bus;
        irq_num = bus->map_irq(pci_dev, irq_num);
        if (bus->set_irq)
            break;
        pci_dev = bus->parent_dev;
    }
    bus->irq_count[irq_num] += change;
    bus->set_irq(bus->irq_opaque, irq_num, bus->irq_count[irq_num] != 0);
}
```

获取 PCI 设备所在的 bus ，调用 bus->map_irq 找到设备对应的 pirq(Programmable Interrupt Router) 号，对于 e1000，其 bus 为 pcie.0 ，map_irq 为 ich9_lpc_map_irq ：

```c
int ich9_lpc_map_irq(PCIDevice *pci_dev, int intx)
{
    BusState *bus = qdev_get_parent_bus(&pci_dev->qdev);
    PCIBus *pci_bus = PCI_BUS(bus);
    PCIDevice *lpc_pdev =
            pci_bus->devices[PCI_DEVFN(ICH9_LPC_DEV, ICH9_LPC_FUNC)];
    ICH9LPCState *lpc = ICH9_LPC_DEVICE(lpc_pdev);

    return lpc->irr[PCI_SLOT(pci_dev->devfn)][intx];
}
```

它首先通过 qdev_get_parent_bus 拿到设备所属的bus对象(pcie.0)，然后从bus上连接的设备数组中找到 ICH9 LPC PCI to ISA bridge ，找到e1000在其irr中对应的 pirq 号，为6。

如果当前一级bus定义了 set_irq 函数，则中断for循环，调用之发送中断，否则设置为bus的父设备，进入下一轮寻找。也就是从发送中断的设备开始，逐级向上查找，直到找到能处理该中断的bus为止。

在这里 set_irq 为 ich9_lpc_set_irq 。于是把中断计数数组中当前中断对应的数值加上change，表示有多少个该类型的中断等待处理。随后调用 ich9_lpc_set_irq 。

```c
void ich9_lpc_set_irq(void *opaque, int pirq, int level)
{
    ICH9LPCState *lpc = opaque;
    int pic_irq, pic_dis;

    assert(0 <= pirq);
    assert(pirq < ICH9_LPC_NB_PIRQS);

    ich9_lpc_update_apic(lpc, ich9_pirq_to_gsi(pirq));
    ich9_lpc_pic_irq(lpc, pirq, &pic_irq, &pic_dis);
    ich9_lpc_update_pic(lpc, pic_irq);
}
```

利用 ich9_pirq_to_gsi 将 pirq 转换成GSI编号，其实就是 pirq + 16 ，e1000为22。然后调用 ich9_lpc_update_apic ，如果中断计数数组中当前中断对应的数值不为0，则level为1。

于是根据GSI编号，从 ICH9LPCState.gsi 中取出对应的 qemu_irq ，调用 qemu_set_irq 将其值设置为level。

#### 考虑e1000、IOAPIC、LAPIC都由QEMU进行模拟的情况(off)

gsi qemu_irq 的 handler 为 gsi_handler ，于是：

qemu_set_irq => irq->handler (gsi_handler) => qemu_set_irq => ioapic_set_irq 设置 IOAPICCommonState 的 irr 。

但这时可能 Remote IRR bit 为 1，因此在设置irr后不会调用 ioapic_service 。

直到某个时刻 LAPIC 处理完毕后发送 EOI 让 IOAPIC 的 Remote IRR bit 变为0，才会在之后的 ioapic_set_irq 中调用 ioapic_service 。

由于此时irr可能积累了多个中断，因此 ioapic_service 会遍历 IOAPIC 上的所有pin，如果 irr 在对应的bit为1，通过 stl_le_phys 修改中断在 IOAPIC AddressSpace 的对应位置。

当对该 AddressSpace 进行写入时，类似MMIO一样最终调用到 MemoryRegion 绑定的 apic_io_ops 。于是调用到 apic_mem_writel ，构造MSI消息后通过 apic_send_msi 发送。

```
apic_deliver_irq => apic_bus_deliver => apic_set_irq => apic_set_bit => apic_set_bit(s->irr, vector_num)   根据中断向量号设置Interrupt Request Register
                                                                     => apic_set_bit(s->tmr, vector_num)   如果是水平触发，设置Trigger Mode Register
                                                     => apic_update_irq         通知CPU
```

##### apic_update_irq

```c
/* signal the CPU if an irq is pending */
static void apic_update_irq(APICCommonState *s)
{
    CPUState *cpu;
    DeviceState *dev = (DeviceState *)s;

    cpu = CPU(s->cpu);
    if (!qemu_cpu_is_self(cpu)) {
        cpu_interrupt(cpu, CPU_INTERRUPT_POLL);
    } else if (apic_irq_pending(s) > 0) {
        cpu_interrupt(cpu, CPU_INTERRUPT_HARD);
    } else if (!apic_accept_pic_intr(dev) || !pic_get_output(isa_pic)) {
        cpu_reset_interrupt(cpu, CPU_INTERRUPT_HARD);
    }
}
```

于是：

```
cpu_interrupt(cpu, CPU_INTERRUPT_POLL) => cpu_interrupt_handler (kvm_handle_interrupt) => cpu->interrupt_request |= mask
                                                                                       => qemu_cpu_kick
```

因此会设置目标cpu的 interrupt_request ，然后 kick 之让其退出到QEMU，回到 kvm_cpu_exec ，由于退出原因是 KVM_EXIT_INTR ，即使进入到 kvm_arch_handle_exit 也无法处理，于是 ret = -1 ，循环中断，退出到上级调用 qemu_kvm_cpu_thread_fn 中，于是在下一次循环中执行 kvm_cpu_exec => kvm_arch_process_async_events ，发现 interrupt_request 的 CPU_INTERRUPT_POLL 为1，调用 apic_poll_irq => apic_update_irq => cpu_interrupt(cpu, CPU_INTERRUPT_HARD) 。如果LAPIC有未处理的中断(apic_irq_pending)，则为 interrupt_request 加上 CPU_INTERRUPT_HARD

于是在接下来的 kvm_arch_pre_run 中，如果中断可以注入，则通过 cpu_get_pic_interrupt => apic_get_interrupt 从 LAPIC 中取出中断号：

```
=> apic_irq_pending(s)              从irr中取出优先级级最高的中断号
=> apic_reset_bit(s->irr, intno)    设置中断号在irr对应的bit为0
=> apic_set_bit(s->isr, intno)      设置中断号在isr对应的bit为1
=> apic_update_irq(s)               如果还有其它中断未处理，再次设置 cpu->interrupt_request 为 CPU_INTERRUPT_HARD
```

在获得中断号后，通过 kvm_vcpu_ioctl(cpu, KVM_INTERRUPT, &intr) 注入中断到KVM。

如果前面还有中断没处理，则此时 cpu->interrupt_request 依然为 CPU_INTERRUPT_HARD ，但我们一次只能注入一个中断，因此设置 request_interrupt_window 为 1，从而保证党guest能够处理下一个中断时立刻退回到QEMU。

这里注入到KVM中的 irq 是 **中断向量号(interrupt vector)** 。

##### KVM

kvm_arch_vcpu_ioctl => kvm_vcpu_ioctl_interrupt => kvm_queue_interrupt(vcpu, irq->irq, false)   将中断设置到 vcpu->arch.interrupt
                                                => kvm_make_request(KVM_REQ_EVENT, vcpu)        产生请求

这样接下来当QEMU通过 `kvm_vcpu_ioctl(cpu, KVM_RUN, 0)` 进入到KVM时，在


kvm_arch_vcpu_ioctl_run => vcpu_run => vcpu_enter_guest 中，检测到有 KVM_REQ_EVENT 请求，调用 inject_pending_event 进行中断注入：

```c
static int inject_pending_event(struct kvm_vcpu *vcpu, bool req_int_win)
{
  ...
  if (vcpu->arch.interrupt.pending) {
      kvm_x86_ops->set_irq(vcpu);
      return 0;
  }
  ...
}
```

最后由 vmx_inject_irq 将中断写入到VMCS中。




#### 考虑e1000、IOAPIC由QEMU进行模拟，LAPIC由KVM进行模拟的情况(split)

中断从e1000发送到 IOAPIC 的流程和上文一致，直到 ioapic_service 。它会用 kvm_irqchip_is_split 判断是否为split模式，如果是，则 LAPIC 由KVM负责模拟，于是通过 kvm_set_irq 设置中断(注意对于on模式，IOAPIC也由KVM模拟，根本不会走到这里，因此这里只判断是否是split)。

于是 kvm_set_irq => kvm_vm_ioctl(s, s->irq_set_ioctl, &event) 向KVM注入中断。 s->irq_set_ioctl 根据 KVM能力可能为 KVM_IRQ_LINE 或 KVM_IRQ_LINE_STATUS ，区别在于后者会返回状态。

这里注入到KVM中的 irq 为中断设备对应的 **GSI**。e1000 的 gsi 是 22 。




##### KVM

kvm_vm_ioctl => kvm_vm_ioctl_irq_line => kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irq_event->irq, irq_event->level, line_status)

从 table 中找到对应的 entry ，调用 kvm_set_ioapic_irq => kvm_ioapic_set_irq => ioapic_set_irq => ioapic_service => kvm_irq_delivery_to_apic => kvm_apic_set_irq => __apic_accept_irq 对目标LAPIC设置中断：

```
=> 根据 delivery_mode 进行对应设置，如 APIC_DM_FIXED 为 kvm_lapic_set_vector + kvm_lapic_set_irr
=> kvm_make_request(KVM_REQ_EVENT, vcpu)
=> kvm_vcpu_kick(vcpu)              让目标vCPU退出来处理请求
```

接下来在 vcpu_run => vcpu_enter_guest 中，由于 LAPIC 在KVM中，先通过 kvm_x86_ops->hwapic_irr_update (vmx_hwapic_irr_update) 更新 irr 中优先级最高的中断？

后检测到有 KVM_REQ_EVENT 请求，调用 inject_pending_event 进行中断注入：

```c
static int inject_pending_event(struct kvm_vcpu *vcpu, bool req_int_win)
{
  ...
  if (vcpu->arch.interrupt.pending) {
      kvm_x86_ops->set_irq(vcpu);
      return 0;
  }
  ...
}
```

最后由 vmx_inject_irq 将中断写入到VMCS中。



#### 考虑e1000由QEMU进行模拟，IOAPIC、LAPIC由KVM进行模拟的情况(on)

此时gsi qemu_irq 的 handler 为 kvm_pc_gsi_handler ，于是：

qemu_set_irq => irq->handler (kvm_pc_gsi_handler) => qemu_set_irq(s->ioapic_irq[n], level) => irq->handler (kvm_ioapic_set_irq) => kvm_set_irq(kvm_state, s->kvm_gsi_base + irq, level) => kvm_vm_ioctl(s, s->irq_set_ioctl, &event) 通过ioctl向KVM注入中断。

这里注入到KVM中的 irq 为中断设备对应的 **GSI**。由于s->kvm_gsi_base 为 0， 因此e1000算出来的gsi s->kvm_gsi_base + irq 依然为 22 。

因此可以发现在split和on情况下，不管IOAPIC在哪模拟，最终都是通过KVM的 KVM_IRQ_LINE / KVM_IRQ_LINE_STATUS 接口注入中断。并且中断的gsi都是22。



##### KVM

KVM中的流程和split中的流程一样。因此和split的区别在于on需要通过接口去查询KVM模式的IOAPIC信息，而split由于QEMU负责模拟了，所以不用查询自己知道。

比如on在 hmp 查询IOAPIC时，需要通过 kvm_ioapic_dump_state => kvm_ioapic_get => kvm_vm_ioctl(kvm_state, KVM_GET_IRQCHIP, &chip) 去查。


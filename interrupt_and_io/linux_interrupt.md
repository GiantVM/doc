# Linux Interrupt
## Facts of Hardware
在x86 CPU上，中断需通过IDT中的表项分发，若通过Interrupt Gate进入中断处理例程，则EFLAGS的IF（Interrupt Flag）会被清零（相当于执行`CLI`指令），从而禁用中断，直到执行`IRET`指令从中断处理例程中返回时，才会重置IF。若通过的是Trap Gate，则不会对IF作任何操作。

## Linux
Linux中，在进入一个中断处理例程后，会立即重新开启中断，但该中断自己会在所有CPU上都被屏蔽，从而防止其自身重入。因此，在Linux中，中断处理例程不必是可重入的。

### SMP Affinity
`/proc/interrupts`显示了中断被各CPU处理的次数。`/proc/irq/IRQ#/smp_affinity`是一个bitmap，表示第IRQ#个irq应被发送到哪些CPU上，每个bit代表一个CPU，如果有多于一个CPU被分配到某个irq，则会使用lowest priority mode，由硬件选取这一组CPU中优先级最低的CPU作为中断的目的地。优先级通常可以通过设置LAPIC的TPR寄存器来变更。

SMP affinity也可以通过`/proc/irq/IRQ#/smp_affinity_list`查询和指定，它是一个CPU list，其形式形如`2`、`1-7`等。SMP affinity为32位、64位或128位宽，默认值通常是全1（即所有CPU都能收到），默认值可以通过`/proc/irq/default_smp_affinity`查询

实践中，通常会将每个IRQ绑定到一个单独的核，而不是一组核，因为同一个中断在同一个核上处理locality更好。并且lowest priority mode也不是完美的，它只保证总能仲裁选出一组CPU中的某一个，不保证能在这一组CPU之间load balance，例如一组CPU的优先级都相同时，可能每次选出的CPU都是同一个。

不幸的是，Linux并不会设置CPU的TPR：
```c
// arch/x86/kernel/apic/apic.c
// 下面这个片段证明Linux只会将TPR设为0，并不再变更

/*
 * Set Task Priority to 'accept all'. We never change this
 * later on.
 */
value = apic_read(APIC_TASKPRI);
value &= ~APIC_TPRI_MASK;
apic_write(APIC_TASKPRI, value);
```
因此当smp affinity为一个IRQ设置了超过一个CPU时，往往只有一个CPU在处理中断。

---

通常采用红帽开发的`irqbalance`工具进行smp affinity的管理，该工具能分析系统的负载，每隔一段时间（默认是10秒）修改一次smp affinity。它会将每个IRQ绑定到一个CPU，并根据负载动态变更这个绑定，使这个IRQ在CPU间的分布均匀。它还有一个Power save模式，当发现系统中断负载较低时，会把所有中断都绑定到同一个CPU上，减少其他CPU的工作，让其他CPU可以进入休眠模式。

---

Linux内核中，设置smp affinity的工作在`kernel/irq/manage.c`中实现，关键函数为`setup_affinity(irq_desc, cpumask)`。该函数调用`irq_do_set_affinity`函数，后者再进一步调用`struct irq_chip`中的`irq_set_affinity`方法，调用到具体硬件的affinity设置代码。

最终，`smp_affinity`这个bitmap被作为`cpumask`参数传入`struct apic`的`cpu_mask_to_apicid_and`方法，得到APIC Destination field用于填入硬件寄存器。APIC的每种配置方法都对应于一个`struct apic`对象，从而就有一种`cpu_mask_to_apicid_and`实现。

### APIC initialization

Linux启动时会选择一个APIC驱动（`struct apic`对象），32位下通过`arch/x86/kernel/apic/probe_32.c`的`generic_apic_probe`函数实现，该函数会调用每个APIC驱动对象的`probe`方法查询是否可用，64位下通过`arch/x86/kernel/apic/probe_64.c`的`default_acpi_madt_oem_check`函数实现，该函数会调用每个APIC驱动对象的`acpi_madt_oem_check`方法，根据ACPI表选择驱动。APIC驱动被检查的顺序就是Makefile中指定的编译顺序。

从编译顺序可知，系统会优先选择x2APIC，而在x2APIC和xAPIC模式内部，默认模式都是Logical + Lowest Priority模式，除非系统不支持从而只能选择Physical + Fixed模式。

> **推论：** 对于Physical + Fixed模式，其`cpu_mask_to_apicid_and`方法会选取`cpumask`中第一个找到的在线CPU，作为APIC Destination。因此假若APIC被配置为该模式，`smp_affinity`会退化到在一组中固定选择一个CPU。

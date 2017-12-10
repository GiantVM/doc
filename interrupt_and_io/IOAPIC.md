# IOAPIC
IOAPIC全称为82093AA I/O Advanced Programmable Interrupt Controller，据此型号可找到其[手册](https://pdos.csail.mit.edu/6.828/2016/readings/ia32/ioapic.pdf)。

IOAPIC通过MMIO访问，实际只有两个32位的寄存器可供访问，分别是I/O Register Select（IOREGSEL，只有低8位有效）和I/O Window（IOWIN），其物理地址分别为0xFEC0xy00和0xFEC0xy10（其中x、y可通过APIC Base Address Relocation Register配置）。前者提供Index，其第0-7位表示要访问的寄存器编号，后者提供Data，用于读写要访问的IOAPIC寄存器。

> **Info:** APIC Base Address Relocation Register是PIIX3芯片上的寄存器，PIIX3是90年代末Intel的南桥，后来南桥进入ICH系列后取消了该寄存器，但后来又增加了新的配置寄存器，将两个寄存器的地址变为0xFEC0x000和0xFEC0x010（其中x可配置）
## Registers
- IOAPIC ID (IOAPICID)，位于Index 0x0，第24-27位表示IOAPIC ID，用于标识IOAPIC
- IOAPIC Version (IOAPICVER)，位于Index 0x1，只读
    - 第0-7位表示APIC Version，取值应为0x11
    - 第16-23位表示Maximum Redirection Entry，即Redirection Table的项数-1，取值应为0x17（即23）
- IOAPIC Arbitration ID (IOAPICARB)，位于Index 0x2，只读，第24-27位表示Arb ID，用于APIC Bus的仲裁
    - LAPIC之间以及LAPIC和IOAPIC之间的通信都是走APIC Bus，每个LAPIC和IOAPIC都有一个Arb ID用于仲裁，Arb ID最高者胜利，并将自己的ID置为0，其余各APIC的Arb ID则加一（除了原本Arb ID为15者，要将Arb ID置位胜利者原本的Arb ID值）。
    > **Note:** APIC Bus是奔腾和P6 family使用的技术，从奔腾4开始LAPIC之间以及LAPIC和IOAPIC之间的通信都是走系统总线，不使用Arb ID进行仲裁，因此现在讨论Arb ID已没有意义。
- Redirection Table (IOREDTBL[0:23])，位于Index 0x10-0x3F（每项64位），负责配置中断转发功能，其中每一项简称RTE（Redirection Table Entry）

## Interrupt Redirection
当一个中断来到IOAPIC时，就要根据Redirection Table发送给CPU(s)，这张转发表中的表项RTE的内容如下：
- 第56-63位为Destination，代表目的CPU(s)，Physical模式则为APIC ID，Logical模式则为MDA
    - Physical模式下，仅第56-59位有效，第60-63位必须取0
- 第16位为Mask，取0表示允许接受中断，取1表示禁止，reset后初始值为1
- 第15位为Trigger Mode，取0表示edge triggered，取1表示level triggered
- 第14位为Remote IRR，只读且只对level triggered中断有意义，取1代表目的CPU已经接受中断，当收到CPU发来的EOI后，变回0表示中断已经完成
    > **Note:** Remote IRR取1时的作用，实际上是阻止Level Triggered的IRQ line上的Active信号再次触发一个中断。设想若Active信号会产生中断，则只要信号保持Active（e.g. 高电平），就会不断触发中断，这显然是不正确的，故需要由Remote IRR位将中断阻塞。由此可见，CPU应该先设法让IRQ line回到Inactive状态，然后再进行EOI，否则该中断将再次产生。
- 第13位为Interrupt Input Pin Polarity，取0表示active high，取1表示active low
- 第12位为Delivery Status（只读），取0表示空闲，取1表示CPU尚未接受中断（尚未将中断存入IRR）
    - 若目的CPU对某Vector已经有两个中断在Pending，IOAPIC等于可以为该Vector提供第三个Pending的中断
- 第11位为Destination Mode，取0表示Physical，取1表示Logical
- 第8-10位为Delivery Mode，有以下几种取值：
    - 000 (Fixed)：按Vector的值向目标CPU(s)发送相应的中断向量号
    - 001 (Lowest Priority)：按Vector的值向Destination决定的所有目标CPU(s)中Priority最低的CPU发送相应的中断向量号
        - 关于该模式，详见Intel IA32手册第三册第十章
    - 010 (SMI)：向目标CPU(s)发送一个SMI，此模式下Vector必须为0，SMI必须是edge triggered的
    - 100 (NMI)：向目标CPU(s)发送一个NMI（走#NMI引脚），此时Vector会被忽略，NMI必须是edge triggered的
    - 101 (INIT)：向目标CPU(s)发送一个INIT IPI，导致该CPU发生一次INIT（INIT后的CPU状态参考Intel IA32手册第三册表9-1），此模式下Vector必须为0，且必须是edge triggered
        > **Info:** CPU在INIT后其APIC ID和Arb ID（只在奔腾和P6上存在）不变
    - 111（ExtINT）：向目标CPU(s)发送一个与8259A兼容的中断信号，将会引起一个INTA周期，CPU(s)在该周期向外部控制器索取Vector，ExtINT必须是edge triggered的
- 第0-7位为Vector，即目标CPU收到的中断向量号，有效范围为16-254（0-15保留，255为全局广播）

### Destination Mode
Physical Mode表示Destination的取值为目的LAPIC的APIC ID。Logical Mode表示Destination的取值为Message Destination Address（MDA），可以用于引用一组LAPIC（即可用于Multicast）。关于MDA，详见Intel IA32手册第三册第十章。

如果使用Logical Mode寻址模式引用了一组CPU，同时选择了Lowest Priority发送模式，则中断最终会发给这组CPU中优先级最低的CPU。（优先级的确定可参考Intel IA32手册第三册第十章，其中一种方法是依据LAPIC中的TPR寄存器）

### Pin
根据手册，IOAPIC的24个中断输入引脚通常如下连接：

- Pin #1连接到键盘中断（IRQ1）
- Pin #2连接到IRQ0
- Pin #3-#11,#14,#15，分别连接到ISA IRQ[3:7,8#,9:11,14:15]
- Pin #12连接到鼠标中断（IRQ12/M）
- Pin #16-#19代表PCI IRQ[0:3]
- Pin #20-#21代表Motherboard IRQ[0:1]
- Pin #23代表SMI中断，若Mask掉，则SMI中断会从IOAPIC的#SMIOUT引脚引出，否则会由IOAPIC根据RTE #23转发

上述描述代表了PIIX3芯片组时期的典型接法，若要了解现在的芯片组是如何连接这些引脚的，还应查询最新芯片组的datasheet。

值得注意的是，若某个设备（如键盘控制器）的中断信号通过IRQ line连接到PIC，则它也会连接到IOAPIC的中断输入引脚。例如键盘控制器通过IRQ1连接到PIC，同时也通过Pin #1连接到IOAPIC。因此，若PIC和IOAPIC同时启用，可能会造成设备产生一个中断，CPU收到**两次**中断，故必须屏蔽其中一个而只用另一个，通常我们会屏蔽PIC（这可以通过写入0xFF到PIC的OCW1实现）。

事实上，约定俗成地，人们通常将系统中的第一个IOAPIC（如果有多个的话）的前16个Pin和PIC的16个IRQ相对应。也就是说Pin #x和IRQ x中的中断信号相同（0 <= x < 16）。其中有两个例外，一是Master PIC的INTR输出引脚要连接到IOAPIC的Pin #0（这也是约定俗成的要求，MP Spec并未规定PIC的INTR输出一定要连接到IOAPIC），故Pin #0不对应与IRQ0，二是Pin #2对应于IRQ0，这是由于IRQ2是Slave PIC，无需对应于IOAPIC Pin，而Pin #0又没有连到IRQ0，正好将Pin #2和IRQ0连接起来。

## IOAPIC till ICH9
随着芯片集成度的提升，IOAPIC芯片已被集成到了南桥内，我们可以从历代南桥手册（Datasheet）中查询到其pin接到的是什么设备，增加了什么功能。此处介绍QEMU模拟的Q35芯片组中的ICH9南桥中的IOAPIC。

### IOxAPIC
从ICH1开始，IOAPIC就被集成到了南桥里，并且ICH中集成的不再是原版的82093AA IOAPIC，而是经过修改的版本，可以称之为IOxAPIC。下面介绍历代IOAPIC经过的改动：

#### ICH1
ICH1虽然仍保持IOAPIC Version为0x11，但实际上进行了一些改动：
- 取消了APIC Base Address Relocation Register，使得Index Register（IOREGSEL）和Data Register（IOWIN）固定位于0xFEC00000和0xFEC00010
- 增加了两个32位的Write Only的寄存器，分别是IRQ Pin Assertion Register和EOI Register，分别位于0xFEC00020和0xFEC00040
- IOAPIC Version Register的第15位表示是否支持IRQ Assertion Register（此前是保留位，默认取0）

IRQ Pin Assertion Register：第0-4位表示IRQ Number，其余位保留。每当对该寄存器写入val时，就会使对应的IRQ发生中断。

> **Info:** IRQ Assertion Register是MSI的早期实现使用的机制。当时MSI Address Register填写0xFEC00020，MSI Data Register填写IRQ Number，即可进行一次MSI。后来MSI改为使用Upstream Memory Write的方式，直接向CPU进行写入，由CPU直接处理MSI，IOAPIC中便取消了该功能。

EOI Register：第0-7位表示interrupt vector，其余位保留。每当对该寄存器写入val时，就会清除interrupt vector为val的RTE表项的Remote IRR位。

#### PCI Age (ICH2 - ICH5)
从ICH2开始IOAPIC Version改为0x20（文档写作0x02，实际上是0x20，Version为0x2X表示支持PCI 2.2），有如下改动：
- 增加了一个间接引用的寄存器，Boot Configuration Register，位于index 0x3，可读写，仅最低位有效，取0表示APIC总线发送中断消息（默认取0），取1表示通过系统总线（FSB）发送中断消息

ICH4未改动IOAPIC Version，但又扩展了RTE表项，从ICH4开始RTE的第48-55位表示EDID（Extended Destination ID），当通过系统总线发送中断消息时，EDID是address的第4-11位。
> **PS:** 文档原文为"They become bits 11:4 of the address"，但LAPIC能接受的Destination只有8位或32位（x2APIC），且IOAPIC的Destination只有在Physical Mode时才只有4位，能与EDID拼接。EDID的具体作用和在OS中的使用尚待考察。

ICH5未改动IOAPIC Version，但删除了Arbitration ID Register和Boot Configuration Register，标志着对APIC Bus的兼容彻底移除（此时已经是2003年，离最后一款P6 family的CPU已经过去了好几年）。

#### PCIe Age (ICH6 - ICH9)
ICH6首次支持PCIe，有如下改动：
- 删除了IRQ Pin Assertion Register

从ICH8开始，IOAPIC寄存器的地址又变为可变，通过Chipset Configuration Register中的OIC（Other Interrupt Control）寄存器第4-7位（APIC Range Select，ASEL）控制IOAPIC地址的第12-15位。此时IOAPIC的地址范围为0xFEC0x000-0xFEC0x040（其中x可配置）

### IOxAPIC Interrupt Delivery
最初，IOAPIC是通过向APIC Bus发送Message实现中断的发送的，到ICH1还是如此，但从ICH2开始就支持了通过系统总线发送中断。这里所谓的通过系统总线发送中断，其实和MSI的方式是相同的，都是向特定地址写入特定数据，CPU会监听到这个写入，于是把它解释成一个中断请求，并让LAPIC处理该请求。事实上，其地址和数据的格式同MSI也是相同的。

这种技术在ICH2中称为Front-Side Interrupt Delivery，在ICH3、ICH4中称为System Bus Interrupt Delivery，在ICH5-ICH9中称为FSB Interrupt Delivery，本质上指的都是同一件事。

由此可见，ICH6中引入的现代意义上的MSI，实际上就是从ICH2开始的FSB Interrupt Delivery，只不过前者直接从PCI Device发到CPU，后者经过IOAPIC转发到CPU，但在CPU看来都是相同的。

需要注意的是，FSB Interrupt Delivery只支持普通中断，不支持SMI、NMI、INIT中断，不能在Delivery Mode field中填写SMI、NMI或INIT（ICH2未提到该规定，ICH3-ICH9都有此规定）。OS可以通过ACPI表（由BIOS构造）查询芯片组是否支持FSB Interrupt Delivery特性。

### 配置

ICH9中IRQ pin的配置如下：

![ich9_ioapic_1](/assets/ich9_ioapic_1.png)
![ich9_ioapic_2](/assets/ich9_ioapic_2.png)

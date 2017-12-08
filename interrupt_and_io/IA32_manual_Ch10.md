# Intel IA32 Manual Chapter 10 APIC
## Overview
Local APIC（以下简称APIC）可以从以下来源接收中断：
1. 通过LINT0和LINT1这两个引脚接收的本地中断
2. 通过IOAPIC接收的外部中断
3. 其他CPU（甚至自己）发来的IPI
4. APIC Timer产生的中断
5. APIC内部错误引发的中断
6. Performance Monitoring Counter产生的中断（c.f. 18.6.3.5.8）
7. 温度传感器产生的中断（c.f. 14.7.2）

其中1、4、5、6、7统称为本地中断源，通过LVT（Local Vector Table）控制其接收。通过写入Interrupt Control Register（ICR），则可以发送IPI。IOAPIC通过interrupt message给APIC发送中断，在message中已经指定了Interrupt Vector，因此不需要APIC通过LVT表来进行配置。

在P6 family（即从奔腾Pro到奔腾3）的时代，各CPU的APIC间通过一条APIC总线通信，IPI message和interrupt message都在该总线上传输。从奔腾4开始，APIC间的通信改为走system bus，IOAPIC发送的interrupt message也通过system bus发到APIC。
> **Note:** APIC中有一些奔腾和P6 family独有的设置项，以下均用:warning:标出。其中部分涉及其特有的APIC Bus，若有兴趣，请查阅手册的10.7节、10.10节、10.13节，对照理解。

APIC最初源自外置的Intel 82489DX芯片，到奔腾和P6时代演化成APIC，自奔腾4开始进一步演变为xAPIC，近年来又扩展为x2APIC，提供更多功能。

## Basic functions
### Overview
APIC的寄存器是通过MMIO访问的，对应的物理地址为从0xFEE00000起的4KB页，寄存器的具体映射关系见手册的表10-1。判断CPU是否支持APIC以及是否支持x2APIC均可通过查询CPUID实现。

> **Info:** 寄存器所在页的物理地址可以通过设置改变，这是P6时代遗留下来的上古特性，当时是为了防止与已经使用了这块地址的程序发生冲突，现在已无人使用。此外还需注意，该页在页表中必须设置为strong uncacheable (UC)。

APIC可以硬件禁用/启用，禁用再启用后相当于断电重启。此外还可以[软件禁用/启用](#spurious-interrupt)，这只是暂时禁用，重新启用后其状态会得到保留。需要注意的是，APIC在通电后默认是处于软件禁用的状态的。

### APIC MSR
与APIC相关的唯一一个MSR为`IA32_APIC_BASE`：
- 第8位代表该CPU是否为BSP（1代表BSP）
- 第11位可以用于硬件启用/禁用APIC（0代表禁用，自奔腾4起禁用后仍可启用，P6则不行），禁用后相当于APIC不存在，CPUID也会显示不支持APIC，重新启用后APIC回到断电重启时的状态
- 第12-35位表示APIC Base，即APIC的寄存器所在物理地址（实际物理地址为APIC Base左移12位，即正好4K对齐）**【注意：该特性只为兼容存在】**

### Local APIC ID Register
位于Base + 0x20，作为CPU的ID使用，其初始值为开机时由硬件自动决定并赋予，会被BIOS和/或OS使用，因此不宜再行修改（且仅在部分型号CPU上支持修改）。若已改动其值，则仍可通过`CPUID.01H.EBX[31:24]`查询到其初始值。

奔腾及P6 family仅使用第24-27位（共4位），故最多支持15个CPU，奔腾四开始的xAPIC模式使用第24-31位（共8位），故最多支持255个CPU。x2APIC使用32位的x2APIC ID，存在MSR中，可以认为不存在最大CPU数量限制。

### Local APIC Version Register
位于Base + 0x30，只读，功能如下：
- 第0-7位为Version Number
- 第16-23位为Max LVT Entry，其值为最大LVT数-1
- 第24位表示是否支持[Suppress EOI-broadcast](#spurious-interrupt-vector-register)

## Handling Local Interrupts
### Local Vector Table
LVT表中共有（最多）七项，每一项负责配置一种来源的中断的分发，分别是：
- LVT CMCI Register，Base + 0x2F0
    - 负责发送Corrected Machine Check Error Interrupt，即被纠正的Machine Check Error累积至超过一个阈值后，便会引起一个CMCI中断（从至强5500起才有此项，该功能默认禁止）
- LVT Timer Register，Base + 0x320
    - 负责发送由APIC Timer产生的中断
- LVT Thermal Monitor Register，Base + 0x330
    - 负责发送由温度传感器产生的中断（从奔腾4起才有此项）
- LVT Performance Counter Register，Base + 0x340（此地址仅供参考，该寄存器具体在何处是implementation specific的）
    - 负责发送由性能计数器Overflow产生的中断（从P6 family起才有此项）
- LVT LINT0 Register，Base + 0x350
    - 负责转发来自LINT0引脚的中断
- LVT LINT1 Register，Base + 0x360
    - 负责转发来自LINT1引脚的中断
- LVT Error Register，Base + 0x370
    - 负责发送APIC内部错误时产生的中断

这些寄存器的功能如下：
- 第0-7位为Vector，即CPU收到的中断向量号，其中0-15号被视为非法，会产生一个Illegal Vector错误（即ESR的bit 6，[详下](#error-handling)）
- 第8-10位为Delivery Mode，有以下几种取值：
    - 000 (Fixed)：按Vector的值向CPU发送相应的中断向量号
    - 010 (SMI)：向CPU发送一个SMI，此模式下Vector必须为0
    - 100 (NMI)：向CPU发送一个NMI，此时Vector会被忽略
    - 111 (ExtINT)：令CPU按照响应外部8259A的方式响应中断，这将会引起一个INTA周期，CPU在该周期向外部控制器索取Vector。
        - APIC只支持一个ExtINT中断源，整个系统中应当只有一个CPU的其中一个LVT表项配置为ExtINT模式
- 第12位为Delivery Status（只读），取0表示空闲，取1表示CPU尚未接受该中断（尚未EOI）
- 第13位为Interrupt Input Pin Polarity，取0表示active high，取1表示active low
- 第14位为Remote IRR Flag（只读），若当前接受的中断为fixed mode且是level triggered的，则该位为1表示CPU已经接受中断（已将中断加入IRR），但尚未进行EOI。CPU执行EOI后，该位就恢复到0。
- 第15位为Trigger Mode，取0表示edge triggered，取1表示level triggered（具体使用时尚有许多注意点，详见手册10.5.1节）
- 第16位为Mask，取0表示允许接受中断，取1表示禁止，reset后初始值为1
- 第17/17-18位为Timer Mode，只有LVT Timer Register有，用于切换APIC Timer的三种模式，详下

注意并不是LVT中每个寄存器都拥有所有的field，具体情况如下所示：

![local_vector_table](/assets/local_vector_table.png)

### Error Handling
APIC发生内部错误时，错误原因会被记录到Error Status Register（ESR），位于Base + 0x280，内容如下：
- :warning: Bit 0：Send Checksum Error，只在奔腾和P6 family上有效，**已过时无需考虑**
- :warning: Bit 1：Receive Checksum Error，只在奔腾和P6 family上有效，**已过时无需考虑**
- :warning: Bit 2：Send Accept Error，只在奔腾和P6 family上有效，**已过时无需考虑**
- :warning: Bit 3：Receive Accept Error，只在奔腾和P6 family上有效，**已过时无需考虑**
- Bit 4：Redirectable IPI，试图发送一个Lowest Priority Mode的IPI，但CPU并不支持
    - 只有部分型号会使用该Bit。由于发送Lowest Priority Mode IPI的能力依具体型号而定，不建议BIOS和OS使用该功能。
- Bit 5：Send Illegal Vector，试图（通过IPI）发送一个0-15范围的Interrupt Vector
- Bit 6：Receive Illegal Vector，接收到了一个0-15范围的Interrupt Vector，包括从自己的LVT接收和通过Interrupt Message接收IPI或外部中断
- Bit 7：Illegal Register Address，试图读取一个不存在的Register

在读取ESR的内容前，必须先为其写入0，才能清空此前的flag，并会自动填入新的flag，以及重新激活错误检测机制。

### APIC Timer
APIC Timer是一个32位的Timer，通过两个32位的Counter寄存器实现：
- Initial Count Register，Base + 0x380
- Current Count Register，Base + 0x390（只读）

Counter的频率由APIC Timer的基准频率除以Divide Configuration Register确定的除数获得。Divide Configuration Register，位于Base + 0x3E0，其第0、1、3位决定了除数：

![divider](/assets/divide_configuration_register.png)

> **Info:** APIC Timer可能会随CPU休眠而停止运作，检查`CPUID.06H.EAX.ARAT[bit 2]`（APIC Timer Always Running bit）可知其是否会永远保持运作。APIC Timer的基准频率是总线频率（外频）或Core晶振频率（如果能通过`CPUID.15H.ECX`查到的话）。

APIC Timer有三种操作模式，可以通过LVT Timer Register的第17-18位设置，分别是：
- One shot（00）：写入Initial Count以启动Timer，Current Count会从Initial Count开始不断减小，直到最后降到零触发一个中断，并停止变化
- Periodic（01）：写入Initial Count以重启Timer，Current Count会反复从Initial Count减小到0，并在减小到0时触发中断
- TSC-Deadline Mode（10）
    - `CPUID.01H.ECX.TSC_Deadline[bit 24]`表示是否支持TSC-Deadline模式，若不支持，第18位为reserved
    - 此模式下，对Initial Count的写入会被忽略，Current Count永远为0。此时Timer受MSR`IA32_TSC_DEADLINE_MSR`控制，为其写入一个非零64位值即可激活Timer，使得在TSC达到该值时触发一个中断。该中断只会触发一次，触发后`IA32_TSC_DEADLINE_MSR`就被重置为零。
    > **Note:** 写入LVT Timer Register切换到TSC-Deadline Mode是一个Memory操作，该操作和接下来的`WRMSR`指令间必须添加一个`MFENCE`以保证不会乱序

注意前两种模式下，为Initial Count写入0即可停止Timer运作，在第三种模式下则是为`IA32_TSC_DEADLINE_MSR`写入0，此外修改模式也会停止Timer运行。当然，也可以通过LVT Timer Register中的Mask屏蔽Timer中断实现同样的效果。

## Handling Interrupts
### Interrupt Acceptance for Fixed Interrupts
与接收中断有关的寄存器主要是Interrupt Request Register（IRR）和In-Service Register（ISR），都是256位宽，每一位代表一个Interrupt Vector。IRR位于Base + 0x200到Base + 0x270（共8个32位寄存器），ISR位于Base + 0x100到Base + 0x170（共8个32位寄存器）。

APIC收到的Fixed Interrupts首先进入IRR，写入对应的位，表示已收到但尚未发送给CPU的中断。当CPU准备好接收新的中断时，APIC选出IRR中优先级最高（即Vector最大）的中断，清除其bit，设置ISR中对应的bit，并发送给CPU。

当CPU执行完当前的中断处理例程时，应对EOI Register（位于Base + 0xB0）写入0（写入非零似乎会引起#GP，但手册没写），从而清除ISR中优先级最高（即Vector最大）的中断的Bit。

当CPU正在处理中断时，可以被更高优先级的中断打断。若IRR中收到的新中断的Priority Class高于CPU的当前Priority Class（见下节），则可以直接发送给CPU并写入到ISR中，打断正在运行的中断处理例程（当然CPU必须没有关中断）。

> **Info:** 当收到Priority Class小于等于CPU的当前Priority Class的中断时，如果IRR中还有空位，可以放入IRR，即使ISR中该Vector已经占了一个位置。也就是说，APIC对同一个Vector支持两个Pending的中断，与此相对，PIC对同一个Vector只支持一个Pending的中断（即In-Service的那一个）。

此外还有一个Trigger Mode Register（TMR），也是256位宽，每一位代表一个Interrupt Vector。当某个中断来临时，除了将其加入IRR，还会根据它是否是Level Triggered设置TMR中的对应位，1代表Level Triggered。Level Triggered的中断会在CPU写入EOI Register后通过总线向所有IOAPIC广播EOI Message。
> **Info:** 这个默认的广播行为是可以禁止的，通过设置[Spurious Interrupt Vector Register](#spurious-interrupt-vector-register)的第12位即可禁止向IOAPIC广播。此时，必须由软件手动设置发送中断的那个IOAPIC的EOI Register，来完成EOI的必要步骤。

### Interrupt, Task, and Processor Priority
手册中将Interrupt Vector的高4位称为Interrupt Priority Class，低4位无特殊称谓。Interrupt之间的优先级纯看其数值大小，大者即优先级高。

CPU当前正在处理的中断是否能被新来的中断打断，取决于以下两个寄存器：
- Task Priority Register（TPR），位于Base + 0x80，第4-7位为Task Priority Class，第0-3位为Task Priority Sub-Class。
    > **Info:** 在IA32-e（即x86-64）模式下，有CR8寄存器，其第0-3位同样代表了Task Priority Class。此时，恒有TPR[7:4] = CR8[3:0]，但OS应在TPR和CR8之间选择一种操作方式，而不应两种方式混用。
- Processor Priority Register（PPR），位于Base + 0xA0，第4-7位为Processor Priority Class，第0-3位为Processor Priority Sub-Class。

它们的含义如下：
- Processor Priority Class代表了CPU的当前Priority Class，只有Interrupt Priority Class大于它的新中断才允许被注入到CPU中，并从IRR进入ISR。Processor Priority Sub-Class并无实际作用。
- PPR是由TPR和ISRV决定的一个只读的值，其中ISRV是ISR中最大的Vector（ISR为0则ISRV也为0）。公式如下：
    - `PPR[7:4] = max(TPR[7:4], ISRV[7:4])`
    - PPR[3:0] 无实际作用，故略

也就是说，优先级的机制实际上就是每16个Interrupt Vector分为一组（Class），后16个比前16个优先级高。Task Priority表明的是CPU希望（暂时）屏蔽该Priority Class及以下的中断。此外，CPU当前正处理的中断只能被高于ISR中最高Priority Class的中断打断。

### 总结：奔腾4和至强以来的处理流程
1. 确认自己是否是Interrupt Message的目标，如果是则继续
2. 如果收到的中断是NMI、SMI、INIT、ExtINT或SIPI，则直接交给CPU处理
3. 否则设置IRR中的适当的位
4. 对于在IRR中pending的中断，APIC根据它们的优先级，以及当前CPU的优先级（存在PRR中）依次分发，一次只给CPU一个中断
5. 中断处理例程执行完毕时，应写入EOI Register，使得APIC从ISR队列中删除中断对应的项，结束该中断的处理（NMI、SMI、INIT、ExtINT及SIPI不需要写入EOI）

## Spurious Interrupt Vector Register
Spurious Interrupt产生的原因如下：当CPU要接受一个ExtINT中断时，第一步获取中断向量号需要经过两个周期，第一个周期INTR引脚收到信号，第二个周期（INTA周期）从外部控制器获取中断向量号。而通常的中断都是在INTR引脚收到信号的周期内，就取得中断向量号。由于这个非原子性，若CPU正好在INTA周期通过LVT表项屏蔽了该ExtINT中断，则APIC会转而发送一个Spurious Interrupt。

Spurious-Interrupt Vector Register（SVR），位于Base + 0xF0，内容如下：
- 第0-7位为Spurious Vector，即APIC产生Spurious Vector时应该发送的Interrupt Vector
    - 对于奔腾和P6 family，第0-3位被hardwire到1，因此Spurious Vector低4位最好设置为全1
- 第8位控制APIC软件启用/禁用，1为启用，0为禁用
    - APIC断电重启后，默认处于软件禁用状态
    - 在软件禁用状态下，IRR和ISR的状态仍会保留
    - 在软件禁用状态下，仍能响应NMI、SMI、INIT、SIPI中断，并且仍可通过ICR发送IPI
    - 在软件禁用状态下，LVT表项的Mask位都被强制设置为1（即屏蔽）
- :warning: 第9位为Focus Processor Checking，取1表示禁用Focus Processor，取0表示启用。Focus Processor是奔腾和P6 family在处理Lowest Priority Mode时会涉及到的概念，**如今已没有用处**
- 第12位为Suppress EOI Broadcasts，设置为1则禁止一个Level Triggered的中断的EOI默认向IOAPIC广播EOI Message的行为
    > **Info:** 并非所有型号的CPU都支持该功能，查询APIC Version Register的第24位可知是否支持。

## Issuing IPIs
### Interrupt Command Register (ICR)
ICR是一个64位寄存器，分为ICR_Low和ICR_High两部分，分别位于Base + 0x300(Low)和Base + 0x310(High)，写入ICR_Low即可发送一个IPI。
> **Info:** ICR_Low的内容可能会在进入深度休眠的C-State后丢失

ICR_Low的内容如下：
- 第0-7位为Vector，即目标CPU收到的中断向量号，其中0-15号被视为非法，会给目标CPU的APIC产生一个Illegal Vector错误（即ESR的bit 6）
- 第8-10位为Delivery Mode，有以下几种取值：
    - 000 (Fixed)：按Vector的值向目标CPU(s)发送相应的中断向量号
    - 001 (Lowest Priority)：按Vector的值向Destination决定的所有目标CPU(s)中Priority最低的CPU发送相应的中断向量号（[详下](#lowest-priority-mode)）
        > **Info:** 发送Lowest Priority模式的IPI的能力取决于CPU型号，不总是有效，因此BIOS和OS不应该发送Lowest Priority模式的IPI
    - 010 (SMI)：向目标CPU(s)发送一个SMI，此模式下Vector必须为0
    - 100 (NMI)：向目标CPU(s)发送一个NMI，此时Vector会被忽略
    - 101 (INIT)：向目标CPU(s)发送一个INIT IPI，导致该CPU发生一次INIT（INIT后的CPU状态参考手册表9-1），此模式下Vector必须为0
        > **Info:** CPU在INIT后其APIC ID和Arb ID（只在奔腾和P6上存在）不变
    - :warning: 101 (INIT Level De-assert)：向所有CPU广播一个特殊的IPI，将所有CPU的APIC的Arb ID（只在奔腾和P6上存在）重置为初始值（初始APIC ID）。要使用此模式，Level必须取0，Trigger Mode必须取1，Destination Shorthand必须设置为All Including Self。
        - 只在奔腾和P6 family上有效，**已过时无需考虑**
    - 110 (Start-up)：向目标CPU(s)发送一个Start-up IPI（SIPI），目标会从0x000VV000开始执行，其中0xVV为Vector的值
- 第11位为Destination Mode，取0表示Physical，取1表示Logical（[详下](#determining-ipi-destination)）
- 第12位为Delivery Status（只读），取0表示空闲，取1表示上一个IPI尚未发送完毕
- :warning: 第13位为Level，取0则Delivery Mode 101表示INIT Level De-assert，否则表示INIT。
    - 只在奔腾和P6 family上有意义，**在奔腾4及以后的CPU上该位必须永远取1**
- :warning: 第14位为Trigger Mode，表示INIT Level De-assert模式下的trigger mode，取0表示edge triggered，取1表示level triggered。
    - 只在奔腾和P6 family上有意义，**在奔腾4及以后的CPU上该位必须永远取0**
- 第18-19位为Destination Shorthand，如果指定了Shorthand，就无需通过ICR_High中的Destination field来指定目标CPU，于是可以只通过一次对ICR_Low的写入发送一次IPI。该field取值如下：
    - 00 (No Shorthand)：目标CPU通过Destination指定
    - 01 (Self)：目标CPU为自身
    - 10 (All Including Self)：向所有CPU广播IPI，此时发送的IPI Message的Destination会被设置为0xF（奔腾和P6）或0xFF（奔腾4及以后），表示是一个全局广播
    - 11 (All Excluding Self)：同上，除了不向自己发送以外

ICR_High只有56-63位有效，用于表示Destination，决定了IPI发送的目标CPU(s)，如何决定见下节。

### Determining IPI Destination
#### Physical Destination Mode
若Destination Mode取0，则为Physical模式。在此模式下，Destination field表示目标CPU的APIC ID，0xF（奔腾和P6）或0xFF（奔腾4及以后）表示全局广播。

#### Logical Destination Mode
若Destination Mode取1，则为Logical模式。在此模式下，Destination field表示Message Destination Address（MDA）。IPI Message发送到总线上后，各个CPU的APIC会根据自己的Logical Destination Register（LDR）和Destination Format Register（DFR）决定是否要接受这个IPI。

Logical Destination Register，位于Base + 0xD0，其第24-31位表示Logical APIC ID

Destination Format Register，位于Base + 0xE0，其第28-31位表示Model，0000表示Cluster Model，1111表示Flat Model。
> **Note:** 所有（软件）启用的APIC的DFR必须设置到同一个模式，应当在尽量早地（在启动的早期阶段）设定好DFR，并且应在设置完DFR模式后再（软件）启用APIC

##### Flat Model
APIC将自己的LDR和总线上的IPI Message的MDA进行`AND`操作，结果非零则接受该IPI

##### Cluster Model【应该用于NUMA，不建议随便使用该模式】
- :warning: Flat Cluster模式，只有奔腾和P6 family支持
    - LDR和MDA各自分成两部分，高4位表示Cluster，只有LDR和MDA的高4位完全相等才表示属于该Cluster，可以进一步比较低4位，低4位用于选择具体的APIC，若LDR和MDA的低4位的`AND`结果非零则表示接受。
    - Cluster取15表示对所有Cluster广播，因此最多只支持0-14共15个Cluster。MDR取0xFF即可实现全局广播。
- Hierarchical Cluster模式，所有型号都支持
    - 系统分为若干个Cluster，最多可以有15个Cluster，每个Cluster需要有一个特殊的硬件，称作Cluster Manager，每个Cluster最多可以有4个Agent，总共最多60个APIC Agent
    - 具体寻址方式手册并未说明，从上述描述看似乎寻址方式和Flat Cluster相同，只是硬件结构不同（手册也没提到有什么软件方式可以切换这两个模式，因此认为寻址方式相同是合理的）

#### Lowest Priority Mode
当使用Logical Destination Mode或使用Destination Shorthand进行IPI群发时，可以使用Lowest Priority Mode，选出目标CPU集合中优先级最低的CPU作为发送对象，最终只有该CPU能收到IPI。

在至强及以后的CPU，选择是通过主板芯片组自动进行的，CPU或无法干涉【还需更多调查 #TODO#】。在奔腾4上，芯片组可以通过一个特殊的总线cycle获得CPU的Task Priority（似乎应为TPR[7:0]【存疑】），从而当需要仲裁时选出Priority最低的CPU。

在奔腾和P6 family上，仲裁依赖于Arbitration Priority Register（APR，位于Base + 0x90），它的取值如下：
```
if (TPR[7:4] >= IRRV[7:4] && TPR[7:4] > ISRV[7:4]) {
    APR[7:0] = TPR[7:0]
} else {
    APR[7:4] = max(TPR[7:4], ISRV[7:4], IRRV[7:4])
    APR[3:0] = 0
}
```
其中ISRV是ISR中最大的Vector，IRRV是IRR中最大的Vector。APR最小的CPU即胜出，获得IPI。

> **Note:** 除了IPI以外，IOAPIC发送的中断也可以是Lowest Priority Mode的，通过MSI发送的中断也可是Lowest Priority Mode的。该特性最初设计的时候应该是配合TPR使用，每当OS切换Task（进程/线程/etc.）时，就更新TPR（Task Priority），这样可以使得高优先级的Task不被中断打断，中断优先发给低优先级的Task。

## Message Signalled Interrupts
MSI的发送是通过向一个特殊的地址（表示目标CPU）进行一次PCI写入事务实现的。当某PCI设备要发送一个中断，它就会从自己的Message Address Register（以及Message Upper Address Register，如果地址是64位的话）读取出中断的目的地址，从Message Data Register读取出要填入中断Message的内容。下面分别介绍这两个寄存器：

Message Address Register：
- 第20-31位，必须为0xFEE
- 第12-19位为Destination ID，相当于IOAPIC中RTE的第56-63位，也相当于ICR_High中的Destination
- 第3位为Redirection Hint Indication（RH），取0相当于fixed mode，取1相当于lowest priority mode
    - RH取1，DM取0时，Destination不允许取0xFF，取其他值则可以正常发送到Physical Destination
    - RH取1，DM取1，且采用Clustered Addressing Model时，Destination不允许取0xFF，即不允许全局广播
- 第2位为Destination Mode（DM），取0表示Physical Mode，取1表示Logical Mode

Message Data Register：
- 第0-7位为Vector，即目标CPU收到的中断向量号，其中0-15号被视为非法，会给目标CPU的APIC产生一个Illegal Vector错误（即ESR的bit 6）
- 第8-10位为Delivery Mode，有以下几种取值：
    - 000 (Fixed)：按Vector的值向目标CPU(s)发送相应的中断向量号
    - 001 (Lowest Priority)：按Vector的值向Destination决定的所有目标CPU(s)中Priority最低的CPU发送相应的中断向量号
    - 010 (SMI)：向目标CPU(s)发送一个SMI，此模式下Vector必须为0，SMI必须是edge triggered的
    - 100 (NMI)：向目标CPU(s)发送一个NMI（走#NMI引脚），此时Vector会被忽略，NMI永远是edge triggered的，无论Trigger Mode设置成什么
    - 101 (INIT)：向目标CPU(s)发送一个INIT IPI，导致该CPU发生一次INIT，此时Vector会被忽略，INIT永远是edge triggered的，无论Trigger Mode设置成什么
    - 111（ExtINT）：向目标CPU(s)发送一个与8259A兼容的中断信号，将会引起一个INTA周期，CPU(s)在该周期向外部控制器索取Vector，ExtINT必须是edge triggered的
- 第14位为Level，若中断是level triggered的，则取1表示Assert，取0表示Deassert
- 第15位为Trigger Mode，取0表示edge triggered，取1表示level triggered

## Extended XAPIC (x2APIC)
将`IA32_APIC_BASE`MSR的第10位设置位1，即可启用x2APIC。断电重启后首先进入的是xAPIC模式，随后才能进入x2APIC模式，一旦进入则无法回到xAPIC模式（否则会引起#GP），必须进行一次重启（硬件禁用再启用）才能回到xAPIC模式。
> **Info:** x2APIC模式下INIT，回到x2APIC模式的初始状态，而不是xAPIC模式。同理xAPIC模式下INIT，回到xAPIC模式的初始状态。不允许在硬件禁用的同时开启x2APIC模式（试图如此做会引起#GP），硬件禁用时INIT仍处于禁用状态。

> 在启动x2APIC模式前，BIOS还应检查并启用VT-d中的Extended Interrupt Mode

x2APIC模式下通过MSR访问其寄存器，0x800到0x8FF的MSR都被预留给x2APIC。在进入x2APIC模式前无法访问这些MSR，否则会引起#GP。同样，进入x2APIC模式后，xAPIC的MMIO区域相当于关闭时的状态。

x2APIC与xAPIC相比，寄存器基本可以一一对应（MSR有64位宽，但只对应到32位的xAPIC寄存器，高32位保留），除了以下变化：
- 取消了DFR寄存器
- ICR_Low和ICR_High合并为了一个64位的寄存器ICR（MSR 0x830）
- 增加了SELF IPI寄存器（MSR 0x83F）

### x2APIC ID
APIC ID被改为32位，扩展后的ID可称之为x2APIC ID，占满了APIC ID Register的32位。寄存器被改为只读，只会在开机时由硬件设置一次，其末8位被作为xAPIC模式下的APIC ID使用。
> **Info:** 实际实现中x2APIC ID可能不足32位，此时不支持的高位永远为0，通过写入0xFFFFFFFF再读出即可确定其实际范围。

若支持x2APIC模式，则通过`CPUID.0BH.EDX`可以获得完整32位的x2APIC ID，从而帮助BIOS在xAPIC模式下判断系统的APIC ID超过了256的上限。若真的超过上限，则BIOS必须(a)在进入OS前就在所有CPU上均开启x2APIC模式或(b)只启用APIC ID小于等于255的CPU（其余CPU令其进入深度睡眠，保证不被OS启动），并且保持在xAPIC模式。

### ICR Operation
ICR的低32位与原本的ICR_Low相同（除了取消了Delivery Status位），高32位的Destination Field从8位扩展到了32位。

Destination取0xFFFFFFFF无论在Physical还是Logical模式都表示广播。Physical模式下的语义显见，下面讨论Logical模式的变化。

LDR的有效内容从8位扩展到了32位，且变为只读，其取值可称之为Logical x2APIC ID。x2APIC模式下取消了Flat Model，于是按照Clustered Model，Logical x2APIC ID分为高16位的Cluster ID和低16位的Logical ID。前者与Destination的高16位匹配，后者与Destination的低16位求`AND`。

实际上，初始化时Logical x2APIC ID就是由x2APIC ID决定的，计算方法为`Logical x2APIC ID = (x2APIC ID[19:4] << 16) | (1 << x2APIC ID[3:0])`。

### Self IPI Register
该寄存器是一个只写的寄存器，试图读取会造成#GP，仅0-7位有效，代表了Interrupt Vector。写入该寄存器的效果等价于通过写入ICR发送一个Edge Triggered、Fixed Interrupt的Self IPI。

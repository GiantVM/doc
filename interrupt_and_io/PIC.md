# PIC
PIC也就是8259A中断控制器，可以配合MCS-80/85（即8080/8085）或8086使用，前者与后者操作模式不同，以下只考虑8086模式，即现代PC所兼容的操作模式。手册[见此](http://heim.ifi.uio.no/~inf3151/doc/8259A.pdf)。

每个8259A芯片具备8个中断引脚，分别记为IRQ0-IRQ7。8259A支持级联，最多可以支持一个Master和八个Slave，但约定俗成的使用方式是一个Master加一个Slave。此时Master的中断记为IRQ0-IRQ7，Slave的中断记为IRQ8-IRQ15，Slave的INT输出信号连到Master的IRQ2，故总共支持15个中断信号。
> **Info:** 最初只有一个PIC芯片时，IRQ2已被占用，当IBM切换到双PIC芯片方案时，当初的IRQ2就被重新连接到了IRQ9上

8259A的两个基本寄存器是IRR和ISR，都是8位寄存器，用于记录当前正在处理的中断的状态。

## 基本流程
1. IRQ0-IRQ7的其中一个引脚收到一个中断信号，则设置IRR中对应的bit
2. 8259A通过INT信号线向CPU发送中断信号
3. CPU收到INT信号后，发出一个INTA信号，输入8259A的INTA输入引脚
4. 将IRR中的bit清除，在ISR中设置对应的bit
5. CPU在下一个周期再发出一个INTA信号，输入8259A的INTA输入引脚，此时8259A通过数据总线向CPU发送Interrupt Vector
6. 最后，若处于AEOI模式，ISR中的bit直接清除，否则要等CPU处理完该中断后，向8259A进行一次EOI，才能将ISR中的bit清除

> **Note:** 若在第四步时中断信号已经消失，无从判断其IRQ号，则会产生一个Spurious Interrupt，其Interrupt Vector和IRQ7一致。因此接受IRQ7（或IRQ15等）时，需先检查PIC的ISR，看其是否是Spurious Interrupt，然后才能进行中断处理。
>
> PS. 显然，这没有LAPIC为Spurious Interrupt单独赋予一个向量号先进。

### 级联模式
在级联模式下，若Slave收到一个中断，它会通过INT输出引脚输出一个中断信号，引起Master在某个IRQ引脚上收到中断。Master由于事先已经被配置好，知道该引脚对应的是Slave（而且知道是哪个），故在给CPU发送INT信号后，会通过一个3位的CAS选择器，选择8个Slave之一。被选中的Slave便在第二个INTA周期向数据总线输出Interrupt Vector，完成中断的发送。

需要注意的是，级联模式下，CPU执行完中断处理例程后，需要进行两次EOI，分别对Master和Slave各进行一次。

## 编程接口
PIC通过Port IO操纵，Master占据0x20和0x21两个端口，Slave占据0xA0和0xA1两个端口。下面我们将前一个端口称为Command端口，后一个端口称为Data端口（这是OSDev上的叫法）。[手册](http://heim.ifi.uio.no/~inf3151/doc/8259A.pdf)中的A_0位表示的就是Port号的最后一位，取0即Command，取1即Data。

### Initialization
在初始化时有四个Word（8位）可以使用，分别为ICW1-ICW4（Initialization Command Word）。写入ICW1就会启动初始化过程，重新初始化PIC。

当A_0位为0（即写入Command端口）且输入值的第四位取1时，就认为是ICW1，由此会开启初始化过程。随后需要输入ICW2-ICW4，它们都要求A_0位1，即从Data端口输入。

ICW1的内容如下：
- 第0位为IC4，表示是否需要ICW4，取1表示需要
- 第1位为SNGL，取1表示Single，取0表示Cascade，若为Single模式则不需要ICW3
- :warning: 第2位为ADI,8080/8085模式才有用，在8086模式下会被忽略
- 第3位为LTIM，取1表示Level Triggered Mode，此时Edge Triggered Interrupt会被忽略
- 第4位必须为1
- :warning: 第5-7位，8080/8085模式才有用，在8086模式下会被忽略

ICW1之后必须紧跟ICW2，内容如下：
- :warning: 第0-2位，8080/8085模式才有用，在8086模式下会被忽略
- 第3-7位，表示Interrupt Vector的第3-7位。也就是说一块PIC的IRQ0-IRQ7，会被映射到Offset+0到Offset+7，其中Offset就是ICW2的值（末三位总是被视为零，设置时填零即可）

> **Info:** 在实模式下，约定俗成的设置为Master的IRQ映射到0x08-0x0F，Slave的IRQ映射到0x70-0x77，BIOS通常会如此设置。但进入保护模式后，0x08-0x0F与CPU默认的Exception范围冲突，故应该将IRQ重映射，一般是映射到0x20-0x2F。

如果处于级联模式，则还需设置ICW3，内容如下：
- Slave模式时，第0-2位表示Slave ID（对应于CAS选择器的值），其余位保留，应均为0
- Master模式时，ICW3是一个8位的bitmap，每一位对应于一个Slave

> **Info:** 在非Buffered模式下，PIC是处于Master还是Slave状态，是由SP/EN输入引脚的电平决定的，若为1则是Master，若为0则是Slave。在Buffered模式下，则是由ICW4的M/S位确定的，取1表示Master，取0表示Slave。

如果IC4取1，则还需要设置ICW4，否则ICW4当做全为0处理，内容如下：
- 第0位为μPM，取0表示8080/8085操作模式，取1表示8086操作模式
    > 由此可见现代机器上若要使用PIC，必须设置ICW4
- 第1位为AEOI，取1表示开启Automatic EOI模式
- 第2位为M/S，在Buffered模式下用于确定Master和Slave
- 第3位为BUF，取1表示开启Buffered模式
    - Buffered模式下，SP/EN引脚作为输出引脚，控制Buffer的开启。所谓的Buffer，就是在PIC和Data Bus之间设置的一个Buffer。
- 第4位为SFNM，取1表示开启Special Fully Nested模式
    - 该模式用于级联配置，应由Master开启Special Fully Nested模式，此时Slave的中断不会被In-Service的它自己屏蔽。在该模式下，BIOS或OS需在Slave的中断处理完时检查Slave中是否有超过一个中断等待EOI，若是则只需对Slave进行EOI而无需对Master进行EOI，否则应对Master也进行EOI
- 第5-7位保留，应全为0

### Operation
初始化完成后，还能通过三个8位的OCW（Operation Command Word）进行运行时的调整。

当A_0位为1时（即Data端口），输入值即为OCW1，代表了Interrupt Mask，其每一位对应于一个IRQ，取1即屏蔽该IRQ。同时，读取该端口即可读到IMR（Interrupt Mask Register）的值。

当A_0位为0时（即Command端口），根据输入值的第3、第4位决定其含义。若第4位为1，则表示ICW1，若第4位为0，则第3位取0代表OCW2，第3位取1代表OCW3。

#### OCW2
在介绍OCW2前，先介绍以下概念：
##### End of Interrupt
- EOI Command：有两种EOI Command，即Specific和Non-Specific，Non-Specific EOI会自动清空ISR中最高优先级位【**手册中为最高位，疑为笔误，似应为最高优先级位**】，Specific EOI则清空ISR中被指定的那个Bit
- AEOI Mode：若ICW4中的AEOI位取1，则进入Automatic EOI模式，在中断收到后（INTA周期结束后）立即自动进行一次Non-Specific EOI

##### Interrupt Priority
- Fully Nested Mode：PIC初始化后的默认状态，此时中断的优先级依照IRQ0-IRQ7的顺序依次降低，高优先级的中断可以打断低优先级的
    > **Info:** 一旦ISR中有高优先级的中断，低优先级的中断就不会进入IRR，而在LAPIC中则还能在IRR中存入一次低优先级的中断
- Rotation Mode：优先级轮转，一次轮转即令某个IRQ优先级变为最低，其余IRQ随之轮转（e.g. 令IRQ4为最低优先级7，则IRQ3优先级变为6，以此类推，最后IRQ5变为最高优先级0）
    - Automatic Rotation：依附在EOI上的选项，令EOI带有使被其清除的Bit对应的IRQ优先级变为最低，并让其余IRQ依次轮转的能力
    - Specific Rotation：手动进行优先级轮转，指定某个IRQ优先级变为最低，并让其余IRQ依次轮转（该操作与EOI无关，但可以依附在Specific EOI上）

##### Operations of OCW2
OCW2的内容如下：
- 第0-2位，配合SL位使用，用于指定某个IRQ
- 第3-4位必须为0
- 第5位为EOI，取1表示这是一个EOI Command，可以清空ISR中的一个Bit
- 第6位为SL，即Specific Level，取1表示指定一个IRQ进行操作（Specific EOI或Specific Rotation）
- 第7位为R，即Rotation，用于决定是否进行优先级轮转

OCW2的第5-7位组合如下：
- EOI=1, SL=0, R=0：Non-Specific EOI Command，清空ISR最高优先级位，优先级不变
- EOI=1, SL=1, R=0：Specific EOI Command，OCW2的第0-2位用于指定清除ISR的哪个Bit
- EOI=1, SL=0, R=1：Non-Specific EOI with Automatic Rotation，清空ISR最高优先级位，优先级轮转
- EOI=0, SL=0, R=1：开启AEOI模式下的Automatic Rotation，使得AEOI下自动进行的Non-specific EOI带有Automatic Rotation功能
- EOI=0, SL=0, R=0：关闭AEOI模式下的Automatic Rotation
- EOI=0, SL=1, R=1：Specific Rotation，OCW2的第0-2位用于指定哪个IRQ为最低优先级
- EOI=1, SL=1, R=1：Specific EOI with Specific Rotation，OCW2的第0-2位用于指定清除ISR的哪个Bit以及指定哪个IRQ为最低优先级
- EOI=1, SL=1, R=0：无效操作，不会发生任何事

#### OCW3
OCW3的内容如下：
- 第0位为RIS，即Read ISR，取1表示从Command端口读出IRR，取0表示从Command端口读出ISR
- 第1位为RR，Read Register，取1时启用RIS位，取0时RIS位会被忽略
- 第2位为P，取1表示这是一个Poll Command，取0则没有任何效果
- 第3位必须为1，第4位必须为0
- 第5位为SMM，Special Mask Mode，取1代表开启Special Mask模式，取0代表关闭
- 第6位为ESMM，Enable Special Mask Mode，取1时启用SMM位，取0时SMM位会被忽略
- 第7位保留，应为0

实际上OCW3共集成了三个功能（可以同时使用），第一个功能是从Command端口（A_0位为0）可以读出IRR或ISR的值，通过RR和RIS位可以选择读取哪个，PIC初始化后默认是读出IRR的值。

第二个功能是Poll模式，在通过OCW3发出一个Poll Command后，下一次读取Command端口（A_0位为0）时，就相当于一次中断Accept。若此时恰有中断Pending，则ISR中的Bit会被设置，读到的值最高位为1，最低3位为IRQ号。否则，则读到的值最高位为0，表示没有中断发生。

第三个功能是Special Mask模式，在此模式下若某个位被IMR屏蔽，则有以下效果：
- 即使该位是ISR中有最高优先级，也不会被Non-Specific EOI清空
- 其余未被屏蔽的位，即使优先级低于该位，其对应的IRQ仍然可以被接受

## PIC in ICH9
随着芯片集成度的提升，8259芯片被集成到了南桥内，我们可以从历代南桥手册中查询到其IRQ pin接到的是什么设备，已经经历了什么小改动。此处介绍QEMU模拟的Q35芯片组中的ICH9南桥中对8259的改动和配置。

### 改动
在ISA总线时代，ICW1的第3位决定了是否是Level Triggered，取1表示整个PIC的中断都是Level Triggered的。进入PCI总线时代后（从PIIX芯片组开始），添加了两个8位的ELCR（Edge/Level Control Register）寄存器（ELCR0、ELCR1），分别位于0x4D0和0x4D1端口，每个位控制一个IRQ是否是Level Triggered的。

### 配置
Slave的ID设置为010b，Master和Slave的IRQ Pin设置如下：

![ich9_8259](/assets/ich9_8259.png)

其中IRQ0、IRQ1、IRQ2、IRQ8、IRQ13必须为Edge Triggered，即ELCR中对应位必须为0


本文将在熟悉QOM和q35架构的基础上，分析QEMU中的PCI设备是如何被初始化和挂载的。


## PCI设备的创建与初始化

```
pci_create_simple_multifunction => pci_create_multifunction => qdev_create(&bus->qbus, name) => object_class_by_name
                                                            => qdev_prop_set_int32(dev, "addr", devfn)
                                => qdev_init_nofail => dc->realize (pci_qdev_realize) => do_pci_register_device
                                                                                      => pc->realize (ich9_lpc_realize)
                                                                                      => pci_add_option_rom              添加rom
```

创建了PCI设备对象，并设置realized为true，于是调用realize函数进行初始化。

这其中关键的一点在于 pci_create_simple_multifunction 传入的 devfn 参数，它是PCI设备的设备功能号，一般通过以下宏算出：

```c
#define PCI_DEVFN(slot, func)   ((((slot) & 0x1f) << 3) | ((func) & 0x07))
```

因此通过该宏算出的 devfn 是一个8bit的数字，高5bit为slot号，低3bit位func号。因此有32个slot，每个slot有8个function。

比如对于ISA桥 "ICH9-LPC" ， 其功能号为 `PCI_DEVFN(ICH9_LPC_DEV, ICH9_LPC_FUNC)` ，即 31 * 2^3 + 0 = 248

在 object_class_by_name 时，会调用类实例构造函数。根据 ICH9-LPC 的 TypeInfo ：

```c
static const TypeInfo ich9_lpc_info = {
    .name       = TYPE_ICH9_LPC_DEVICE,
    .parent     = TYPE_PCI_DEVICE,
    .instance_size = sizeof(struct ICH9LPCState),
    .instance_init = ich9_lpc_initfn,
    .class_init  = ich9_lpc_class_init,
    .interfaces = (InterfaceInfo[]) {
        { TYPE_HOTPLUG_HANDLER },
        { TYPE_ACPI_DEVICE_IF },
        { }
    }
};
```

其类实例构造函数为 ich9_lpc_class_init ，在里面对这款设备的属性进行了初始化：

```c
static void ich9_lpc_class_init(ObjectClass *klass, void *data)
{
    ...
    k->realize = ich9_lpc_realize;
    k->config_write = ich9_lpc_config_write;
    dc->desc = "ICH9 LPC bridge";
    k->vendor_id = PCI_VENDOR_ID_INTEL;
    k->device_id = PCI_DEVICE_ID_INTEL_ICH9_8;
    k->revision = ICH9_A2_LPC_REVISION;
    k->class_id = PCI_CLASS_BRIDGE_ISA;
    ...
}
```

### do_pci_register_device
对设备实例对象进行设置。

```
=> 查看 bus->devices[devfn] 是否为空，如果不为空，表示位置被人占了，报错返回
=> 如果设备是hotplugged的， pci_get_function_0 如果非空，表示slot被人占了，报错返回
    => 如果bus有upstream PCIe port，则只能放在第一个slot的第一个设备，即devfn=0。否则可以放在devfn对应slot的第一个设备处
    => pci_init_bus_master
=> pci_config_alloc(pci_dev)                                                                                   分配配置空间，PCI设备为256B，PCIe为4096B
=> pci_config_set_vendor_id / pci_config_set_device_id / pci_config_set_revision / pci_config_set_revision     根据初始化时的数据设置设备标识
=> 如果设备是bridge(is_bridge)， pci_init_mask_bridge
=> pci_init_multifunction
=> 设置 pci_dev->config_read 和 pci_dev->config_write ，如果在类构造函数中设置了则用设置的，否则使用默认函数 pci_default_read_config / pci_default_write_config
```

## BAR(base address register)

根据PCI规范，BAR用于描述PCI设备需要占用的地址空间大小。比如网卡等设备需要占用较大的地址空间，而一些串口设备则占用较少的地址空间。其位于PCI配置空间中0x10-0x27这24个byte。如果使用32bit的BAR，则每个设备最多能设置6个。如果使用64bit，则最多只能设置3个。

对于每个BAR，bit0指定了映射的类型，0为Memory，1为I/O。在真实硬件上，bit0是readonly的，由设备制造商决定，其他人无法修改。

对于Memory类型，bit1-2为Locatable，0为表示寄存器大小为32bit，能映射到32bit内存空间中的任何位置；2为64bit；1保留给PCI Local Bus Specification 3修订版。bit3为Prefetchable，0为no，1为yes。bit4-end为Base Address(16-byte aligned)。

对于I/O类型，bit1为Reserved，bit2-end为Base Address(4-byte aligned)。

每一个BAR对应的长度由硬件决定，BIOS/OS根据获取到的长度动态的为其分配一段内存空间，然后将内存空间的起始地址写入BAR中作为address base，于是[base, base+len]这段范围将作为软件(OS)和PCI设备交流的信道。

根据qemu monitor查看PCI设备的BAR ：

```
(qemu) info pci
  Bus  0, device   0, function 0:
    Host bridge: PCI device 8086:29c0
      id ""
  Bus  0, device   1, function 0:
    VGA controller: PCI device 1234:1111
      BAR0: 32 bit prefetchable memory at 0xfd000000 [0xfdffffff].
      BAR2: 32 bit memory at 0xfebf0000 [0xfebf0fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0000fffe].
      id ""
  Bus  0, device   2, function 0:
    Ethernet controller: PCI device 8086:100e
      IRQ 11.
      BAR0: 32 bit memory at 0xfebc0000 [0xfebdffff].
      BAR1: I/O at 0xc000 [0xc03f].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0003fffe].
      id ""
  Bus  0, device  31, function 0:
    ISA bridge: PCI device 8086:2918
      id ""
  Bus  0, device  31, function 2:
    SATA controller: PCI device 8086:2922
      IRQ 10.
      BAR4: I/O at 0xc080 [0xc09f].
      BAR5: 32 bit memory at 0xfebf1000 [0xfebf1fff].
      id ""
  Bus  0, device  31, function 3:
    SMBus: PCI device 8086:2930
      IRQ 10.
      BAR4: I/O at 0x0700 [0x073f].
      id ""
```

可以发现只有 VGA controller 、 Ethernet controller 、SATA controller 和 SMBus 有BAR，而 Host bridge 和 PCI/ISA bridge (也就是前面提到的ICH9-LPC) 没bar。因此猜测是bridge设备都不需要映射地址空间来和OS进行通信。

对应到OS中，通过`lspci -x`可以找到对应的bar，比如对于 PCI/ISA bridge，其 0x10 - 0x27 就是全0，而对于 Ethernet controller ，0x10 - 0x27 为：

```
10: 00 00 bc fe 01 c0 00 00 00 00 00 00 00 00 00 00
20: 00 00 00 00 00 00 00 00
```

一个bar占4个byte，因此可以看出(**x86为小端**)：

* BAR 0 值为 0xfebc0000 ，对应Memory类型，32bit，no Prefetchable，base = 0xfebc0000
* BAR 1 值为 0x0000c001 ，对应I/O类型，32bit，no Prefetchable，base = 0x0000c000
* BAR 6 值为 0xffffffffffffffff ，对应Memory类型，对应的是ROM。

可以发现lspci的结果和从QEMU monitor查询而得的结果是一致的。


### 设置BAR

那么设备的BAR在QEMU是什么时候被设置的呢？经过一番搜索，找到 pci_register_bar ：

```c
void pci_register_bar(PCIDevice *pci_dev, int region_num,
                      uint8_t type, MemoryRegion *memory)
{
    PCIIORegion *r;
    uint32_t addr; /* offset in pci config space */
    uint64_t wmask;
    pcibus_t size = memory_region_size(memory);

    assert(region_num >= 0);
    assert(region_num < PCI_NUM_REGIONS);
    if (size & (size-1)) {
        fprintf(stderr, "ERROR: PCI region size must be pow2 "
                    "type=0x%x, size=0x%"FMT_PCIBUS"\n", type, size);
        exit(1);
    }

    r = &pci_dev->io_regions[region_num];
    r->addr = PCI_BAR_UNMAPPED;
    r->size = size;
    r->type = type;
    r->memory = memory;
    r->address_space = type & PCI_BASE_ADDRESS_SPACE_IO
                        ? pci_dev->bus->address_space_io
                        : pci_dev->bus->address_space_mem;

    wmask = ~(size - 1);
    if (region_num == PCI_ROM_SLOT) {
        /* ROM enable bit is writable */
        wmask |= PCI_ROM_ADDRESS_ENABLE;
    }

    addr = pci_bar(pci_dev, region_num);
    pci_set_long(pci_dev->config + addr, type);

    if (!(r->type & PCI_BASE_ADDRESS_SPACE_IO) &&
        r->type & PCI_BASE_ADDRESS_MEM_TYPE_64) {
        // 64bit base address
        pci_set_quad(pci_dev->wmask + addr, wmask);
        pci_set_quad(pci_dev->cmask + addr, ~0ULL);
    } else {
        // 32bit
        pci_set_long(pci_dev->wmask + addr, wmask & 0xffffffff);
        pci_set_long(pci_dev->cmask + addr, 0xffffffff);
    }
}
```

其根据传入的 MemoryRegion ，初始化 PCIIORegion 结构，其表示了BAR中对应的一段地址空间：

```c
typedef struct PCIIORegion {
    pcibus_t addr; /* current PCI mapping address. -1 means not mapped */
#define PCI_BAR_UNMAPPED (~(pcibus_t)0)
    pcibus_t size;
    uint8_t type;
    MemoryRegion *memory;
    MemoryRegion *address_space;
} PCIIORegion;
```

这里设置了映射类型(type)、大小(size)、地址(addr，初始值为 PCI_BAR_UNMAPPED )等，然后设置为 PCIDevice.io_regions 数组region_num对应项。也就是说，所有BAR的相关信息都被存储在 io_regions 数组中。

同时，将信息写入config中BAR对应的位置，这里只设置了BAR的type(bit0)。

比如对于网卡e1000来说，在 pci_e1000_realize 时，设置了：

```c
pci_register_bar(pci_dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &d->mmio);
pci_register_bar(pci_dev, 1, PCI_BASE_ADDRESS_SPACE_IO, &d->io);
```

d->mmio 和 d->io 都是MemoryRegion，在 e1000_mmio_setup 中通过 memory_region_init_io 创建，前者大小为 PNPMMIO_SIZE(0x20000)，后者大小为 IOPORT_SIZE(0x40)。


### MemoryRegion 的映射

既然我们创建了MemoryRegion并设置到 io_regions 中，那么其必然需要将对应的内存区域注册到虚拟机中，我们以 io_regions 作为关键字在代码中进行搜索，找到了 pci_do_device_reset ，它是在什么时候被调用呢？通过跟踪可以发现，当QEMU在main函数中将设备都注册完成，一切初始化都准备就绪，即将调用 main_loop 进入主循环之时，调用了 qemu_system_reset ，堆栈如下：

```
(gdb) bt
#0  pci_do_device_reset (dev=0x7ffff22a0010) at hw/pci/pci.c:278
#1  0x0000555555a1a657 in pcibus_reset (qbus=0x5555569a5640) at hw/pci/pci.c:304
#2  0x00005555559720ef in qbus_reset_one (bus=0x5555569a5640, opaque=0x0) at hw/core/qdev.c:304
#3  0x0000555555976e53 in qbus_walk_children (bus=0x5555569a5640, pre_devfn=0x0, pre_busfn=0x0, post_devfn=0x55555597206c <qdev_reset_one>, post_busfn=0x55555597208f <qbus_reset_one>, opaque=0x0) at hw/core/bus.c:68
#4  0x0000555555972c50 in qdev_walk_children (dev=0x55555694ec00, pre_devfn=0x0, pre_busfn=0x0, post_devfn=0x55555597206c <qdev_reset_one>, post_busfn=0x55555597208f <qbus_reset_one>, opaque=0x0) at hw/core/qdev.c:602
#5  0x0000555555976e17 in qbus_walk_children (bus=0x555556769d20, pre_devfn=0x0, pre_busfn=0x0, post_devfn=0x55555597206c <qdev_reset_one>, post_busfn=0x55555597208f <qbus_reset_one>, opaque=0x0) at hw/core/bus.c:59
#6  0x00005555559721a2 in qbus_reset_all (bus=0x555556769d20) at hw/core/qdev.c:321
#7  0x00005555559721c5 in qbus_reset_all_fn (opaque=0x555556769d20) at hw/core/qdev.c:327
#8  0x00005555558d94ec in qemu_devices_reset () at vl.c:1765
#9  0x000055555582832c in pc_machine_reset () at /home/binss/work/qemu/qemu-2.8.1.1/hw/i386/pc.c:2181
#10 0x00005555558d9589 in qemu_system_reset (report=false) at vl.c:1778
#11 0x00005555558e0eb0 in main (argc=19, argv=0x7fffffffe498, envp=0x7fffffffe538) at vl.c:4656
```

调用链为 main => qemu_system_reset => mc->reset (pc_machine_reset) => qemu_devices_reset 。它会遍历所有的 reset_handlers ，拿到每一个QEMUResetEntry，调用它们的回调函数 re->func 。 QEMUResetEntry 在 qemu_register_reset 时加入到该队列中。

`#8` 对应的 QEMUResetEntry 为 main-system-bus ，也就是系统总线，其在 vl.c 中 `qemu_register_reset(qbus_reset_all_fn, sysbus_get_default());` 注册了 qbus_reset_all_fn 为reset handler。于是这里调用 qbus_reset_all_fn

qbus_reset_all_fn => qbus_reset_all => qbus_walk_children 会遍历该bus上的所有dev，对它们调用 qdev_walk_children 。对于 e1000 来说，这里自然是 q35-pcihost (如果不记得q35上dev和bus是怎么连接的，请参考q35) 。然后 qdev_walk_children 又对 q35-pcihost 上的所有child_bus都调用 qbus_walk_children ，于是来到 pcie.0 ，注意e1000就是挂在这个bus上。于是这里暂时打住，不考虑 pcie.0 的 qbus_walk_children 递归，而是关系递归完成后，调用了 post_busfn (qbus_reset_one) 对 pcie.0 这个bus进行reset。

于是 qbus_reset_one => bc->reset (pcibus_reset) ，会遍历 bus->devices ，对每一个设备调用 pci_do_device_reset


顾名思义， pci_do_device_reset 主要做了重置工作：

```
=> pci_word_test_and_clear_mask     清除config中 PCI_COMMAND 对应的2个byte(0x04-0x06)
=> pci_word_test_and_clear_mask     清除config中 PCI_STATUS 对应的2个byte(0x06-0x08)
=> 清除config中的 PCI_CACHE_LINE_SIZE 和 PCI_INTERRUPT_LINE 分别对应的1个byte
=> 遍历 io_regions ，如果存在，设置 config 中BAR对应位置的值，只写入了类型，剩余部分留给虚拟机内的BIOS/OS进行设备
=> pci_update_mappings
=> msi_reset
=> msix_reset
```

这里的关键是 pci_update_mappings ，其负责检查 io_regions 数组中 PCIIORegion 的 addr ，如果 addr 不等于 PCI_BAR_UNMAPPED ，表示地址已经映射完成，于是通过 memory_region_add_subregion_overlap / memory_region_del_subregion 利用ioctl将BAR对应的 MemoryRegion 注册到KVM中(如果使用了KVM的话)：

```c
static void pci_update_mappings(PCIDevice *d)
{
    PCIIORegion *r;
    int i;
    pcibus_t new_addr;

    // 第7个io region(BAR 6)是 PCI_ROM_SLOT
    for(i = 0; i < PCI_NUM_REGIONS; i++) {
        r = &d->io_regions[i];

        /* this region isn't registered */
        if (!r->size)
            continue;

        // 从config中读取该BAR中存储的base address
        new_addr = pci_bar_address(d, i, r->type, r->size);

        /* This bar isn't changed */
        if (new_addr == r->addr)
            continue;

        /* now do the real mapping */
        if (r->addr != PCI_BAR_UNMAPPED) {
            trace_pci_update_mappings_del(d, pci_bus_num(d->bus),
                                          PCI_SLOT(d->devfn),
                                          PCI_FUNC(d->devfn),
                                          i, r->addr, r->size);
            memory_region_del_subregion(r->address_space, r->memory);
        }
        r->addr = new_addr;
        if (r->addr != PCI_BAR_UNMAPPED) {
            trace_pci_update_mappings_add(d, pci_bus_num(d->bus),
                                          PCI_SLOT(d->devfn),
                                          PCI_FUNC(d->devfn),
                                          i, r->addr, r->size);
            memory_region_add_subregion_overlap(r->address_space,
                                                r->addr, r->memory, 1);
        }
    }

    pci_update_vga(d);
}
```

遗憾的是，在 qemu_system_reset 时，BAR的base address还没被设置，依然处于 unmap 的状态。于是 MemoryRegion 也就没有被注册到KVM中。

那么e1000的这两个 MemoryRegion 到底在何时才会被注册到KVM中呢？答案是，进入虚拟机且其开始运行后，执行设备发现流程，给e1000配置一个address base，然后写到e1000的配置空间(config)中，这时才会将其注册，这会在下文再进行分析。



### 配置空间写入和读取

于是我们的配置空间初始化好了，接下来就是等待虚拟机来读写了。前面提到过，PCI设备设置了 config_write 和 config_read ，就是干这事的。理论上，BIOS/OS 应该通过 config_read 读取config，获得PCI设备的配置空间信息，然后为PCI设备分配地址，写回配置空间的BAR。


由于前面发现 ICH9-LPC 不带BAR，因此接下来选取 e1000 来进行分析（在写此文时一直抓着ICH9-LPC一通分析，最后才发现由于它不带bar导致很多过程直接continue了，不能较好的反应PCI设备的注册流程，泪崩）。

根据 e1000_base_info ，其类构造函数为 e1000_class_init ，里面没有配置自己的config函数，但在初始化函数 pci_e1000_realize 设置了 `pci_dev->config_write = e1000_write_config` ，因此其 config_write 为 e1000_write_config， config_read 使用默认的 pci_default_read_config 。

我们找到 config 的地址，然后设置观察点，观察其在读写的时刻。

1. 在 do_pci_register_device 中分配内存，对config内容进行设置，如 pci_config_set_vendor_id
2. 在 pci_e1000_realize 中继续设置config，包括 pci_register_bar 中将BAR base address设置为全f
3. 由于有ROM(efi-e1000.rom)，于是调用 pci_add_option_rom ，注册 PCI_ROM_SLOT 为BAR6
4. pci_do_device_reset (调用链前面提过) 进行清理和设置
5. KVM_EXIT_IO
    QEMU => KVM => VM 后，当VM运行port I/O指令访问config信息时，发生VMExit，VM => KVM => QEMU：

    kvm_cpu_exec => kvm_handle_io => address_space_rw => address_space_read => address_space_read_full => address_space_read_continue => memory_region_dispatch_read => memory_region_dispatch_read1 => access_with_adjusted_size => memory_region_read_accessor => mr->ops->read (pci_host_data_read) => pci_data_read => pci_host_config_read_common => pci_default_read_config

    于是e1000的config_read被调用，读取对应位置的配置空间信息返回给 KVM => VM。

    同理当VM需要写入config时，发生VMExit，于是 VM => KVM => QEMU，其调用链如下：

    vm_cpu_exec => kvm_handle_io => address_space_rw => address_space_write => address_space_write_continue => memory_region_dispatch_write => access_with_adjusted_size => memory_region_write_accessor => mr->ops->write (pci_host_data_write) => pci_data_write => pci_host_config_write_common => pci_dev->config_write (e1000_write_config) => pci_default_write_config

    当退回到QEMU时，QEMU根据 exit_reason 得知原因是 KVM_EXIT_IO ，于是从 cpu->kvm_run 中取出 io 信息，如 `{direction = 1, size = 4, port = 3324, count = 1, data_offset = 4096}` 。于是对 address_space_io 进行操作，根据 direction 判断是读还是写，调用 address_space_read(write) 。其通过 address_space_translate 找到地址(port)对应的 MemoryRegion 和 xlat(偏移量?)，对其进行读写。于是 address_space_read(write)_continue => memory_region_dispatch_read(write) 根据读写设置 accessor 。

    最后来到 access_with_adjusted_size ，其调用传入的accessor，于是从 MemoryRegion 的ops成员中取出对应操作，比如 pci_host_data_le_ops ：

    ```c
    const MemoryRegionOps pci_host_data_le_ops = {
        .read = pci_host_data_read,
        .write = pci_host_data_write,
        .endianness = DEVICE_LITTLE_ENDIAN,
    };
    ```

    对于写操作，则调用 pci_host_data_write ，其会将 MemoryRegion 的opaque转换为 PCIHostState ，调用 pci_data_write 。以参数 addr=2147487760, val=4294967295, len=4 为例，此时传入的 PCIBus 为 pcie.0 ：

    * pci_dev_find_by_addr  将addr(2147487760=0x80001010)右移16位截断，得到bus号，0x00；右移8位截断，得到devfn，0x10。由于当前bus的bus号就是0，因此直接取bus的devices[16]返回。该设备就是e1000
    * 通过 `addr & (PCI_CONFIG_SPACE_SIZE - 1)`计算出 config_addr ，即要修改的值在该设备config配置空间中的偏移量。比如说 2147487760 & 0xff = 16，因此这里要修改的就是 BAR0 。
    * pci_host_config_write_common => pci_dev->config_write (e1000_write_config) => pci_default_write_config 将 4294967295 写入 config[16:19]

6. KVM_EXIT_MMIO

    设置完config后，在Linux完成了了对设备的初始化后，就可以进行通信了。当VM对映射的内存区域进行访问时，发生VMExit，VM => KVM => QEMU：

    address_space_rw => address_space_write => address_space_write_continue => memory_region_dispatch_write => access_with_adjusted_size => memory_region_write_accessor => e1000_mmio_write

    当退回到QEMU时，QEMU根据 exit_reason 得知原因是 KVM_EXIT_MMIO ，于是从 cpu->kvm_run 中取出 mmio 信息，如 `$64 = {phys_addr = 4273733840, data = "\235\000\000\000\000\000\000", len = 4, is_write = 1}` 。于是对 address_space_memory 进行操作，根据 is_write 判断是读还是写，调用 address_space_read(write) 。其通过 address_space_translate 找到地址(phys_addr)对应的 MemoryRegion 和 xlat(偏移量?)，对其进行读写。于是 address_space_read(write)_continue => memory_region_dispatch_read(write) 根据读写设置 accessor 。

    最后来到 access_with_adjusted_size ，其调用传入的accessor，于是从 MemoryRegion 的ops成员中取出对应操作，比如 e1000_mmio_ops ：

    ```c
    static const MemoryRegionOps e1000_mmio_ops = {
        .read = e1000_mmio_read,
        .write = e1000_mmio_write,
        .endianness = DEVICE_LITTLE_ENDIAN,
        .impl = {
            .min_access_size = 4,
            .max_access_size = 4,
        },
    };
    ```

    对于写操作，则调用 e1000_mmio_write ，其会将 MemoryRegion 的opaque转换为 E1000State ，调用 macreg_writeops[index]。以参数addr=208, val=157, size=4为例，此时操作的地址是4273733840(0xfebc00d0)：

    `index = (addr & 0x1ffff) >> 2`，于是 index = 52。macreg_writeops[52]为 set_ims ，其在设置IMS后又调用 set_ics ，其负责设置中断，于是 set_interrupt_cause => pci_set_irq => pci_irq_handler => pci_update_irq_status ，将设备配置空间(config)的 PCI_STATUS([6:7]) 的 PCI_STATUS_INTERRUPT bit置1，表示收到INTx#信号。

    对于MMIO读，过程类似，只是在读后将 PCI_STATUS_INTERRUPT bit置0。



对于e1000而言，在Linux启动之前(没有任何启动信息输出)，进行了以下写操作：

* 将addr[16, 19]，即BAR0，写为 4294967295(0xffffffff)
* 将addr[16, 19]，即BAR0，写为 0(0x0)
* 将addr[16, 19]，即BAR0，写为 4273733632(0xfebc0000)
* 将addr[20, 23]，即BAR1，写为 4294967295(0xffffffff)
* 将addr[20, 23]，即BAR1，写为 1(0x1)
* 将addr[20, 23]，即BAR1，写为 49152(0xc000)
* 将addr[4, 5]，即COMMAND，写为 259(0x103，100000011)，响应Memory Space和I/O Space的访问。启用SERR# driver。

Linux启动后，进行了以下写操作：

* 将addr[4, 5]，即COMMAND，写为 256(0x100，100000000)，启用SERR# driver。
* 将addr[4, 5]，即COMMAND，写为 259(0x103，100000011)，响应Memory Space和I/O Space的访问。启用SERR# driver。

然后发现Linux又重新做长度检测，然后写入值。将BAR1写为49153(0xc001)。目测是感知到它是IO port，将其最后一bit修正为1。

将addr[4, 5]，即COMMAND，写为 263(0x103，100000111)，响应Memory Space和I/O Space的访问。启用SERR# driver。成为bus master。

此后在e1000的工作流程中，会在发包时调用 e1000x_rx_ready 检查是否就绪，这也需要读取 config[PCI_COMMAND] ，只有其为bus master，才能算是ready。



#### pci_default_write_config

在设置 config 时，最终都会调用到 pci_default_write_config 。根据：

```c
    if (ranges_overlap(addr, l, PCI_BASE_ADDRESS_0, 24) ||
        ranges_overlap(addr, l, PCI_ROM_ADDRESS, 4) ||
        ranges_overlap(addr, l, PCI_ROM_ADDRESS1, 4) ||
        range_covers_byte(addr, l, PCI_COMMAND))
        pci_update_mappings(d);
```

如果修改的 [PCI_BASE_ADDRESS_0:PCI_BASE_ADDRESS_0+24] 表示的 BAR0-BAR5，[PCI_ROM_ADDRESS:PCI_ROM_ADDRESS+4]、[PCI_ROM_ADDRESS1:PCI_ROM_ADDRESS1+4] 表示的BAR6，或则改动了 PCI_COMMAND ，则触发 pci_update_mappings

但经过调试，对 io_regions 中 PCIIORegion.address ，即base address的更新并不是在更新对应BAR的 pci_default_write_config 里面通过 pci_update_mappings 更新的，因为此时 PCI_COMMAND 中bit0和bit1为0，表示不响应Memory Space和I/O Space的访问，因此在 pci_bar_address 转换地址的过程中cmd为0，因此都会返回 PCI_BAR_UNMAPPED 。

而是要等到将 PCI_COMMAND 从 256(0x100) 更新为 259(0x103) 时，表示开启了响应Memory Space和I/O Space的访问，此时一样会调用 pci_update_mappings ，遍历7个region，从config中依次读出，比如在前面的 pci_default_write_config 中已经更新了config中BAR0的base address为0xfebc0000，于是更新 io_regions[0].address 为0xfebc0000。然后其对应的 MemoryRegion (e1000-mmio) 作为offset添加到父级 MemoryRegion (pci) 中。触发address space的listener，最后更新flatview，通过ioctl更新到KVM。



#### 小结
PCI设备通过自己的配置空间来和上层的BIOS和OS进行信息交换，实际在硬件实现上配置空间的每个字段对应一个个寄存器或ROM。

BIOS/OS在启动时，会执行设备发现逻辑，在发现到当前设备时，需要读取配置空间获取相关信息，于是发生VMExit到KVM，再到QEMU，QEMU会调用设备初始化时设置的 config_read 读取设备信息。读取完毕后，会对BAR写入全1检测大小，然后分配base address填入。填入后修改command位，允许响应Memory Space和I/O Space的访问。

于是触发mapping的更新，将对应的IO MemoryRegion 设置到 KVM 中。此后OS通过PIO/MMIO和设备进行交互。




### 映射区域长度监测

BIOS/OS在为BAR分配地址时，需要知道BAR所需的长度。

根据PCI规范，BIOS/OS往对应的BAR寄存器中写全1，然后读出，实现对BAR对应的size。比如size为4k，则写入 0xffffffff 后，会读得 0xfffff00X ，最低一位(4bit)为X表示它是固定的，不会因写入f而改变，因为前文提到过，PCI规范规定，最低4个bit都是存放META信息的，因此被设备制造商写死，其他人无法修改：

```
For Memory BARs
0     Region Type       0 = Memory
2-1   Locatable         0 = any 32-bit  /  1 = < 1 MiB  /  2 = any 64-bit
3     Prefetchable      0 = no  /  1 = yes
31-4  Base Address      16-byte aligned

For I/O BARs (Deprecated)
0     Region Type       1 = I/O (deprecated)
1     Reserved
31-2  Base Address      4-byte aligned
```

于是BIOS/OS在读到 0xffffff00X 后，将X对应的4个bit设置为0，得到 0xfffff000 ，然后进行取反，得到 0x00000fff ，再加1，得到 0x00001000 ，因此得出大小即为 0x1000 ，即4kb

至于写入 0xffffffff 为什么会读到 0xfffff00X ，这是由硬件特性保证的。那么在QEMU中，模拟的设备是怎么模拟这样的效果呢？

实际上，这是通过mask来实现的。影响BAR的值有两个mask，分别为 wmask 和 w1cmask 。在设备初始化过程中，调用了 pci_qdev_realize => do_pci_register_device 对它们进行初始化：

```c
pci_config_alloc(pci_dev);
pci_init_wmask(pci_dev);
pci_init_w1cmask(pci_dev);
```

前者为 wmask 和 w1cmask 分配了一片 config 大小的内存，对于e1000这个PCI设备来说是256byte。这说明 wmask 和 w1cmask 将覆盖整个配置空间，负责对整个配置空间的写入作mask。

后者对 wmask 和 w1cmask 的一些byte进行初始化，保证对应位置为1，从而避免config在写入时被mask屏蔽掉。但在此时，BAR0-BAR6对应的位置依然是0。

直到在注册BAR的函数 pci_register_bar 中，设置了 `uint64_t wmask = ~(size - 1)` ，并在最后，如果BAR长度为32bit，则执行 ：

```
pci_set_long(pci_dev->wmask + addr, wmask & 0xffffffff);
```

比如对于e1000的BAR0，type为0，size为 PNPMMIO_SIZE(0x20000)，于是局部变量 wmask 为 0xfffffffffffe0000 ，而由于BAR0的长度只有32bit，因此要和 0xffffffff 与一下，得到 0xfffe0000 。于是 wmask[16:19] 为 00 00 fe ff ，即 0xfffe0000 。

于是当 BIOS/OS 对BAR0写入 0xffffffff 时，最终调用到 pci_default_write_config ：

```c
void pci_default_write_config(PCIDevice *d, uint32_t addr, uint32_t val_in, int l)
{
    int i, was_irq_disabled = pci_irq_disabled(d);
    uint32_t val = val_in;

    for (i = 0; i < l; val >>= 8, ++i) {
        uint8_t wmask = d->wmask[addr + i];
        uint8_t w1cmask = d->w1cmask[addr + i];
        assert(!(wmask & w1cmask));
        d->config[addr + i] = (d->config[addr + i] & ~wmask) | (val & wmask);
        d->config[addr + i] &= ~(val & w1cmask); /* W1C: Write 1 to Clear */
    }
    if (ranges_overlap(addr, l, PCI_BASE_ADDRESS_0, 24) ||
        ranges_overlap(addr, l, PCI_ROM_ADDRESS, 4) ||
        ranges_overlap(addr, l, PCI_ROM_ADDRESS1, 4) ||
        range_covers_byte(addr, l, PCI_COMMAND))
        pci_update_mappings(d);

    if (range_covers_byte(addr, l, PCI_COMMAND)) {
        pci_update_irq_disabled(d, was_irq_disabled);
        memory_region_set_enabled(&d->bus_master_enable_region,
                                  pci_get_word(d->config + PCI_COMMAND)
                                    & PCI_COMMAND_MASTER);
    }

    msi_write_config(d, addr, val_in, l);
    msix_write_config(d, addr, val_in, l);
}
```

此时addr为16，长度l为4。于是循环4次，一次设置一个byte。对于每个byte，会取出 wmask 和 w1cmask 在对应位置的值，于是：

1. i=0，wmask = 0 ， w1cmask = 0，于是 config[16] = (0 & 0xffff) | (0xffffffff & 0) = 0
                                    config[16] &= ~(0xffffffff & 0) = 0
2. i=1，wmask = 0 ， w1cmask = 0，于是 config[17] = (0 & 0xffff) | (0xffffffff & 0) = 0
                                    config[17] &= ~(0xffffffff & 0) = 0
3. i=1，wmask = 0xfe ， w1cmask = 0，于是 config[18] = (0 & 0x1) | (0xffffffff & 0xfe) = 0xfe
                                        config[18] &= ~(0xffffffff & 0) = 0xfe
4. i=1，wmask = 0xff ， w1cmask = 0，于是 config[19] = (0 & 0) | (0xffffffff & 0xff) = 0xff
                                        config[19] &= ~(0xffffffff & 0) = 0xff

于是在经过mask后，要写入的 0xffffffff 实际上写入的是 0xfffe0000 。由于读取函数 pci_default_read_config 只是简单地做 memcpy ，因此BIOS/OS读到的值就是 0xfffe0000。

于是BIOS/OS在拿到后，设置最后1位为0后取反得到 0x0001ffff 。然后加1，得到 0x00020000 ，即 0x20000 ，符合BAR0设置的size。

同理对于BAR1，其size为 IOPORT_SIZE(0x40) ，wmask[16:19]为 c0 ff ff ff，即0xffffffc0 ，于是最后实际写入的是 0xffffffc1 ，于是BIOS/OS在读到后，设置最后1位为0后取反得到 0x0000003f 。然后加1，得到 0x00000040 ，即 0x40 ，符合BAR0设置的size。

在确定了BAR的size后，BIOS/OS需要为其分配base address，需要和BAR的size对齐。


### 小结

BIOS/OS通过write and read的手段发现BAR的长度，然后为其分配base address。而QEMU是通过mask的机制实现了对BAR最大值的限制，模拟了硬件上的实现。





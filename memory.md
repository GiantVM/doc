## 基础

### 页表
负责将 VA 转换为 PA。VA 的地址由页号和页内偏移量组成，转换时，先从页表的基地址寄存器 (CR3) 中读取页表的起始地址，然后加上页号得到对应页的页表项。从中取出页的物理地址，再加上偏移量得到 PA。

随着寻址范围的扩大 (64 位 CPU 支持 48 位的虚拟地址寻址空间，和 52 位的物理地址寻址空间)，页表需要占用越来越多连续的内存空间，再加上每个进程都要有自己的页表，系统光是维护页表就需要耗费大量内存。为此，利用程序使用内存的局部化特征，引进了多级页表。

目前版本的 Linux 使用了四级页表：

Page Map Level 4(PML4) => Page Directory Pointer Table(PDPT) => Page Directory(PD) => Page Table(PT)

在某些地方被称为： Page Global Directory(PGD) => Page Upper Directory(PUD) => Page Middle Directory(PMD) => Page Table(PT)

在 x86_64 下，一个普通 page 的大小为 4KB，由于地址为 64bit，因此一个页表项占 8 Byte，于是一张页表中只能存放 512 个表项。因此每级页表索引使用 9 个 bit，加上页内索引 (offset) 使用 12 个 bit，因此一个 64bit 地址中只有 0-47bit 被用到。

在 64 位下，EPT 采用了和传统页表相同的结构，于是如果不考虑 TLB，进行一次 GVA 到 HVA 需要经过 4 * 4 次 (考虑访问每一级 page 都 fault 的情况) 页表查询。

有多少次查询就要访问多少次内存，在 walk 过程中不断对内存进行访问无疑会对性能造成影响。为此引入 TLB(Translation Lookaside Buffer) ，用来缓存常用的 PTE。这样在 TLB 命中的情况下就无需到内存去进行查找了。利用程序使用内存的局部化特征，TLB 的命中率往往很高，改善了在多级页表下的的访问速度。




### 内存虚拟化
QEMU 利用 mmap 系统调用，在进程的虚拟地址空间中申请连续的大小的空间，作为 Guest 的物理内存。

在这样的架构下，内存地址访问有四层映射：

GVA - GPA - HVA - HPA

GVA - GPA 的映射由 guest OS 负责维护，而 HVA - HPA 由 host OS 负责维护。于是我们需要一种机制，来维护 GPA - HVA 的映射。常用的实现有 SPT(Shadow Page Table) 和 EPT/NPT ，前者通过软件维护影子页表，后者通过硬件特性实现二级映射。


### 影子页表
KVM 通过维护 GVA 到 HPA 的页表 SPT ，实现了直接映射。于是页表可被物理 MMU 寻址使用。如何实现的呢：

KVM 将 Guest OS 的页表设置为 read-only ，当 Guest OS 进行修改时会触发 page fault， VMEXIT 到 KVM 。 KVM 会对 GVA 对应的页表项进行访问权限检查，结合错误码进行判断:

1. 如果是由 Guest OS 引起的，则将该异常注入回去。 Guest OS 调用自己的 page fault 处理函数 (申请一个 page ，将 page 的 GPA 填充到 上级页表项中)
2. 如果是 Guest OS 的页表和 SPT 不一致引起的，则同步 SPT ，根据 Guest OS 页表和 mmap 映射找到 GVA 到 GPA 再到 HVA 的映射关系，然后在 SPT 中增加 / 更新 GVA - HPA 的表项

当 Guest OS 切换进程时，会把待切换进程的页表基址载入 Guest 的 CR3，导致 VM EXIT 回到 KVM。KVM 通过哈希表找到对应的 SPT ，然后加载机器的 CR3 中。

缺点：需要为每个进程都维护一张 SPT ，带来额外的内存开销。需要保持 Guest OS 页表和 SPT 的同步。每当 Guest 发生 page fault ，即使是 guest 自身缺页导致的，都会导致 VMExit ，开销大。


### EPT / NPT
Intel EPT 技术 引入了 EPT(Extended Page Table) 和 EPTP(EPT base pointer) 的概念。 EPT 中维护着 GPA 到 HPA 的映射，而 EPT base pointer 负责指向 EPT 。在 Guest OS 运行时，该 VM 对应的 EPT 地址被加载到 EPTP ，而 Guest OS 当前运行的进程页表基址被加载到 CR3 ，于是在进行地址转换时，首先通过 CR3 指向的页表实现 GVA 到 GPA 的转换，再通过 EPTP 指向的 EPT 实现从 GPA 到 HPA 的转换。

在发生 EPT page fault 时，需要 VMExit 到 KVM，更新 EPT 。

AMD NPT(Nested Page Table) 是 AMD 搞出的解决方案，它原理和 EPT 类似，但描述和实现上略有不同。Guest OS 和 Host 都有自己的 CR3 。当进行地址转换时，根据 gCR3 指向的页表从 GVA 到 GPA ，然后根据 nCR3 指向的页表从 GPA 到 HPA 。

优点：Guest 的缺页在 guest 内处理，不会 vm exit。地址转换基本由硬件 (MMU) 查页表完成。

缺点：两级页表查询，只能寄望于 TLB 命中。




## 实现

### QEMU

#### 内存设备模拟

##### PCDIMMDevice

```c
typedef struct PCDIMMDevice {
    /* private */
    DeviceState parent_obj;

    /* public */
    uint64_t addr;                  // 映射到的起始 GPA
    uint32_t node;                  // 映射到的 numa 节点
    int32_t slot;                   // 插入的内存槽编号，默认为 -1，表示自动分配
    HostMemoryBackend *hostmem;     // 对应的 backend
} PCDIMMDevice;
```

通过 QOM(qemu object model) 定义的虚拟内存条。可通过 QMP 或 QEMU 命令行进行管理。通过增加 / 移除该对象实现 VM 中内存的热插拔。


##### HostMemoryBackend

```c
struct HostMemoryBackend {
    /* private */
    Object parent;

    /* protected */
    uint64_t size;                                  // 提供内存大小
    bool merge, dump;
    bool prealloc, force_prealloc, is_mapped;
    DECLARE_BITMAP(host_nodes, MAX_NODES + 1);
    HostMemPolicy policy;

    MemoryRegion mr;                                // 拥有的 MemoryRegion
};
```

通过 QOM 定义的一段 Host 内存，为虚拟内存条提供内存。可通过 QMP 或 QEMU 命令行进行管理。


#### 内存初始化

在开启 KVM 的前提下， QEMU 通过以下流程初始化内存：


```
main => configure_accelerator => kvm_init => kvm_memory_listener_register(s, &s->memory_listener, &address_space_memory, 0) 初始化
kvm_state.memory_listener
                                          => kml->listener.region_add = kvm_region_add                      为 listener 设置操作
                                          => memory_listener_register                                       初始化 listener 并绑定到 address_space_memory
                                          => memory_listener_register(&kvm_io_listener, &address_space_io)  初始化 kvm_io_listener 并绑定到 address_space_io
     => cpu_exec_init_all => memory_map_init                                        创建 system_memory("system") 和 system_io("io") 两个全局 MemoryRegion
                                 => address_space_init                              初始化 address_space_memory("memory") 和 address_space_io("I/O") AddressSpace，并把 system_memory 和 system_io 作为 root
                                    => memory_region_transaction_commit             提交修改，引起地址空间的变化
```

在进行进一步分析之前，我们先介绍下涉及的三种结构： AddressSpace 、 MemoryRegion 和 MemoryRegionSection ：

#### AddressSpace

```c
struct AddressSpace {
    /* All fields are private. */
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root;
    int ref_count;
    bool malloced;

    /* Accessed via RCU.  */
    struct FlatView *current_map;                               // 指向当前维护的 FlatView，在 address_space_update_topology 时作为 old 比较

    int ioeventfd_nb;
    struct MemoryRegionIoeventfd *ioeventfds;
    struct AddressSpaceDispatch *dispatch;                      // 负责根据 GPA 找到 HVA
    struct AddressSpaceDispatch *next_dispatch;
    MemoryListener dispatch_listener;
    QTAILQ_HEAD(memory_listeners_as, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};
```

顾名思义，用来表示虚拟机的一片地址空间，如内存地址空间，IO 地址空间。每个 AddressSpace 一般包含一系列 MemoryRegion ： AddressSpace 的 root 指向根级 MemoryRegion ，该 MemoryRegion 有可能有自己的若干个 subregion ，于是形成树状结构。

如上文所述，在内存初始化流程中调用了 memory_map_init ，其初始化了 address_space_memory 和 address_space_io ，其中：

* address_space_memory 的 root 为 system_memory
* address_space_io 的 root 为 system_io



#### MemoryRegion

```c
struct MemoryRegion {
    Object parent_obj;                                                  // 继承自 Object

    /* All fields are private - violators will be prosecuted */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool rom_device;                                                    // 是否只读
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;                                             // dirty map 类型
    RAMBlock *ram_block;                                                // 指向对应的 RAMBlock
    Object *owner;
    const MemoryRegionIOMMUOps *iommu_ops;

    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;                                            // 指向父 MemoryRegion
    Int128 size;                                                        // 内存区域大小
    hwaddr addr;                                                        // 在父 MemoryRegion 中的偏移量 (见 memory_region_add_subregion_common)
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;                                                // 指向实体 MemoryRegion
    hwaddr alias_offset;                                                // 起始地址 (GPA) 在实体 MemoryRegion 中的偏移量
    int32_t priority;
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;                   // subregion 链表
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
    QLIST_HEAD(, IOMMUNotifier) iommu_notify;
    IOMMUNotifierFlag iommu_notify_flags;
};
```

MemoryRegion 表示在 Guest memory layout 中的一段内存，具有逻辑 (Guest) 意义。

在初始化 VM 的过程中，建立了相应的 MemoryRegion ：

```
pc_init1 / pc_q35_init => pc_memory_init => memory_region_allocate_system_memory                        初始化 MemoryRegion 并为其分配内存
                                         => memory_region_init_alias => memory_region_init              初始化 alias MemoryRegion
                                         => memory_region_init                                          初始化 MemoryRegion
                                         => memory_region_init_ram => memory_region_init                初始化 MemoryRegion 并分配 Ramblock
```


##### memory_region_allocate_system_memory

对于非 NUMA 架构的 VM ，直接分配内存

```
=> allocate_system_memory_nonnuma => memory_region_init_ram_from_file / memory_region_init_ram          分配 MemoryRegion 对应 Ramblock 的内存
=> vmstate_register_ram                                                                                 根据 region 的名称 name 设置 RAMBlock 的 idstr
```

对于 NUMA，分配后需要设置 HostMemoryBackend

```
=> memory_region_init
=> memory_region_add_subregion                          遍历所有 NUMA 节点的内存 HostMemoryBackend ，依次把那些 mr 成员不为空的作为当前 MemoryRegion 的 subregion，偏移量从 0 开始递增
=> vmstate_register_ram_global => vmstate_register_ram  根据 region 的名称 name 设置 RAMBlock 的 idstr
```

##### MemoryRegion 类型

可将 MemoryRegion 划分为以下三种类型：

* 根级 MemoryRegion: 直接通过 memory_region_init 初始化，没有自己的内存，用于管理 subregion。如 system_memory
* 实体 MemoryRegion: 通过 memory_region_init_ram 初始化，有自己的内存 (从 QEMU 进程地址空间中分配)，大小为 size 。如 ram_memory(pc.ram) 、 pci_memory(pci) 等
* 别名 MemoryRegion: 通过 memory_region_init_alias 初始化，没有自己的内存，表示实体 MemoryRegion(如 pc.ram) 的一部分，通过 alias 成员指向实体 MemoryRegion，alias_offset 为在实体 MemoryRegion 中的偏移量。如 ram_below_4g 、ram_above_4g 等

代码中常见的 MemoryRegion 关系为：

```
                  alias
ram_memory (pc.ram) - ram_below_4g(ram-below-4g)
                    - ram_above_4g(ram-above-4g)

             alias
system_io(io) - (pci0-io)
              - (isa_mmio)
              - (isa-io)
              - ...

                     sub
system_memory(system) - ram_below_4g(ram-below-4g)
                      - ram_above_4g(ram-above-4g)
                      - pcms->hotplug_memory.mr        热插拔内存

          sub
rom_memory - isa_bios(isa-bios)
           - option_rom_mr(pc.rom)

```

同时将 AddressSpace 映射到 FlatView ，得到若干个 MemoryRegionSection ，调用 kvm_region_add ，将 MemoryRegionSection 注册到 KVM 中。


##### MemoryRegionSection

```c
struct MemoryRegionSection {
    MemoryRegion *mr;                           // 指向所属 MemoryRegion
    AddressSpace *address_space;                // 所属 AddressSpace
    hwaddr offset_within_region;                // 起始地址 (HVA) 在 MemoryRegion 内的偏移量
    Int128 size;
    hwaddr offset_within_address_space;         // 在 AddressSpace 内的偏移量，如果该 AddressSpace 为系统内存，则为 GPA 起始地址
    bool readonly;
};
```

MemoryRegionSection 指向 MemoryRegion 的一部分 ([offset_within_region, offset_within_region + size])，是注册到 KVM 的基本单位。

将 AddressSpace 中的 MemoryRegion 映射到线性地址空间后，由于重叠的关系，原本完整的 region 可能会被切分成片段，于是产生了 MemoryRegionSection。

回头再看内存初始化的流程，做的工作很简单：创建一些 AddressSpace ，绑定 listener 。创建相应的 MemoryRegion，作为 AddressSpace 的根。最后提交修改，让地址空间的发生变化，更新到 KVM 中。下面将分点介绍。



##### KVMMemoryListener

在初始化过程中，我们为 address_space_memory 和 address_space_io 分别注册了 memory_listener 和 kvm_io_listener 。前者类型为 KVMMemoryListener ，后者类型为 MemoryListener：

```c
typedef struct KVMMemoryListener {
    MemoryListener listener;
    KVMSlot *slots;
    int as_id;
} KVMMemoryListener;

struct MemoryListener {void (*begin)(MemoryListener *listener);
    void (*commit)(MemoryListener *listener);
    void (*region_add)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_del)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_nop)(MemoryListener *listener, MemoryRegionSection *section);
    void (*log_start)(MemoryListener *listener, MemoryRegionSection *section,
                      int old, int new);
    void (*log_stop)(MemoryListener *listener, MemoryRegionSection *section,
                     int old, int new);
    void (*log_sync)(MemoryListener *listener, MemoryRegionSection *section);
    void (*log_global_start)(MemoryListener *listener);
    void (*log_global_stop)(MemoryListener *listener);
    void (*eventfd_add)(MemoryListener *listener, MemoryRegionSection *section,
                        bool match_data, uint64_t data, EventNotifier *e);
    void (*eventfd_del)(MemoryListener *listener, MemoryRegionSection *section,
                        bool match_data, uint64_t data, EventNotifier *e);
    void (*coalesced_mmio_add)(MemoryListener *listener, MemoryRegionSection *section,
                               hwaddr addr, hwaddr len);
    void (*coalesced_mmio_del)(MemoryListener *listener, MemoryRegionSection *section,
                               hwaddr addr, hwaddr len);
    /* Lower = earlier (during add), later (during del) */
    unsigned priority;
    AddressSpace *address_space;
    QTAILQ_ENTRY(MemoryListener) link;
    QTAILQ_ENTRY(MemoryListener) link_as;
};
```

可以看到 KVMMemoryListener 主体就是 MemoryListener ，而 MemoryListener 包含大量函数指针，用来指向 address_space 成员发生变化时调用的回调函数。

address_space_io 上绑有 kvm_io_listener 和 dispatch_listener 。因此 AddressSpace 和 listener 存在一对多的关系，当 AddressSpace 发生变化时，其绑定的所有 listener 都会被触发。这是如何实现的呢？

实际上，任何对 AddressSpace 和 MemoryRegion 的操作，都以 memory_region_transaction_begin 开头，以 memory_region_transaction_commit 结尾。

这些操作包括：启用、析构、增删 eventfd、增删 subregion、改变属性 (flag)、设置大小、开启 dirty log 等，如：

* memory_region_add_subregion
* memory_region_del_subregion
* memory_region_set_readonly
* memory_region_set_enabled
* memory_region_set_size
* memory_region_set_address
* memory_region_set_alias_offset
* memory_region_readd_subregion
* memory_region_update_container_subregions
* memory_region_set_log
* memory_region_finalize
* ...

对 AddressSpace 的 root MemoryRegion 进行操作：

* address_space_init
* address_space_destroy

##### memory_region_transaction_begin

```
=> qemu_flush_coalesced_mmio_buffer => kvm_flush_coalesced_mmio_buffer
=> ++memory_region_transaction_depth
```

KVM 中对某些 MMIO 做了 batch 优化：KVM 遇到 MMIO 而 VMEXIT 时，将 MMIO 操作记录到 kvm_coalesced_mmio 结构中，然后塞到 kvm_coalesced_mmio_ring 中，不退出到 QEMU 。直到某一次退回到 QEMU ，要更新内存空间之前的那一刻，把 kvm_coalesced_mmio_ring 中的 kvm_coalesced_mmio 取出来做一遍，保证内存的一致性。这事就是 kvm_flush_coalesced_mmio_buffer 干的。


##### memory_region_transaction_commit

```
=> --memory_region_transaction_depth
=> 如果 memory_region_transaction_depth 为 0 且 memory_region_update_pending 大于 0
    => MEMORY_LISTENER_CALL_GLOBAL(begin, Forward)        从前向后调用全局列表 memory_listeners 中所有 listener 的 begin 函数
    => 对 address_spaces 中的所有 address space，调用 address_space_update_topology ，更新 QEMU 和 KVM 中维护的 slot 信息。
    => MEMORY_LISTENER_CALL_GLOBAL(commit, Forward)       从后向前调用全局列表 memory_listeners 中所有 listener 的 commit 函数
```

调用 listener 对应的函数来实现对地址空间的更新。

##### address_space_update_topology

```
=> address_space_get_flatview                             获取原来 FlatView(AddressSpace.current_map)
=> generate_memory_topology                               生成新的 FlatView
=> address_space_update_topology_pass                     比较新老 FlatView，对其中不一致的 FlatRange，执行相应的操作。
```

由于 AddressSpace 是树状结构，于是调用 address_space_update_topology ，使用 FlatView 模型将树状结构映射 (压平) 到线性地址空间。比较新老 FlatView，对其中不一致的 FlatRange，执行相应的操作，最终操作的 KVM。

##### generate_memory_topology

```
=> addrrange_make                   创建起始地址为 0，结束地址为 2^64 的地址空间，作为 guest 的线性地址空间
=> render_memory_region             从根级 region 开始，递归将 region 映射到线性地址空间中，产生一个个 FlatRange，构成 FlatView
=> flatview_simplify                将 FlatView 中连续的 FlatRange 进行合并为一个
```

AddressSpace 的 root 成员是该地址空间的根级 MemoryRegion ，generate_memory_topology 负责将它的树状结构进行压平，从而能够映射到一个线性地址空间，得到 FlatView 。

##### address_space_update_topology_pass

比较该 AddressSpace 的新老 FlatRange 是否有变化，如果有，从前到后或从后到前遍历 AddressSpace 的 listeners，调用对应 callback 函数。

```
=> MEMORY_LISTENER_UPDATE_REGION => section_from_flat_range      根据 FlatRange 的范围构造 MemoryRegionSection
                                 => MEMORY_LISTENER_CALL
```

举个例子，前面提到过，在初始化流程中，注册了 kvm_state.memory_listener 作为 address_space_memory 的 listener，它会被加入到 AddressSpace 的 listeners 中。于是如果 address_space_memory 发生了变化，则调用会调用 memory_listener 中相应的函数。

例如 MEMORY_LISTENER_UPDATE_REGION 传入的 callback 参数为 region_add ，则调用 memory_listener.region_add (kvm_region_add)。



##### kvm_region_add

```
=> kvm_set_phys_mem => kvm_lookup_overlapping_slot
                    => 计算起始 HVA
                    => kvm_set_user_memory_region => kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem)
```

kvm_lookup_overlapping_slot 用于判断新的 region section 的地址范围 (GPA) 是否与已有 KVMSlot(kml->slots) 有重叠，如果重叠了，需要进行处理：

假设原 slot 可以切分成三个部分：prefix slot + overlap slot + suffix slot，重叠区域为 overlap

对于完全重叠的情况，既有 prefix slot 又有 suffix slot。无需注册新 slot。

对于部分重叠的情况，prefix slot = 0 或 suffix slot = 0。则执行以下流程：

1. 删除原有 slot
2. 注册 prefix slot 或 suffix slot
3. 注册 overlap slot

当然如果没有重叠，则直接注册新 slot 即可。然后将 slot 通过 kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem) 更新 KVM 中对应的 kvm_memory_slot 。

QEMU 中维护 slot 结构也需要更新，对于原有的 slot，因为它是 kml->slots 数组的项，所以在 kvm_set_phys_mem 直接修改即可。对于 kml->slots 中没有的 slot，如 prefix、suffix、overlap，则需要调用 kvm_alloc_slot => kvm_get_free_slot ，它会在 kml->slots 找一个空白的 (memory_size = 0) 为 slot 返回，然后对该 slot 进行设置。

##### kvm_set_phys_mem => kvm_set_user_memory_region

KVM 规定了更新 memory slot 的参数为 kvm_userspace_memory_region ：

```c
struct kvm_userspace_memory_region {
    __u32 slot;                                                             // 对应 kvm_memory_slot 的 id
    __u32 flags;
    __u64 guest_phys_addr;                                                  // GPA
    __u64 memory_size; /* bytes */                                          // 大小
    __u64 userspace_addr; /* start of the userspace allocated memory */     // HVA
};
```

它会在 kvm_set_phys_mem => kvm_set_user_memory_region 的过程中进行计算并填充，流程如下：

1. 根据 region 的起始 HVA(memory_region_get_ram_ptr) + region section 在 region 中的偏移量 (offset_within_region) + 页对齐修正 (delta) 得到 section 真正的起始 HVA，填入 userspace_addr

    在 memory_region_get_ram_ptr 中，如果当前 region 是另一个 region 的 alias，则会向上追溯，一直追溯到非 alias region(实体 region) 为止。将追溯过程中的 alias_offset 加起来，可以得到当前 region 在实体 region 中的偏移量。

    由于实体 region 具有对应的 RAMBlock，所以调用 qemu_map_ram_ptr ，将实体 region 对应的 RAMBlock 的 host 和总 offset 加起来，得到当前 region 的起始 HVA。

2. 根据 region section 在 AddressSpace 内的偏移量 (offset_within_address_space) + 页对齐修正 (delta) 得到 section 真正的 GPA，填入 start_addr

3. 根据 region section 的大小 (size) - 页对齐修正 (delta) 得到 section 真正的大小，填入 memory_size





### RAMBlock

前面提到，MemoryRegion 表示在 guest memory layout 中的一段内存，具有逻辑意义。那么实际意义，也是就是这段内存所对应的实际内存信息是由谁维护的呢？

我们可以发现在 MemoryRegion 有一个 ram_block 成员，它是一个 RAMBlock 类型的指针，由 RAMBlock 来负责维护实际的内存信息，如 HVA、GPA。比如在刚刚计算 userspace_addr 的流程中，计算 region 的起始 HVA 需要找到对应的 RAMBlock ，然后获取其 host 成员来得到。

RAMBlock 定义如下：

```c
struct RAMBlock {
    struct rcu_head rcu;                                        // 用于保护 Read-Copy-Update
    struct MemoryRegion *mr;                                    // 对应的 MemoryRegion
    uint8_t *host;                                              // 对应的 HVA
    ram_addr_t offset;                                          // 在 ram_list 地址空间中的偏移 (要把前面 block 的 size 都加起来)
    ram_addr_t used_length;                                     // 当前使用的长度
    ram_addr_t max_length;                                      // 总长度
    void (*resized)(const char*, uint64_t length, void *host);  // resize 函数
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];                                            // id
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;                                 // 指向在 ram_list.blocks 中的下一个 block
    int fd;                                                     // 映射文件的文件描述符
    size_t page_size;                                           // page 大小，一般和 host 保持一致
};
```

前文提到过， MemoryRegion 会调用 memory_region_* 对 MemoryRegion 结构进行初始化。常见的函数有以下几个：

* memory_region_init_ram => qemu_ram_alloc
    通过 qemu_ram_alloc 创建的 RAMBlock.host 为 NULL

* memory_region_init_ram_from_file => qemu_ram_alloc_from_file
    通过 qemu_ram_alloc_from_file 创建的 RAMBlock 会调用 file_ram_alloc 使用对应路径的 (设备) 文件来分配内存，通常是由于需要使用 hugepage，会通过 `-mem-path` 参数指定了 hugepage 的设备文件 (如 /dev/hugepages)

* memory_region_init_ram_ptr => qemu_ram_alloc_from_ptr
    RAMBlock.host 为传入的指针地址，表示从该地址指向的内存分配内存

* memory_region_init_resizeable_ram => qemu_ram_alloc_resizeable
    RAMBlock.host 为 NULL，但 resizeable 为 true，表示还没有分配内存，但可以 resize。


qemu_ram_alloc_* (qemu_ram_alloc / qemu_ram_alloc_from_file / memory_region_init_ram_ptr / memory_region_init_resizeable_ram) 最后都会调用到  qemu_ram_alloc_internal => ram_block_add 。它如果发现 host 为 NULL ，则会调用 phys_mem_alloc (qemu_anon_ram_alloc) 分配内存。让 host 有所指向后，将该 RAMBlock 插入到 ram_list.blocks 中。


##### qemu_anon_ram_alloc

=> qemu_ram_mmap(-1, size, QEMU_VMALLOC_ALIGN, false) => mmap

通过 mmap 在 QEMU 的进程地址空间中分配 size 大小的内存。





### RAMList

ram_list 是一个全局变量，以链表的形式维护了所有的 RAMBlock 。

```c
RAMList ram_list = {.blocks = QLIST_HEAD_INITIALIZER(ram_list.blocks) };

typedef struct RAMList {
    QemuMutex mutex;
    RAMBlock *mru_block;
    /* RCU-enabled, writes protected by the ramlist lock. */
    QLIST_HEAD(, RAMBlock) blocks;                              // RAMBlock 链表
    DirtyMemoryBlocks *dirty_memory[DIRTY_MEMORY_NUM];          // 记录脏页信息，用于 VGA / TCG / Live Migration
    uint32_t version;                                           // 每更改一次加 1
} RAMList;
extern RAMList ram_list;
```

注：

* VGA: 显卡仿真通过 dirty_memory 跟踪 dirty 的视频内存，用于重绘界面
* TCG: 动态翻译器通过 dirty_memory 追踪自调整的代码，当上游指令发生变化时对其重新编译
* Live Migration: 动态迁移通过 dirty_memory 来跟踪 dirty page，在 dirty page 被改变之后重传




##### AddressSpaceDispatch

根据：

```
address_space_init => address_space_init_dispatch => as->dispatch_listener = (MemoryListener) {
                                                                            .begin = mem_begin,
                                                                            .commit = mem_commit,
                                                                            .region_add = mem_add,
                                                                            .region_nop = mem_add,
                                                                            .priority = 0,
                                                                        };
                                                  => memory_listener_register(as->dispatch_listener)
```

address_space_memory 上除了绑有 kvm_state.memory_listener ，还会创建并绑定 dispatch_listener 。该 listener 实现了为了在虚拟机退出时根据 GPA 找到对应的 HVA 。

当 memory_region_transaction_commit 调用各个 listener 的 begin 函数时， mem_begin 被调用

```
=> g_new0(AddressSpaceDispatch, 1)                  创建 AddressSpaceDispatch 结构作为 AddressSpace 的 next_dispatch 成员
```

AddressSpaceDispatch 结构如下：

```c
struct AddressSpaceDispatch {
    struct rcu_head rcu;

    MemoryRegionSection *mru_section;
    /* This is a multi-level map on the physical address space.
     * The bottom level has pointers to MemoryRegionSections.
     */
    PhysPageEntry phys_map;
    PhysPageMap map;            // GPA -> HVA 的映射，通过多级页表实现
    AddressSpace *as;
};
```

map 成员是一个多级 (6 级) 页表，最后一级页表指向 MemoryRegionSection 。

当 address_space_update_topology_pass => address_space_update_topology_pass 处理 add 时， mem_add 被调用：

于是调用 register_subpage / register_multipage 将 page 注册到页表中。

```
=> 如果 MemoryRegionSection 所属的 MemoryRegion 的 subpage 不存在
    => subpage_init                                         创建 subpage
    => phys_page_set => phys_map_node_reserve               分配页目录项
                     => phys_page_set_level                 填充页表，从 L5 填到 L0
=> 如果存在
    => container_of(existing->mr, subpage_t, iomem)         取出
=> subpage_register                                         设置 subpage
```

因此从 KVM 中退出到 QEMU 之后，通过 AddressSpaceDispatch.map 可以找到对应的 MemoryRegionSection ，继而找到对应的 HVA



## KVM


### kvm_vm_ioctl_set_memory_region

添加内存。在 KVM 收到 KVM_SET_USER_MEMORY_REGION(取代了 KVM_SET_MEMORY_REGION ，因为其不支持细粒度控制) 的 ioctl 时调用。

传入参数如下：

```c
struct kvm_userspace_memory_region {
    __u32 slot;                                                             // 对应 kvm_memory_slot 的 id
    __u32 flags;
    __u64 guest_phys_addr;                                                  // GPA
    __u64 memory_size; /* bytes */                                          // 大小
    __u64 userspace_addr; /* start of the userspace allocated memory */     // HVA
};
```

flags 可选：

* KVM_MEM_LOG_DIRTY_PAGES 声明需要跟踪对该 Region 的写，提供给 KVM_GET_DIRTY_LOG 时读取
* KVM_MEM_READONLY        如果支持 readonly(KVM_CAP_READONLY_MEM)，则当写该 Region 时触发 VMEXIT (KVM_EXIT_MMIO)

于是 kvm_vm_ioctl_set_memory_region => kvm_set_memory_region => __kvm_set_memory_region

该函数将根据 npages(region 所包含的数树) 和原来的 npages 判断用户操作：

#### KVM_MR_CREATE
现在有页而原来没有，则为新增内存区域，创建并初始化 slot 。

#### KVM_MR_DELETE
现在没有页而原来有，则为删除内存区域，将 slot 标记为 KVM_MEMSLOT_INVALID

#### KVM_MR_FLAGS_ONLY / KVM_MR_MOVE
现在有页且原来也有，则为修改内存区域，如果只有 flag 变了，则为 KVM_MR_FLAGS_ONLY ，目前只有可能是 KVM_MEM_LOG_DIRTY_PAGES ，则根据 flag 选择是要创建还是释放 dirty_bitmap。

如果 GPA 有变，则为 KVM_MR_MOVE ，需要进行移动。其实就直接将原来的 slot 标记为 KVM_MEMSLOT_INVALID，然后添加新的。

新增 / 修改后的 slot 通过 install_new_memslots 更新。

#### kvm_memory_slot

在 __kvm_set_memory_region 操作的 slot 是 KVM 中内存管理中的基本单位，定义如下：

```c
struct kvm_memory_slot {
    gfn_t base_gfn;                     // slot 的起始 gfn
    unsigned long npages;               // page 数
    unsigned long *dirty_bitmap;        // 脏页 bitmap
    struct kvm_arch_memory_slot arch;   // 结构相关，包括 rmap 和 lpage_info 等
    unsigned long userspace_addr;       // 对应的起始 HVA
    u32 flags;
    short id;
};


struct kvm_arch_memory_slot {struct kvm_rmap_head *rmap[KVM_NR_PAGE_SIZES];              // 反向链接
    struct kvm_lpage_info *lpage_info[KVM_NR_PAGE_SIZES - 1];   // 维护下一级页表是否关闭 hugepage
    unsigned short *gfn_track[KVM_PAGE_TRACK_MAX];
};
```

slot 保存在 kvm->memslots[as_id]->memslots[id] 中，其中 as_id 为 address space id，其实通常的架构都只有一个地址空间，as_id 总是取 0，唯独x86需要两个地址空间，as_id = 0为普通的地址空间，as_id = 1为SMM模式专用的SRAM空间，id 为 slot id 。这些结构的内存都在 kvm_create_vm 中就分配好了。这里对其进行初始化。


### 内存管理单元 (MMU)

#### 初始化

```
kvm_init => kvm_arch_init => kvm_mmu_module_init => 建立 mmu_page_header_cache 作为 cache
                                                 => register_shrinker(&mmu_shrinker)                注册回收函数


kvm_vm_ioctl_create_vcpu =>
kvm_arch_vcpu_create => kvm_x86_ops->vcpu_create (vmx_create_vcpu) => init_rmode_identity_map       为实模式建立 1024 个页的等值映射
                                                                   => kvm_vcpu_init => kvm_arch_vcpu_init => kvm_mmu_create
kvm_arch_vcpu_setup => kvm_mmu_setup => init_kvm_mmu => init_kvm_tdp_mmu                            如果支持 two dimentional paging(EPT)，初始化之，设置 vcpu->arch.mmu 中的属性和函数
                                                     => init_kvm_softmmu => kvm_init_shadow_mmu     否则初始化 SPT
```


##### kvm_mmu_create

以 vcpu 为单位初始化 mmu 相关信息。它们在 vcpu 中的相关定义包含：

```c
struct kvm_vcpu_arch {
    ...
    /*
     * Paging state of the vcpu
     *
     * If the vcpu runs in guest mode with two level paging this still saves
     * the paging mode of the l1 guest. This context is always used to
     * handle faults.
     */
    struct kvm_mmu mmu;

    /*
     * Paging state of an L2 guest (used for nested npt)
     *
     * This context will save all necessary information to walk page tables
     * of the an L2 guest. This context is only initialized for page table
     * walking and not for faulting since we never handle l2 page faults on
     * the host.
     */
    struct kvm_mmu nested_mmu;

    /*
     * Pointer to the mmu context currently used for
     * gva_to_gpa translations.
     */
    struct kvm_mmu *walk_mmu;

    // 以下为 cache，用于提升常用数据结构的分配速度
    // 用于分配 pte_list_desc ，它是反向映射链表 parent_ptes 的链表项，在 mmu_set_spte => rmap_add => pte_list_add 中分配
    struct kvm_mmu_memory_cache mmu_pte_list_desc_cache;
    // 用于分配 page ，作为 kvm_mmu_page.spt
    struct kvm_mmu_memory_cache mmu_page_cache;
    // 用于分配 kvm_mmu_page ，作为页表页
    struct kvm_mmu_memory_cache mmu_page_header_cache;
    ...
}
```

其中 cache 用于提升页表中常用数据结构的分配速度。这些 cache 会在初始化 MMU(kvm_mmu_load)、发生 page fault(tdp_page_fault) 等情况下调用 mmu_topup_memory_caches 来保证各 cache 充足。

```c
// 保证各 cache 充足
static int mmu_topup_memory_caches(struct kvm_vcpu *vcpu)
{
    // r 不为 0 表示从 slab 分配 /__get_free_page 失败，直接返回错误
    int r;
    // 如果 vcpu->arch.mmu_pte_list_desc_cache 不足，从 pte_list_desc_cache 中分配
    r = mmu_topup_memory_cache(&vcpu->arch.mmu_pte_list_desc_cache,
                   pte_list_desc_cache, 8 + PTE_PREFETCH_NUM);
    if (r)
        goto out;
    // 如果 vcpu->arch.mmu_page_cache 不足，直接通过 __get_free_page 分配
    r = mmu_topup_memory_cache_page(&vcpu->arch.mmu_page_cache, 8);
    if (r)
        goto out;
    // 如果 vcpu->arch.mmu_page_header_cache 不足，从 mmu_page_header_cache 中分配
    r = mmu_topup_memory_cache(&vcpu->arch.mmu_page_header_cache,
                   mmu_page_header_cache, 4);
out:
    return r;
}
```

pte_list_desc_cache 和 mmu_page_header_cache 两块全局 slab cache 在 kvm_mmu_module_init 中被创建，作为 vcpu->arch.mmu_pte_list_desc_cache 和 vcpu->arch.mmu_page_header_cache 的 cache 来源。

可以在 host 通过 `cat /proc/slabinfo` 查看到分配的 slab：

```
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
kvm_mmu_page_header    576    576    168   48    2 : tunables    0    0    0 : slabdata     12     12      0
```



#### 加载页表

kvm_vm_ioctl_create_vcpu 仅仅是对 mmu 进行初始化，比如将 vcpu->arch.mmu.root_hpa 设置为 INVALID_PAGE ，直到要进入 VM(VMLAUNCH/VMRESUME) 前才真正设置该值。

```
vcpu_enter_guest => kvm_mmu_reload => kvm_mmu_load => mmu_topup_memory_caches                       保证各 cache 充足
                                                   => mmu_alloc_roots => mmu_alloc_direct_roots     如果根页表不存在，则分配一个 kvm_mmu_page
                                                   => vcpu->arch.mmu.set_cr3 (vmx_set_cr3)          对于 EPT，将该页的 spt(strcut page) 的 HPA 加载到 VMCS
                                                                                                    对于 SPT，将该页的 spt(strcut page) 的 HPA 加载到 cr3
                 => kvm_x86_ops->run (vmx_vcpu_run)
                 => kvm_x86_ops->handle_exit (vmx_handle_exit)
```


#### kvm_mmu_page

页表页，详细解释见 Documentation/virtual/kvm/mmu.txt

```c
struct kvm_mmu_page {
    struct list_head link;                          // 加到 kvm->arch.active_mmu_pages 或 invalid_list ，表示当前页处于的状态
    struct hlist_node hash_link;                    // 加到 vcpu->kvm->arch.mmu_page_hash ，提供快速查找

    /*
     * The following two entries are used to key the shadow page in the
     * hash table.
     */
    gfn_t gfn;                                      // 管理地址范围的起始地址对应的 gfn
    union kvm_mmu_page_role role;                   // 基本信息，包括硬件特性和所属层级等

    u64 *spt;                                       // 指向 struct page 的地址，其包含了所有页表项 (pte)。同时 page->private 会指向该 kvm_mmu_page
    /* hold the gfn of each spte inside spt */
    gfn_t *gfns;                                    // 所有页表项 (pte) 对应的 gfn
    bool unsync;                                    // 用于最后一级页表页，表示该页的页表项 (pte) 是否与 guest 同步 (guest 是否已更新 tlb)
    int root_count;          /* Currently serving as active root */ // 用于最高级页表页，统计有多少 EPTP 指向自身
    unsigned int unsync_children;                   // 页表页中 unsync 的 pte 数
    struct kvm_rmap_head parent_ptes; /* rmap pointers to parent sptes */ // 反向映射 (rmap)，维护指向自己的上级页表项

    /* The page is obsolete if mmu_valid_gen != kvm->arch.mmu_valid_gen.  */
    unsigned long mmu_valid_gen;                    // 代数，如果比 kvm->arch.mmu_valid_gen 小则表示已失效

    DECLARE_BITMAP(unsync_child_bitmap, 512);       // 页表页中 unsync 的 spte bitmap

#ifdef CONFIG_X86_32
    /*
     * Used out of the mmu-lock to avoid reading spte values while an
     * update is in progress; see the comments in __get_spte_lockless().
     */
    int clear_spte_count;                           // 32bit 下，对 spte 的修改是原子的，因此通过该计数来检测是否正在被修改，如果被改了需要 redo
#endif

    /* Number of writes since the last time traversal visited this page.  */
    atomic_t write_flooding_count;                  // 统计从上次使用以来的 emulation 次数，如果超过一定次数，会把该 page 给 unmap 掉
};

union kvm_mmu_page_role {
    unsigned word;
    struct {
        unsigned level:4;           // 页所处的层级
        unsigned cr4_pae:1;         // cr4.pae，1 表示使用 64bit gpte
        unsigned quadrant:2;        // 如果 cr4.pae=0，则 gpte 为 32bit，但 spte 为 64bit，因此需要用多个 spte 来表示一个 gpte，该字段指示是 gpte 的第几块
        unsigned direct:1;
        unsigned access:3;          // 访问权限
        unsigned invalid:1;         // 失效，一旦 unpin 就会被销毁
        unsigned nxe:1;             // efer.nxe
        unsigned cr0_wp:1;          // cr0.wp，写保护
        unsigned smep_andnot_wp:1;  // cr4.smep && !cr0.wp
        unsigned smap_andnot_wp:1;  // cr4.smap && !cr0.wp
        unsigned :8;

        /*
         * This is left at the top of the word so that
         * kvm_memslots_for_spte_role can extract it with a
         * simple shift.  While there is room, give it a whole
         * byte so it is also faster to load it from memory.
         */
        unsigned smm:8;             // 处于 system management mode
    };
};
```


#### EPT Violation

当 Guest 第一次访问某个页面时，由于没有 GVA 到 GPA 的映射，触发 Guest OS 的 page fault。于是 Guest OS 会建立对应的 pte 并修复好各级页表，最后访问对应的 GPA。由于没有建立 GPA 到 HVA 的映射，于是触发 EPT Violation，VMEXIT 到 KVM。 KVM 在 vmx_handle_exit 中执行 kvm_vmx_exit_handlers[exit_reason]，发现 exit_reason 是 EXIT_REASON_EPT_VIOLATION ，因此调用 handle_ept_violation 。

##### handle_ept_violation

```
=> vmcs_readl(EXIT_QUALIFICATION)                       获取 EPT 退出的原因。EXIT_QUALIFICATION 是 Exit reason 的补充，详见 Vol. 3C 27-9 Table 27-7
=> vmcs_read64(GUEST_PHYSICAL_ADDRESS)                  获取发生缺页的 GPA
=> 根据 exit_qualification 内容得到 error_code，可能是 read fault / write fault / fetch fault / ept page table is not present
=> kvm_mmu_page_fault => vcpu->arch.mmu.page_fault (tdp_page_fault)
```

##### tdp_page_fault

```
=> gfn = gpa >> PAGE_SHIFT      将 GPA 右移 pagesize 得到 gfn(guest frame number)
=> mapping_level                计算 gfn 在页表中所属 level，不考虑 hugepage 则为 L1
=> try_async_pf                 将 gfn 转换为 pfn(physical frame number)
        => kvm_vcpu_gfn_to_memslot => __gfn_to_memslot                  找到 gfn 对应的 slot
        => __gfn_to_pfn_memslot                                         找到 gfn 对应的 pfn
                => __gfn_to_hva_many => __gfn_to_hva_memslot            计算 gfn 对应的起始 HVA
                => hva_to_pfn                                           计算 HVA 对应的 pfn，同时确保该物理页在内存中

=> __direct_map                                                         更新 EPT，将新的映射关系逐层添加到 EPT 中
    => for_each_shadow_entry                                            从 level4(root) 开始，逐层补全页表，对于每一层：
        => mmu_set_spte                                                 对于 level1 的页表，其页表项肯定是缺的，所以不用判断直接填上 pfn 的起始 hpa
        => is_shadow_present_pte                                        如果下一级页表页不存在，即当前页表项没值 (*sptep = 0)
            => kvm_mmu_get_page                                         分配一个页表页结构
            => link_shadow_page                                         将新页表页的 HPA 填入到当前页表项 (sptep) 中
```

可以发现主要有两步，第一步会获取 GPA 所对应的物理页，如果没有会进行分配。第二步是更新 EPT。

##### try_async_pf
1. 根据 gfn 找到对应的 memslot
2. 用 memslot 的起始 HVA(userspace_addr) + (gfn - slot 中的起始 gfn(base_gfn) ) * 页大小 (PAGE_SIZE)，得到 gfn 对应的起始 HVA
3. 为该 HVA 分配一个物理页，有 hva_to_pfn_fast 和 hva_to_pfn_slow 两种， hva_to_pfn_fast 实际上是调用 __get_user_pages_fast ，会尝试去 pin 该 page，即确保该地址所在的物理页在内存中。如果失败，退化到 hva_to_pfn_slow ，会先去拿 mm->mmap_sem 的锁然后调用 __get_user_pages 来 pin。
4. 如果分配成功，对其返回的 struct page 调用 page_to_pfn 得到对应的 pfn

该函数建立了 gfn 到 pfn 的映射，同时将该 page pin 死在 host 的内存中。


##### __direct_map

通过迭代器 kvm_shadow_walk_iterator 将 EPT 中与该 GPA 相关的页表补充完整。

```c
struct kvm_shadow_walk_iterator {
    u64 addr;                   // 发生 page fault 的 GPA，迭代过程就是要把 GPA 所涉及的页表项都填上
    hpa_t shadow_addr;          // 当前页表项的 HPA，在 shadow_walk_init 中设置为 vcpu->arch.mmu.root_hpa
    u64 *sptep;                 // 指向当前页表项，在 shadow_walk_okay 中更新
    int level;                  // 当前层级，在 shadow_walk_init 中设置为 4 (x86_64 PT64_ROOT_LEVEL)，在 shadow_walk_next 中减 1
    unsigned index;             // 在当前 level 页表中的索引，在 shadow_walk_okay 中更新
};
```

在每轮迭代中，sptep 都会指向 GPA 在当前级页表中所对应的页表项，我们的目的就是把下一级页表的 GPA 填到该页表项内 (即设置 *sptep)。因为是缺页，可能会出现下一级的页表页不存在的问题，这时候需要分配一个页表页，然后再将该页的 GPA 填进到 *sptep 中。

举个例子，对于 GPA(如 0xfffff001)，其二进制为：

```
000000000 000000011 111111111 111111111 000000000001
  PML4      PDPT       PD        PT        Offset
```

初始化状态：level = 4，shadow_addr = root_hpa，addr = GPA

执行流程：

1. index = addr 在当前 level 分段的值。如在 level = 4 时为 0(000000000)，在 level = 3 时为 3(000000011)
2. sptep = va(shadow_addr) + index，得到 GPA 在当前地址中所对应的页表项 HVA
3. 如果 *sptep 没值，分配一个 page 作为下级页表，同时将 *sptep 设置为该 page 的 HPA
4. shadow_addr = *sptep，进入下级页表，循环

开启 hugepage 时，由于页表项管理的范围变大，所需页表级数减少，在默认情况下 page 大小为 2M，因此无需 level 1。



##### mmu_set_spte

```
=> set_spte => mmu_spte_update => mmu_spte_set => __set_spte                                设置物理页 (pfn) 起始 HPA 到 *sptep，即设置最后一级页表中某个 pte 的值
=> rmap_add => page_header(__pa(spte))                                                      获取 spetp 所在的页表页
            => kvm_mmu_page_set_gfn                                                         将 gfn 设置到该页表页的 gfns 中
            => gfn_to_rmap => __gfn_to_memslot                                              获取 gfn 对应的 slot
                           => __gfn_to_rmap => gfn_to_index                                 通过 gfn 和 slot->base_gfn，算出该页在 slot 中的 index
                                    => slot->arch.rmap[level - PT_PAGE_TABLE_LEVEL][idx]    从该 slot 中取出对应的 rmap
            => pte_list_add                                                                 将当前项 (spetp) 的地址加入到 rmap 中，做反向映射
```

作用于 1 级页表 (PT)。负责设置最后一级页表中的 pte(*spetp) 的值，同时将当前项 (spetp) 的地址加入到 slot->arch.rmap[level - PT_PAGE_TABLE_LEVEL][idx] 中作为反向映射，此后可以通过 gfn 快速找到该 kvm_mmu_page 。

在大多数情况下，gfn 对应单个 kvm_mmu_page，于是 rmap_head 直接指向 spetp 即可。但由于一个 gfn 对应多个 kvm_mmu_page，因此在该情况下 rmap 采用链表 + 数组来维护。一个链表项 pte_list_desc 能存放三个 spetp。由于 pte_list_desc 频繁被分配，因此也是从 cache (vcpu->arch.mmu_pte_list_desc_cache) 中分配的。


##### kvm_mmu_get_page

获取 gfn 对应的 kvm_mmu_page 。会通过 gfn 尝试从 vcpu->kvm->arch.mmu_page_hash 中找到对应的页表页，如果以前分配过该页则直接返回即可。否则需要通过 kvm_mmu_alloc_page 从 cache 中分配，然后以 gfn 为 key 将其加到 vcpu->kvm->arch.mmu_page_hash 中。

kvm_mmu_alloc_page 会通过 mmu_memory_cache_alloc 从 vcpu->arch.mmu_page_header_cache 和 vcpu->arch.mmu_page_cache 分配 kvm_mmu_page 和 page 对象，在 mmu_topup_memory_caches 中保证了这些 cache 的充足，如果发现余量不够，会通过全局变量的 slab 补充，这点前面也提到了。



##### link_shadow_page

```
=> mmu_spte_set => __set_spte                               为当前页表项的值 (*spetp) 设置下一级页表页的 HPA
=> mmu_page_add_parent_pte => pte_list_add                  将当前项的地址 (spetp) 加入到下一级页表页的 parent_ptes 中，做反向映射
```

作用于 2-4 级页表 (PML4 - PDT)，在遍历过程中如果发现下一级页表缺页，需要在分配一个页表页后更新当前的迭代器指向的页表项 (spetp)，设置为下一级该页表页的 HPA，这样下次就能够通过页表项访问到该页表页了。同时需要将当前页表项 (spetp) 的地址加入到下一级页表页的 parent_ptes 中作为反向映射。

利用两套反向映射，在右移 GPA 算出 gfn 后，可以通过 rmap 得到在 L1 中的页表项，然后通过 parent_ptes 可以依次得到在 L2-4 中的页表项。当 Host 需要将 Guest 的某个 GPA 的 page 换出时，直接通过反向索引找到该 gfn 相关的页表项进行修改，而无需再次走 EPT 查询。






## 总结

### QEMU
创建一系列 MemoryRegion ，分别表示 Guest 中的 ROM、RAM 等区域。 MemoryRegion 之间通过 alias 或 subregion 的方式维护相互之间的关系，从而进一步细化区域的定义。

对于一个实体 MemoryRegion(非 alias)，在初始化内存的过程中会创建它所对应的 RAMBlock 。 RAMBlock 通过 mmap 的方式从 QEMU 的进程空间中分配内存，并负责维护该 MemoryRegion 管理内存的起始 HVA/GPA/size 等信息。

AddressSpace 表示 VM 的物理地址空间。如果 AddressSpace 中的 MemoryRegion 发生变化，则 listener 被触发，将所属 AddressSpace 的 MemoryRegion 树展平，形成一维的 FlatView ，比较 FlatRange 是否发生了变化。如果是调用相应方法如 region_add 对变化的 section region 进行检查，更新 QEMU 内的 KVMSlot，同时填充 kvm_userspace_memory_region 结构，作为 ioctl 的参数更新 KVM 中的 kvm_memory_slot 。


### KVM
当 QEMU 通过 ioctl 创建 vcpu 时，调用 kvm_mmu_create 初始化 mmu 相关信息，为页表项结构分配 slab cache。

当 KVM 要进入 Guest 前， vcpu_enter_guest => kvm_mmu_reload 会将根级页表地址加载到 VMCS，让 Guest 使用该页表。

当 EPT Violation 发生时， VMEXIT 到 KVM 中。如果是缺页，则拿到对应的 GPA ，根据 GPA 算出 gfn，根据 gfn 找到对应的 memory slot ，得到对应的 HVA 。然后根据 HVA 找到对应的 pfn，确保该 page 位于内存。在把缺的页填上后，需要更新 EPT，完善其中缺少的页表项。于是从 L4 开始，逐层补全页表，对于在某层上缺少的页表页，会从 slab 中分配后将新页的 HPA 填入到上一级页表中。

除了建立上级页表到下级页表的关联外，KVM 还会建立反向映射，可以直接根据 GPA 找到 gfn 相关的页表项，而无需再次走 EPT 查询。


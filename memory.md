## 基础

### 页表
将VA转换为PA。VA的地址由页号和页内偏移量组成，转换时，先从页表的基地址寄存器(CR3)中读取页表的起始地址，将起始地址加上页号得到页查询，查询得到物理页地址。物理地址再加上偏移量得到PA。

随着寻址范围的扩大(64位CPU支持48位的虚拟地址寻址空间，和52位的物理地址寻址空间)，页表需要大量且连续的内存空间，同时每个进程都有自己的页表，系统光是维护页表就需要耗费大量内存。为此，利用程序使用内存的局部化特征，引进多级页表。Linux使用了四级页表：

Page Map Level 4(PML4) => Page Directory Pointer Table(PDPT) => Page Directory(PD) => Page Table(PT)
PGD PUD PMD PTE Offset

在x86_64下，一个普通page的大小为4KB，由于地址为64bit，因此一个页表项占8 Byte，于是一张页表中只能存放512个表项。因此每级页表索引使用9个bit，加上页内索引(offset)使用12个bit，因此一个64bit地址中只有0-47bit被用到。

在64位下，EPT采用了和传统页表相同的结构，于是如果不考虑TLB，进行一次GVA到HVA需要经过 ??? 4*4次页表查询。

有多少次查询就要访问多少次内存，在walk过程中不断对内存进行访问无疑会对性能造成影响。为此引入TLB(Translation Lookaside Buffer)，用来缓存常用的PTE。这样在TLB命中的情况下就无需到内存去进行查找了。利用程序使用内存的局部化特征，TLB的命中率往往很高，改善了在多级页表下的的访问速度。




### 内存虚拟化
QEMU利用mmap系统调用，在进程的虚拟地址空间中申请连续的大小的空间，映射为guest的物理内存。

因此内存有四层映射：

GVA - GPA - HVA - HPA

GVA - GPA 的映射由guest OS负责维护，而 HVA - HPA 由host OS负责维护，现需要一种机制维护 GPA - HVA 的映射。常用的实现有SPT(Shadow Page Table)和EPT/NPT，前者通过软件维护影子页表，后者通过硬件特性实现二级映射。


### 影子页表
KVM通过维护 GVA 到 HPA 的页表SPT，实现了直接映射。于是可以被物理MMU寻址使用。

guest OS的页表被设置为read-only，当guest OS进行修改时会触发page fault，VMEXIT到KVM。KVM会对GVA对应的页表项进行访问权限检查，结合错误码进行判断:如果是由guest OS引起的，则将该异常注入回去。客户机调用客户机自己的page_fault处理函数，申请一个page，将page的GPA填充到客户机页表项中。如果是guest OS的页表和SPT不一致引起的，则同步SPT，根据guest OS页表和mmap映射找到GVA到GPA再到HVA的映射关系，???然后在SPT中增加/更新 GVA - HVA 的表项。

??? 当guest OS切换进程时，会把待切换进程的页表基址载入CR3，触发VM EXIT到KVM，通过哈希表找到对应的SPT，然后加载到guest的CR3。

缺点：每个进程都有一张SPT，带来额外的内存开销。需要维护guest OS页表和SPT的同步。每当guest发送page fault都会VM exit(即使是guest自身缺页导致的)，开销大。




### EPT / NPT
Intel EPT(Extended Page Table)引入了EPT页表和EPTP(EPT base pointer)，EPT中维护着GPA到HPA的映射，而EPT base pointer负责指向EPT。在guest OS运行时，该VM对应的EPT地址被加载到EPTP，而guest OS当前运行的进程页表基址被加载到CR3，于是在进行地址转换时，通过CR3指向的页表从GVA到GPA，再通过EPTP指向的EPT从GPA到HPA。

在page fault时，更新 EPT。

AMD NPT(Nested Page Table)原理类似，但实现上略有不同。Guest OS和Host都有自己的CR3。当进行地址转换时，根据gCR3指向的页表从GVA到GPA，然后根据nCR3指向的页表从GPA到HPA。

优点：guest的缺页在guest内处理，不会vm exit。地址转换基本由硬件(MMU)查页表完成。
缺点：两级页表查询，只能寄望于TLB命中。




## 实现

### QEMU


#### PCDIMMDevice

```c
typedef struct PCDIMMDevice {
    /* private */
    DeviceState parent_obj;

    /* public */
    uint64_t addr;                  // 映射到的起始GPA
    uint32_t node;                  // 映射到的numa节点
    int32_t slot;                   // 插入的内存槽编号，默认为-1，表示自动分配
    HostMemoryBackend *hostmem;     // 对应的 backend
} PCDIMMDevice;
```

通过QOM(qemu object model)定义的虚拟内存条。可通过QMP或QEMU命令行进行管理。通过增加/移除该对象实现VM中内存的热插拔。


#### HostMemoryBackend

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

通过QOM定义的一段host内存，为虚拟内存条提供内存。可通过QMP或QEMU命令行进行管理。





#### 初始化

```
main => configure_accelerator => kvm_init => kvm_memory_listener_register(s, &s->memory_listener, &address_space_memory, 0) 初始化
kvm_state.memory_listener
     => kml->listener.region_add = kvm_region_add                  为listener设置操作
     => memory_listener_register                                   初始化listener并绑定到 address_space_memory
     => memory_listener_register(&kvm_io_listener, &address_space_io)  初始化 kvm_io_listener 并绑定到 address_space_io
     => cpu_exec_init_all => memory_map_init                       创建 system_memory("system") 和 system_io("io") 两个全局                                                                        MemoryRegion
     => address_space_init                                         初始化 address_space_memory("memory") 和 
                                                                   address_space_io("I/O")  AddressSpace，并设置 system_memory 
                                                                   和 system_io 作为 root
```

在初始化流程中，注册了 memory_listener 和 kvm_io_listener ，在AddressSpace address_space_memory 和 address_space_io 发生变化时会调用相应的回调函数。

同时将 AddressSpace 映射到 FlatView ，得到一个个 MemoryRegionSection ，调用 kvm_region_add ，将 MemoryRegionSection 注册到KVM中。

##### AddressSpace

```c
struct AddressSpace {
    /* All fields are private. */
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root;
    int ref_count;
    bool malloced;

    /* Accessed via RCU.  */
    struct FlatView *current_map;                               // 指向当前维护的FlatView，在address_space_update_topology时作为old比较

    int ioeventfd_nb;
    struct MemoryRegionIoeventfd *ioeventfds;
    struct AddressSpaceDispatch *dispatch;                      // 负责根据GPA找到HVA
    struct AddressSpaceDispatch *next_dispatch;
    MemoryListener dispatch_listener;
    QTAILQ_HEAD(memory_listeners_as, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};
```

一个 AddressSpace 由多个 MemoryRegion 构成，由于 MemoryRegion 可以有subregion(MemoryRegionSection)，因此形成树状结构。

初始化函数为 address_space_init ，在该函数中需要设置根级MemoryRegion(root)，同时将该 AddressSpace 加入到 address_spaces 中。

在 memory_map_init 中初始化了 address_space_memory 和 address_space_io，其中：

* address_space_memory 的 root 为 system_memory
* address_space_io 的 root 为 system_io




##### MemoryRegion

```c
struct MemoryRegion {
    Object parent_obj;                                                  // MemoryRegion可嵌套

    /* All fields are private - violators will be prosecuted */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool rom_device;                                                    // 只读
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;                                             // dirty map类型
    RAMBlock *ram_block;                                                // 指向对应的 RAMBlock
    Object *owner;
    const MemoryRegionIOMMUOps *iommu_ops;

    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;                                            // 指向父MemoryRegion
    Int128 size;                                                        // 内存区域大小
    hwaddr addr;                                                        // 在父MemoryRegion中的偏移量(memory_region_add_subregion_common)
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;                                                // 指向实体MemoryRegion
    hwaddr alias_offset;                                                // 起始地址(GPA)在实体MemoryRegion中的偏移量
    int32_t priority;
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;                   // subregion链表
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
    QLIST_HEAD(, IOMMUNotifier) iommu_notify;
    IOMMUNotifierFlag iommu_notify_flags;
};
```

表示在guest memory layout中的一段内存，具有逻辑(guest)意义。

* 根级 MemoryRegion 有 system_memory ，没有自己的内存，用于管理subregion
* 实体 MemoryRegion 有 ram_memory(pc.ram) 、 pci_memory(pci) 等，有自己的内存(从QEMU进程地址空间中分配)，大小为size
* 别名 MemoryRegion 有 ram_below_4g 、ram_above_4g 等，没有自己的内存，表示实体MemoryRegion(如pc.ram)的一部分，通过alias成员指向实体MemoryRegion，alias_offset为在实体MemoryRegion中的偏移量。


关系为：

```
                  alias
ram_memory (pc.ram) - ram_below_4g(ram-below-4g)
                    - ram_above_4g(ram-above-4g)
                     sub
system_memory(system) - ram_below_4g(ram-below-4g)
                      - ram_above_4g(ram-above-4g)
                      - pcms->hotplug_memory.mr        热插拔内存
          sub
rom_memory - isa_bios(isa-bios)
           - option_rom_mr(pc.rom)

system_io(io)
```



在初始化VM的过程中，建立了相应的 MemoryRegion ：

pc_init1 / pc_q35_init => pc_memory_init => memory_region_allocate_system_memory
                                         => memory_region_init_alias => memory_region_init        初始化alias的 MemoryRegion
                                         => memory_region_init                                    初始化 MemoryRegion
                                         => memory_region_init_ram                      分配 MemoryRegion 对应 Ramblock 的内存


##### memory_region_allocate_system_memory

对于非NUMA，直接分配内存

```
=> allocate_system_memory_nonnuma => memory_region_init_ram_from_file / memory_region_init_ram          分配 MemoryRegion 对应 
                                                                                                        Ramblock 的内存
=> vmstate_register_ram                                                                                 根据region的名称name设
                                                                                                        置RAMBlock的idstr
```

对于NUMA，分配后需要设置HostMemoryBackend

```
=> memory_region_init
=> memory_region_add_subregion                                  遍历所有NUMA节点的内存 HostMemoryBackend ，依次把那些mr成员不为空的
                                                                作为当前 MemoryRegion 的 subregion，偏移量从0开始递增

=> vmstate_register_ram_global => vmstate_register_ram          根据region的名称name设置RAMBlock的idstr
```




##### MemoryRegionSection

```c
struct MemoryRegionSection {
    MemoryRegion *mr;                           // 指向所属MemoryRegion
    AddressSpace *address_space;                // 所属AddressSpace
    hwaddr offset_within_region;                // 起始地址(HVA)在MemoryRegion内的偏移量
    Int128 size;
    hwaddr offset_within_address_space;         // 在AddressSpace内的偏移量，如果该AddressSpace为系统内存，则为GPA起始地址
    bool readonly;
};
```

将AddressSpace中的MemoryRegion映射到线性地址空间后，由于重叠的关系，原本完整的region可能会被切分成片段，即为 MemoryRegionSection。可以通过 section_from_flat_range 根据FlatRange截取MemoryRegionSection。

指向 MemoryRegion 的一部分(offset_within_region 到 offset_within_region + size)，是注册到KVM的基本单位。



##### KVMMemoryListener

回忆在初始化过程中，我们为 address_space_memory 和 address_space_io 分别注册了 memory_listener 和 kvm_io_listener 。前者类型为 KVMMemoryListener ，后者类型为 MemoryListener：

```c
typedef struct KVMMemoryListener {
    MemoryListener listener;
    KVMSlot *slots;
    int as_id;
} KVMMemoryListener;

struct MemoryListener {
    void (*begin)(MemoryListener *listener);
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

可以看到 KVMMemoryListener 主体就是 MemoryListener ，而 MemoryListener 包含大量函数指针，设置后用于在 address_space 成员发生变化时调用。

在初始化 AddressSpace 时，还会创建并绑定listener。

```
address_space_init => address_space_init_dispatch => as->dispatch_listener = ...
                                                  => memory_listener_register(as->dispatch_listener)
```

此时 address_space_memory 上有 kvm_state.memory_listener 和 dispatch_listener
address_space_io 上有 kvm_io_listener 和 dispatch_listener


注意AddressSpace和listener是一对多的关系，当AddressSpace发生变化时，其绑定的所有listener都会被触发。是怎么使得listener触发的呢？

其实，就是任何对 AddressSpace 和 MemoryRegion 的操作，都以 memory_region_transaction_begin 开头，以 memory_region_transaction_commit 结尾。

这些操作包括：启用、析构、增删eventfd、增删subregion、改变属性(flag)、设置大小、开启dirty log等，即：

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

其中 memory_region_transaction_begin 负责 flush_coalesced_mmio_buffer ，然后 ++memory_region_transaction_depth


##### memory_region_transaction_commit

```
=> --memory_region_transaction_depth
=> 如果 memory_region_transaction_depth 为0 且 memory_region_update_pending 大于0
    => MEMORY_LISTENER_CALL_GLOBAL(begin, Forward)        从前向后调用全局列表 memory_listeners 中所有listener的 begin 函数
    => 对 address_spaces 中的所有address space，调用 address_space_update_topology ，更新QEMU和KVM中维护的slot信息。
    => MEMORY_LISTENER_CALL_GLOBAL(commit, Forward)       从后向前调用全局列表 memory_listeners 中所有listener的 commit 函数
```



##### address_space_update_topology

```
=> address_space_get_flatview                                                           获取原来FlatView(AddressSpace.current_map)
=> generate_memory_topology                                                             生成新的FlatView
=> address_space_update_topology_pass                                                   比较新老FlatView，对其中不一致的FlatRange，执行相应的操作。
```

由于 AddressSpace 是树状结构，调用 address_space_update_topology ，使用FlatView模型将树状结构映射(压平)到线性地址空间。比较新老FlatView，对其中不一致的FlatRange，执行相应的操作，最终操作的KVM。

##### generate_memory_topology

```
=> addrrange_make                   创建起始地址为0，结束地址为2^64的地址空间，作为guest的线性地址空间
=> render_memory_region             从根级region开始，递归将region映射到线性地址空间中，产生一个个FlatRange，构成FlatView
=> flatview_simplify                将FlatView中连续的FlatRange进行合并为一个
```

AddressSpace的root成员是该地址空间的根级 MemoryRegion ，generate_memory_topology 负责将它的树状结构进行压平，从而能够映射到一个线性地址空间，得到 FlatView 。



##### address_space_update_topology_pass

比较该 AddressSpace 的新老FlatRange是否有变化，如果有，从前到后或从后到前遍历AddressSpace的listeners，调用对应callback函数。

```
=> MEMORY_LISTENER_UPDATE_REGION => section_from_flat_range      根据 FlatRange 的范围构造 MemoryRegionSection
                                 => MEMORY_LISTENER_CALL
```

例如，前面提到过，在初始化流程中，注册了 kvm_state.memory_listener 作为 address_space_memory 的listener，它会被加入到AddressSpace的listeners中。于是如果address_space_memory发生了变化，则调用会调用memory_listener中相应的函数。

例如 MEMORY_LISTENER_UPDATE_REGION 传入的callback参数为 region_add ，则调用 memory_listener.region_add (kvm_region_add)。



##### kvm_region_add

```
=> kvm_set_phys_mem => kvm_lookup_overlapping_slot
                    => 计算起始 HVA
                    => kvm_set_user_memory_region => kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem)
```

kvm_lookup_overlapping_slot 用于判断新的region section的地址范围(GPA)是否与已有KVMSlot(kml->slots)有重叠，如果重叠了，需要进行处理：

假设原slot可以切分成三个部分：prefix slot + overlap slot + suffix slot，重叠区域为overlap

对于完全重叠的情况，既有prefix slot又有suffix slot。无需注册新slot。

对于部分重叠的情况，prefix slot = 0 或 suffix slot = 0。则执行以下流程：

1. 删除原有slot
2. 注册prefix slot 或 suffix slot
3. 注册overlap slot

当然如果没有重叠，则直接注册新slot即可。然后将slot通过 kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem) 更新KVM中对应的 kvm_memory_slot 。

QEMU中维护slot结构也需要更新，对于原有的slot，因为它是kml->slots数组的项，所以在 kvm_set_phys_mem 直接修改即可。对于kml->slots中没有的slot，如prefix、suffix、overlap，则需要调用 kvm_alloc_slot => kvm_get_free_slot ，它会在 kml->slots 找一个空白的(memory_size = 0)为slot返回，然后对该slot进行设置即可。

##### kvm_set_phys_mem => kvm_set_user_memory_region

KVM规定了更新memory slot的参数：

```c
struct kvm_userspace_memory_region {
    __u32 slot;                                                             // 对应kvm_memory_slot的id
    __u32 flags;
    __u64 guest_phys_addr;                                                  // GPA
    __u64 memory_size; /* bytes */                                          // 大小
    __u64 userspace_addr; /* start of the userspace allocated memory */     // HVA
};
```

它会在 kvm_set_phys_mem => kvm_set_user_memory_region 的过程中进行计算并填充，流程如下：

1. 根据region的起始HVA(memory_region_get_ram_ptr) + region section在region中的偏移量(offset_within_region) + 页对齐修正(delta) 得到section真正的起始HVA，作为 userspace_addr 。

    在 memory_region_get_ram_ptr 中，如果当前region是另一个region的alias，则会向上追溯，一直追溯到非alias region(实体region)为止。将追溯过程中的 alias_offset 加起来，可以得到当前region在实体region中的偏移量。

    由于实体region具有对应的RAMBlock，所以调用 qemu_map_ram_ptr ，将实体region对应的ram_block的 host 和总offset加起来，得到当前region起始HVA。

2. 根据region section在AddressSpace内的偏移量(offset_within_address_space) + 页对齐修正(delta) 得到section真正的GPA，作为 start_addr

3. 根据region section的大小(size) - 页对齐修正(delta) 得到section真正的大小，作为 memory_size







### RAMBlock

前面提到，MemoryRegion表示在guest memory layout中的一段内存，具有逻辑意义。那么实际意义，也是就是这段内存所对应的实际内存信息是由谁维护的？

我们可以发现在MemoryRegion有一个ram_block成员，它是一个RAMBlock类型的指针，由RAMBlock来负责维护实际的内存信息，如HVA、GPA。比如在刚刚计算userspace_addr的流程中，计算region的起始HVA需要找到对应的RAMBlock，然后获取其host成员来得到。

RAMBlock定义如下：

```c
struct RAMBlock {
    struct rcu_head rcu;                                        // 用于保护Read-Copy-Update
    struct MemoryRegion *mr;                                    // 对应的 MemoryRegion
    uint8_t *host;                                              // 对应的HVA
    ram_addr_t offset;                                          // 在ram_list地址空间中的偏移(要把前面block的size都加起来)
    ram_addr_t used_length;                                     // 当前使用的长度
    ram_addr_t max_length;                                      // 总长度
    void (*resized)(const char*, uint64_t length, void *host);  // resize函数
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];                                            // id
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;                                 // 指向在ram_list.blocks中的下一个block
    int fd;                                                     // 映射文件的文件描述符
    size_t page_size;                                           // page大小，一般和host保持一致
};
```

在 MemoryRegion 的创建流程可以发现，它们一般会先调用 memory_region_init 初始化 MemoryRegion 结构，然后调用相应的函数创建相应的 RAMBlock。常见的函数有以下几个：

* memory_region_init_ram => qemu_ram_alloc
* memory_region_init_ram_from_file => qemu_ram_alloc_from_file
* memory_region_init_ram_ptr => qemu_ram_alloc_from_ptr
* memory_region_init_resizeable_ram => qemu_ram_alloc_resizeable

qemu_ram_alloc / qemu_ram_alloc_from_file / memory_region_init_ram_ptr / memory_region_init_resizeable_ram => qemu_ram_alloc_internal => ram_block_add

通过 qemu_ram_alloc 创建的 RAMBlock.host 为NULL，会调用 phys_mem_alloc (qemu_anon_ram_alloc) 分配内存，然后插入到 ram_list.blocks 中

通过 qemu_ram_alloc_from_file 创建的 RAMBlock 会调用 file_ram_alloc 使用对应路径的(设备)文件来分配内存，通常是由于需要使用hugepage，会通过`-mem-path`参数指定了hugepage的设备文件(如 /dev/hugepages )

通过 memory_region_init_ram_ptr 创建的 RAMBlock.host 为传入的指针地址

通过 memory_region_init_resizeable_ram 创建的 RAMBlock.host 为NULL，但resizeable为true，表示还没有分配内存，但可以resize。




##### qemu_anon_ram_alloc

=> qemu_ram_mmap(-1, size, QEMU_VMALLOC_ALIGN, false) => mmap

通过mmap在QEMU的进程地址空间中分配size大小的内存。





### RAMList

ram_list 是一个全局变量，以链表的形式维护了所有的RAMBlock，构成一个地址空间。

```c
RAMList ram_list = { .blocks = QLIST_HEAD_INITIALIZER(ram_list.blocks) };

typedef struct RAMList {
    QemuMutex mutex;
    RAMBlock *mru_block;
    /* RCU-enabled, writes protected by the ramlist lock. */
    QLIST_HEAD(, RAMBlock) blocks;                              // RAMBlock链表
    DirtyMemoryBlocks *dirty_memory[DIRTY_MEMORY_NUM];          // 记录脏页信息，用于 VGA / TCG / Live Migration
    uint32_t version;                                           // 每更改一次加1
} RAMList;
extern RAMList ram_list;
```

* VGA: 显卡仿真通过 dirty_memory 跟踪脏的视频内存，用于重绘界面
* TCG: 动态翻译器通过 dirty_memory 追踪自调整的代码，当上游指令发生变化时对其重新编译
* Live Migration: 动态迁移通过 dirty_memory 来跟踪dirty page，在dirty page被改变之后重传




##### AddressSpaceDispatch

为了在虚拟机退出时根据GPA找到对应的HVA，定义了 AddressSpaceDispatch 结构：

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

同时创建并注册对应的listener：

```c
    as->dispatch_listener = (MemoryListener) {
        .begin = mem_begin,
        .commit = mem_commit,
        .region_add = mem_add,
        .region_nop = mem_add,
        .priority = 0,
    };
```

于是在添加新的 MemeryRegion 后， mem_add => register_subpage / register_multipage


register_subpage 注册的是 iomem ，多个MemoryRegionSection共用一个 subpage

```
=> 如果 MemoryRegionSection 所属的 MemoryRegion 的 subpage 不存在
    => subpage_init                                         创建subpage
    => phys_page_set => phys_map_node_reserve               分配页目录项
                     => phys_page_set_level                 填充页表，从l5填到l0
=> 如果存在
    => container_of(existing->mr, subpage_t, iomem)         取出
=> subpage_register                                         设置 subpage
```

每个 AddressSpace 的 AddressSpaceDispatch 的 map 都是一个多级(6级)页表，最后一级页表指向 MemoryRegionSection

```
register_multipage => phys_page_set => phys_map_node_reserve               分配页目录项
                                    => phys_page_set_level                 填充页表，从l5填到l0
```

当kvm exit(如 KVM_EXIT_IO )退到qemu之后，通过 AddressSpaceDispatch.map 可以找到对应的 MemoryRegionSection，继而找到对应的HVA











## KVM


### kvm_vm_ioctl_set_memory_region

添加内存。在收到 KVM_SET_USER_MEMORY_REGION(取代了 KVM_SET_MEMORY_REGION ，因为其不支持细粒度控制) 时调用。

如果开启了 KVM_CAP_SYNC_MMU ，则在外部(如QEMU)对 Region的修改(如mmap/madvise)将立刻被同步

传入参数如下：

```c
struct kvm_userspace_memory_region {
    __u32 slot;                                                             // 对应kvm_memory_slot的id
    __u32 flags;
    __u64 guest_phys_addr;                                                  // GPA
    __u64 memory_size; /* bytes */                                          // 大小
    __u64 userspace_addr; /* start of the userspace allocated memory */     // HVA
};
```

flags：

* KVM_MEM_LOG_DIRTY_PAGES 声明需要跟踪对该 Region 的写，提供给 KVM_GET_DIRTY_LOG 时读取
* KVM_MEM_READONLY        如果支持readonly(KVM_CAP_READONLY_MEM)，则当写该 Region 时触发 VMEXIT (KVM_EXIT_MMIO)

调用流程如下：

=> kvm_vm_ioctl_set_memory_region => kvm_set_memory_region => __kvm_set_memory_region

根据npages和原来的npages判断用户操作，创建新slot，然后通过 install_new_memslots 更新。

#### KVM_MR_CREATE
现在有页而原来没有，则为新增内存区域，创建并初始化slot

#### KVM_MR_DELETE
现在没有页而原来有，则为删除内存区域，将slot标记为KVM_MEMSLOT_INVALID

#### KVM_MR_FLAGS_ONLY / KVM_MR_MOVE
现在有页且原来也有，则为修改内存区域，如果只有flag变了，则为 KVM_MR_FLAGS_ONLY ，目前只有可能是 KVM_MEM_LOG_DIRTY_PAGES ，则根据flag选择是要创建还是释放dirty_bitmap。

如果GPA有变，则为 KVM_MR_MOVE ，需要进行移动。其实就直接将原来的slot标记为KVM_MEMSLOT_INVALID，然后添加新的。




#### kvm_memory_slot

在 __kvm_set_memory_region 中初始化了region对应的slot，它是KVM中内存管理中的基本单位。

```c
struct kvm_memory_slot {
    gfn_t base_gfn;                     // slot的起始gfn
    unsigned long npages;               // page数
    unsigned long *dirty_bitmap;        // 脏页bitmap
    struct kvm_arch_memory_slot arch;   // 结构相关，包括rmap和lpage_info等
    unsigned long userspace_addr;       // 对应的起始HVA
    u32 flags;
    short id;
};


struct kvm_arch_memory_slot {
    struct kvm_rmap_head *rmap[KVM_NR_PAGE_SIZES];              // 反向链接
    struct kvm_lpage_info *lpage_info[KVM_NR_PAGE_SIZES - 1];   // 维护下一级页表是否关闭hugepage
    unsigned short *gfn_track[KVM_PAGE_TRACK_MAX];
};
```

slot保存在 kvm->memslots[as_id]->memslots[id] 中，其中as_id为AddressSpace id(KVM中只有两个)，id为slot id。它们的内存都在 kvm_create_vm 中就分配好了。这里做初始化。









### 内存管理单元(MMU)

#### 初始化

```
kvm_init => kvm_arch_init => kvm_mmu_module_init => 建立 mmu_page_header_cache 作为cache
                                                 => register_shrinker(&mmu_shrinker)              注册回收函数


kvm_vm_ioctl_create_vcpu =>
kvm_arch_vcpu_create => kvm_x86_ops->vcpu_create (vmx_create_vcpu) => init_rmode_identity_map     为实模式建立1024个页的等值映射
                                                                   => kvm_vcpu_init => kvm_arch_vcpu_init => kvm_mmu_create

kvm_arch_vcpu_setup => kvm_mmu_setup => init_kvm_mmu => init_kvm_tdp_mmu                如果支持two dimentional   
                                                                                        paging(EPT)，初始化之，设置 
                                                                                        vcpu->arch.mmu 中的属性和函数
                                                     
                                                     => init_kvm_softmmu => kvm_init_shadow_mmu   否则初始化SPT
```


##### kvm_mmu_create


以vcpu为单位初始化mmu相关信息。它们在 vcpu中的相关定义包含：

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

    // 以下为cache，用于提升常用数据结构的分配速度
    // 用于分配 pte_list_desc ，它是反向映射链表 parent_ptes 的链表项，在 mmu_set_spte => rmap_add => pte_list_add 中分配
    struct kvm_mmu_memory_cache mmu_pte_list_desc_cache;
    // 用于分配 page ，作为kvm_mmu_page.spt
    struct kvm_mmu_memory_cache mmu_page_cache;
    // 用于分配 kvm_mmu_page ，作为页表页
    struct kvm_mmu_memory_cache mmu_page_header_cache;
    ...
}
```

其中 cache 用于提升页表中常用数据结构的分配速度。这些cache会在初始化MMU(kvm_mmu_load)、发生page fault(tdp_page_fault)等情况下调用 mmu_topup_memory_caches 来保证各cache充足。

```c
// 保证各cache充足
static int mmu_topup_memory_caches(struct kvm_vcpu *vcpu)
{
    // r不为0表示从slab分配/__get_free_page失败，直接返回错误
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

pte_list_desc_cache 和 mmu_page_header_cache 两块全局slab cache 在 kvm_mmu_module_init 中被创建，作为 vcpu->arch.mmu_pte_list_desc_cache 和 vcpu->arch.mmu_page_header_cache 的cache来源。

可以在host通过 `cat /proc/slabinfo` 查看到分配的slab：

```
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
kvm_mmu_page_header    576    576    168   48    2 : tunables    0    0    0 : slabdata     12     12      0
```





#### 加载页表

在 kvm_vm_ioctl_create_vcpu 中仅仅是对mmu进行初始化，比如将 vcpu->arch.mmu.root_hpa 设置为 INVALID_PAGE ，直到要进入VM(VMLAUNCH/VMRESUME)前才真正设置该值。

```
vcpu_enter_guest => kvm_mmu_reload => kvm_mmu_load => mmu_topup_memory_caches                       保证各cache充足
                                                   => mmu_alloc_roots => mmu_alloc_direct_roots     如果根页表不存在，则分配一个
                                                                                                    kvm_mmu_page
                                                   
                                                   => vcpu->arch.mmu.set_cr3 (vmx_set_cr3)          对于EPT，将该页的spt(strcut 
                                                                                                    page)的HPA加载到VMCS
                                                                                                    
                                                                                                    对于SPT，将该页的spt(strcut 
                                                                                                    page)的HPA加载到cr3
                 => kvm_x86_ops->run (vmx_vcpu_run)
                 => kvm_x86_ops->handle_exit (vmx_handle_exit)
```


#### kvm_mmu_page

用于表示页表，详细解释见 Documentation/virtual/kvm/mmu.txt


```c
struct kvm_mmu_page {
    struct list_head link;                          // 加到 kvm->arch.active_mmu_pages 或 invalid_list ，表示当前页处于的状态
    struct hlist_node hash_link;                    // 加到 vcpu->kvm->arch.mmu_page_hash ，提供快速查找

    /*
     * The following two entries are used to key the shadow page in the
     * hash table.
     */
    gfn_t gfn;                                      // 管理地址范围的起始地址对应的gfn
    union kvm_mmu_page_role role;                   // 基本信息，包括硬件特性和所属层级等

    u64 *spt;                                       // 指向struct page的地址，其包含了所有页表项(pte)。同时page->private会指向该 kvm_mmu_page
    /* hold the gfn of each spte inside spt */
    gfn_t *gfns;                                    // 所有页表项(pte)对应的gfn
    bool unsync;                                    // 用于最后一级页表页，表示该页的页表项(pte)是否与guest同步(guest是否已更新tlb)
    int root_count;          /* Currently serving as active root */ // 用于最高级页表页，统计有多少EPTP指向自身
    unsigned int unsync_children;                   // 页表页中unsync的pte数
    struct kvm_rmap_head parent_ptes; /* rmap pointers to parent sptes */ // 反向映射(rmap)，维护指向自己的上级页表项

    /* The page is obsolete if mmu_valid_gen != kvm->arch.mmu_valid_gen.  */
    unsigned long mmu_valid_gen;                    // 代数，如果比 kvm->arch.mmu_valid_gen 小则表示已失效

    DECLARE_BITMAP(unsync_child_bitmap, 512);       // 页表页中unsync的spte bitmap

#ifdef CONFIG_X86_32
    /*
     * Used out of the mmu-lock to avoid reading spte values while an
     * update is in progress; see the comments in __get_spte_lockless().
     */
    int clear_spte_count;                           // 32bit下，对spte的修改是原子的，因此通过该计数来检测是否正在被修改，如果被改了需要redo
#endif

    /* Number of writes since the last time traversal visited this page.  */
    atomic_t write_flooding_count;                  // 统计从上次使用以来的emulation次数，如果超过一定次数，会把该page给unmap掉
};

union kvm_mmu_page_role {
    unsigned word;
    struct {
        unsigned level:4;           // 页所处的层级
        unsigned cr4_pae:1;         // cr4.pae，1表示使用64bit gpte
        unsigned quadrant:2;        // 如果cr4.pae=0，则gpte为32bit，但spte为64bit，因此需要用多个spte来表示一个gpte，该字段指示是gpte的第几块
        unsigned direct:1;
        unsigned access:3;          // 访问权限
        unsigned invalid:1;         // 失效，一旦unpin就会被销毁
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
        unsigned smm:8;             // 处于system management mode
    };
};
```


#### EPT Violation

当guest第一次访问某个页面时，由于没有gva到gpa的映射，会触发guest os的page fault。于是guest os会建立对应的pte并修复好各级页表，最后访问对应的GPA。由于没有建立gpa到hva的映射，于是触发EPT Violation，VMEXIT会到KVM，在 vmx_handle_exit 中执行kvm_vmx_exit_handlers[exit_reason]，由于exit_reason是 EXIT_REASON_EPT_VIOLATION ，因此调用 handle_ept_violation 。

##### handle_ept_violation

```
=> vmcs_readl(EXIT_QUALIFICATION)                       获取EPT退出的原因。EXIT_QUALIFICATION是Exit reason的补充，详情见 Vol. 3C 27-9 Table 27-7
=> vmcs_read64(GUEST_PHYSICAL_ADDRESS)                  获取发生缺页的GPA
=> 根据exit_qualification内容得到error_code，可能是 read fault / write fault / fetch fault / ept page table is not present
=> kvm_mmu_page_fault => vcpu->arch.mmu.page_fault (tdp_page_fault)
```

##### tdp_page_fault

```
=> gfn = gpa >> PAGE_SHIFT      将GPA右移pagesize得到gfn(guest frame number)
=> mapping_level                计算gfn在页表中所属level，不考虑hugepage则为1
=> try_async_pf                 将gfn转换为pfn(physical frame number)
        => kvm_vcpu_gfn_to_memslot => __gfn_to_memslot                  找到gfn对应的slot
        => __gfn_to_pfn_memslot                                         找到gfn对应的pfn
                => __gfn_to_hva_many => __gfn_to_hva_memslot            计算gfn对应的起始hva
                => hva_to_pfn                                           计算hva对应的pfn，同时确保该物理页在内存中

=> __direct_map                                                         更新EPT，将新的映射关系逐层添加到EPT中
    => for_each_shadow_entry                                            从level4(root)开始，逐层补全页表，对于每一层：
        => mmu_set_spte                                                 对于level1的页表，其页表项肯定是缺的，所以不用判断直接填上pfn的起始hpa
        => is_shadow_present_pte                                        如果下一级页表页不存在，即当前页表项没值(*sptep = 0)
            => kvm_mmu_get_page                                         分配一个页表页
            => link_shadow_page                                         将新页表页的HPA填入到当前页表项(sptep)中
```

可以发现主要有两步，一步获取GPA所对应的物理页，如果没有会进行分配。另一步是更新EPT。

##### try_async_pf
1. 根据gfn找到对应的memslot
2. 用memslot的起始hva(userspace_addr) + (gfn - slot中的起始gfn(base_gfn) ) * 页大小(PAGE_SIZE)，得到gfn对应的起始hva。
3. 为该hva分配一个物理页，有 hva_to_pfn_fast 和 hva_to_pfn_slow 两种， hva_to_pfn_fast 实际上是调用 __get_user_pages_fast ，会尝试去pin该page，即确保该地址所在的物理页在内存中。如果失败，退化到 hva_to_pfn_slow ，会先去拿 mm->mmap_sem 的锁然后调用 __get_user_pages 来pin。
4. 如果分配成功，对其返回的struct page调用 page_to_pfn 得到对应的pfn

该函数建立了gfn到hfn的映射，同时将该page pin在host的内存中。


##### __direct_map

通过迭代器 kvm_shadow_walk_iterator 将EPT中和该GPA相关的页表补充完整。

```c
struct kvm_shadow_walk_iterator {
    u64 addr;                   // 发生page fault的GPA，迭代过程就是要把GPA所涉及的页表项都填上
    hpa_t shadow_addr;          // 当前页表项的HPA，在 shadow_walk_init 中设置为 vcpu->arch.mmu.root_hpa
    u64 *sptep;                 // 指向当前页表项，在 shadow_walk_okay 中更新
    int level;                  // 当前层级，在 shadow_walk_init 中设置为4 (x86_64 PT64_ROOT_LEVEL)，在 shadow_walk_next 中减1
    unsigned index;             // 在当前level页表中的索引，在 shadow_walk_okay 中更新
};
```

在每轮迭代中，sptep都会指向GPA在当前级页表中所对应的页表项，我们的目的就是把下一级页表的GPA填到该页表项内(即设置*sptep)。因为是缺页，可能会出现下一级的页表页不存在的问题，这时候需要分配一个页表页，然后再将该页的GPA填进到*sptep中。

```
举个例子，对于GPA(如 0xfffff001)，其二进制为 000000000 000000011 111111111 111111111 000000000001
                                           PML4      PDPT       PD        PT        Offset
```

初始化状态：level = 4，shadow_addr = root_hpa，addr = GPA

执行流程：

1. index = addr 在当前level分段的值，如在level = 4时为0(000000000)，在level = 3时为3(000000011)
2. sptep = va(shadow_addr) + index，得到GPA在当前地址中所对应的页表项HVA
3. 如果*sptep没值，分配一个page作为下级页表，同时将*sptep设置为该page的HPA
4. shadow_addr = *sptep，进入下级页表

开启hugepage时，由于页表项管理的范围变大，所需页表级数减少，在默认情况下page大小为2M，因此无需level 1。



##### mmu_set_spte

```
=> set_spte => mmu_spte_update => mmu_spte_set => __set_spte                                设置物理页(hfn)起始HPA到*sptep，即设置最后一级页表中的pte的值
=> rmap_add => page_header(__pa(spte))                                                      获取spetp所在的页表页
            => kvm_mmu_page_set_gfn                                                         将gfn设置到该页表页的gfns中
            => gfn_to_rmap => __gfn_to_memslot                                              获取gfn对应的slot
                           => __gfn_to_rmap => gfn_to_index                                 通过gfn和slot->base_gfn，算出该页在
                                                                                            slot中的index
                                    => slot->arch.rmap[level - PT_PAGE_TABLE_LEVEL][idx]    从该slot中取出对应的rmap
            => pte_list_add                                                                 将当前项(spetp)的地址加入到rmap中，做
                                                                                            反向映射
```


作用于1级页表(PT)。负责设置最后一级页表中的pte(*spetp)的值，同时将当前项(spetp)的地址加入到 slot->arch.rmap[level - PT_PAGE_TABLE_LEVEL][idx] 中作为反向映射，此后可以通过gfn快速找到该 kvm_mmu_page 。

在大多数情况下，gfn对应单个kvm_mmu_page，于是rmap_head直接指向spetp即可。但由于一个gfn对应多个kvm_mmu_page，因此在该情况下rmap采用链表+数组来维护。一个链表项 pte_list_desc 能存放三个spetp。由于pte_list_desc频繁被分配，因此也是从cache (vcpu->arch.mmu_pte_list_desc_cache)中分配的。




##### kvm_mmu_get_page

获取gfn对应的 kvm_mmu_page 。会通过gfn尝试从 vcpu->kvm->arch.mmu_page_hash 中找到对应的页表页，如果以前分配过该页则直接返回即可。否则需要通过 kvm_mmu_alloc_page 从cache中分配，然后以gfn为key将其加到vcpu->kvm->arch.mmu_page_hash中。

kvm_mmu_alloc_page 会通过 mmu_memory_cache_alloc 从 vcpu->arch.mmu_page_header_cache 和 vcpu->arch.mmu_page_cache 分配 kvm_mmu_page 和 page 对象，在 mmu_topup_memory_caches 中保证了这些cache的充足，如果发现余量不够，会通过全局变量的slab补充，这点前面也提到了。



##### link_shadow_page

```
=> mmu_spte_set => __set_spte                               为当前页表项的值(*spetp)设置下一级页表页的HPA
=> mmu_page_add_parent_pte => pte_list_add                  将当前项的地址(spetp)加入到下一级页表页的parent_ptes中，做反向映射
```

作用于2-4级页表(PML4 - PDT)，在遍历过程中如果发现下一级页表缺页，需要在分配一个页表页后更新当前的迭代器指向的页表项(spetp)，设置为下一级该页表页的HPA，这样下次就能够通过页表项访问到该页表页了。同时需要将当前页表项(spetp)的地址加入到下一级页表页的parent_ptes中作为反向映射。


利用两套反向映射，在利用GPA可以算出gfn后，可以通过rmap得到在1级中的页表项，通过parent_ptes又可以得到在2-4级中的页表项。当host需要将guest的某个GPA的page换出时，直接通过反向索引操作该gfn相关的页表项，而无需再次走EPT查询。






## 总结

### QEMU
创建一系列 MemoryRegion ，分别表示guest中的rom、ram等区域。 MemoryRegion 之间可通过alias或subregion的方式定义相互之间的关系，从而进一步细化区域的定义。

对于一个实体MemoryRegion(非alias)，在初始化内存的过程中会创建它所对应的RAMBlock。RAMBlock通过mmap的方式从QEMU的进程空间中分配内存，并负责维护该MemoryRegion管理内存的起始HVA/GPA/size等信息。

所有的MemoryRegion构成一个 AddressSpace ，表示VM的物理地址空间。如果AddressSpace中的MemoryRegion发生变化，则listener被触发，将 AddressSpace 下的 MemoryRegion 树展平，形成一维的FlatView，比较FlatRange是否发生了变化。如果是调用相应方法如 region_add 对变化的section region进行检查，更新QEMU内的KVMSlot，同时填充 kvm_userspace_memory_region 结构，作为ioctl的参数更新KVM中的 kvm_memory_slot 。


### KVM

当 QEMU 通过 ioctl 创建vcpu时，调用 kvm_mmu_create 初始化mmu相关信息，为页表项结构分配slab cache。

当 KVM 要进入 guest 前， vcpu_enter_guest => kvm_mmu_reload 会将根级页表地址加载到VMCS，使guest使用该页表。

当发生 EPT Violation 时， VMEXIT 到 KVM 中。如果是缺页，则拿到对应的GPA，根据GPA算出gfn，根据gfn找到对应的memory slot并根据其信息找到对应的hva，再根据hva找到对应的pfn，确保该page位于内存。在把缺的页填上后，需要更新EPT，完善其中缺少的页表项。于是从level4开始，逐层补全页表，对于在某层上缺少的页表页，会从slab中分配后将新页表页的HPA填入到上一级页表中。

除了建立上级页表到下级页表的关联外，还会建立反向映射，可以直接根据GPA找到gfn相关的页表项，而无需再次走EPT查询。


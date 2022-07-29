## qemu代表mem结构体的关系
别人的讲的很好了<a>https://oenhan.com/qemu-memory-struct</a>

### address space
cpuMM地址总线和IO总线对应了qemu中address_space_memory和address_space_io<br>
physmem.c
```
RAMList ram_list = { .blocks = QLIST_HEAD_INITIALIZER(ram_list.blocks) };

static MemoryRegion *system_memory;
static MemoryRegion *system_io;

AddressSpace address_space_io;
AddressSpace address_space_memory;

static MemoryRegion io_mem_unassigned;
```
kvm在x86下返回的KVM_CAP_MULTI_ADDRESS_SPACE也是2<br>
\arch\x86\include\asm\kvm_host.h
```
#define __KVM_VCPU_MULTIPLE_ADDRESS_SPACE
#define KVM_ADDRESS_SPACE_NUM 2
```
可以通过得到平面视图
```
view = address_space_get_flatview(as);
```
### memory region
```
struct MemoryRegion {
    Object parent_obj;
    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool nonvolatile;
    bool rom_device;
    bool flush_coalesced_mmio;
    uint8_t dirty_log_mask;
    bool is_iommu;
    RAMBlock *ram_block;
    Object *owner;

    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;
    int mapped_via_alias; /* Mapped via an alias, container might be NULL */
    Int128 size;
    hwaddr addr;
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;
    hwaddr alias_offset;
    int32_t priority;
    QTAILQ_HEAD(, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
    RamDiscardManager *rdm; /* Only for RAM */
};
```
MemRegion是一段抽象的内存，可以不对应实际的ramblock，而下面挂着其他的subRegions（也是MemRegion）
###### 例子
x86_bios_rom_init
```
/* 在ram上创建了一段mem region(有ramblock对应的) */
memory_region_init_ram(bios, NULL, "pc.bios", bios_size, &error_fatal);
/* 做了一个别名，将Memregion bios中后128kB称作为MemRegion isa_bios(所以isa_bios是没有实际ramblock对应的) */
memory_region_init_alias(isa_bios, NULL, "isa-bios", bios,
                             bios_size - isa_bios_size, isa_bios_size);
memory_region_add_subregion_overlap(rom_memory,
                                    0x100000 - isa_bios_size,
                                    isa_bios,
                                    1);
if (!isapc_ram_fw) {
    memory_region_set_readonly(isa_bios, true);
}

/* bios的container->rom_memory, 将bios按prio插入到rom_memory->subregions */
/* map all the bios at the top of memory */
memory_region_add_subregion(rom_memory,
                            (uint32_t)(-bios_size),
                            bios);
return;
```
### ramblock
这个是表示guest phy mem中的一段ram空间，所以offset是GPA, host是HVA<br>
```
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;                                                
    uint8_t *host;                                                          /* qemu进程mapping地址空间中的一段大小为max_length的内存 */
    ram_addr_t offset;                                                      /* qemu phymem的地址 */
    ram_addr_t used_length;
    ram_addr_t max_length;                                                  /* block大小 */
    QLIST_ENTRY(RAMBlock) next;
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers;
    int fd;
    size_t page_size;
};
```
#### ram_block_add
ramblock通过ram_block_add分配并加入到<b>ram_list</b>全局链表中
1. find_ram_offset搜寻ramlist满足大小符合new->max_length的ram空间(从0到RAM_ADDR_MAX)，插入ramlist尾端，返回offset填充(RamBlock)new->offset
2. qemu_anon_ram_alloc在host上mapping一段大小为max_length的空间, new->host
```
RAMBLOCK_FOREACH(block) {
        ram_addr_t candidate, next = RAM_ADDR_MAX;

        /* Align blocks to start on a 'long' in the bitmap
         * which makes the bitmap sync'ing take the fast path.
         */
        candidate = block->offset + block->max_length;
        candidate = ROUND_UP(candidate, BITS_PER_LONG << TARGET_PAGE_BITS);

        /* Search for the closest following block
         * and find the gap.
         */
        RAMBLOCK_FOREACH(next_block) {
            if (next_block->offset >= candidate) {
                next = MIN(next, next_block->offset);
            }
        }

        /* If it fits remember our place and remember the size
         * of gap, but keep going so that we might find a smaller
         * gap to fill so avoiding fragmentation.
         */
        if (next - candidate >= size && next - candidate < mingap) {
            offset = candidate;
            mingap = next - candidate;
        }

        trace_find_ram_offset_loop(size, candidate, offset, next, mingap);
    }

```

### MemoryRegionSection、kvmSlot和kvm_userspace_memory_region
MemoryRegionSection->kvmSlot->kvm_userspace_memory_region <br>
转换为kvm_userspace_memory_region后通过kvm_set_user_memory_region插入到vm的物理mem中<br>
##### kvm_userspace_memory_region
kvm_userspace_memory_region是kvm定义的结构体，表征的是kvm中定义的一个物理mem <b>slot</b>
```
struct kvm_userspace_memory_region {
	__u32 slot;
	__u32 flags;
	__u64 guest_phys_addr;
	__u64 memory_size; /* bytes */
	__u64 userspace_addr; /* start of the userspace allocated memory */
};
```
##### MemoryRegionSection
一个MemoryRegion中的一段，就是从一个FlagRange(扁平化视图中的一段)变过来的, 不是MemRegion，没有关联ramblock哦<br>
其中的offset_within_address_space就是物理ram地址，但是要转成kvmslot（再转成kvm_userspace_memory_region）要做页地址对齐
```
section_from_flat_range(FlatRange *fr, FlatView *fv)
{
    return (MemoryRegionSection) {
        .mr = fr->mr,
        .fv = fv,
        .offset_within_region = fr->offset_in_region,
        .size = fr->addr.size,
        .offset_within_address_space = int128_get64(fr->addr.start),
        .readonly = fr->readonly,
        .nonvolatile = fr->nonvolatile,
    };
}
```
##### kvmslot






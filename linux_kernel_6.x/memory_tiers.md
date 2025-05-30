# memory_tiers


> @author Jiahao Lu @ XMU  (Xiamen University)

> Feel free to contact me : `lujhcoconut@foxmail.com`


## Associated Directories

```
linux_kernel/
    |-include/linux/
        |-memory_tiers.h
    |-mm/
        |-memory_tiers.c
```

## Core Data Structure(s)

### memory_dev_type

```c
struct memory_dev_type {
	/* list of memory types that are part of same tier as this type */
	struct list_head tier_sibling;
	/* list of memory types that are managed by one driver */
	struct list_head list;
	/* abstract distance for this specific memory type */
	int adistance;
	/* Nodes of same abstract distance */
	nodemask_t nodes;
	struct kref kref;
};
```
* The Linux kernel employs a core data structure to represent memory device types (e.g., DRAM, PMem, HBM), with each field specifically designed to support the hierarchical management of tiered memory systems.
    > Linux 内核中用于描述内存设备类型（如 DRAM、PMem、HBM 等）的核心数据结构，其每个字段的设计均服务于异构内存系统的分层管理。
    * `tier_sibling` :  `tier_sibling` 是 `struct list_head` 类型，通过内核的 `list.h` 实现双向循环链表
        * `tier_sibling` 
            
            list of memory types that are part of same tier as this type 与该类型处于同一层级的内存类型列表

        * `list` 
        
            list of memory types that are managed by one driver 由同一个驱动管理的内存类型列表

      ```c
      // 层级结构示意
        struct memory_tier {
            struct list_head memory_types;  // Head node, 头节点
            ...
        };
        struct memory_dev_type {
            struct list_head tier_sibling;  // member nodes, 成员节点
            ...
        };
      ```
      * **Head node**: represented by `struct memory_tier->memory_types` as the head of the linked list.
        > **头节点**​​：由 `struct memory_tier->memory_types` 作为链表头
      * **member nodes**: memory devices of the same tier are linked together via the `tier_sibling` pointer.
        > **成员节点**​​：同层级的 memory_dev_type 通过 `tier_sibling` 串联

      * **More** about `list_head`:
        * The essence of the linked list design: **decoupling and reuse**
          > 链表结构设计的本质：解耦与复用​
            * Separation of data and operations
                > 数据与操作分离

                * list_head only contains prev and next pointers (occupying 16 bytes) and does not store any application-specific data.
                  > `list_head` 仅包含 prev/next 指针（占 16 字节），不存储业务数据;

                * The host structure (e.g., `memory_dev_type`) gains linked list capability by embedding a list_head member (e.g., `tier_sibling`), enabling zero-copy linkage.
                  > 宿主结构（如 `memory_dev_type`）通过嵌入 `list_head` 成员（`tier_sibling`）获得链表能力，实现​​零拷贝链接​​。

                    ```c
                    // 内存类型加入层级链表
                    list_add_tail(&new_type->tier_sibling, &memtier->memory_types);
                    ```
            * Type-agnostic operation interface
                > 类型无关的操作接口

                * All linked list operations (insertion, deletion, modification, and lookup) are implemented via generic functions such as `list_add()` and `list_del()`.

                  > 所有链表操作（增删改查）通过通用函数实现（如 `list_add()`、`list_del()`）

                * Avoid rewriting linked list logic for each data type.
                  > 避免为每种数据类型重写链表逻辑
            
            * Reverse mapping to the host object
                > 宿主对象反向定位

                * Using the `container_of` macro to reverse-calculate the host structure’s address from a `list_head` pointer.
                  > 通过 `container_of` 宏从 `list_head` 指针反向计算宿主结构地址 

                  ```c
                  struct memory_dev_type *memtype = container_of(ptr, struct memory_dev_type, tier_sibling);
                  ```
      * **More** about `Linux Memory Management`:
        > Memory management mechanisms: the collaboration between the buddy system and the slab allocator   
        内存管理机制：伙伴系统与 Slab 分配器的协同​

        * Memory allocation for the host structure 宿主结构的内存分配​   
            * Large memory blocks (`≥ PAGE_SIZE`) are allocated as contiguous physical pages using the buddy system.
            
                ```c
                // 为 memory_dev_type 分配大内存（假设其大小 >4KB）
                struct memory_dev_type *memtype = (struct memory_dev_type*)__get_free_pages(GFP_KERNEL, order);
                ```
            * Small memory allocations (`< PAGE_SIZE`) use the slab allocator to cache objects.
            
                ```c
                struct memory_dev_type *memtype = kmalloc(sizeof(struct memory_dev_type), GFP_KERNEL);
                ```
              Slab 将伙伴系统分配的页框分割为小对象缓存（如 memory_dev_type），显著减少碎片

        * Memory of the linked list node itself   链表节点自身的内存
            * **Zero overhead**: Since `list_head` is a member of the host structure, its memory is allocated along with the host and requires no separate management.
                > ​​**零开销​**​：`list_head` 作为宿主结构的成员，其内存已随宿主分配，无需单独管理
            * **Allocation-free operations**: Functions like `list_add()` and `list_del()` only manipulate pointers without triggering memory allocation or deallocation.
                > **操作无分配​​**：`list_add()`、`list_del()` 等函数仅修改指针，不触发内存分配/释放


            ```
            |  CPU Core  |        Buddy System          |        Slab Allocator         |   Link Operation   |
            |____________|______________________________|_______________________________|____________________|
            |            |                              |                               |                    |
            | 2^{order}  |                              |                               |                    |
            | ---------> |                              |                               |                    |
            | seq_pages  |                              |                               |                    |
            | <--------- |                              |                               |                    |
            |            |                              |                               |                    |
            | Splitting large pages into slab objects   |                               |                    |
            | ----------------------------------------> |                               |                    |
            | <---------------------------------------- |                               |                    |
            |            |                              |                               |                    |
            |                             list_add_tail() ...                           |                    |
            | ------------------------------------------------------------------------> |          |         |
            |                                                                           |          |         |
            |                                                                           |  update prev/next  |
            |            |                              |                               |                    |
            |____________|______________________________|_______________________________|____________________|
            |  CPU Core  |        Buddy System          |        Slab Allocator         |   Link Operation   |
            ```


        * **More** about `list_head` in `memory_tiers`:
            * Initialize tiered list head 初始化层级链表头
                * `memory_tier->memory_types` is initialized as a self-referencing (circular) list using `INIT_LIST_HEAD`().
                    > `memory_tier->memory_types` 通过 `INIT_LIST_HEAD()` 初始化为自环链表
                  ```c
                  INIT_LIST_HEAD(&memtier->memory_types);  // next/prev均指向自身
                  ```

            * Dynamically inserting memory device types 动态插入设备类型​​
                * When a new memory type (e.g., a CXL device) is detected, `alloc_memory_type()` is called to create the corresponding object.
                    > 当检测到新型内存（如 CXL 设备），调用 alloc_memory_type() 创建对象。
                * The object is then linked into the appropriate tier's memory_types list using `list_add_tail()`.
                    > list_add_tail() 将其链入对应层级的 memory_types 链表。

            * Traverse devices within the same tier  遍历同层设备
                * The scheduler needs to scan devices at the same tier to select a target node.
                    > 调度器需扫描同层设备选择目标节点

    * `adistance` : abstract distance 抽象距离值
        
        *  Smaller abstract distance values imply faster (higher) memory tiers. Offset the DRAM adistance so that we can accommodate devices with a slightly lower adistance value (slightly faster) than default DRAM adistance to be part of the same memory tier.

            > 较小的抽象距离（abstract distance）值意味着更快（更高层次）的内存层级。
            通过调整（下调）DRAM 的抽象距离值，使得那些抽象距离略小于默认 DRAM 值的设备（即略快的设备）也能被归入同一内存层级。

        * Tiering logic
            * The system defines one memory tier per 128 abstract distance units (MEMTIER_CHUNK_SIZE).
                > 每 128 单位为一个层级（`MEMTIER_CHUNK_SIZE`）
            * The baseline abstract distance for DRAM is MEMTIER_ADISTANCE_DRAM = 576 (calculated as 4 × 128 + 64).
                > DRAM 基准值 `MEMTIER_ADISTANCE_DRAM=576`（即 4 * 128 + 64）

            *    When a new memory device is registered, it is automatically assigned to the appropriate memory tier via `find_create_memory_tier()`
                 > 设备注册时通过 `find_create_memory_tier()` 自动归属对应层级
    

    * `nodemask_t nodes` : NUMA node mask
        * Bitmap management
            * Use `nodemask_t` (i.e., `unsigned long[MAX_NUMNODES]`) to record the NUMA nodes that utilize this memory type.

                ```c
                node_set(node_id, &memtype->nodes);  // 添加节点
                node_clear(node_id, &memtype->nodes); // 移除节点
                ```
        * Dynamic binding
            * Upon node initialization, the `set_node_memory_tier()` function associates the node with the appropriate `memory_dev_type`.
                > 节点上线时：`set_node_memory_tier()` 将其绑定到匹配的 `memory_dev_type`
            * Upon node offline event, `clear_node_memory_tier()` removes the binding and reconstructs the downgrade (fallback) path.
                > 节点离线时：clear_node_memory_tier() 解除绑定并重建降级路径

    * `kref` : Reference counting
        * Lifecycle management ​​生命周期管理
            * **Initialization**: `kref_init(&memtype->kref)` sets the reference count to 1.
                > **初始化**​​：`kref_init(&memtype->kref)` 设置计数为 1
            * **Increment reference**: `kref_get(&memtype->kref)` (e.g., when binding a node)
                > **增加引用**​​：kref_get(&memtype->kref)（如节点绑定时）
            * **Release determination** L Destruction is triggered when the count reaches zero.
                ```c
                if (kref_put(&memtype->kref, release_memtype)) 
                        kfree(memtype); // 计数归零时触发销毁
                ```
    
    * Workflow:
        * Step 1 : Driver registration
        * Step 2 : alloc_memory_type
        * Step 3 : set `adistance`
        * Step 4 : Add (or insert) into the driver list
        * Step 5 : Node binding, `init_node_memory_type`
        * Step 6 : Insert into the memory tier list `tier_sibling`
        * Step 7 : `establish_demotion_targets`
        * Step 8 : Resource release, `put_memory_type`

    * **Step #1, #2 and #8**:
        
        **Register new memory type**
        ```c
        // Allocate memory type (e.g. CXL memory with adistance = 640)
        struct memory_dev_type *cxl_type = alloc_memory_type(640); 
        // Bind to a NUMA node
        init_node_memory_type(cxl_node_id, cxl_type);

        // Release (upon node unbinding)
        clear_node_memory_type(cxl_node_id, cxl_type);
        put_memory_type(cxl_type);
        ```

        * Related source code

            ```c
            // Allocate memory type
            struct memory_dev_type *alloc_memory_type(int adistance)
            {
                struct memory_dev_type *memtype;

                memtype = kmalloc(sizeof(*memtype), GFP_KERNEL);
                if (!memtype)
                    return ERR_PTR(-ENOMEM);

                memtype->adistance = adistance;
                INIT_LIST_HEAD(&memtype->tier_sibling);
                memtype->nodes  = NODE_MASK_NONE;
                kref_init(&memtype->kref);
                return memtype;
            }
            EXPORT_SYMBOL_GPL(alloc_memory_type);

            // Bind to a NUMA node
            void init_node_memory_type(int node, struct memory_dev_type *memtype)
            {

                mutex_lock(&memory_tier_lock);
                __init_node_memory_type(node, memtype);
                mutex_unlock(&memory_tier_lock);
            }

            // Release (upon node unbinding)
            void clear_node_memory_type(int node, struct memory_dev_type *memtype)
            {
                mutex_lock(&memory_tier_lock);
                if (node_memory_types[node].memtype == memtype || !memtype)
                    node_memory_types[node].map_count--;
                /*
                * If we umapped all the attached devices to this node,
                * clear the node memory type.
                */
                if (!node_memory_types[node].map_count) {
                    memtype = node_memory_types[node].memtype;
                    node_memory_types[node].memtype = NULL;
                    put_memory_type(memtype);
                }
                mutex_unlock(&memory_tier_lock);
            }
            EXPORT_SYMBOL_GPL(clear_node_memory_type);

            void put_memory_type(struct memory_dev_type *memtype)
            {
                kref_put(&memtype->kref, release_memtype);
            }
            EXPORT_SYMBOL_GPL(put_memory_type);
            ```

### memory_tier

```c
struct memory_tier {
	/* hierarchy of memory tiers */
	struct list_head list;
	/* list of all memory types part of this tier */
	struct list_head memory_types;
	/*
	 * start value of abstract distance. memory tier maps
	 * an abstract distance  range,
	 * adistance_start .. adistance_start + MEMTIER_CHUNK_SIZE
	 */
	int adistance_start;
	struct device dev;
	/* All the nodes that are part of all the lower memory tiers. */
	nodemask_t lower_tier_mask;
};
```

* The `memory_tier` structure represents a memory tier. Each tier contains a set of memory types (`memory_dev_type`) with similar performance. The members of this structure are as follows:
    * `list`: Used to link the current tier into the global memory tiers linked list (`memory_tiers`).
    * `memory_types`: The list head linking all memory types (memory_dev_type) that belong to this tier.
    * `adistance_start`: The starting abstract distance value of this tier. The abstract distance range covered by a tier is:
        > `[adistance_start, adistance_start + MEMTIER_CHUNK_SIZE)`  where MEMTIER_CHUNK_SIZE defaults to 128.

    * `dev`: A device structure representing this tier in sysfs, allowing user space to view tier information via sysfs.
    * `lower_tier_mask`: A nodemask representing all nodes contained in tiers lower (worse-performing) than the current tier. This mask is used as a candidate target during page demotion.

* `memory_tier`结构体代表一个内存层级。每个层级包含一组具有相似性能的内存类型（`memory_dev_type`）。以下是该结构体的成员
    * `list`: 用于将当前层级链接到全局的内存层级链表（memory_tiers）中。
    * `memory_types`: 链表头，链接属于该层级的所有内存类型（memory_dev_type）。
    * `adistance_start`: 该层级的起始抽象距离值。一个层级覆盖的抽象距离范围是   ;（其中MEMTIER_CHUNK_SIZE默认为128）。
        > `[adistance_start, adistance_start + MEMTIER_CHUNK_SIZE)`
    * `dev`: 一个设备结构体，用于在sysfs中表示该层级，这样用户空间就可以通过sysfs查看层级信息。
    * `lower_tier_mask`: 一个节点掩码（nodemask），表示所有比当前层级更低（性能更差）的层级中包含的节点。这个掩码用于在页面降级（demotion）时作为备选目标。
    

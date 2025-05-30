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


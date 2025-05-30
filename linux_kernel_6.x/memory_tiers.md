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
            struct list_head memory_types;  // 头节点
            ...
        };
        struct memory_dev_type {
            struct list_head tier_sibling;  // 成员节点
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

                list_head only contains prev and next pointers (occupying 16 bytes) and does not store any application-specific data.
                > `list_head` 仅包含 prev/next 指针（占 16 字节），不存储业务数据;

                The host structure (e.g., `memory_dev_type`) gains linked list capability by embedding a list_head member (e.g., `tier_sibling`), enabling zero-copy linkage.
                > 宿主结构（如 `memory_dev_type`）通过嵌入 `list_head` 成员（`tier_sibling`）获得链表能力，实现​​零拷贝链接​​。

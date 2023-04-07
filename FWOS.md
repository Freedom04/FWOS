# FWOS

## 1 内存管理

实现RISC-V64三级页表机制的基本步骤如下：

1. 定义页表结构体

首先需要定义一个页表结构体，包含页表项和一些其他的信息，例如页表的大小和当前使用的页表级别等。

```c
#define PAGE_SHIFT 12
#define PAGE_SIZE (1 << PAGE_SHIFT)
#define PAGE_MASK (~(PAGE_SIZE-1))

#define PTE_V     0x001
#define PTE_R     0x002
#define PTE_W     0x004
#define PTE_X     0x008
#define PTE_U     0x010
#define PTE_G     0x020
#define PTE_A     0x040
#define PTE_D     0x080
#define PTE_RSW   0x300
#define PTE_PPN_SHIFT 10
#define PTE_PPN_MASK  0xFFFFFFFFFFFFF000ull

typedef struct {
    uintptr_t ppn;  // 物理页号
    int flags;      // 页表项标志
} PTE;

typedef struct {
    PTE *table;     // 指向下一级页表的指针
    int level;      // 当前页表级别
    int count;      // 页表项数量
} PageTable;
```

2. 实现页表初始化函数

在页表初始化函数中，需要为每个页表项分配内存，并将页表项清零。

```c
void init_page_table(PageTable *pt, int level) {
    int i;

    pt->table = (PTE*)malloc(sizeof(PTE) * pt->count);
    for (i = 0; i < pt->count; i++) {
        pt->table[i].ppn = 0;
        pt->table[i].flags = 0;
    }
    pt->level = level;
}
```

3. 实现页表映射函数

页表映射函数用于将虚拟地址映射到物理地址，并设置相应的页表项标志。

```c
int map_page(PageTable *root, uintptr_t va, uintptr_t pa, int flags) {
    int i;
    PageTable *pt = root;

    for (i = 2; i >= 0; i--) {
        int index = (va >> (PAGE_SHIFT + i * 9)) & 0x1FF;
        if (!pt->table[index].ppn) {
            PageTable *new_pt = (PageTable*)malloc(sizeof(PageTable));
            new_pt->count = 512;
            init_page_table(new_pt, i);
            pt->table[index].ppn = (uintptr_t)new_pt >> PAGE_SHIFT;
            pt->table[index].flags = PTE_V | PTE_R | PTE_W | PTE_X | PTE_U;
        }
        pt = (PageTable*)((pt->table[index].ppn << PAGE_SHIFT) | 0xFFFFF000);
    }
    pt->table[(va >> PAGE_SHIFT) & 0x1FF].ppn = pa >> PAGE_SHIFT;
    pt->table[(va >> PAGE_SHIFT) & 0x1FF].flags = flags;
    return 0;
}
```

4. 实现页表解除映射函数

页表解除映射函数用于将虚拟地址从物理地址中解除映射，并清除相应的页表项标志。

```c
int unmap_page(PageTable *root, uintptr_t va) {
    int i;
    PageTable *pt = root;

    for (i = 2; i >= 0; i--) {
        int index = (va >> (PAGE_SHIFT + i * 9)) & 0x1FF;
        if (!pt->table[index].ppn) {
            return -1;
        }
        pt = (PageTable*)((pt->table[index].ppn << PAGE_SHIFT) | 0xFFFFF000);
    }
    pt->table[(va >> PAGE_SHIFT) & 0x1FF].ppn = 0;
    pt->table[(va >> PAGE_SHIFT) & 0x1FF].flags = 0;
    return 0;
}
```

5. 实现页表查找函数

页表查找函数用于查找虚拟地址对应的物理地址和页表项标志。

```c
int lookup_page(PageTable *root, uintptr_t va, uintptr_t *pa, int *flags) {
    int i;
    PageTable *pt = root;

    for (i = 2; i >= 0; i--) {
        int index = (va >> (PAGE_SHIFT + i * 9)) & 0x1FF;
        if (!pt->table[index].ppn) {
            return -1;
        }
        pt = (PageTable*)((pt->table[index].ppn << PAGE_SHIFT) | 0xFFFFF000);
    }
    *pa = (pt->table[(va >> PAGE_SHIFT) & 0x1FF].ppn << PAGE_SHIFT) | (va & PAGE_MASK);
    *flags = pt->table[(va >> PAGE_SHIFT) & 0x1FF].flags;
    return 0;
}
```

6. 实现页表销毁函数

页表销毁函数用于释放页表占用的内存。

```c
void destroy_page_table(PageTable *pt) {
    int i;

    for (i = 0; i < pt->count; i++) {
        if (pt->table[i].ppn) {
            PageTable *sub_pt = (PageTable*)((pt->table[i].ppn << PAGE_SHIFT) | 0xFFFFF000);
            destroy_page_table(sub_pt);
        }
    }
    free(pt->table);
}
```

通过实现上述函数，即可实现RISC-V64的三级页表机制。

首先，需要了解一下 RISC-V 三级页表机制的基本原理：

RISC-V 三级页表机制中，虚拟地址被分为三个部分：页全局偏移量（PGOFF）、页表偏移量（PTEOFF）和页内偏移量（OFFSET）。其中，PGOFF 表示虚拟地址所在的页在一级页表中的索引，PTEOFF 表示虚拟地址所在的页在二级页表中的索引，OFFSET 表示虚拟地址在页内的偏移量。

在实现过程中，需要定义一些数据结构，包括页表项（Page Table Entry，PTE）、页表（Page Table，PT）和页表目录（Page Table Directory，PTD）等。

以下是一个简单的示例代码，实现了 RISC-V 三级页表机制的基本功能：

```c
#include <stdio.h>
#include <stdlib.h>

#define PTE_NUM 512
#define PT_NUM 512
#define PTD_NUM 512
#define PAGE_SIZE 4096

// 定义页表项
typedef struct {
    unsigned int valid: 1; // 是否有效
    unsigned int r: 1; // 是否可读
    unsigned int w: 1; // 是否可写
    unsigned int x: 1; // 是否可执行
    unsigned int pfn: 28; // 物理页帧号
} PTE;

// 定义页表
typedef struct {
    PTE pte[PTE_NUM]; // 页表项数组
} PT;

// 定义页表目录
typedef struct {
    PT *pt[PT_NUM]; // 页表指针数组
} PTD;

// 获取页全局偏移量
#define PGOFF(va) ((va) & 0x1FF)

// 获取页表偏移量
#define PTEOFF(va) (((va) >> 9) & 0x1FF)

// 获取页表目录偏移量
#define PTD_OFF(va) (((va) >> 18) & 0x1FF)

// 获取物理地址
#define PA(ptd, va) ((ptd)->pt[PTD_OFF(va)]->pte[PTEOFF(va)].pfn * PAGE_SIZE + PGOFF(va))

// 创建页表项
PTE *new_pte() {
    PTE *pte = (PTE *) malloc(sizeof(PTE));
    pte->valid = 0;
    pte->r = 0;
    pte->w = 0;
    pte->x = 0;
    pte->pfn = 0;
    return pte;
}

// 创建页表
PT *new_pt() {
    PT *pt = (PT *) malloc(sizeof(PT));
    for (int i = 0; i < PTE_NUM; i++) {
        pt->pte[i] = *new_pte();
    }
    return pt;
}

// 创建页表目录
PTD *new_ptd() {
    PTD *ptd = (PTD *) malloc(sizeof(PTD));
    for (int i = 0; i < PT_NUM; i++) {
        ptd->pt[i] = NULL;
    }
    return ptd;
}

// 映射虚拟地址到物理地址
void map(PTD *ptd, unsigned int va, unsigned int pa, unsigned int perm) {
    // 获取页全局偏移量、页表偏移量和页表目录偏移量
    int pgo = PGOFF(va);
    int pteo = PTEOFF(va);
    int ptdo = PTD_OFF(va);

    // 如果页表目录项不存在，则创建一个新的页表
    if (ptd->pt[ptdo] == NULL) {
        ptd->pt[ptdo] = new_pt();
    }

    // 更新页表项
    PTE *pte = &(ptd->pt[ptdo]->pte[pteo]);
    pte->valid = 1;
    pte->r = perm & 0x1;
    pte->w = (perm >> 1) & 0x1;
    pte->x = (perm >> 2) & 0x1;
    pte->pfn = pa / PAGE_SIZE;
}

int main() {
    // 创建页表目录
    PTD *ptd = new_ptd();

    // 映射虚拟地址 0x80000000 到物理地址 0x10000000，权限为可读可写
    map(ptd, 0x80000000, 0x10000000, 0x3);

    // 计算虚拟地址 0x80000123 对应的物理地址
    unsigned int pa = PA(ptd, 0x80000123);

    printf("0x80000123 -> 0x%x\n", pa);

    return 0;
}
```

该示例代码创建了一个页表目录，然后调用 `map` 函数将虚拟地址 `0x80000000` 映射到物理地址 `0x10000000`，权限为可读可写。最后，通过 `PA` 宏计算出虚拟地址 `0x80000123` 对应的物理地址，并输出到控制台。

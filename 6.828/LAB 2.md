## mem_init()

- 首先调用`i386_detect_memory()`来检测有多少物理内存：

  ```
  Physical memory: 131072K available, base = 640K, extended = 130432K
  ```

  结果显示共有131072KB，也就是128MB，其中包括1MB里没有用到的640KB和1MB之上的Extended Memory部分（不清楚为什么是128MB），也就是本实验的物理内存大小为128MB

- 在没有给`PageInfo`分配内存来管理物理页时，我们先用`boot_alloc()`函数来进行内存分配

- 我们先给内核的页目录分配一页（页目录中有1024个页目录项，每个页目录项占4B，所以页目录占4KB）

  ```
  kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
  memset(kern_pgdir, 0, PGSIZE);
  ```

- 然后给内核的页目录的第`PDX(UVPT)`项初始化，与其他页目录项不同的是，该页目录项所指向的地址为**页目录的地址**，而不是页表的地址，有关这一操作的解释，见LAB 4B

  ```
  kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
  ```

- 我们使用`PageInfo`来管理物理页，每个页对应一个`PageInfo`，可以看到其包含两个成员：指向下一个free page的`PageInfo`和该页的引用数.

  ```
  struct PageInfo {
     // Next page on the free list.
     struct PageInfo *pp_link;
  
     // pp_ref is the count of pointers (usually in page table entries)
     // to this page, for pages allocated using page_alloc.
     // Pages allocated at boot time using pmap.c's
     // boot_alloc do not have valid reference count fields.
  
     uint16_t pp_ref;
  };
  ```

- 我们需要给这些`PageInfo`来分配内存。注意：kern_pgdir和page都是虚拟地址，此时页表还是采用lab 1中的页表，所以**此时虚拟地址 = 物理地址 + KERNBASE**

  ```
  pages = (struct PageInfo *)boot_alloc(npages * sizeof(struct PageInfo));
  memset(pages , 0 , npages * sizeof(struct PageInfo));
  ```

- 然后调用page_init()函数来对刚分配好的`PageInfo`进行初始化，也就是将free page进行链接，由于是头插法，所以在page_init()结束后，page_free_list指向了最后一个free page（按地址由高到低），并且此时`PageInfo`是高地址指向低地址，在check_page_free_list(1)中将链表的指向搞反，使得page_free_list指向第一个free page。

- 这样，我们就实现了最基本的物理内存管理功能：page_alloc()分配一个页，page_free()回收一个页。一系列操作都是对于free list中`PageInfo`的操作。

- 然后继续映射，不断充实kern_pgdir。

  ```
  boot_map_region(kern_pgdir , UPAGES , PTSIZE , PADDR(pages) , PTE_U);
  
  boot_map_region(kern_pgdir , KSTACKTOP - KSTKSIZE , KSTKSIZE , PADDR(bootstack) , PTE_W);
  
  boot_map_region(kern_pgdir , KERNBASE , 0x100000000 - KERNBASE , 0x0 , PTE_W);
  ```

## Physical Page Management

- 在lab1中，我们经过了BIOS和boot leader后，进入内核，并且在最后初始化了一些console devices，比如CGA、serial等等
- 在lab 2中，我们将实现一些基本的内核内存分配功能，并且初始化一个新的页目录，构建`memlayout.h`那样的虚拟内存映射，对内存进行初始化
- 在内核初始化代码中，调用了`mem_init()`来进行内存初始化

#### 虚拟地址结构

- 前10位用于在页目录中寻找页目录项，然后找到页表的起始地址，中间10位用于在页表中找到页表项，从页表项中找到页的起始地址，然后和后12位相加得到物理地址

  ```
  // +--------10------+-------10-------+---------12----------+
  // | Page Directory |   Page Table   | Offset within Page  |
  // |      Index     |      Index     |                     |
  // +----------------+----------------+---------------------+
  //  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
  //  \---------- PGNUM(la) ----------/
  ```

#### boot_alloc()

- 这个函数首先需要找到第一个没有被使用过的在Extended Memory部分的页。`nextfree`表示下一次分配时可用的第一个页的首地址

  ```
  if (!nextfree) {
     extern char end[];
     nextfree = ROUNDUP((char *) end, PGSIZE);
  }
  ```

  当第一次调用`boot_alloc()`时，我们初始化`nextfree`为end的下一页。

  ```
  .bss : {
     PROVIDE(edata = .);
     *(.bss)
     PROVIDE(end = .);
     BYTE(0)
  }
  ```

  end为bss段的结束地址，在内核的段的分配表中，我们可以看到.bss段位于内核的末尾，也就是end的地址就是内核程序的末尾地址，所以end的下一页就是Extended Memory部分的第一个free page.

  ```
  Idx Name          Size      VMA       LMA       File off  Algn
    0 .text         000040fd  f0100000  00100000  00001000  2**4
                    CONTENTS, ALLOC, LOAD, READONLY, CODE
    1 .rodata       000011d0  f0104100  00104100  00005100  2**5
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    2 .stab         000072c1  f01052d0  001052d0  000062d0  2**2
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    3 .stabstr      00001e56  f010c591  0010c591  0000d591  2**0
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    4 .data         00009300  f010f000  0010f000  00010000  2**12
                    CONTENTS, ALLOC, LOAD, DATA
    5 .got          00000008  f0118300  00118300  00019300  2**2
                    CONTENTS, ALLOC, LOAD, DATA
    6 .got.plt      0000000c  f0118308  00118308  00019308  2**2
                    CONTENTS, ALLOC, LOAD, DATA
    7 .data.rel.local 00001000  f0119000  00119000  0001a000  2**12
                    CONTENTS, ALLOC, LOAD, DATA
    8 .data.rel.ro.local 00000060  f011a000  0011a000  0001b000  2**5
                    CONTENTS, ALLOC, LOAD, DATA
    9 .bss          00000674  f011a060  0011a060  0001b060  2**5
                    CONTENTS, ALLOC, LOAD, DATA
   10 .comment      0000002b  00000000  00000000  0001b6d4  2**0
                    CONTENTS, READONLY
  ```

- 然后这个函数返回此次分配的第一个free page的首地址，并且更新`nextfree`。

#### page_alloc()

- 该函数就是从free list中取一个free `PageInfo`返回，相当于从链表的头部取下一个元素。

  ```
  struct PageInfo *
  page_alloc(int alloc_flags)
  ```

#### page_free()

- 该函数就是将一个free `PageInfo`插入到free list的头部。**注意：这里不是按顺序插入，而是先调用page_free()的先插入**，所以有可能第1，2，3号页分配了，但是2先free，那么2就指向了4。那么一次分配多个页时得到的可能不是连续的页

  ```
  void
  page_free(struct PageInfo *pp)
  ```

  问：如何判断是几号页？

  答：每个struct PageInfo *pp不同，也就是每个PageInfo的地址不同，根据地址可以算出几号页

## Virtual Memory

- 虚拟地址转化为物理地址的过程：![1693307767928](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308291916263.png)

  虚拟地址先进行段的选择，得到Linear地址，然后根据页表得到物理地址，在JOS中，由于我们在初始化GDT表时代码段和数据段的起始地址都设为了0，所以我们**这里的虚拟地址和Linear地址相同**，所以在JOS中只需要考虑虚拟地址到物理地址的过程，也就是建立页表的过程

- 下图为页表项（以及页目录表项）的结构，由于页表和页的大小都是4KB，所以高20位为(**页的起始地址 >> 12**)，且为**物理地址**，第12位为各种标志位，其中P为存在位，可以判断当前页表项是否合法

  ![img](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-10.gif)

### pgdir_walk()

- 给定一个页目录地址pgdir，找到虚拟地址va所对应的页表项的地址，如果发现页表不存在，且create == 1，那么我们需要给这个页表分配一个页，并且修改页目录表项.

  ```
  pte_t *
  pgdir_walk(pde_t *pgdir, const void *va, int create)
  {
      // 页目录表项地址
      pde_t *dic_entry_ptr = pgdir + PDX(va);
      struct PageInfo *new_page;
      // 检查present位
      if (!(*dic_entry_ptr & PTE_P)) {
          if (create) {
              new_page = page_alloc(1);
              if (!new_page) {
                  return NULL;
              }
              ++new_page->pp_ref;
              // 修改页目录表项，page2pa(new_page)得到页表的起始物理地址
              *dic_entry_ptr = (page2pa(new_page) | PTE_P | PTE_W | PTE_U);
          }
          else {
              return NULL;
          }
      }
      // 如果页表在内存中
      // page_base是页表的基地址，是虚拟地址
      pte_t *page_base = (pte_t *)KADDR(PTE_ADDR(*dic_entry_ptr));
      // 返回va对应的页表项的地址
      return page_base + PTX(va);
  }
  ```

- 由于一个页表项为32位，而`pde_t`是32位整数，所以pgdir+1表示第一个表项，page_base同理。
- 由于dic_entry_ptr指向了一个页目录表项，所以*dic_entry_ptr就表示该页目录表项，通过和PTE_P做与运算，得出该页目录表项的P标志位
- 问：如何给一个页表分配页？

​		答：通过**page_alloc函数**。但其实我们发现page_alloc函数得到的是一个`PageInfo`，所以我们需要将这个`PageInfo`的**物理地址写入页目录**，这样，就算是给页表分配了一页。

### boot_map_region()

- 将[va , va + size)这块虚拟内存与[pa , pa + size)这块物理内存进行映射，并将映射关系写入pgdir

  ```
  static void
  boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
  {
     // Fill this function in
      for (size_t i = 0 ; i < size ; i += PGSIZE) {
          pte_t *entry = pgdir_walk(pgdir , (void *)va , 1);
          *entry = (pa | perm | PTE_P);
          va += PGSIZE;
          pa += PGSIZE;
      }
  }
  ```

- 由于我们是二级页表，所以第一级映射关系是页目录->页表，其中页目录项的索引是PDX(va)，其所指向的页表的地址和va无关，以及页表项的索引是PTX(va)。最终pa所在页的物理地址保存在页表项中。其实不是va与pa的映射，而是**va所在的虚拟页和pa所在的物理页的映射**，因为页表项里包含的地址是页的地址。
- 有了boot_map_region()，我们就可以不断地给虚拟地址va映射物理地址pa
- **注意：**这个函数是用于内核在boot time分配内存，所以不会涉及pp_ref的加减，因为这些页一旦映射了，就再也不会free，所以统计这个没意义.

### page_lookup()

- 寻找虚拟地址**va所在的物理页**所对应的`PageInfo`，并将**其对应的页表项**保存在*pte_store中。

  ```
  struct PageInfo *
  page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
  {
     // Fill this function in
      pte_t *entry = pgdir_walk(pgdir , va , 1);
      if (!entry || !(*entry & PTE_P)) {
          return NULL;
      }
      struct PageInfo *pp = pa2page(PTE_ADDR(*entry));
      if (pte_store) {
          *pte_store = entry;
      }
      return pp;
  }
  ```

### page_remove()

- 取消对于va的内存映射，第一步：如果这个页的pp_ref为0了，那么就将这个页free；第二步：将页表项赋0

  ```
  void
  page_remove(pde_t *pgdir, void *va)
  {
     // Fill this function in
      pte_t *pte = NULL;
      struct PageInfo *pp = page_lookup(pgdir , va , &pte);
      if (!pp) {
          return ;
      }
      page_decref(pp);
      tlb_invalidate(pgdir , va);
      *pte = 0;
  }
  ```

### page_insert()

- 这个就是常规的页面映射，将物理页pp映射到虚拟地址va处，**注意这里有pp_ref的计算**

  ```
  int
  page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
  {
     // Fill this function in
      pte_t *entry = pgdir_walk(pgdir , va , 1);
      if (!entry) {
          return -E_NO_MEM;
      }
      ++pp->pp_ref;
      if ((*entry & PTE_P)) {
          tlb_invalidate(pgdir , va);
          page_remove(pgdir , va);
      }
      *entry = (page2pa(pp) | perm | PTE_P);
  //    pgdir[PDX(va)] |= perm;           // ?
      return 0;
  }
  ```

### 如何给一个虚拟地址va分配内存

- 首先我们需要在free list中找到物理内存块，即调用page_alloc函数
- 找到物理块后，我们调用page_insert()来实现地址映射，这样就给va分配了内存
- 地址映射的目的是能利用va找到物理地址，调用page_alloc的目的是管理物理块

### 地址映射和分配内存的关系

- 分配内存需要调用page_alloc函数从free list中得到一个free `PageInfo`，此时我们只知道其物理地址，没有其对应的虚拟地址

- 而地址映射将虚拟地址和物理地址对应起来，这个物理地址可以是已经分配了的，比如：

  ```
  boot_map_region(kern_pgdir , UPAGES , PTSIZE , PADDR(pages) , PTE_U);
  
  boot_map_region(kern_pgdir , KSTACKTOP - KSTKSIZE , KSTKSIZE , PADDR(bootstack) , PTE_W);
  ```

  其中pages和bootstack都是已经分配好内存了。

  也可以是没有分配内存，比如：

  ```
  boot_map_region(kern_pgdir , KERNBASE , 0x100000000 - KERNBASE , 0x0 , PTE_W);
  ```

  **这里有个问题：(0x100000000 - KERNBASE)这么大的内存里很多内存是没有进行内存分配的，那么为什么还能映射呢？**

  答：映射只是确定了对应关系，当我们分配内存后，得到物理地址可以很容易转为虚拟地址，而如果不进行提前映射，那么我们很难确定这个物理块要映射到哪个虚拟地址

- 那么我们什么时候需要分配内存呢？

​		比如：给一个进程的页目录分配一页，首先我们不知道页目录的虚拟地址，所以我们先分配物理页，得到物理地址，然后**根据已有的映射关系转为虚拟地址**，只有这种方法才能快速找到可用的虚拟地址

```
// Allocate a page for the page directory
if (!(p = page_alloc(ALLOC_ZERO)))
    return -E_NO_MEM;
    
e->env_pgdir = (pde_t *)page2kva(p);   
```


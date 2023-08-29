## Physical Page Management

- 在lab1中，我们经过了BIOS和boot leader后，进入内核，并且在最后初始化了一些console devices，比如CGA、serial等等
- 在lab 2中，我们将实现一些基本的内核内存分配功能，并且初始化一个新的页目录，构建`memlayout.h`那样的虚拟内存映射，对内存进行初始化
- 在内核初始化代码中，调用了`mem_init()`来进行内存初始化

### mem_init()

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
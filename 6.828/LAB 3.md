## User Environments

- 

### env_init()

- 初始化env free list，并且调用`env_init_percpu()`来初始化GDTR和各种段寄存器

  ```
  void
  env_init(void)
  ```

  新的GDT，所有的段基址都是0，并且用户的代码段和数据段的dpl都是3

  ```
  
  struct Segdesc gdt[] =
          {
                  // 0x0 - unused (always faults -- for trapping NULL far pointers)
                  SEG_NULL,
  
                  // 0x8 - kernel code segment
                  [GD_KT >> 3] = SEG(STA_X | STA_R, 0x0, 0xffffffff, 0),
  
                  // 0x10 - kernel data segment
                  [GD_KD >> 3] = SEG(STA_W, 0x0, 0xffffffff, 0),
  
                  // 0x18 - user code segment
                  [GD_UT >> 3] = SEG(STA_X | STA_R, 0x0, 0xffffffff, 3),
  
                  // 0x20 - user data segment
                  [GD_UD >> 3] = SEG(STA_W, 0x0, 0xffffffff, 3),
  
                  // 0x28 - tss, initialized in trap_init_percpu()
                  [GD_TSS0 >> 3] = SEG_NULL
          };
  
  struct Pseudodesc gdt_pd = {
          sizeof(gdt) - 1, (unsigned long) gdt
  };
  ```

### env_setup_vm()

- 给一个进程的页目录分配一个页，并且初始化该进程的页目录：UTOP以下的虚拟内存不进行映射，UTOP以上的虚拟内存映射和内核相同

  ```
  static int
  env_setup_vm(struct Env *e)
  {
      int i;
      struct PageInfo *p = NULL;
  
      // Allocate a page for the page directory
      if (!(p = page_alloc(ALLOC_ZERO)))
          return -E_NO_MEM;
          
      // LAB 3: Your code here.
      e->env_pgdir = (pde_t *)page2kva(p);
      p->pp_ref++;
      // The initial VA below UTOP is empty.
      for (i = 0 ; i < PDX(UTOP) ; ++i) {
          e->env_pgdir[i] = 0;
      }
      // The VA space of all envs is identical above UTOP
      for (i = PDX(UTOP) ; i < NPDENTRIES ; ++i) {
          e->env_pgdir[i] = kern_pgdir[i];
      }
  
      // UVPT maps the env's own page table read-only.
      // Permissions: kernel R, user R
      e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
  
      return 0;
  }
  ```

### region_alloc()

- 给环境e分配len大小的物理内存，并且将这块内存映射到虚拟内存[va , va + len)处。这个函数是给进程分配内存，**改变的是进程的空间**

  ```
  static void
  region_alloc(struct Env *e, void *va, size_t len)
  {
      // 给环境e分配len大小的物理内存，并且将这块内存映射到虚拟内存[va , va + len)处
      // LAB 3: Your code here.
      // (But only if you need it for load_icode.)
      //
      // Hint: It is easier to use region_alloc if the caller can pass
      //   'va' and 'len' values that are not page-aligned.
      //   You should round va down, and round (va + len) up.
      //   (Watch out for corner-cases!)
      // 首先明确所分配的虚拟页地址边界
      void *begin = (void *)ROUNDDOWN((uint32_t)va , PGSIZE);
      void *end = (void *)ROUNDUP((uint32_t)va + len , PGSIZE);
      struct PageInfo *p;
      // 依照页边界分配页，并且实现地址映射
      for (void *i = begin ; i < end ; i += PGSIZE) {
          p = page_alloc(0);
          if (!p) {
              panic("region alloc, allocation failed.");
          }
          int r = page_insert(e->env_pgdir , p , i , PTE_W | PTE_U);
          if (r) {
              panic("region alloc, allocation failed.");
          }
      }
  }
  ```

### load_icode()

- 进程的执行需要具体的代码和数据，而现在我们还没有实现文件系统，所以我们无法直接选一个可执行文件来让一个进程进行执行。

- JOS想出的办法是首先在user/目录下写好一些.c文件，然后编译阶段将这些.c文件与`lib/entry.S`文件一起编译成可执行文件：

  ```
  $(OBJDIR)/user/%: $(OBJDIR)/user/%.o $(OBJDIR)/lib/entry.o $(USERLIBS:%=$(OBJDIR)/lib/lib%.a) user/user.ld
     @echo + ld $@
     $(V)$(LD) -o $@ $(ULDFLAGS) $(LDFLAGS) -nostdlib $(OBJDIR)/lib/entry.o $@.o -L$(OBJDIR)/lib $(USERLIBS:%=-l%) $(GCC_LIB)
     $(V)$(OBJDUMP) -S $@ > $@.asm
     $(V)$(NM) -n $@ > $@.sym
  ```

  然后将这些可执行文件"嵌入到"内核中，也就是说这些用户可执行文件在boot leader阶段都加载进内核空间了：-b binary表示二进制文件，在编译链接后，发现`kernel.sym`中出现了`_binary_obj_user_hello_start`等符号，这些符号表示这些用户的可执行文件的开始或者结束地址。

  ```
  # How to build the kernel itself
  $(OBJDIR)/kern/kernel: $(KERN_OBJFILES) $(KERN_BINFILES) kern/kernel.ld \
       $(OBJDIR)/.vars.KERN_LDFLAGS
     @echo + ld $@
     $(V)$(LD) -o $@ $(KERN_LDFLAGS) $(KERN_OBJFILES) $(GCC_LIB) -b binary $(KERN_BINFILES)
     $(V)$(OBJDUMP) -S $@ > $@.asm
     $(V)$(NM) -n $@ > $@.sym
  ```

- binary表示的就是某一个用户可执行文件的开始地址，由于**现在可执行文件在内核空间**，我们需要将其代码段等**转移到用户空间**，并且实现地址映射。

  通过对于虚拟地址的观察，我们发现这些可执行文件位于**内核的代码段**

  ![1693394200519](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308301916533.png)<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308301917339.png" alt="1693394244565" style="zoom: 67%;" />

  这里我们可以发现我们需要先切换为进程的页目录进行地址映射，这是因为**所有进程的代码段等的虚拟地址是相同的**，但是**物理地址不同**，所以我们可以实现多进程。每个进程有自己的进程空间、页目录、**栈**。

  ```
  static void
  load_icode(struct Env *e, uint8_t *binary)
  {
      // LAB 3: Your code here.
      // 给一个环境e通过二进制Elf文件来初始化program binary, stack, and processor flags
      struct Elf *header = (struct Elf *)binary;
      if (header->e_magic != ELF_MAGIC) {
          panic("load_icode failed: The binary we load is not elf.\n");
      }
      if (header->e_entry == 0) {
          panic("load_icode failed: The elf file can't be excuterd.\n");
      }
  	
      lcr3(PADDR(e->env_pgdir));
  
      struct Proghdr *ph , *eph;
      ph = (struct Proghdr *)((uint8_t *)header + header->e_phoff);
      eph = ph + header->e_phnum;
      for ( ; ph < eph ; ++ph) {
          if (ph->p_type == ELF_PROG_LOAD) {
              if (ph->p_filesz > ph->p_memsz) {
                  panic("load icode failed : p_memsz < p_filesz.\n");
              }
  //            cprintf("ph->p_va: %x\n" , ph->p_va);
              region_alloc(e , (void *)ph->p_va , ph->p_memsz);
              // 在region_alloc中，给环境e分配了ph->p_memsz大小的物理内存，并且映射到了虚拟地址ph->p_va中
              // 其中，映射的页目录是e->env_pgdir，kern_pgdir里是没有这一映射信息的，所以需要lcr3(PADDR(e->env_pgdir));
              // 使得在memmove和memset时候使用的ph->p_va虚拟地址能够找到对应的物理地址
              memmove((void *)ph->p_va , binary + ph->p_offset , ph->p_filesz);
              memset((void *)(ph->p_va + ph->p_filesz) , 0 , ph->p_memsz - ph->p_filesz);
          }
      }
      lcr3(PADDR(kern_pgdir));
      e->env_status = ENV_RUNNABLE;
      // 该进程所要执行的代码的入口地址
      e->env_tf.tf_eip = header->e_entry;
      // Now map one page for the program's initial stack
      // at virtual address USTACKTOP - PGSIZE.
  
      // LAB 3: Your code here.
      region_alloc(e , (void *)(USTACKTOP - PGSIZE) , PGSIZE);
  }
  ```

### env_alloc()

- 分配一个进程，并将进程信息返回到*newenv_store中。

- 具体来说，首先在env free list中找到一个空闲的`Env`，然后初始化该进程的页目录，给该进程分配一个进程id，以及初始化`Env`结构中的各种信息。其中给该进程所保存的各种段选择子的RPL设置为3

  ```
  int
  env_alloc(struct Env **newenv_store, envid_t parent_id)
  
  e->env_tf.tf_ds = GD_UD | 3;
  e->env_tf.tf_es = GD_UD | 3;
  e->env_tf.tf_ss = GD_UD | 3;
  e->env_tf.tf_esp = USTACKTOP;
  e->env_tf.tf_cs = GD_UT | 3;
  ```

### env_create()

- 根据所调用的可执行文件来创建一个进程

- 分两步：首先从env free list中找到空闲的`Env e`，初始化各种信息（页表、栈、eip、各种段寄存器等）后，将该可执行文件的各种段写入e的空间中。并且e保存有该可执行文件的入口地址

  ```
  void
  env_create(uint8_t *binary, enum EnvType type)
  {
      // LAB 3: Your code here.
      struct Env *e;
      int res = env_alloc(&e , 0);
      if (res) {
          panic("env_create failed: env_alloc failed.\n");
      }
      cprintf("binary: %x\n", binary);
      load_icode(e , binary);
      e->env_type = type;
  }
  ```

### env_run()

- 选择一个进程e来执行。我们首先需要将当前运行的进程`curenv`变为就绪态，然后将e变为当前运行的进程，并且将e变为运行态，注意需要将页表切换为e的页表

  ```
  void
  env_run(struct Env *e)
  {
      if (curenv && curenv->env_status == ENV_RUNNING) {
          curenv->env_status = ENV_RUNNABLE;
      }
      curenv = e;
      curenv->env_status = ENV_RUNNING;
      ++curenv->env_runs;
      lcr3(PADDR(curenv->env_pgdir));
  
      env_pop_tf(&curenv->env_tf);
  
  }
  
  void
  env_pop_tf(struct Trapframe *tf)
  {
      // 将*tf赋给esp，那么popa指令时会将*tf内的PushRegs结构体里的值pop出来赋给相应的寄存器
      // 也就是恢复进程环境
      // iret: 中断返回，进入用户态
      asm volatile(
              "\tmovl %0,%%esp\n"
              "\tpopal\n"
              "\tpopl %%es\n"
              "\tpopl %%ds\n"
              "\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
              "\tiret\n"
              : : "g" (tf) : "memory");
      panic("iret failed");  /* mostly to placate the compiler */
  }
  
  ```

  在切换进程时，我们需要将进程e的**环境进行恢复**，也就是**恢复各种寄存器的值**，这些值保存在进程的`struct Trapframe env_tf`中，由于栈是从高地址向低地址，所以我们让esp指向`struct Trapframe`的开始，然后进行pop，将这些值恢复到寄存器中。最后iret指令将cs, eip, eflags, ss, esp等弹出，这样，就**从内核态转化为了用户态**，恢复到原来执行的地方继续执行。

  ```
  struct PushRegs {
     /* registers as pushed by pusha */
     uint32_t reg_edi;
     uint32_t reg_esi;
     uint32_t reg_ebp;
     uint32_t reg_oesp;    /* Useless */
     uint32_t reg_ebx;
     uint32_t reg_edx;
     uint32_t reg_ecx;
     uint32_t reg_eax;
  } __attribute__((packed));
  
  struct Trapframe {
     struct PushRegs tf_regs;
     uint16_t tf_es;
     uint16_t tf_padding1;
     uint16_t tf_ds;
     uint16_t tf_padding2;
     uint32_t tf_trapno;     // trap number
     /* below here defined by x86 hardware */
     uint32_t tf_err;        // error code
     uintptr_t tf_eip;
     uint16_t tf_cs;
     uint16_t tf_padding3;
     uint32_t tf_eflags;
     /* below here only when crossing rings, such as from user to kernel */
     uintptr_t tf_esp;
     uint16_t tf_ss;
     uint16_t tf_padding4;
  } __attribute__((packed));
  ```

### env_free()

- 在进程e执行时，会给e分配页目录、页表，以及给e的进程空间分配内存，并将映射关系写入页表中。所以我们需要遍历e的每个页表的每个页表项，将这些分配出去的内存回收，并且删除映射关系，并且把每个页表也进行回收，最终把页目录回收，把e变为free Env。

  ```
  void
  env_free(struct Env *e)
  ```
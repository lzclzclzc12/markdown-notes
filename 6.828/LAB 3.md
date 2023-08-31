## User Environments

- 在编译用户可执行文件时，我们发现除了.c源码文件，参与编译的还有`lib/entry.S`。原来每个用户可执行文件的入口都是`lib/entry.S`，源码如下。可见每个可执行文件都有其各自的代码段和数据段，这里我们发现定义了`envs`和`pages`等全局符号，这是与内核的`envs`和`pages`等符号相区别的，我们知道内核的数据段都在很高的内核虚拟地址（KERNBASE之上），这块地址处于用户态的普通进程是没有读写权限的，但是进程有时候会用到这些信息，所以我们需要在用户可读的区域进行虚拟内存映射，即UENVS所映射的物理地址和内核的`envs`处在的物理地址相同。![1693472324011](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308311658723.png)

  ```
  boot_map_region(kern_pgdir , UPAGES , PTSIZE , PADDR(pages) , PTE_U);
  boot_map_region(kern_pgdir , UENVS , PTSIZE , PADDR(envs) , PTE_U);
  ```

  ```
  #include <inc/mmu.h>
  #include <inc/memlayout.h>
  
  .data
  // Define the global symbols 'envs', 'pages', 'uvpt', and 'uvpd'
  // so that they can be used in C as if they were ordinary global arrays.
     .globl envs
     .set envs, UENVS
     .globl pages
     .set pages, UPAGES
     .globl uvpt
     .set uvpt, UVPT
     .globl uvpd
     .set uvpd, (UVPT+(UVPT>>12)*4)
  
  
  // Entrypoint - this is where the kernel (or our parent environment)
  // starts us running when we are initially loaded into a new environment.
  .text
  .globl _start
  _start:
     // See if we were started with arguments on the stack
     cmpl $USTACKTOP, %esp
     jne args_exist
  
     // If not, push dummy argc/argv arguments.
     // This happens when we are loaded by the kernel,
     // because the kernel does not know about passing arguments.
     pushl $0
     pushl $0
  
  args_exist:
     call libmain
  1: jmp 1b
  ```

  也就是说，进程在用户态下运行时，所运行的代码段和数据段都是该可执行文件所编译好的，并且是**独立的**，和内核没关系，在**运行时的符号也和内核没关系**。这也就解释了为什么`libmain()`中可以准确找到各种.c`umain()`的入口地址。

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

## Interrupts and Exceptions

- 中断和异常都是protected control transfers，因为发生了特定的中断和异常后，如果是用户态则转为内核态，由内核来进行处理

- 中断一般是异步事件造成的，比如IO中断；异步一般是由同步时间造成的，比如除0错误、非法内存访问等

- 操作系统将这些中断和异常按照类型进行编号，在JOS中，最多有256个，其中异常编号为0-31；其余为中断，即可以通过软中断，也就是是可以通过int指令来产生，又可以通过异步硬件中断，比如外部设备来产生。系统调用为48号（0x30）。

- 当发生中断或异常时，如果是用户态下发生，那么则转为内核态，然后由内核去执行中断处理程序。那么如何得到中断处理程序的入口地址呢？操作系统按照编号给每个中断或异常分配一个中断向量，这些中断向量都存放在中断向量表IDT中，由IDTR来存放IDT的长度和首地址，与GDT类似。

- 中断向量的结构图：依旧是**16位的段选择子和32位的偏移**作为中断处理程序的入口地址。

- ```
  // Gate descriptors for interrupts and traps
  struct Gatedesc {
     unsigned gd_off_15_0 : 16;   // low 16 bits of offset in segment
     unsigned gd_sel : 16;        // segment selector
     unsigned gd_args : 5;        // # args, 0 for interrupt/trap gates
     unsigned gd_rsv1 : 3;        // reserved(should be zero I guess)
     unsigned gd_type : 4;        // type(STS_{TG,IG32,TG32})
     unsigned gd_s : 1;           // must be 0 (system)
     unsigned gd_dpl : 2;         // descriptor(meaning new) privilege level
     unsigned gd_p : 1;           // Present
     unsigned gd_off_31_16 : 16;  // high bits of offset in segment
  };
  ```

  ![1693467355927](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308311536383.png)

- 那么我们首先应该**对IDT表进行初始化**：每个中断处理程序入口地址的段选择子都是内核代码段，并且注意**T_SYSCALL的DPL要设置为3**，这是因为只能是高特权代码访问低特权代码，我们需要在用户态下调用int指令来进入内核态，那么我们就需要访问IDT表来获取要跳转的CS:IP，此时如果DPL为0，那么用户态是无法访问这个中断向量的，也就无法获取CS:IP，那么也就无法进入内核态。

  ```
  // 初始化IDT表
  void
  trap_init(void)
  {
     extern struct Segdesc gdt[];
     // LAB 3: Your code here.
      SETGATE(idt[T_DIVIDE] , 0 , GD_KT , t_divide , 0);
      SETGATE(idt[T_DEBUG] , 0 , GD_KT , t_debug , 3);
      SETGATE(idt[T_NMI] , 0 , GD_KT , t_nmi , 0);
      SETGATE(idt[T_BRKPT] , 0 , GD_KT , t_brkpt , 3);
      SETGATE(idt[T_OFLOW] , 0 , GD_KT , t_oflow , 0);
      SETGATE(idt[T_BOUND] , 0 , GD_KT , t_bound , 0);
      SETGATE(idt[T_ILLOP] , 0 , GD_KT , t_illop , 0);
      SETGATE(idt[T_DEVICE] , 0 , GD_KT , t_device , 0);
      SETGATE(idt[T_DBLFLT] , 0 , GD_KT , t_dblflt , 0);
      SETGATE(idt[T_TSS] , 0 , GD_KT , t_tss , 0);
      SETGATE(idt[T_SEGNP] , 0 , GD_KT , t_segnp , 0);
      SETGATE(idt[T_STACK] , 0 , GD_KT , t_stack , 0);
      SETGATE(idt[T_GPFLT] , 0 , GD_KT , t_gpflt , 0);
      SETGATE(idt[T_PGFLT] , 0 , GD_KT , t_pgflt , 0);
      SETGATE(idt[T_FPERR] , 0 , GD_KT , t_fperr , 0);
      SETGATE(idt[T_ALIGN] , 0 , GD_KT , t_align , 0);
      SETGATE(idt[T_MCHK] , 0 , GD_KT , t_mchk , 0);
      SETGATE(idt[T_SIMDERR] , 0 , GD_KT , t_simderr , 0);
      SETGATE(idt[T_SYSCALL] , 0 , GD_KT , t_syscall , 3);
  
     // Per-CPU setup 
     trap_init_percpu();
  }
  
  // Initialize and load the per-CPU TSS and IDT
  void
  trap_init_percpu(void)
  {
  	// Setup a TSS so that we get the right stack
  	// when we trap to the kernel.
  	ts.ts_esp0 = KSTACKTOP;
  	ts.ts_ss0 = GD_KD;
  	ts.ts_iomb = sizeof(struct Taskstate);
  
  	// Initialize the TSS slot of the gdt.
  	gdt[GD_TSS0 >> 3] = SEG16(STS_T32A, (uint32_t) (&ts),
  					sizeof(struct Taskstate) - 1, 0);
  	gdt[GD_TSS0 >> 3].sd_s = 0;
  
  	// Load the TSS selector (like other segment selectors, the
  	// bottom three bits are special; we leave them 0)
  	ltr(GD_TSS0);
  
  	// Load the IDT
  	lidt(&idt_pd);
  }
  ```

  用户通过查询IDT来获取内核中断处理程序的CS:IP，但是切换到内核态还需要切换为内核栈，这需要内核栈的SS:ESP，这个信息由TSS提供，访问TSS的方式如下图：也就是说我们可以通过访问GDT来访问TSS。所以在`trap_init_percpu(void)`函数中添加TSS段的信息，并且由于我们只有一个内核，所以内核的SS和ESP存放在SS0和ESP0处。<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308311558529.png" alt="1693468720759"  />

- **定义中断处理程序**。我们**通过宏定义`TRAPHANDLER`和`TRAPHANDLER_NOEC`来定义全局符号`t_divide`等为函数**，并且每个中断处理函数都统一跳转到`_alltraps`处。即在`trapentry.S`中定义好中断处理函数，在`trap_init()`函数中引用这些函数的地址

  ```
  // int 指令先将TSS中指示的内核ss,esp取出，转入内核的栈，然后把ss,esp,eflags, cs,eip以及一个可选的error code入栈
  // TSS中的ss, esp信息已经存在了GDT表的相关表项中
  // 得到的中断向量和IDTR中存好的IDT基地址从IDT表中取出中断处理程序的入口地址
  #define TRAPHANDLER(name, num)                \
     .globl name;      /* define global symbol for 'name' */  \
     .type name, @function; /* symbol type is function */     \
     .align 2;     /* align function definition */       \
     name:        /* function starts here */    \
     pushl $(num);                    \
     jmp _alltraps
  
  /* Use TRAPHANDLER_NOEC for traps where the CPU doesn't push an error code.
   * It pushes a 0 in place of the error code, so the trap frame has the same
   * format in either case.
   */
  #define TRAPHANDLER_NOEC(name, num)                \
     .globl name;                     \
     .type name, @function;                \
     .align 2;                    \
     name:                       \
     pushl $0;                    \
     pushl $(num);                    \
     jmp _alltraps
  
  .text
  
  /*
   * Lab 3: Your code here for generating entry points for the different traps.
   */
  // 真正对于中断服务程序的实现，global symbol
  TRAPHANDLER_NOEC(t_divide, T_DIVIDE)
  TRAPHANDLER_NOEC(t_debug, T_DEBUG)
  TRAPHANDLER_NOEC(t_nmi, T_NMI)
  TRAPHANDLER_NOEC(t_brkpt, T_BRKPT)
  TRAPHANDLER_NOEC(t_oflow, T_OFLOW)
  TRAPHANDLER_NOEC(t_bound, T_BOUND)
  TRAPHANDLER_NOEC(t_illop, T_ILLOP)
  TRAPHANDLER_NOEC(t_device, T_DEVICE)
  TRAPHANDLER(t_dblflt, T_DBLFLT)
  TRAPHANDLER(t_tss, T_TSS)
  TRAPHANDLER(t_segnp, T_SEGNP)
  TRAPHANDLER(t_stack, T_STACK)
  TRAPHANDLER(t_gpflt, T_GPFLT)
  TRAPHANDLER(t_pgflt, T_PGFLT)
  TRAPHANDLER_NOEC(t_fperr, T_FPERR)
  TRAPHANDLER(t_align, T_ALIGN)
  TRAPHANDLER_NOEC(t_mchk, T_MCHK)
  TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)
  
  TRAPHANDLER_NOEC(t_syscall, T_SYSCALL)
  ```

  在进入内核之前，用户态代码通常调用int指令来切换到内核态，int 指令先将TSS中指示的内核ss,esp取出，转入内核的栈，然后把ss,esp,eflags, cs,eip以及一个可选的error code入栈，下面这是int之后内核栈的变化：<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308311616523.png" alt="1693469811679" style="zoom: 80%;" />

  但是光保存这些信息对于进程恢复来说是不够的，我们需要参照`inc/trap.h`中的`struct Trapframe`来进行信息保存。

  ```
  _alltraps:
  	//首先保存进程的各种寄存器信息
      pushl %ds
      pushl %es
      pushal
  
  	// 修改DS和ES为内核数据段
      movl $GD_KD , %eax
      movl %eax , %ds
      movl %eax , %es
  
      pushl %esp
      call trap
  ```

  由于`trap`函数需要参数`struct Trapframe *`，而现在esp指向的正是一个完整的`struct Trapframe`的首地址，所以把esp push进去。

  ```
  void
  trap(struct Trapframe *tf)
  ```

### trap()

- JOS中所有的中断处理程序都要进入trap函数中，并且中断号保存在`tf->tf_trapno`中。但是现在有个问题，就是现在保存的进程环境是在内核的栈中，而内核在不断运行时会不断进行压栈等操作，如果要进行进程环境恢复，直接使用栈里保存的值是很不方便的，所以我们需要将栈里保存的值复制到专门保存该进程的空间中：

  ```
  // 拷贝一份
  curenv->env_tf = *tf;
  // The trapframe on the stack should be ignored from here on.
  tf = &curenv->env_tf;
  ```

  ```
  void
  trap(struct Trapframe *tf)
  {
     // The environment may have set DF and some versions
     // of GCC rely on DF being clear
     asm volatile("cld" ::: "cc");
  
     // Check that interrupts are disabled.  If this assertion
     // fails, DO NOT be tempted to fix it by inserting a "cli" in
     // the interrupt path.
     assert(!(read_eflags() & FL_IF));
  
     cprintf("Incoming TRAP frame at %p\n", tf);
  
     if ((tf->tf_cs & 3) == 3) {
        // Trapped from user mode.
        assert(curenv);
  
        // Copy trap frame (which is currently on the stack)
        // into 'curenv->env_tf', so that running the environment
        // will restart at the trap point.
        curenv->env_tf = *tf;
        // The trapframe on the stack should be ignored from here on.
        tf = &curenv->env_tf;
     }
  
     // Record that tf is the last real trapframe so
     // print_trapframe can print some additional information.
     last_tf = tf;
  
     // Dispatch based on what type of trap occurred
     trap_dispatch(tf);
  
     // Return to the current environment, which should be running.
     assert(curenv && curenv->env_status == ENV_RUNNING);
     env_run(curenv);
  }
  ```

  之后调用`trap_dispatch(tf);`来根据`tf->tf_trapno`中保存的中断向量号执行相应的处理程序。最后调用`env_run(curenv);`返回用户态。
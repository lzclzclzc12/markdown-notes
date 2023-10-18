## Copy-on-Write Fork

- 在我们之前实现的`dumbfork`中，子进程不光是各种寄存器的值和父进程相同，子进程的进程空间和栈空间通过父进程的进程空间和栈空间拷贝而来，也就是说子进程和父进程的进程空间和栈空间的虚拟地址相同，但是物理地址不同（新给子进程分配了物理空间，但内容是拷贝父进程的）。但是通常情况下子进程在fork出来之后要经过exec()来执行新的可执行文件，也就是说子进程从父进程中拷贝过来的内容很快就被新执行的可执行文件所覆盖，这样的话前面所进行的拷贝操作就没有意义了。

- 所以JOS采用COW Fork。参考：[Linux fork的写时复制(copy on write)](https://blog.csdn.net/qq_34556414/article/details/108399543)

- 具体来说，子进程并不先拷贝父进程的空间内容，而是**映射相同的虚拟地址到相同的物理地址**，这篇区域称为共享区，并且父进程和子进程中关于这片区域的权限设置为PTE_COW，如果父进程或子进程想要对共享区进行写操作，那么由于没有PTE_W权限但是有PTE_COW权限，所以触发COW Fork中特有的页错误处理函数，给父进程或子进程在想要写入的共享虚拟空间新分配一块物理空间，并且权限为PTE_W。

- 子进程和父进程在相同虚拟进程空间映射到相同的物理空间。对于父进程只读的空间还是保留成只读，对于可读空间权限变为PTE_COW

  ```
  static int
  duppage(envid_t envid, unsigned pn)
  {
     int r;
  
     // LAB 4: Your code here.
      void *va = (void *)(pn * PGSIZE);
      if ((uvpt[pn] & PTE_W) || (uvpt[pn] & PTE_COW)) {
  
          if ((r = sys_page_map(0 , va , envid , va , PTE_P | PTE_U | PTE_COW)) < 0) {
              return r;
          }
          if ((r = sys_page_map(0 , va , 0 , va , PTE_P | PTE_U | PTE_COW)) < 0) {
              return r;
          }
      } else {
          if ((r = sys_page_map(0 , va , envid , va , PTE_P | PTE_U)) < 0) {
              return r;
          }
      }
     return 0;
  }
  ```

- 如果是想要对PTE_COW区域进行写，那么触发页错误处理程序，先检查是否是由于对PTE_COW区域进行写而产生的页错误，然后给产生错误的进程分配新的物理空间，并且把PTE_COW区域的内容拷贝过去，修改权限为PTE_W

  ```
  static void
  pgfault(struct UTrapframe *utf)
  {
     void *addr = (void *) utf->utf_fault_va;
     uint32_t err = utf->utf_err;
     int r;
  
     // LAB 4: Your code here.
      if (!((err & FEC_WR) && (uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P) &&
              (uvpt[PGNUM(addr)] & PTE_COW))) {
          panic("must be above reason");
      }
  
     // LAB 4: Your code here.
      // Allocate a new page, map it at a temporary location (PFTEMP)
      if ((r = sys_page_alloc(0 , PFTEMP , PTE_W | PTE_P | PTE_U)) < 0) {
          panic("alloc map error, error nubmer is %d\n",r);
      }
      addr = ROUNDDOWN(addr , PGSIZE);
      // copy the data from the old page to the new page
      memcpy((void *)PFTEMP , addr , PGSIZE);
  
      // move the new page to the old page's address.
      if ((r = sys_page_map(0 , (void *)PFTEMP , 0 , addr , PTE_W | PTE_P | PTE_U)) < 0) {
          panic("sysmap error");
      }
      if ((r = sys_page_unmap(0 , (void *)PFTEMP)) < 0) {
          panic("page_unmap error");
      }
  }
  ```

## User-level page fault handling

- 当发生页错误时，会触发页错误中断，从用户态转为内核态。但是页错误的原因有很多，那么处理函数也相应因该有很，如果这些都让内核来实现的话，那么内核需要知道很多信息来判断执行哪个处理程序，引用LAB 4中的一段话：

  > It's **common** to set up an address space so that page faults indicate when some action needs to take place. For example, most Unix kernels initially map only a single page in a new process's stack region, and allocate and map additional stack pages later "on demand" as the process's stack consumption increases and causes page faults on stack addresses that are not yet mapped. **A typical Unix kernel must keep track of what action to take when a page fault occurs in each region of a process's space**. For example, **a fault in the stack region** will typically allocate and map new page of physical memory. **A fault in the program's BSS region** will typically allocate a new page, fill it with zeroes, and map it. In systems with demand-paged executables, **a fault in the text region** will read the corresponding page of the binary off of disk and then map it.
  >
  > This is a lot of information for the kernel to keep track of. Instead of taking the traditional Unix approach, you will decide what to do about each page fault in user space, where bugs are less damaging. **This design has the added benefit of allowing programs great flexibility in defining their memory regions**; you'll use user-level page fault handling later for mapping and accessing files on a disk-based file system.

- 所以JOS采用在用户级别进行页处理，这样用户就可以自定义页错误处理程序了。也就是说**页错误处理程序在用户态下执行**，而不是内核态

- 我们给每个进程分配一个异常处理栈UXSTACK，当执行页错误处理程序时，就切换为UXSTACK。并且这个栈保留有关于页错误的各种信息，这些信息整合起来构成结构体`struct UTrapframe`，并且每个进程的信息块Env中都保留有`void *env_pgfault_upcall;`信息，这个指针指向**所有用户级别页错误程序的入口地址**。

- 首先我们需要修改内核态调用的页错误处理程序，也就是在发生页错误时，首先进入内核态，内核先调用这个函数。可以看到，内核判断出是用户态下引发的页错误并且发生错误的进程已经实现了用户级别的页错误处理程序时，就先在UXSTACK上写入页错误相关的各种信息（**utf指向的是UXSTACK空间**），比如错误地址、发生错误时进程的上下文环境等等。

  值得注意的是：这个函数的参数tf是发生页错误的进程的上下文信息，我们在恢复进程时要用到，但是这里却**修改了其esp和eip**，当我们执行`env_run(curenv);`时，从内核态返回为了用户态，**返回的栈是UXSTACK，返回地址为用户自定义的页错误处理函数**。那么我们就开始能开始执行用户级别的页错误处理程序了。

  ```
  tf->tf_eip = (uintptr_t)curenv->env_pgfault_upcall;
  tf->tf_esp = (uintptr_t)utf;
  ```

  ```
  void
  page_fault_handler(struct Trapframe *tf)
  {
     uint32_t fault_va;
  
     // Read processor's CR2 register to find the faulting address
     fault_va = rcr2();
  
     // Handle kernel-mode page faults.
  
     // LAB 3: Your code here.
      if ((tf->tf_cs & 0x3) == 0) {
          panic("page_fault in kernel mode, fault address %d\n", fault_va);
      }
  
     // Destroy the environment that caused the fault.
      if (curenv->env_pgfault_upcall) {
          uintptr_t xstack_top = UXSTACKTOP;
          if (tf->tf_esp < UXSTACKTOP && tf->tf_esp > UXSTACKTOP - PGSIZE) {
              // 这种情况是当异常发生时，用户环境已经在异常栈上了，也就是处理异常时发生了异常
              xstack_top = tf->tf_esp;
          }
          uint32_t size = sizeof(struct UTrapframe) + sizeof(uint32_t);
          // 检查XSTACK是否越界
          user_mem_assert(curenv , (void *)(xstack_top - size) , size , PTE_U | PTE_W);
  
          //结构体内存分配是从低到高的
          struct UTrapframe *utf = (struct UTrapframe *)(xstack_top - size);
          utf->utf_fault_va = fault_va;
          utf->utf_err = tf->tf_err;
          utf->utf_regs = tf->tf_regs;
          // 返回地址，从异常栈到用户栈，这里和tf的eip和esp相同，因为发生异常时
          // 先从用户态变为内核态，tf里已经保存了之前的返回地址了
          utf->utf_eip = tf->tf_eip;
          utf->utf_eflags = tf->tf_eflags;
          utf->utf_esp = tf->tf_esp;
  
          // 执行异常处理程序
          tf->tf_eip = (uintptr_t)curenv->env_pgfault_upcall;
          tf->tf_esp = (uintptr_t)utf;
          env_run(curenv);
      }
      else {
          // Destroy the environment that caused the fault.
          cprintf("[%08x] user fault va %08x ip %08x\n",
                  curenv->env_id, fault_va, tf->tf_eip);
          print_trapframe(tf);
          env_destroy(curenv);
      }
  }
  ```

- 我们可以在用户态下设置该进程所执行程序的用户级别的页错误处理程序。`sys_env_set_pgfault_upcall`系统调用是设置当前进程的页错误中断处理函数的入口地址，保存在`env->env_pgfault_upcall`中；

  ```
  void
  set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
  {
     int r;
  
     if (_pgfault_handler == 0) {
        // First time through!
        // LAB 4: Your code here.
        if (sys_page_alloc(0, (void *)(UXSTACKTOP-PGSIZE), PTE_W | PTE_U | PTE_P) < 0) {
           panic("set_pgfault_handler:sys_page_alloc failed！\n");
        }
        sys_env_set_pgfault_upcall(0, _pgfault_upcall);
     }
  
     // Save handler pointer for assembly to call.
     _pgfault_handler = handler;
  }
  ```

- 接下来执行用户级别的页错误处理程序入口，可知`_pgfault_handler`才是真正的用户级别页错误处理程序，`_pgfault_upcall`起到统一分发的作用。

  首先esp指向的是UXSTACKTOP中保存的错误信息的底部，也对应着`struct UTrapframe`的首地址，所以我们`pushl %esp`把`_pgfault_handler`的入口参数入栈，执行完之后，我们把入栈的参数出栈，此时esp依然指向的是UXSTACKTOP中保存的错误信息的底部

  ```
                      <-- UXSTACKTOP
  trap-time esp
  trap-time eflags
  trap-time eip
  trap-time eax       start of struct PushRegs
  trap-time ecx
  trap-time edx
  trap-time ebx
  trap-time esp
  trap-time ebp
  trap-time esi
  trap-time edi       end of struct PushRegs
  tf_err (error code)
  fault_va            <-- %esp when handler is run
  ```

  接下来看注释吧：

  ```
  .text
  .globl _pgfault_upcall
  _pgfault_upcall:
     // Call the C page fault handler.
     pushl %esp       // function argument: pointer to UTF
     movl _pgfault_handler, %eax
     call *%eax
     addl $4, %esp        // pop function argument
     
     // 到这里页错误处理程序完成了，我们应该返回到发生错误的进程的程序处继续执行
     // 所以我们需要恢复上下文环境
     // 我们用ret来返回，注意ret需要从esp指向的地址中取到eip，所以我们先要保存eip到返回的栈底
     // 也就是在返回的栈底预留4B来保存eip
      addl $8 , %esp       // 忽略utf_fault_va和utf_err
      // 此时先不急着popa，因为我们需要用到eax和ebx这些通用寄存器
      movl 0x20(%esp) , %eax   // utf_eip到 eax
      movl 0x28(%esp) , %ebx   // utf_esp到 ebx
      subl $4 , %ebx      // 用户栈底 - 4，也就是用户栈预留4字节的空间
      movl %ebx , 0x28(%esp)   // 改变用户栈的栈底指针
      movl %eax , (%ebx)  // 将返回地址填入用户栈的预留4字节的空间内
  
      popal
  
      add $4 , %esp
      popfl
  
      popl %esp
  
      ret
  ```

- ret 是普通的子程序的返回指令。
  也可以叫做近返回，即段内返回。处理器从堆栈中弹出IP或者EIP，然后根据当前的CS：IP跳转到新的执行地址。如果之前压栈的还有其余的参数，则这些参数也会被弹出。所以上述汇编除了我们需要自行切换为用户栈（popl %esp），还需要我们将用户程序的返回地址写入到栈里（movl %eax , (%ebx)），因为正常的函数调用esp是不需要我们自己赋值的，它经过pop和push就自然正确了，相反需要修改ebp

  iret 是中断服务子程序的返回指令。
  用于从中断返回，会弹出IP/EIP，ss和esp，然后CS，以及一些标志。然后从CS：IP执行，也就是切换栈和eip。

- 这样，我们就可以在不同的用户程序段实现不同的用户级别的页错误处理程序，而不需要内核来追踪这些信息了

## Exercise 8-11

- `sys_env_set_pgfault_upcall`系统调用是设置当前环境的页错误中断处理函数的入口地址，保存在`env->env_pgfault_upcall`中；
- 首先我们在用户态下调用`set_pgfault_handler(handler);`来设置当前环境的页错误中断处理函数的入口地址，并且为当前环境分配exception stack；并且设置用户自定义的handler函数`_pgfault_handler`
- 当发生页错误时，进入内核态，并且根据`IDT`调用`TRAPHANDLER(t_pgflt, T_PGFLT)`函数，然后根据`T_PGFLT`调用`page_fault_handler`函数
- `page_fault_handler`函数将当前用户的`UTrapframe`保存在exception stack中，当页错误处理程序结束想要返回原程序时，需要`UTrapframe`提供的上下文信息来恢复环境。之后设置当前的eip为页处理程序入口地址，esp为exception stack的栈底，然后调用`env_run(curenv);`来切换到执行异常处理程序
- 页错误处理程序的统一入口为`pfentry.S`中的`_pgfault_upcall`，该函数调用`_pgfault_handler`，之后使用保存好的`UTrapframe`来恢复到原程序

## `UVPT`的理解

- 首先页目录的每个页目录项占4B，其值为其所指向页表的页的起始地址；

- 页目录和页表一样也是占4KB，即一页大小

- 所以`kern_pgdir`等变量其低12位都是0，因为其指向的是一个页的起始地址

- ```
  // 下面这句代码是给内核的页目录添加了一个页目录项，其值为页目录所在页的起始地址
  kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
  ```

## 访问`UVPT[PGNUM(va)]`的过程

- `uvpt[PGNUM(va)] = uvpt +( va >> 12 * 4)`
- `UVPT`的值为0xef400000，变为二进制为1110111101 0000000000 000000000000
- 也就是说，`UVPT`的后22位都是0，那么`UVPT[PGNUM(va)]` = 3BD <> PDX(va) <> PTX(va) <> 00(二进制)
- 首先CR3先根据PDX（这里是3BD）找到对应的页目录项，其值为PADDR(kern_pgdir) | PTE_U | PTE_P，即页目录所在页的起始地址
- 然后找到页目录所在页，再根据PTX（这里为PDX(va)）找到对应的页目录表项，其值为`va`所对应的页表的起始地址
- 最后根据OFF（这里是PTX(va) <> 00(二进制)）从该页表中找到`va`所对应的页表项的地址，最后有两个0是因为每个页表项是4B
- 所以`UVPT[PGNUM(va)]`最终得到的是`va`所对应的页表项的地址
- 这里之所以会找到页目录项的原因是一般情况下CR3先根据PDX找到的是页表的起始地址，但是**这里的3BD页目录项所指向的是页目录的起始地址**
- 根据上述分析，`UVPT[0]`即`UVPT`所指向的是**第0个页表的第0个页表项的地址**

## 关于`UVPD`的理解

- `uvpt[PGNUM(va)] = uvpt +( va >> 12 * 4)` = PDX(va) <> PTX(va)所表示的就是第PDX(va)个页表的第PDX(va)个页表项的地址
- `pde_t uvpd[] = UVPT+(UVPT>>12)*4;`
- 那么`uvpd[0]`即`uvpd`就是第PDX(uvpt)个页表的第PDX(uvpt)个页表项的地址，这里PDX(uvpt) = 3BD，PDX(uvpt) = 0，也就是**第0个页目录表项的起始地址**
- `UVPD[PDX(va)]`表示va所对应的页目录项的起始地址

## Inter-Process communication(IPC)

- 实现进程的相互交流，我们需要实现两个系统调用`sys_ipc_recv`和`sys_ipc_try_send`，分别用于**准备接受**和发送信息

- 注意：`sys_ipc_recv`并不是我一开始理解的一直while循环来接受，而是设置进程信息块`Env`的相关变量，**告诉发送进程该进程正在等待别的进程发送信息**。如下面代码所示，这个系统调用就是用来设置接收者为准备接收状态，然后进程状态改为阻塞态，切换为其他进程。其中如果`dstva`为合法地址的话，表示可以接受一个**共享页**，也就是如果发送方发来一页，那么接收方虚拟地址`dstva`映射到和发送方传来的页的物理地址上，发送方和接收方都映射到了同一物理地址，实现了共享。

  ```
  static int
  sys_ipc_recv(void *dstva)
  {
     // LAB 4: Your code here.
      if (((uintptr_t) dstva < UTOP) && ((uintptr_t)dstva & (PGSIZE  - 1))) {
          return -E_INVAL;
      }
      struct Env *cur_env;
      envid2env(0,&cur_env,0);
      assert(cur_env);
  	// 阻塞态
      cur_env->env_status = ENV_NOT_RUNNABLE;
      // 设置接收者为准备接收状态
      cur_env->env_ipc_recving = 1;
      cur_env->env_ipc_dstva = dstva;
      sys_yield();
      return 0;
  }
  ```

- 发送方先检查接收方是否为准备接收状态，并且如果接收方同意接受一页数据，那么传递一页数据（将接收方的虚拟地址映射到所传递的页的物理地址上），并且修改接收方的相关信息，将接收方的状态改为就绪态，相当于**唤醒**操作。

  ```
  static int
  sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
  {
     // LAB 4: Your code here.
      struct Env *rec_env, *cur_env;
      int r;
      // get current env
      envid2env(0,&cur_env,0);
      assert(cur_env);
      if ((r = envid2env(envid, &rec_env,0)) < 0) {
          return r;
      }
      if (!rec_env->env_ipc_recving) {
          return -E_IPC_NOT_RECV;
      }
      if ((uintptr_t)srcva < UTOP) {
          struct PageInfo *pg;
          pte_t *pte;
          // if srcva is not page-aligned
          // 0x1000 - 1=  0x0111
          // if srcva any bit is 1 is not page-aligned
          if ((uintptr_t) srcva & (PGSIZE - 1)) {
              return -E_INVAL;
          }
          // perm is inappropriate is same as the sys_page_alloc
          if(!(perm & PTE_U) || !(perm & PTE_P) || (perm & (~PTE_SYSCALL))){
              return -E_INVAL;
          }
          // srcva is mapped in the caller's address spcae
          if (!(pg = page_lookup(cur_env->env_pgdir,srcva,&pte))) {
              return -E_INVAL;
          }
          // if (perm & PTE_W), but srcva is read-only
          if ((perm & PTE_W) && !((*pte) & PTE_W)) {
              return -E_INVAL;
          }
          if ((uintptr_t)rec_env->env_ipc_dstva < UTOP) {
              if ((r = page_insert(rec_env->env_pgdir,pg,rec_env->env_ipc_dstva,perm)  < 0)) {
                  return -r;
              }
          }
      }
      rec_env->env_ipc_perm = perm;
      rec_env->env_ipc_value = value;
      rec_env->env_ipc_recving = 0;
      rec_env->env_ipc_from = cur_env->env_id;
      rec_env->env_status = ENV_RUNNABLE;
      rec_env->env_tf.tf_regs.reg_eax = 0;
      return 0;
  }
  ```

- 在用户态下，发送方不断发送，直到发送成功。

  ```
  void
  ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
  {
     // LAB 4: Your code here.
      if (!pg) {
          pg = (void *)UTOP;
      }
      int r;
      while((r = sys_ipc_try_send(to_env,val,pg,perm)) < 0) {
          if (r != -E_IPC_NOT_RECV) {
              panic("sys_ipc_try_send error %e\n",r);
          }
          sys_yield();
      }
  }
  ```

- 用户态下，接收方接收到信息后返回接收到的信息

  ```
  int32_t
  ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
  {
     // LAB 4: Your code here.
      int error;
      if(!pg)pg = (void *)UTOP; //
      if((error = sys_ipc_recv(pg)) < 0){
          if(from_env_store)*from_env_store = 0;
          if(perm_store)*perm_store = 0;
          return error;
      }
  
      if(from_env_store)*from_env_store = thisenv->env_ipc_from;
      if(perm_store)*perm_store = thisenv->env_ipc_perm;
      return thisenv->env_ipc_value;
  }
  ```

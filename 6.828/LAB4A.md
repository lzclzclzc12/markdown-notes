### JOS多处理器支持

- JOS支持“symmetric multiprocessing”(SMP)，所有CPU能够平等地共享系统资源
- 这些CPU在boot阶段分为两种：the bootstrap processor(BSP) 和 application processors (APs)
- 其中BSP负责boot 操作系统(boot load 和加载内核)，之后激活APs
- 哪一个CPU是BSP由硬件和BIOS决定，JOS初始化内核代码啥的在BSP上运行

### LAPIC

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20191208110634629.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nOTg3MjQ2NTEw,size_16,color_FFFFFF,t_70)
- LAPIC位于CPU核心，不属于外设，它接收IOAPIC经Host桥发送来的中断信息，对其进行处理。
- IOAPIC属于外设，不在CPU核内，接收来自ISA设备或者其它pci设备的中断信号，通过它和LAPIC模块之间专用的APIC bus将中断信息传递给LAPIC。
- IOAPIC发送的中断消息包含两个关键信息：中断要发到哪个CPU（APIC ID），中断向量是多少（VECTOR）。
- 每个CPU都有一个LAPIC单元，LAPIC中提供了本CPU的ID
- LAPIC存在于物理内存的0xFEE00000，并且存在于MMIO区域，JOS将MMIO区域映射到`MMIOBASE`

### Application Processor Bootstrap

- 在BSP启动APs之前，需要知道一些信息，比如CPU的数量，CPU的APIC ID，以及每个CPU的LAPIC地址
- 在**kern/mpconfig.c中的mp_init()函数**获取了这些信息，通过读取BIOS区域的MP configuration table，the MP configuration table contains explicit configuration information about APICs, processors, buses, and interrupts.  The table consists of a **header**, followed by **a number of entries of various types**.  The format and length of each entry depends on its type.
- The MP configuration table is filled in by the BIOS after it executes a CPU identification procedure on each of the processors in the system.
- kern/init.c中的boot_aps()函数负责启动APs，在boot/boot.S启动操作系统时，将BSP从实模式变为保护模式。APs一开始也运行在实模式下，然后APs的Boot Loader **kern/mpentry.S**将其变为保护模式，boot/boot.S的入口地址是由BIOS设置的，同样，boot_aps()需要将kern/mpentry.S的入口地址放到一个实模式可以检测到的地址，在JOS中是0x7000(MPENTRY_PADDR)物理地址
- 在入口地址放入内存后boot_aps()开始向LAPIC发送STARTUP IPIs以及MPENTRY_PADDR
- **kern/mpentry.S**将APs从实模式变为保护模式，开启分页，设置AP的栈，最后跳转到**kern/init.c的mp_main()函数**中



### 一些疑惑

- 关于MP config的查找为什么EBDA的物理地址要是实模式下的20位？

### **Exercise 6**：在创建2个yield环境后，使用1个CPU后的输出结果为：

- ![1692016829016](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308142101718.png)
- 当创建好环境之后，内核启动环境0，环境0执行用户态代码，其中sys_yield函数启动系统调用，所以进入内核态，**返回地址为进入内核态之前指令的下一条地址**，所以当环境1再次切换为环境0时，执行的是”Back in environment...“而不是”Hello, I am environment...“。
- 当环境0做完执行exit()之后，此时处于内核态，并且调用sched_yield()启动环境1
- 当环境1也退出之后，执行sched_yield()中的sched_halt()，发现没有可用的环境，就启动monitor(NULL)函数进入命令行

#### 收获

##### 收获1：切换环境时，环境的上下文如何保存与恢复？

- 保存：当我们的环境要从用户态转为内核态时，首先在进入内核态后将环境的各种寄存器入内核栈，然后将内核栈保存的上下文环境拷贝到`curenv->env_tf`中，其中`curenv`表示当前CPU所运行的环境
- 恢复：`env_pop_tf`函数中利用`curenv->env_tf`进行pop来恢复寄存器
- 环境切换（比如环境1切换到环境2）：需要系统调用，进入内核态，将环境1的上下文保存到`cpus[cpunum()].env_tf`中，然后恢复环境2。

##### 收获2：int指令进入内核态干了什么？

- 首先我们在int指令之前加入断点。![1692018332758](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308142106178.png)
- 可以看到当前esp和eip的值，都是用户态下的
- 然后我们执行int指令。![1692018478337](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308142108317.png)
- 可以看到esp已经是内核态下的高地址了，并且之前用户态下的eip，esp等寄存器(称为old eip和old esp)的值都存在内核的栈中了，其中old eip为用户态int指令的下一条地址
- 因此，中断通过int指令后，内核栈发生的变化为：![img](https://pic3.zhimg.com/v2-d345f8032b7ca104c8c85a2989938712_r.jpg)
- 其中内核的esp和ss由TSS提供
- 每个环境的都有**各自的页目录**，其**用户栈的虚拟地址都是一样的，但是指向不同的物理地址**，`region_alloc`给用户栈分配一个页的空间![1692018561837](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308142109627.png)
- 每个环境（包括内核）的页目录都的地址映射都存在虚拟地址UVPT处,：![1692172332673](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308161552915.png)<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308161553748.png" alt="1692172363101" style="zoom:67%;" />

### **Exercise 7**：关于Fork系统调用

- 过程为首先创建一个用户环境00001000，然后调用`env_run()`进入这个用户，执行`dumbfork.c`的用户态代码，该代码调用系统调用`sys_exofork()`来创建一个子环境，并且子环境的tf和父环境的tf相同。

- 结果为：![1692018865742](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308142114404.png)

- 其中，在内核态下的sys_exofork()函数实现为：

  ```
  static envid_t
  sys_exofork(void)
  {
  	// Create the new environment with env_alloc(), from kern/env.c.
  	// It should be left as env_alloc created it, except that
  	// status is set to ENV_NOT_RUNNABLE, and the register set is copied
  	// from the current environment -- but tweaked so sys_exofork
  	// will appear to return 0.
  
      // LAB 4: Your code here.
      struct Env *child;
      int ret = env_alloc(&child , thiscpu->cpu_env->env_id);
  //    cprintf("ret = %d\n" , ret);
  //    cprintf("child->env_id = %d\n" , child->env_id);
  
      if (ret < 0) {
          return ret;
      }
      child->env_status = ENV_NOT_RUNNABLE;
      child->env_tf = thiscpu->cpu_env->env_tf;
      // ?
      child->env_tf.tf_regs.reg_eax = 0;
      return child->env_id;
  }
  ```

- 其中，child->env_tf.tf_regs.reg_eax设置为0，这是为什么呢？

- 答：在用户态进入内核态之前的代码为：

  ```
  static inline envid_t __attribute__((always_inline))
  sys_exofork(void)
  {
  	envid_t ret;
  	asm volatile("int %2"
  		     : "=a" (ret)
  		     : "a" (SYS_exofork), "i" (T_SYSCALL));
  	return ret;
  }
  
  // 对应的反汇编代码为：
  	asm volatile("int %2"
    8000e1:	b8 07 00 00 00       	mov    $0x7,%eax
    8000e6:	cd 30                	int    $0x30
    8000e8:	89 c3                	mov    %eax,%ebx
  ```

- 其中在`int 0x30`之后，将$eax的值赋给了ret，也就是说，在环境0调用这个系统调用时，返回地址为`eip = 0x8000e8`，而在内核态下创建环境1之后，环境1的tf和环境0的tf相同，那么环境1的`eip = 0x8000e8`，所以当环境0切换到环境1之后，环境1其实是从0x8000e8开始执行的，此时环境1的eax被我们设置为了0，所以环境1的`sys_exofork()`返回值就是0。

- 也就是说，之后父环境真正执行了`sys_exofork()`，而子环境只是执行了`mov    %eax,%ebx`

- 这也就实现了课程所要求的： In the parent, `sys_exofork` will return the `envid_t` of the newly created environment (or a negative error code if the environment allocation failed). In the child, however, it will return 0.

### 问题1：

- Compare `kern/mpentry.S` side by side with `boot/boot.S`. Bearing in mind that `kern/mpentry.S` is compiled and linked to run above `KERNBASE` just like everything else in the kernel, what is the purpose of macro `MPBOOTPHYS`? Why is it necessary in `kern/mpentry.S` but not in `boot/boot.S`? In other words, what could go wrong if it were omitted in `kern/mpentry.S`?
- 因为此时`kern/mpentry.S` is compiled and linked to run above `KERNBASE`，而start32，mpentry_start都是虚拟高地址，但此时汇编代码运行在实模式下，此时未开启分页，也就没有什么虚拟地址和物理地址了，而且我们已经把`kern/mpentry.S`的代码移动到物理地址为`MPENTRY_PADDR`的地方了，所以我们需要访问其在实模式下的物理地址。

### 问题2：

- It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.
- 其实在**上锁之前，处理器就已经处于内核态了**，比如中断，我们先进行int指令进入内核态，然后调用trap函数上锁，在int指令时，会进行内核栈的压栈，如果所有CPU都共享一个内核栈，那么当多个处理器进入内核态时，会在一个内核栈中压很多次。

### 疑惑

- 每个CPU都有自己的LAPIC，所以每个CPU启动时都要调用函数`lapic_init()`，进行地址映射`lapic = mmio_map_region(lapicaddr, 4096);`，其中`lapicaddr`是物理地址，但是好像每个CPU的`lapicaddr`都一样，也就是不同的虚拟地址映射到了相同的物理地址
- `lapic_init()`和`lapic_startap(c->cpu_id, PADDR(code));`内部没有深究
- 为什么`&addr`直接指向了用户的栈底？![1692079599215](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202308151406148.png)
- 创建子环境时，子环境和父环境的esp相同，为什么？在一开始实现的fork函数dumbfork中，我们将父环境的内存块复制了一份，包括栈，所以虽然子环境和父环境的esp相同，但是这是**虚拟地址相同**，子环境和父环境的相同虚拟地址指向了不同的物理地址。在实现了COW的fork中，用户栈页当成了共享区域，**猜测**（之后测试一下）如果父环境先修改用户栈时会触发缺页处理函数，给父环境重新分配一个用户栈


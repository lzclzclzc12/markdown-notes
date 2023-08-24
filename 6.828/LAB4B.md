### Exercise 8-11

- `sys_env_set_pgfault_upcall`系统调用是设置当前环境的页错误中断处理函数的入口地址，保存在`env->env_pgfault_upcall`中；
- 首先我们在用户态下调用`set_pgfault_handler(handler);`来设置当前环境的页错误中断处理函数的入口地址，并且为当前环境分配exception stack；并且设置用户自定义的handler函数`_pgfault_handler`
- 当发生页错误时，进入内核态，并且根据`IDT`调用`TRAPHANDLER(t_pgflt, T_PGFLT)`函数，然后根据`T_PGFLT`调用`page_fault_handler`函数
- `page_fault_handler`函数将当前用户的`UTrapframe`保存在exception stack中，当页错误处理程序结束想要返回原程序时，需要`UTrapframe`提供的上下文信息来恢复环境。之后设置当前的eip为页处理程序入口地址，esp为exception stack的栈底，然后调用`env_run(curenv);`来切换到执行异常处理程序
- 页错误处理程序的统一入口为`pfentry.S`中的`_pgfault_upcall`，该函数调用`_pgfault_handler`，之后使用保存好的`UTrapframe`来恢复到原程序

### `UVPT`的理解

- 首先页目录的每个页目录项占4B，其值为其所指向页表的页的起始地址；

- 页目录和页表一样也是占4KB，即一页大小

- 所以`kern_pgdir`等变量其低12位都是0，因为其指向的是一个页的起始地址

- ```
  // 下面这句代码是给内核的页目录添加了一个页目录项，其值为页目录所在页的起始地址
  kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
  ```

#### 访问`UVPT[PGNUM(va)]`的过程

- `uvpt[PGNUM(va)] = uvpt +( va >> 12 * 4)`
- `UVPT`的值为0xef400000，变为二进制为1110111101 0000000000 000000000000
- 也就是说，`UVPT`的后22位都是0，那么`UVPT[PGNUM(va)]` = 3BD <> PDX(va) <> PTX(va) <> 00(二进制)
- 首先CR3先根据PDX（这里是3BD）找到对应的页目录项，其值为PADDR(kern_pgdir) | PTE_U | PTE_P，即页目录所在页的起始地址
- 然后找到页目录所在页，再根据PTX（这里为PDX(va)）找到对应的页目录表项，其值为`va`所对应的页表的起始地址
- 最后根据OFF（这里是PTX(va) <> 00(二进制)）从该页表中找到`va`所对应的页表项的地址，最后有两个0是因为每个页表项是4B
- 所以`UVPT[PGNUM(va)]`最终得到的是`va`所对应的页表项的地址
- 这里之所以会找到页目录项的原因是一般情况下CR3先根据PDX找到的是页表的起始地址，但是**这里的3BD页目录项所指向的是页目录的起始地址**
- 根据上述分析，`UVPT[0]`即`UVPT`所指向的是**第0个页表的第0个页表项的地址**

#### 关于`UVPD`的理解

- `uvpt[PGNUM(va)] = uvpt +( va >> 12 * 4)` = PDX(va) <> PTX(va)所表示的就是第PDX(va)个页表的第PDX(va)个页表项的地址
- `pde_t uvpd[] = UVPT+(UVPT>>12)*4;`
- 那么`uvpd[0]`即`uvpd`就是第PDX(uvpt)个页表的第PDX(uvpt)个页表项的地址，这里PDX(uvpt) = 3BD，PDX(uvpt) = 0，也就是**第0个页目录表项的起始地址**
- `UVPD[PDX(va)]`表示va所对应的页目录项的起始地址


### AT&T语法

- 本实验采用的是GNU汇编器（GNU assembler，GAS），GNU汇编器使用AT&T语法。NASM汇编器使用的是Intel语法
- 两者之间的不同见： [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html).
- 比如寄存器前必须加%，立即数和常量前必须加$，操作数左边为源操作数，右边为目的操作数
- 寄存器间接寻址：AT&T:  (%eax)
- [asm: gcc - c语言的内联汇编学习](https://www.cnblogs.com/lnlidawei/p/17016142.html).

### obj/kern/kernel.img

- kern/Makefrag中有着构建`obj/kern/kernel.img`的命令：

  ```
  # How to build the kernel disk image
  $(OBJDIR)/kern/kernel.img: $(OBJDIR)/kern/kernel $(OBJDIR)/boot/boot
     @echo + mk $@
     $(V)dd if=/dev/zero of=$(OBJDIR)/kern/kernel.img~ count=10000 2>/dev/null
     $(V)dd if=$(OBJDIR)/boot/boot of=$(OBJDIR)/kern/kernel.img~ conv=notrunc 2>/dev/null
     $(V)dd if=$(OBJDIR)/kern/kernel of=$(OBJDIR)/kern/kernel.img~ seek=1 conv=notrunc 2>/dev/null
     $(V)mv $(OBJDIR)/kern/kernel.img~ $(OBJDIR)/kern/kernel.img
  ```

  可以看出，首先我们需要构建好`obj/kern/kernel`和`obj/boot/boot`，然后

  ```
  $(V)dd if=/dev/zero of=$(OBJDIR)/kern/kernel.img~ count=10000 2>/dev/null
  ```

  这一命令给`obj/kern/kernel.img`分配了10000 * 512B = 5120000B的空间，其中512B为磁盘的一个扇区，`obj/kern/kernel.img`这一文件就是**模拟磁盘**，所以现在磁盘大小为10000个扇区。之后

  ```
  $(V)dd if=$(OBJDIR)/boot/boot of=$(OBJDIR)/kern/kernel.img~ conv=notrunc 2>/dev/null
  ```

  这一命令将`obj/boot/boot`的内容放入磁盘的第一个扇区（called the ***boot sector***），对应seek = 0，再然后

  ```
  $(V)dd if=$(OBJDIR)/kern/kernel of=$(OBJDIR)/kern/kernel.img~ seek=1 conv=notrunc 2>/dev/null
  ```

  将`obj/kern/kernel`的内容放入磁盘的第二个扇区（从第二个扇区开始放）

- `if=/dev/zero`是一个用于输入的特殊设备文件。它的作用是生成一连串的零字节，可以用于测试、填充文件或者设备。这个命令行参数通常会用在Unix或Linux系统的dd命令中。举个例子，使用以下命令可以将`/dev/zero`设备上的内容写入到一个文件中：

  ```
  dd if=/dev/zero of=output.txt bs=1M count=10
  ```

  这会创建一个名为`output.txt`的文件，文件大小为10MB，文件内容全部都是零字节。

- `obj/kern/kernel`由各种.o文件链接而成，成为可执行文件

  ```
  ld -o obj/kern/kernel -m elf_i386 -T kern/kernel.ld -nostdlib obj/kern/entry.o obj/kern/entrypgdir.o obj/kern/init.o obj/kern/console.o obj/kern/monitor.o obj/kern/printf.o obj/kern/kdebug.o  obj/kern/printfmt.o  obj/kern/readline.o  obj/kern/string.o /usr/lib/gcc/x86_64-linux-gnu/9/32/libgcc.a -b binary
  ```

- `boot/Makefrag`文件：

  ```
  $(OBJDIR)/boot/boot: $(BOOT_OBJS)
     @echo + ld boot/boot
     $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o $@.out $^
     $(V)$(OBJDUMP) -S $@.out >$@.asm
     $(V)$(OBJCOPY) -S -O binary -j .text $@.out $@
     $(V)perl boot/sign.pl $(OBJDIR)/boot/boot
  ```

  可以看出，`boot.out`是由 `boot/boot.S` 和 `boot/main.c`编译链接后生成的 ELF 可执行文件，而 `boot.asm` 是从可执行文件` boot.out `反编译的包含源码的汇编文件，而最后通过` objcopy` 拷贝` boot.out`中的 .text 代码节生成最终的二进制引导文件 boot (380个字节)，最后通过 `sign.pl`这个perl脚本填充 boot 文件到**512字节**（最后两个字节设置为 55 aa，代表这是一个引导扇区）。

### BIOS的作用

- 在处理器上电时，进入实模式，并且设置cs = 0xf000，ip = 0xfff0，由于是实模式，所以真实的物理地址为：16 * 0xf000 + 0xfff0 = 0xffff0，这正是BIOS的区域，所以开始执行BIOS
- When the BIOS runs, it sets up an **interrupt descriptor table** and **initializes various devices** such as the VGA display.
- After **initializing the PCI bus** and all the important devices the BIOS knows about, it **searches for a bootable device** such as a floppy, hard drive, or CD-ROM. Eventually, when it finds a bootable disk, the BIOS **reads the *boot loader* from the disk and transfers control to it**.

## The Boot Loader

- 在JOS中，BIOS从磁盘的第一个扇区读取出boot loader代码，放到内存的**[0x7c00 , 0x7dff]**处，最后使用jmp指令设置cs:ip为0000:7c00，该地址为boot loader的入口地址，至此，处理器的控制权转向了boot loader.
- 主要是将操作系统从实模式转化为保护模式，然后将内核写入内存，并且跳转到内核的入口地址
- 参考链接：[操作系统实模式、保护模式的深入浅出](https://blog.csdn.net/qq_45507768/article/details/131417014)和[操作系统学习 — 启动操作系统：进入保护模式](https://zhuanlan.zhihu.com/p/596820045).

#### .code16和.code32

- 在实模式下，寄存器为16位，地址线为20个，.code16伪指令就是**告诉编译器将该汇编语言汇编成16位的机器指令**，然后CPU就将当前机器码按照16位模式执行；.code32同理
- CPU 在16位模式和32位模式处理同一条机器码，转化后的汇编指令是不同的。所以我们需要使用.code16和.code32来区分实模式代码和保护模式代码，不然汇编器会全部汇编成16位机器码，造成执行的结果不同。

#### 开启A20

- 在[操作系统学习 — 启动操作系统：进入保护模式](https://zhuanlan.zhihu.com/p/596820045)中有介绍，这里简单了解一下，就是为了与16位CPU兼容而设计的
- 在实模式下，A20是关闭状态，在`boot.S`代码中，我们通过在0x64端口写入0xd1和在0x60端口写入0xdf来开启A20

#### 引入GDT

- 在实模式下，操作系统和用户属于同一特权级别，而且都是访问物理地址，这样的模式下对于内存的保护很弱

- 在保护模式下，我们引入了GDT，也就是全局描述符表，每个表项（也就是描述符）大小为8B，**段描述符**结构如图：![img](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-3.gif)

- 可以看到每个段描述符包含base、limit、DPL等，base为32位，limit为20位，base表示段的位置，**DPL**为Descriptor Privilege Level

- limit表示段的大小，其单位在JOS中为4KB，也就是说段的大小限制在4GB

- 在`mmu.h`中我们可以找到其定义：

  ```
  #define SEG(type,base,lim)              \
     .word (((lim) >> 12) & 0xffff), ((base) & 0xffff); \
     .byte (((base) >> 16) & 0xff), (0x90 | (type)),       \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
  ```

  可以看到，lim右移了12位，这说明其单位是4KB

- 我们初始化GDT为3个描述符，第一个描述符不用

  ```
  gdt:
    SEG_NULL                          # null seg
    SEG(STA_X|STA_R, 0x0, 0xffffffff)        # code seg
    SEG(STA_W, 0x0, 0xffffffff)          # data seg
  ```

- 那么如何访问段描述符呢？我们通过**段选择子**来访问，其结构为：

  ![img](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-6.gif)

  可以看出，段选择子为16位，**RPL**为请求特权级别，index为后12位，这是由于每个描述符占8B，为$2^3$

- 在`boot.S`中定义了两个段选择子

  ```
  .set PROT_MODE_CSEG, 0x8         # kernel code segment selector
  .set PROT_MODE_DSEG, 0x10        # kernel data segment selector
  ```

  可以看到代码段的index为1，数据段的index为2，CPU不用index = 0的描述符

- 然后我们将GDT的大小和地址传入GDTR（GDT寄存器）中

  ```
  lgdt    gdtdesc
  
  gdtdesc:
    .word   0x17                            # sizeof(gdt) - 1
    .long   gdt                             # address gdt
  ```

  GDTR是一个48位寄存器，在代码中可以看出，GDTR的前16位（.word）为GDT的大小，后32位（.long 双字）为GDT的位置

- 有了GDT之后，我们**访问内存的方式就变了**，在实模式下，我们访问内存是通过(cs << 4) + ip，在保护模式下，cs存放的不再是段基址了，而是段选择子，我们通过段选择子查询GDT得到段基址，然后段基址 + ip得到地址
- 然后我们将**CR0**寄存器的protected mode enable flag置为1

  ```
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
  ```

- 在保护模式下，我们有了DPL和RPL，有了对于段的保护机制

#### bootmain

- `boot.S`的最后一条指令跳转到了bootmain函数，这个函数的作用是将内核的每个段从磁盘写入内存

- 我们知道，在编译链接阶段，各种.o文件链接成`obj/kern/kernel`，这个就是内核文件，而且是一个ELF可执行文件，要运行内核，我们就需要将该ELF文件的各个段写入内存

- [计算机原理系列之二 -------- 详解ELF文件](https://mp.weixin.qq.com/s/RRx5KGvoiuKIHYeiEw5TKw)，这篇文章介绍了ELF文件的结构和数据结构

- 首先，我们从磁盘的第一个扇区开始读取一个页，第一个页存放的是ELF Header

  ```
  // read 1st page off disk，0表示内核文件的偏移
  readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
  
  
  // translate from bytes to sectors, and kernel starts at sector 1，+1表示从第二个扇区开始
  offset = (offset / SECTSIZE) + 1;
  
  // 0x1f3端口指明了磁盘的盘块号
  outb(0x1F3, offset);
  ```

- 然后通过ELF Header找到Program Header Table（ph）的首地址，将内核的每个段读入内存

  ```
  // load each program segment (ignores ph flags)
  ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
  eph = ph + ELFHDR->e_phnum;
  for (; ph < eph; ph++) {
     // p_pa is the load address of this segment (as well
     // as the physical address)
     readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
     // 这里由于p_memsz很可能比p_filesz要大，而p_filesz是该段实际的大小，所以我们需要将多分配的内存置0
     for (i = 0; i < ph->p_memsz - ph->p_filesz; i++) {
        *((char *) ph->p_pa + ph->p_filesz + i) = 0;
     }
  }
  ```

- 最后跳转到内核的入口地址

  ```
  ((void (*)(void)) (ELFHDR->e_entry))();
  7d9a:	ff 15 18 00 01 00    	call   *0x10018
  ```

  反汇编之后可以看到，跳转地址为*0x10018，\*指的是间接寻址，我们可以查看0x10018中存放的数据，发现是**0x0010000c**，这个就是**内核的入口物理地址**

  ```
  (gdb) x /1xw 0x10018
  0x10018:        0x0010000c
  ```

  0x10018应该就是`ELFHDR->e_entry`的物理地址，因为在bootmain中我们将ELF Header放到了0x10000处：

  ```
  #define ELFHDR     ((struct Elf *) 0x10000) // scratch space
  ```

- 之所以以上这些都是物理地址的原因是此时还未开启分页，所以没有虚拟地址




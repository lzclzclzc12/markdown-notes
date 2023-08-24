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

  将`obj/kern/kernel`的内容放入磁盘的第二个扇区

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
- 

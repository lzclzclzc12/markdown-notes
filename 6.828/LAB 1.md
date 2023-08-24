### AT&T语法

- 本实验采用的是GNU汇编器（GNU assembler，GAS），GNU汇编器使用AT&T语法。NASM汇编器使用的是Intel语法
- 两者之间的不同见： [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html).
- 比如寄存器前必须加%，立即数和常量前必须加$，操作数左边为源操作数，右边为目的操作数
- 寄存器间接寻址：AT&T:  (%eax)
- [asm: gcc - c语言的内联汇编学习](https://www.cnblogs.com/lnlidawei/p/17016142.html).


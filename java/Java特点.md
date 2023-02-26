# Java特点

## 1. Java语言是跨平台性的

- 首先编写java文件Test.java
- 之后对其进行**编译**：javac Test.java，得到Test.class
- Test.class文件既可以在win上运行，又可以在linux上运行，所以是跨平台的（基于jvm）

## 2. Java语言是解释型语言

- 编译后生成的.class文件不能直接执行，需要解释器来解释执行

## 3. JDK

- (Java Development Kit)，是java开发工具包
- JDK = JRE + Java的开发工具（java,javac,javadoc,javap等）

## 4. JRE

- Java Runtime Environment ，是Java运行环境
- JRE = JVM + Java的核心类库

## 5. 为什么要配置java环境变量path

- 如果不配置，那么javac和java会报错
- 原因是当前执行的程序在当前目录下如果不存在，那么win系统会在系统中已有的一个名为path的环境变量指定的目录中查找，如果仍未找到，那么会报错，所以需要配置jdk的bin目录

## 6. 注意事项

- 一个源文件中最多有一个public类，其他类的个数不限
- 编译后每一个类都对应一个.class文件,下图中编译后会生成三个.class文件<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230210104753565.png" alt="image-20230210104753565" style="zoom: 50%;" />
- 文件名必须和文件中public类的名字相同，比如上图的文件名是Hello.java
- 也可以将main方法写在非public类中，然后指定非public类，这样入口方法就是非public的main方法
- shift+tab向左移动

## 7. 转义字符

- \t: 制表位（Tab健）
- \n: 换行符
- \r: 回车（将光标移动到本输出行的第一个字符）如下图输出是 北京平教育![image-20230210111309204](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230210111309204.png)
- \: 想要输出\\\，那么需要写\\\\\\\

## 8. 数据类型

- ![image-20230212111921536](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212111921536.png)
- java的整型常量**默认为int类型**，声明long类型常量需要后面加'l'或'L'
- 浮点数=符号位+指数位+尾数位，尾数部分可能丢失
- java的浮点型常量**默认为double型**，声明float型常量后面需加'f'或'F'<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230211111221838.png" alt="image-20230211111221838" style="zoom:50%;" />
- 当我们对运算结果是小数的变量进行想等判断时，要小心![image-20230211112116725](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230211112116725.png)
- 正确比较法应该是以两个数的差值的绝对值，在某个精度范围内判断![image-20230211112758802](C:\Users\lzc\AppData\Roaming\Typora\typora-user-images\image-20230211112758802.png)
- java中，char的本质是一个整数，输出时，是**unicode码**对应的字符；char类型是可以运算的，相当于一个整数
- 字符类型的本质![image-20230212111723755](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212111723755.png)
- 不可以用0或非0来代替false和true，必须是boolean

### 8.1. 自动类型转换

- ![image-20230212113017738](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212113017738.png)
- 细节和注意事项：![image-20230212115448912](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212115448912.png)
- 例子：![image-20230212114523924](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212114523924.png)
- ![image-20230212114723434](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212114723434.png)
- ![image-20230212115614097](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212115614097.png)
- ![image-20230212130309462](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212130309462.png)![image-20230216110701797](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216110701797.png)



### 8.2. 基本数据类型和String类型的转换

- ![image-20230212131504929](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230212131504929.png)

## 9. 运算符 

### 9.1 算术运算符

- 除法：![image-20230213103021139](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213103021139.png)
- 取模：![image-20230213103413507](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213103413507.png)![image-20230214154728797](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214154728797.png)
- ![image-20230213104054214](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213104054214.png)
- 原本b+2是int类型，复合赋值运算进行了类型转换![image-20230213113121105](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213113121105.png)

### 9.2. 逻辑运算符

- 与![image-20230213110211775](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213110211775.png)
-  ![image-20230213110541274](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213110541274.png)
- ![image-20230213110944159](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213110944159.png)
- ![image-20230213112515495](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213112515495.png)

### 9.3. 三元运算符

- ![image-20230213113621782](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213113621782.png)

### 9.4. 运算符优先级

<img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213115136008.png" alt="image-20230213115136008" style="zoom:50%;" />

## 10. 标识符命名规范

- ![image-20230213124836446](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213124836446.png)

## 11. 键盘输入

使用Scanner类![image-20230213130421598](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213130421598.png)

![image-20230214161522035](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214161522035.png)

## 12. 进制

![image-20230213131029926](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213131029926.png)

## 13. java中的原码、反码、补码

- 运算用补码，显示用原码![image-20230213133200890](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230213133200890.png)
- 位运算符：![image-20230214153954135](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214153954135.png)

## 14. switch

- ![image-20230214162258466](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214162258466.png)

  <img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214162351992.png" alt="image-20230214162351992" style="zoom: 67%;" />

- 注意：**如果case之后没写break，那么代码会直接执行后面的语句而绕过后面的判断语句**

- ![image-20230214164258396](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230214164258396.png)

- ![image-20230215104445587](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230215104445587.png)

## 15. for循环

- 注意：执行完循环操作后才执行 循环变量迭代![image-20230215104945935](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230215104945935.png)
- ![image-20230215130439262](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230215130439262.png)

## 16. 数组

- ![image-20230216105256680](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216105256680.png)

- 获取数组长度：数组名.length

- ![image-20230216110059176](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216110059176.png)

- ![image-20230216110245971](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216110245971.png)

- **数组的赋值机制**：![image-20230216111236115](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216111236115.png)

- 创建数组，**java中可以列数不确定**![image-20230216125951973](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216125951973.png)

- 创建时先只定义第一维，第二维先声明，后赋值 ![image-20230216125854969](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216125854969.png)

- ![image-20230216132052731](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/image-20230216132052731.png)

  

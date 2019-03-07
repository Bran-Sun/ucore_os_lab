# lab1实验报告

## 练习1

### 1. 镜像文件ucore.img的生成

```makefile
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

* 镜像文件需要依赖```bin/bootblock```和```bin/kernel```两个文件
* 第一条命令生成一个全是字符0的文件，一共拷贝10000次，共5120000byte
* 第二条命令将bootblock文件拷贝到ucore.img的开头位置，而且保持其原有长度
* 第三条命令将kernel文件拷到之前bootblock之后。因为bootblock本身是512个字节，所以正好跳过一个block

### 2. 主引导扇区的特征

一共有512个字节，最后两个字节是```0x55AA```



## 练习2

### 1. 从加电后的第一条指令开始单步调试BIOS

按照文档的提示，先将```tools/gdbinit```的内容修改成

```
set architecture i8086
target remote :1234
```

然后直接运行```make debug```即可

在gdb调试窗口中，使用```si```命令进行单步调试即可

### 2. 在初始化位置0x7c00设置断点，测试断点是否正常

修改文件```gdbinit```为如下

```
file obj/bootblock.o
set architecture i8086
target remote :1234
b *0x7c00
continue
```

即可跳转到bootloader入口处执行代码

### 3. 比较反汇编的代码和bootasm.s和bootblock.asm

可以发现三者的汇编代码均相同

### 4. 找一个bootloader或内核中的代码位置设置断点，并进行测试

选在进入保护模式的位置设置断点，将gdbinit文件进行如下修改

```
file obj/bootblock.o
set architecture i8086
target remote :1234
b protcseg
continue
```

即可使程序在运行到该位置时暂停。



## 练习3

### 分析bootloader进入保护模式的过程

* 初始化寄存器

```assembly
    cli                                             # IF设成0，此时无视中断
    cld                                             # DF设成0，表示字符从低地址处理到高地址

    # 将段寄存器设置成0
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

* 为了访问4G内存，需要将A20打开

```assembly
seta20.1:
    inb $0x64, %al                                  # 读status寄存器，直到input寄存器无数据
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64, 表示要开始写数据
    outb %al, $0x64                                 

seta20.2:											# 等待input寄存器为空
    inb $0x64, %al                       
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60, 打开A20
    outb %al, $0x60                                 

```

* 加载gdt，并转换到保护模式

```assembly
    lgdt gdtdesc
    movl %cr0, %eax 								#将CR0的PE设置成1，开启保护模式
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    
    ljmp $PROT_MODE_CSEG, $protcseg 				#通过长跳转更新CS寄存器
```

* 设置段寄存器，并建立堆栈，然后跳转到主方法
  ```assembly
  		movw $PROT_MODE_DSEG, %ax
  	    movw %ax, %ds
  	    movw %ax, %es
  	    movw %ax, %fs
  	    movw %ax, %gs
  	    movw %ax, %ss
  	    movl $0x0, %ebp
  	    movl $start, %esp
  	    
  	    call bootmain
  ```



## 练习4

### 1. bootloader是如何读取硬盘扇区的？

在bootmain.c中，readsect函数用来读取硬盘扇区。

* 首先，等待扇区不是忙状态
* 0x1F2地址输入需要读取的扇区数目
* 0x1F3～0x1F6输入的均为LBA的数值
* 0x1F7输入读取的命令
* 等待扇区准备好
* 从扇区中读出512字节的内容

### 2. bootloader是如何加载elf格式的文件的？

* 先读取elf文件头

* 验证文件头中的魔数(magic number)

* 读取程序头部，即一个program header结构的数组，从中获得程序各个的信息，且读取到指定的地址处

* 跳转执行该文件

* 其中，如下语句

  ```C
  ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))()
  ```

  表示将该地址强制转换成一个参数为空的函数，并执行该函数。

## 练习5

### 实现函数调用堆栈跟踪函数

* 具体函数实现见kdebug.c文件。实现过程比较简单，按照提示获取ebp和eip，输出当前栈帧内容，然后再获得前一个栈帧的ebp和eip(用当前的ebp值作为地址)，进行递归即可。

* 输出的最后一行为

  ```
  ebp:00007bf8 eip:00007d6e args:c031fcfa c08ed88e 64e4d08e fa7502a8 
      <unknow>: -- 0x00007d6d --
  ```

  * 该eip指向的是bootblock.asm中的0x7d6e地址的语句。其中0x7d6c的语句为call语句，表示跳转到kern_init运行。
  * ebp的初始值在0x7c45处被设置成0x7c00，然后跳转到bootmain运行
  * 在bootmain中进行了一些压栈操作，所以后面的这些参数就是在bootmain函数中被压入的值

## 练习6

### 1. 中断描述符表的一个表项占多少字节？其中哪几位表示中断处理程序的入口？

* 一个表项占8个字节
* 其中0-1位和6-7位合起来表示地址，2-3位为段选择子。段选择子从GDT或者LDT中取得基址，相加后得到程序入口

### 2. 完善trap.c中的idt_init函数

* 先在外面声明__vectors，里面存储着每个中断处理代码的offset
* 然后将idt中每一项用SETGATE来设置
* 最后调用lidt()函数即可

### 3. 完善trap.c中的trap函数

* 设置一个静态变量进行累加，当等于100时清零并打印100 ticks。




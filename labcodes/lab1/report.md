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

### 分析bootloader加载OS的过程


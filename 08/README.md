# 第 8 章 硬盘和显卡的访问控制

|本期版本|上期版本
|:---:|:---:|
`Sat Dec 31 21:27:57 CST 2022` | 

## 8.2 用户程序的结构


### 8.2.2 用户程序头部

* 用户程序的尺寸，即以字节为单位的大小
* 用户程序的入口点，包括段地址和偏移地址
* 段重定位表

## 8.3 加载程序（器）的工作流程

### 8.3.1    初始化和决定加载位置

* 从那个物理内存地址开始加载用户程序
* 用户程序位于硬盘上的什么位置

### 8.3.4 I/O端口和端口访问

* `in` 指令是从端口读。目的操作数必须是寄存器AL或者AX
* `out` 指令。源操作数必须是寄存器AL或者AX


### 8.3.5 通过硬盘控制器端口读扇区数据

> [15 硬盘读写](https://www.bilibili.com/video/BV1b44y1k7mT?p=15)

| Primary 通道            | Secondary 通道 | in 操作      | out 操作     |
| ----------------------- | -------------- | ------------ | ------------ |
| 0x1F0                   | 0x170          | Data         | Data         |
| 0x1F1                   | 0x171          | Error        | Features     |
| 0x1F2                   | 0x172          | Sector count | Sector count |
| 0x1F3                   | 0x173          | LBA low      | LBA low      |
| 0x1F4                   | 0x174          | LBA mid      | LBA mid      |
| 0x1F5                   | 0x175          | LBA high     | LBA high     |
| 0x1F6                   | 0x176          | Device       | Device       |
| 0x1F7                   | 0x177          | Status       | Command      |


**1 - 准备阶段**

* 读取扇区的数量: `0x1f2`(每读一个扇区，这个数值就减一)
* 28位的扇区号: `0x1f3`、`0x1f4`、`0x1f5`、`0x1f6`

<img src="8-11.png" />

```
mov al, 0xe0
or al, ah
```

**2 - 等待阶段**

*  向端口 `0x1f7` 写入 `0x20` ，请求硬盘读
* `0x1f7` 既是命令端口，又是状态端口

<img src="8-12.png" />

```
and al, 0x88 			;10001000 保留第7位和第3位
cmp al, 0x08			;00001000 可以退出等待端口
```

**3 - 获取数据阶段**

* `0x1f0` 是硬盘的数据端口，而且是一个 16 位端口

### 8.3.6 过程调用

* `ret` 是近返回指令，从栈中弹出一个字到指令指针寄存器IP中
* `fetf` 是远返回指令，从栈中弹出两个字到指令指针寄存器IP和代码段寄存器CS中


### 8.3.7 加载用户程序

* `jnz` 不相等则跳转 / `jz` 相等则跳转
* 一个逻辑段最大也才 64 KB， 当用户程序特别大的时候，根本容纳不下
* 每次往内存中加载一个扇区前，都重新在前面的数据尾部构造一个新的逻辑段，并把要读取的数据加载到这个新段内
* 每个段的大小是 `512` 字节，即十六进制的 `0x200`，右移4位后是 `0x20`

### 8.3.8 用户程序重定位

* 尽管 `DX:AX` 中是 32 位的用户程序起始物理内存地址，理论上，它只有 20 位是有效的， 低 16 位在寄存器 AX中，高 4 位在寄存器 DX 的低 4 位


## 8.4 用户程序的工作流程

**8.4.1 初始化段寄存器和栈切换**

* `resb（Reserve Byte）`: 从当前位置开始，保留指定数量的字节，但不初始化他们的值


**8.4.2 调用字符串显示例程**

* 将字车推到最左边， 也就是一行的开始，叫做回车(Carriage Return)。拧一下滚动，将纸上卷一行，叫作换行(Line Feed)
* 这个过程通常叫做回车换行(CRLF)
* 回车分配的 ASCII 码是 `0x0d`, 换行分配的则是 `0x0a`
* `0` 终止的字符串

**8.4.4 屏幕光标控制**

* 光标是在屏幕上有规律地闪动的一条小横线
* 光标在屏幕上的位置保存在显卡内部的两个光标寄存器中，每个寄存器是 8 位，合起来形成一个 16 位的数值
* 光标寄存器是可读写的


**8.4.5 取当前光标的位置**

* 很多寄存器只能通过索引寄存器间接访问
* 索引寄存器的端口号是 `0x3d4`
* `14(0x0e)` 和 `15(0x0f)` 粉笔用于提供光标位置的高8位和低8位
* 要对它进行读写，这可以通过数据端口: `0x3d5`

**8.4.6 处理回车和换行字符**

* 如果是回车符 `0x0d`, 那么，应将光标移动道当前行首：用当前光标位置除以80，可以得到当前行数，接着再乘以 80， 就是当前行行首的光标数值
* 换行的意图是向下一行，只需增加 80 即可得到新的光标位置数据


### 8.4.8 滚动屏幕内容

*  实质上就是将屏幕上 2\~25行的内容整体往上提一行，最后用黑底白字的空白字符填充低25行


### 8.4.10 切换到另一个代码段中执行

* 可以使用 `retf` 来模拟段间的返回
* 尽管 `call` 和 `call far` 指令分别依赖 `ret` 和 `retf` 指令, 单后者并不依赖与前者

### 8.5 编译和运行程序并观察结果

```bash
dd if=c08_mbr.bin of=c.img bs=512 count=1 conv=notrunc
dd if=c08.bin of=c.img seek=100 bs=512 count=1 conv=notrunc
```


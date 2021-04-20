
ARM 和 Thumb 的不同之处：
- 条件执行指令
    所有的 ARM 状态指令都支持条件执行。一些版本的 ARM 处理器上允许在 Thumb 模式下通过 IT 汇编指令进行条件执行。条件执行减少了要被执行的指令数量，以及用来做分支跳转的语句，所以具有更高的代码密度。
- 32位 ARM 和 Thumb 指令集
    Thumb 的32位汇编指令都有类似于 a.w 的扩展后缀
- 滚筒移位 ——> 独特的 ARM 模式特性
    可以被用来减少指令数量。比如说，为了减少使用乘法所需的两条指令(乘法操作需要先乘 2 然后再把结果用 MOV 存储到另一个寄存器中)，就可以使用在 MOV 中自带移位乘法操作的左移指令(Mov R1, R0, LSL #1)

ARM模式与Thumb模式间切换:
- 分支跳转指令BX(branch and exchange)或者分支链接跳转指令BLX(branch,link and exchange)时，将目的寄存器的最低位置1.  -> 再偏移处加一
- 在CPSR当前程序状态寄存器中，T标志位用来代表当前程序是不是在Thumb模式下运行的

``MNEMONIC{S}{condition} {Rd}, Operand1, Operand2``
助记符{是否使用CPSR}{是否条件执行以及条件} {目的寄存器}, 操作符1, 操作符2
每条ARM指令的宽度是32位
    每个条件在机器码中的占位都是4位
    2位来做为目的寄存器 
    2位作为第一操作寄存器   
    1位用作设置状态的标记位
    加上比如操作码(opcode)
    每条指令留给我们存放立即数的空间只有12位宽。也就是4096个不同的值
    8位用作加载0-255中的任意值，4位用作对这个值做0~30位的循环右移

---
![](https://p3.ssl.qhimg.com/t012c4f64db0e000b91.png)

LOAD AND STORE 的内存指令集
![](截屏2020-08-06%20下午8.48.51.png)

LDM和STM有多种形式
```
IA(increase after)
IB(increase before)
DA(decrease after)
DB(decrease before)
```

条件执行
![](https://p1.ssl.qhimg.com/t01614edecda79f64b7.png)

允许在Thumb模式下条件执行最多四条指令
指令格式：Syntax: IT{x{y{z}}} cond
cond 代表在IT指令后第一条条件执行执行指令的需要满足的条件。
x 代表着第二条条件执行指令要满足的条件逻辑相同还是相反。
y 代表着第三条条件执行指令要满足的条件逻辑相同还是相反。
z 代表着第四条条件执行指令要满足的条件逻辑相同还是相反。

条件指令后缀含义以及他们的逻辑相反指令：
![](https://p5.ssl.qhimg.com/t0159a38754c0e1e375.png)

分支指令：
![](截屏2020-08-07%20下午4.32.26.png)

不同的栈实现，可以用不同情形下的多次存取指令来表示：
![](https://p0.ssl.qhimg.com/t01992f4f147ff35a38.png)

![](https://azeria-labs.com/downloads/cheatsheetv1.3-1920x1080.png)



参考资料：
[Azeria Labs](https://azeria-labs.com/writing-arm-assembly-part-1/)
[Azeria Labs 汇编基础部分中文翻译](https://www.anquanke.com/post/id/86383)

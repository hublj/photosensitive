FLASH 部分：
对于 22.1184MHz 和 24MHz，其预设值分别存储在 FLASH 的 0xFDF4 和 0xFDF3 地址。这些地址位于code存储区，这意味着它们存储的是常量数据，在程序运行过程中通常是只读的。
可以通过*(char code *)0xfdf3这样的指针形式来访问 24MHz 对应的预设值。不过要注意，
使用 STC - ISP 烧录程序时需要勾选 “在程序区结束处添加重要测试参数” 选项，否则这个地址的值会是 0xFF，就无法获取到正确的预设值。
RAM 部分：
22.1184MHz 和 24MHz 在 RAM 中的预设值地址分别是 0xFA 和 0xFB，这些地址位于idata存储区。idata存储区通常用于存储内部数据，包括一些需要在程序运行过程中可能会被修改的变量。
可以通过*(unsigned char idata *)0xFB这样的指针形式来访问 24MHz 对应的 RAM 中的预设值。
这个预设值上电即可读取，并且相对 FLASH 中的预设值更加可靠，但前提是通过 STC - ISP 刷写程序，若是通过stcgal写入的程序，这两个地址的数值为 0。
理解预设值和 IRTRIM 的作用
预设值：从之前提到的 FLASH（如 0xFDF3）和 RAM（如 0xFB）地址读取的预设值，很可能是用于对内部时钟进行校准的值。这些值是芯片出厂时标定好的，
用于在特定频率（如 22.1184MHz 和 24MHz）下对内部时钟电路进行微调，以确保时钟的准确性。
IRTRIM 寄存器：IRTRIM 是用于对内部时钟频率进行调整的寄存器。通过将从特定地址读取的预设值赋给 IRTRIM 寄存器，可以精细地调整内部时钟的频率，使其更接近目标频率（24MHz 或 22.1184MHz）。
CLKDIV 寄存器的作用
CLKDIV 寄存器用于设置时钟分频。当 CLKDIV = 0 时，表示不分频，这意味着系统时钟将以其原始的、经过 IRTRIM 调整后的频率运行。
例如，如果没有正确设置 CLKDIV，即使 IRTRIM 已经根据预设值调整好了内部时钟频率，也可能因为时钟分频而导致实际供给其他模块（如定时器）的时钟频率与预期不符。
代码实现步骤推测
读取预设值并赋值给 IRTRIM：
首先，在代码中需要按照之前提到的地址访问方式来读取预设值。例如，从 RAM 地址 0xFB 读取 24MHz 对应的预设值（假设定义为READ_RAM_IRTRIM_24MHZ宏），然后将这个值赋给 IRTRIM 寄存器。

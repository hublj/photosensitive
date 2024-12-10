FLASH 部分：


对于 22.1184MHz 和 24MHz，其预设值分别存储在 FLASH 的 0xFDF4 和 0xFDF3 地址。这些地址位于code存储区，这意味着它们存储的是常量数据，在程序运行过程中通常是只读的。
可以通过*(char code *)0xfdf3这样的指针形式来访问 24MHz 对应的预设值。不过要注意，
使用 STC - ISP 烧录程序时需要勾选 “在程序区结束处添加重要测试参数” 选项，否则这个地址的值会是 0xFF，就无法获取到正确的预设值。


RAM 部分：


22.1184MHz 和 24MHz 在 RAM 中的预设值地址分别是 0xFA 和 0xFB，这些地址位于idata存储区。idata存储区通常用于存储内部数据，包括一些需要在程序运行过程中可能会被修改的变量。
可以通过*(unsigned char idata *)0xFB这样的指针形式来访问 24MHz 对应的 RAM 中的预设值。
这个预设值上电即可读取，并且相对 FLASH 中的预设值更加可靠，但前提是通过 STC - ISP 刷写程序，若是通过stcgal写入的程序，这两个地址的数值为 0。


理解预设值和 IRTRIM 的作用


预设值：

从之前提到的 FLASH（如 0xFDF3）和 RAM（如 0xFB）地址读取的预设值，很可能是用于对内部时钟进行校准的值。这些值是芯片出厂时标定好的，
用于在特定频率（如 22.1184MHz 和 24MHz）下对内部时钟电路进行微调，以确保时钟的准确性。
IRTRIM 寄存器：IRTRIM 是用于对内部时钟频率进行调整的寄存器。通过将从特定地址读取的预设值赋给 IRTRIM 寄存器，可以精细地调整内部时钟的频率，使其更接近目标频率（24MHz 或 22.1184MHz）。

CLKDIV 寄存器的作用

CLKDIV 寄存器用于设置时钟分频。当 CLKDIV = 0 时，表示不分频，这意味着系统时钟将以其原始的、经过 IRTRIM 调整后的频率运行。
例如，如果没有正确设置 CLKDIV，即使 IRTRIM 已经根据预设值调整好了内部时钟频率，也可能因为时钟分频而导致实际供给其他模块（如定时器）的时钟频率与预期不符。

代码实现步骤推测

读取预设值并赋值给 IRTRIM：
首先，在代码中需要按照之前提到的地址访问方式来读取预设值。例如，从 RAM 地址 0xFB 读取 24MHz 对应的预设值（假设定义为READ_RAM_IRTRIM_24MHZ宏），然后将这个值赋给 IRTRIM 寄存器。


1. 硬件初始化与配置
 
- 定时器0初始化（10毫秒定时）
- 通过 Timer0_Init 函数将定时器0配置为12T模式，工作在模式1（16位定时器模式）。具体是对 AUXR 和 TMOD 寄存器进行操作来设置定时器模式。
- 设置了定时器0的初始值（ TL0 = 0xE0 和 TH0 = 0xB1 ），使得定时器每10毫秒产生一次中断。同时，清除定时器0溢出标志 TF0 ，启动定时器0（ TR0 = 1 ），使能定时器0中断（ ET0 = 1 ），并开启全局中断（ EA = 1 ），确保定时器0能正常工作并在定时结束（溢出）时触发中断服务函数。
- 输出引脚初始化
- 在 main 函数中，将输出引脚（假设为P3.3）初始化为低电平（ outputPin = 0 ），这个输出引脚可能用于后续控制其他外部设备（例如连接一个指示灯来表示光敏状态）。
 
2. 光敏传感器防抖处理
 
- 电平状态监测与防抖计时启动
- 在定时器0中断服务函数 Timer0_ISR 中，每次定时器中断发生时，先重新设置定时器初值，以保证定时器能持续以10毫秒为间隔进行定时。然后获取当前光敏传感器连接的输入引脚（假设为P5.4）的电平状态（ currentInput = inputPin ）。
- 当检测到输入引脚电平与上一次记录的电平 currentInputLevel 不同时，更新 currentInputLevel ，并开启防抖计时（ startTimer = 1 ， timerCount = 0 ）。
- 防抖计时与输出控制
- 如果防抖计时已经开始（ startTimer == 1 ），每次定时器中断 timerCount 就加1。当 timerCount 达到3000（这是根据定时器10毫秒定时间隔计算得到的30秒对应的计数次数）时，代表经过了30秒防抖时间。
- 此时根据当前输入引脚的最终稳定电平来控制输出引脚电平。如果 currentInput 为1，则将输出引脚（ outputPin ）置为1；如果 currentInput 为0，则将 outputPin 置为0。之后，重置 startTimer 和 timerCount ，等待下一次输入引脚电平变化来触发新的防抖计时过程。
 
总体而言，这段代码实现了利用定时器0进行定时，通过中断服务函数实现对光敏传感器输入电平的防抖处理，并根据稳定后的电平状态控制一个输出引脚的功能。


1. 正常工作时的逻辑
 
- 是的，在防抖时间（30秒）结束后，代码的逻辑是输入低电平，输出也是低电平；输入高电平，输出也是高电平。
- 在定时器0中断服务函数 Timer0_ISR 中，当防抖计时达到30秒（ timerCount >= THIRTY_SECONDS_COUNT ）后，会进行判断。如果当前输入引脚电平 currentInput 为1（高电平），就将输出引脚 outputPin 设置为1（高电平）；如果 currentInput 为0（低电平），就将 outputPin 设置为0（低电平）。
2. 防抖阶段的特殊情况
 
- 在防抖计时过程中（从输入引脚电平发生变化开始，到防抖时间30秒结束之前），输出引脚的电平不会立即跟随输入引脚电平变化。这是因为代码在等待30秒的防抖时间结束，以确定输入电平是否稳定，避免因为输入信号的抖动等不稳定因素导致输出频繁变化。



这两组代码虽然都是配置强推挽输出，但它们的应用场景和精细程度有所不同。
 
第一组代码（ P3M1 = 0x00; P3M0 = 0xFF; ）是对整个P3口进行批量设置。将P3口所有引脚一次性配置为强推挽输出模式，就像是给一整排开关统一进行操作，让它们全部开启强推挽输出功能。
 
第二组代码（ P3M0 |= 0x08; P3M1 &= ~0x08; ）是针对P3口的某个特定引脚（P3.3）进行设置。这种方式更精细，在不影响其他引脚原有设置（如果有）的情况下，单独将P3.3引脚配置为强推挽输出，就好比在一排开关中，只调整其中一个开关的设置，而让其他开关保持不变。
 
它们实现的最终目标都是配置强推挽输出，但第一组是粗粒度的全局设置，第二组是细粒度的单个引脚设置。
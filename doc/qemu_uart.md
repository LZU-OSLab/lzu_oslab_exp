# QEMU UART 介绍

## FIFO

FIFO 是“First-In First-Out”的缩写，在此处语境下，它是一个具有先入先出特点的缓冲区，设计目的是提高串口的通讯性能，避免频繁进入中断占用CPU资源，或因如果没有及时读走数据，下一个字节数据覆盖先前数据，导致数据丢失。

使用 FIFO，可以在连续收发若干个数据后才产生一次中断，然后一起进行处理，提高接收效率，缺点是实时性受影响。如果 FIFO 中的数据没有达到指定长度而无法产生中断，可以利用接收超时中断，即在一定的时间内没有接收到数据会进入中断，把不足 FIFO 长度的数据读取完。

解决实时性的问题：保持一个字节进一次接收中断，以保证实时性。把 FIFO 打开，在接收中断处理完后去判断缓冲区中是否仍有数据，若有则继续读入，防止数据丢失，此时不会有数据过短无法产生中断的问题。

D1 开发板的 UART 芯片有以下几个特性：

- 兼容 16550 UART
- 有两个独立的 FIFO：接收 FIFO 和发送 FIFO，UART0 中每个都是 64 bytes
- 支持 RS-232 字符的 5 到 8 位数据位，或 RS-485 格式的 9 位
- 支持 1、1.5 或 2 位停止位
- 可编程的校验（奇校验、偶校验或不校验）
- 支持 DMA
- 支持软件/硬件流控制

## 寄存器及数据结构设计

QEMU 的 ns16550a 有如下寄存器，每个寄存器宽度为 8bit，我们会使用全部这些寄存器。

```c
// QEMU
struct uart_16550a_regs {
    uint8_t RBR_THR_DLL; // 0x00, Receiver Buffer Register/Transmitter Holding Register/Divisor Latch LSB
    uint8_t IER_DLM; // 0x01, Interrupt Enable Register/Divisor Latch MSB
    uint8_t IIR_FCR; // 0x02, Interrupt Identification Register/FIFO Control Register
    uint8_t LCR; // 0x03, Line Control Register
    uint8_t MCR; // 0x04, Modem Control Register
    uint8_t LSR; // 0x05, Line Status Register
};
```

下面会根据几个比较重要的功能和设置来介绍一些寄存器。

RBR 是 UART 模式下串行输入端口上**接收**的数据。仅当 UART_LSR 中的数据就绪（DR）位为 1 时数据有效。如果在 FIFO 模式，则该寄存器访问接收 FIFO 的头部。THR 是在 UART 模式下的串行输出端口上**发送**的数据。对于这两个寄存器，当 FIFO 已满时，后续到来的数据会丢失。

DLL 寄存器存放分频除数的低位，DLM 寄存器存放分频除数的高位。

### 中断使能寄存器（IER）

要想使 16550a 能产生中断，需要将 IER 寄存器的对应为置 1。

| 位域 | 位域名称 | 位宽 | 访问 | 描述 |
| - | - | - | - | - |
| 7:4 | Reserved | 4 | 读/写 | 保留 |
| 3 | EDSSI | 1 | 读/写 | Modem 状态改变中断使能 |
| 2 | ELSI | 1 | 读/写 | 接收器线路状态改变中断使能 |
| 1 | ETBEI | 1 | 读/写 | 发送保存寄存器为空中断使能 |
| 0 | ERBFI | 1 | 读/写 | 接收有效数据中断使能 |

### 中断标识寄存器（IIR）

中断标识寄存器的格式如下：

| 位域 | 位域名称 | 位宽 | 访问 | 描述 |
| - | - | - | - | - |
| 7:6 | FEFLAG   | 2 | 读 | 00:未启用 FIFO, 11:已启用 FIFO, 01: FIFO 不可用 |
| 5:4 | Reserved | 2 | 读 | 保留 |
| 3:1 | IID | 3 | 读 | 中断源表示位，详见下表 |
| 0   | INTp| 1 | 读 | 中断待处理位，0 表示有中断待处理 |

16550a 可产生 5 种中断，这 5 种中断的发生由上述 IER 寄存器的位控制：

| 中断源表示位 | 优先级（1 为最高） | 中断类型 | 中断源 | 清除中断方法 | 需开启的 IER 中断 |
| - | - | - | - | - | - |
| 011 | 1 | 接收线路状态改变 (Receiver line status interrupt) | 线路状态改变（奇偶、溢出或帧错误，或打断中断） | 读 LSR | 接收器线路状态中断使能 |
| 010 | 2 | 接收到有效数据 (Receiver data interrupt) | 接收 FIFO 字符个数达到触发阈值 | 接收 FIFO 的字符个数低于触发阈值 | 接收有效数据中断使能 |
| 110 | 2 | 接收超时 (Character Timeout Indication) | 在FIFO 至少有一个字符，但在 4 个字符时间内没有任何操作，包括读和写操作 | 读接收 FIFO | 接收有效数据中断使能 |
| 001 | 3 | 发送保存寄存器为空 (Transmitter holding register empty) | 发送保存寄存器为空 | 写数据到 THR 或读 IIR | 发送保存寄存器为空中断使能 |
| 000 | 4 | Modem 状态改变 (Modem status interrupt) | MSR 寄存器改变 | 读 MSR | Modem 状态中断使能 |

### FIFO 控制寄存器（FCR）

这个寄存器主要控制 FIFO 的设置与清除。

| 位域 | 位域名称 | 位宽 | 访问 | 描述 |
| - | - | - | - | - |
| 7:6 | TL | 2 | 写 | 接收 FIFO 中断的发生阈值，'00': 1 字节, '01': 1/4满(QEMU: 4B), '10': 1/2满(QEMU: 8B), '11':距满差 2 字节(QEMU: 14B) |
| 5:3 | Reserved | 3 | 写 | 保留 |
| 2 | Txset | 1 | 写 | 写 1 以清除发送 FIFO 的内容，复位其逻辑 |
| 1 | Rxset | 1 | 写 | 写 1 以清除接收 FIFO 的内容，复位其逻辑 |
| 0 | FIFOE | 1 | 写 | FIFO 使能位，'1': 开启 FIFO, '0': 关闭 FIFO。改变此位会自动复位 FIFO |

### 线路控制寄存器（LCR）

这个寄存器主要控制寄存器访问与收发时数据帧的格式，包括校验、停止位与字符的长度。

| 位域 | 位域名称 | 位宽 | 访问 | 描述 |
| - | - | - | - | - |
| 7 | dlab | 1 | 读/写 | 分频锁存器访问位, '1': 访问操作分频锁存器, '0': 访问操作正常寄存器 |
| 6 | bcb | 1 | 读/写 | 打断控制位, '1': 此时串口的输出被置为 0(打断状态), '0': 正常操作 |
| 5 | spb | 1 | 读/写 | 指定奇偶校验位, '0': 正常奇偶校验, '1': 校验位恒为 LCR[4] 取反 |
| 4 | eps | 1 | 读/写 | 奇偶校验位选择, '0': 奇校验, '1': 偶校验 |
| 3 | pe | 1 | 读/写 | 奇偶校验位使能, '0': 没有奇偶校验位, '1': 在发送时生成奇偶校验位，接收时检查奇偶校验位 |
| 2 | sb | 1 | 读/写 | 停止位的位数, '0': 1 个停止位, '1': 在 5 位字符长度时是 1.5 个停止位，其他长度是 2 个停止位 |
| 1:0 | bec | 2 | 读/写 | 设定每个字符的位数, '00': 5 位, '01': 6 位, '10': 7 位, '11': 8 位 |

### 线路状态寄存器（LSR）

| 位域 | 位域名称 | 位宽 | 访问 | 描述 |
| - | - | - | - | - |
| 7 | ERROR | 1 | R | 错误表示位, '1': 至少有奇偶校验位错误，帧错误或打断中断的一个, '0': 没有错误 |
| 6 | TE | 1 | R | 发送为空且线路空闲表示位, '1': 发送寄存器/FIFO 和发送移位寄存器都为空, '0': 有数据 |
| 5 | THRE | 1 | R | 发送寄存器/FIFO 为空表示位, '1': 当前发送寄存器/FIFO 为空, '0': 有数据 |
| 4 | BI | 1 | R | 打断中断表示位, '1': 接收到 起始位＋数据＋奇偶位＋停止位都是 0，即有打断中断, '0': 没有打断 |
| 3 | FE | 1 | R | 帧错误表示位, '1': 接收的数据没有停止位, '0': 没有错误, 读取 LSR 寄存器后自动清空 |
| 2 | PE | 1 | R | 奇偶校验位错误表示位, '1': 当前接收数据有奇偶错误, '0': 没有奇偶错误, 读取 LSR 寄存器后自动清空 |
| 1 | OE | 1 | R | 数据溢出表示位, '1': 有数据溢出, '0': 无溢出, 读取 LSR 寄存器后自动清空 |
| 0 | DR | 1 | R | 接收数据有效表示位, '0': 在 FIFO 中无数据, '1': 在 FIFO 中有数据 |

其余寄存器如 MCR、MSR 我们用不到，就不再介绍。

## UART 操作模式

### 数据帧格式

UART_LCR 寄存器可以设置数据帧的基本参数：数据宽度(5-8 bits)、停止位（1、1.5 或 2 位停止位）、校验形式。

UART 的帧传输包括启动信号、数据信号、奇偶校验位和停止信号。最低有效位（LSB）首先传输。

- 开始信号（开始位）：数据帧的开始标志。根据 UART 协议，开始位传输信号为低电平。
- 数据信号（数据位）：数据位宽度可以配置为 5 - 8 位。
- 奇偶校验位：1 位的错误校正信号。奇偶校验位可通过 UART_LCR 设置奇校验、偶校验或无校验。
- 停止信号（停止位）：数据帧的停止位。UART_LCR 寄存器可以设置停止位为 1 位、1.5 位或 2 位。结束位传输信号为高电平。

### 波特率

波特率计算如下：波特率 = SCLK /（16 * 除数）。*D1 开发板中，SCLK 通常是 APB1，可在 CCU 中设置。*

其中除数（divisor, 分频）有 16 位，低 8 位在 UART_DLL 寄存器中，高 8 位在 UART_DLH 寄存器中。

如果要设置波特率为 115200，时钟源的频率为 24 MHz，根据公式可得除数约为 13.

### DLAB 定义

UART_LCR 的第 7 位为 DLAB(Divisor Latch Access Bit)，控制分频锁存器的访问。

- DLAB = 0，第一、二个寄存器是 UART_RBR/UART_THR (RX/TX FIFO) 寄存器、UART_IER 寄存器。
- DLAB = 1，第一、二个寄存器是 UART_DLL（除数低位）寄存器、UART_DLH（除数高位）寄存器

初始化设置除数时须设 DLAB = 1，结束配置时须设 DLAB = 0 以便访问数据。

## 程序设计

### UART 控制器初始化

1. 等待发送缓冲区清空（所有数据发送完）
2. 设置 UART_IER = 0 以关闭所有 UART 中断，避免在未设置完成时就触发中断。

波特率配置:
3. 计算 UART 波特率
4. 设置 UART_LCR[DLAB] = 1，其他位保留默认配置，使第一第二个端口进入波特率设置模式以访问分频锁存器
5. 向 UART_DLH 寄存器写入分频的高八位，向 UART_DLL 寄存器写入分频的低八位

控制参数配置：
6. 写入 UART_LCR 以设置数据宽度、停止位、奇偶校验方式，并写 UART_LCR[DLAB] = 0，恢复正常模式
7. 写入 UART_FCR 寄存器设置 FIFO 触发条件并启用 FIFO

中断配置：

8. 设置 UART_IER[ERBFI] = 1 以打开接收有效中断

### 轮询模式下发送数据

在我们的实验中 UART 一次发送的量通常较小，因此选择简单的轮询模式发送数据。

1. 检查 UART_LSR[THRE] 以检查发送 FIFO 是否为空。若 UART_LSR[THRE] = 0，则表明发送 FIFO 中的仍有数据，等待数据传输完成
2. 将要被传送的数据写入至 UART_THR 寄存器

### 中断模式中接收数据

1. 设置 UART_IER[ERBFI] = 1 以启用 UART 接收中断
2. 当接收 FIFO 的数据达到触发条件时（如达到触发阈值或超时)，UART 接受中断信号将会产生
3. 读取 UART_LSR[DR] 检查 RX_FIFO 状态，若该位为 1，继续从 UART_RBR 读取数据，直至该位变为 0，接收结束

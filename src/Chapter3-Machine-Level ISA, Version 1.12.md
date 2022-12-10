# Machine-Level ISA, Version 1.12

本章介绍了机器模式（M-mode）中可用的机器级操作，这是 RISC-V 系统中最高权限的模式。M 模式用于对硬件平台的低级访问，是复位时进入的第一个模式。M 模式也可以用来实现那些在硬件中直接实现过于困难或成本高昂的功能。RISC-V 的机器级 ISA 包含一个共同的核心，根据支持的其他权限级别和硬件实现的其他细节来扩展。

## Machine-Level CSRs

除了本节中描述的机器级 CSRs 外，M-mode 代码还可以访问较低特权级别的所有 CSRs。

### Machine ISA Register `misa`

misa CSR 是 WARL 读写寄存器，报告硬件 (hart) 支持的 ISA。该寄存器在任何实现中都必须是可读的，但是可以返回零值以指示未实现 misa 寄存器，这就需要通过一个单独的非标准机制确定 CPU 功能。

![Machine ISA register (misa)](../pic/Pic-3-1.png "Machine ISA register (misa)")

MXL（机器 XLEN）字段编码本机基本整数 ISA 宽度，如表 3.1 所示。MXL 字段在支持多个基本 ISA 宽度的实现中可能是可写的。M-mode 下的有效 XLEN, MXLEN，由 MXL 的设置给出，如果 misa 为零，则有一个固定的值。重置时，MXL 字段始终设置为最广泛支持的 ISA 变种。

![](../pic/Table3-1.pngs)

misa CSR 为 MXLEN 位宽。如果从 misa 读取的值不为零，该值的 MXL 字段总是表示当前的 MXLEN。如果对 misa 的写操作导致 MXLEN 发生更改，则 MXL 的位置将以新的宽度移动到 misa 的最高有效两位。

> 可以使用返回的 misa 值的符号上的分支，以及可能在符号上左移一个分支和第二个分支，来快速确定基本宽度。这些检查可以用汇编代码编写，而无需知道机器的寄存器宽度（XLEN）。基本宽度由 XLEN = 2^(MXL + 4) 给出。如果 misa 为零，则可以通过将立即数 4 放置在一个寄存器中，然后一次将寄存器左移 31 位来找到基本宽度。如果在一次移位后为零，则该机器为 RV32。如果两次移位后为零，则机器为 RV64，否则为 RV128。

Extensions 字段编码了目前存有的标准扩展，其每个 bit 都对应了字母表中的一个字母（bit 0 编码扩展“A”是否存在，bit 1 编码扩展“B”是否存在... 直至 bit 25 编码“Z”）。如果基础 ISA 是 RV32I、RV64I 或 RV128I，则置位“I”bit，否则如果基础 ISA 是 RV32E，则置位“E”bit。

Extensions 字段是一个能包含可写位的 WARL 字段（如果实现允许修改所支持的 ISA）。

复位（reset）时，Extensions 应包含所支持扩展的最大集，如果 E 和 I 都可用，则优先选择 I。

在通过清除 misa 中相应 bit 来禁止一个标准扩展时，由该扩展所定义或修改的指令和 CSR 将恢复为该扩展未实现时的定义，或者保留行为（revert to their defined or reserved behaviors as if the extension is not implemented）。

RV128 base ISA 的设计尚未完工，尽管预计本 specification 中大部分的剩余部分都将适用于 RV128，但本版本的文档仅关注 RV32 和 RV64。

如果支持用户模式（user mode），则将“U”bit 置位；如果支持主管模式（supervisor mode），则将“S”bit 置位。

如果存在任何非标准扩展（non-standard extensions），则将“X”bit 置位。

![](../pic/Table3-2.png)

“E”位是只读的。除非将 misa 硬连线为零，否则“E”位始终读取为“I”位的补码（补集？）。同时支持 RV32E 和 RV32I 的实现可以通过清除“I”位来选择 RV32E。

如果 ISA 功能 x 依赖 ISA 功能 y，则尝试启用功能 x 但禁用功能 y 会导致两个功能都被禁用。例如，设置“F” = 0 和“D” = 1 会导致同时清除“F”和“D”。

具体实现可能会在 2 或多个 misa 字段的集体设置上施加额外约束，此时将它们的集体看作是一个 WARL 字段。试图向其中写入一个不支持的组合会导致这些 bits 被置为某个支持的组合。

写 misa 可能会增加 IALIGN，例如，通过禁用 C 扩展。如果要写入 misa 的指令增加了 IALIGN，而后一条指令的地址未按 IALIGN 位对齐，则将抑制对 misa 的写入，从而使 misa 保持不变。

在软件启用一个之前被禁用的扩展时，除该扩展另有规定（specified），否则所有单独与该扩展有关的状态都将是未指定的（unspecified）。

### Machine Vendor ID Register `mvendorid`

`mvendorid` CSR 是一个 32 位只读寄存器，提供内核供应商的 JEDEC 制造商 ID。此寄存器在任何实现中都必须是可读的，但可以返回 0，表示该字段未实现或这是非商业实现。

![厂商 ID 寄存器 mvendorid](../pic/Pic-3-2.png "厂商ID寄存器 mvendorid")

JEDEC 制造商 ID 通常编码为单字节连续的 `0x7f` 代码的序列，以不等于 `0x7f` 的单字节 ID 终止，并且在每个字节的最高有效位中带有奇校验位。`mvendorid` 在 Bank 字段中编码单字节的连续代码，并在 `Offset` 字段中编码最后一个字节，丢弃奇偶校验位。例如，JEDEC 制造商 ID `0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x8a`（十二个连续代码，后跟 0x8a）将在 mvendorid 字段中编码为 `0x60a`。

> 译者注：JEDEC 固态技术协会（JEDEC Solid State Technology Association）是固态及半导体工业界的一个标准化组织，它由约 300 家公司成员组成，约 3300 名技术人员通过 50 个不同的委员会运作，制定固态电子方面的工业标准。JEDEC 曾经是电子工业联盟（EIA）的一部分：联合电子设备工程委员会（Joint Electron Device Engineering Council，JEDEC）。该协会制定了一个制造商标识码的标准：[Standard Manufacturer’s Identification Code](http://www.softnology.biz/pdf/JEP106AV.pdf)，通过读取`mvendorid`寄存器值，查阅该标准即可确定制造商。

> 注：用 JEDEC 的话来说，Bank 编号比 Continuation 的数量大 1；因此，mvendorid Bank 字段编码的值比 JEDEC Bank 编号小一。

> 注：以前，供应商 ID 是 RISC-V 基金会分配的编号，但这与 JEDEC 在维护制造商 ID 标准方面的工作重复。在撰写本文时，向 JEDEC 注册制造商 ID 的一次性费用为 500 美元。

## Machine-Level Memory-Mapped Registers

## Machine-Mode Privileged Instructions

### Environment Call and Breakpoint

![](../pic/Pic-Reg-ECALL.jpg "")

### Trap-Return Instruction

### Wait for Interrupt

等待中断指令 (WFI) 为实现提供了一个提示，即当前的 hart 可以停止，直到需要服务中断。WFI 指令的执行也可以用来通知硬件平台合适的中断应该优先路由到这个 hart。WFI 在所有特权模式下都可用，并且可用于 U 模式 (可选地)。当 mstatus 中的 TW = 1 时，该指令可能会引发非法指令异常，如第 3.1.6.5 节所述。

![](../pic/Pic-Reg-WFI.jpg)

如果在 hart 停止时存在或稍后出现启用的中断，则中断 trap 将在以下指令上执行，即在 trap 处理程序中恢复执行并且 `mepc = pc + 4`。

### Custom SYSTEM Instructions
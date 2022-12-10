# Control and Status Registers (CSRs)

### 机器 Trap 委托寄存器 (medeleg and mideleg)

默认情况下，任何特权级别的所有 Trap 都在机器模式下处理，尽管机器模式处理程序可以使用 MRET 指令（第 3.3.2 节）将 Trap 重定向回适当的级别。

为了提高性能，实现可以在 `medeleg` 和 `mideleg` 中提供单独的读/写位，以指示某些异常和中断应由较低的特权级别直接处理。机器异常委托寄存器（machine exception delegation register,medeleg）和机器中断委托寄存器（machine interrupt
delegation register, mideleg）为 MXLEN 位读写寄存器。

在具有 S-mode 的系统中，`medeleg` 和 `mideleg` 寄存器必须存在，并且在 medeleg 或 mideleg 中设置一个位将在 S-mode 或 U-mode 中发生时将相应的 Trap 委托给 S-mode Trap 处理程序。在没有 S-mode 的系统中，`medeleg` 和 `mideleg` 寄存器不应该存在。

> 在 1.9.1 及更早的版本中，这些寄存器存在但仅在 M 模式或没有 N 系统的 M/U 中硬接线为零。在这些情况下没有理由要求它们返回零，因为 `misa` 寄存器指示它们是否存在。

当 Trap 委托给 S 模式时，将 Trap 原因写入 `scause` 寄存器；`sepc` 寄存器写入捕获 Trap 的指令的虚拟地址；`stval` 寄存器写入异常特定数据；`mstatus` 的 `SPP` 字段写入 trap 时的 active 特权模式；`mstatus` 的 `SPIE` 字段写入 trap 时 `SIE` 字段的值；`mstatus` 的 `SIE` 字段被清除。`mcause`、`mepc` 和 `mtval` 寄存器以及 `mstatus` 的 `MPP` 和 `MPIE` 字段没有写入。

实现可以选择对可委托 Trap 进行子集化，通过向每个位位置写入 `1` 找到支持的可委托位，然后读回 `medeleg` 或 `mideleg` 中的值以查看哪些位位置为 1。

一个实现不应有任何 `medeleg` 位是只读的，即，任何可以委托的同步 Trap 必须支持不被委托。类似地，一个实现不应将对应于机器级中断的 `mideleg` 的任何位固定为只读（但可以为较低级别的中断这样做）。

> 1.11 版及更早版本禁止将 `mideleg` 的任何位设为只读。平台标准可能总是添加此类限制。

Trap 永远不会从特权更高的模式转换到特权更低的模式。例如，如果 M-mode 已将非法指令异常委托给 S-mode，并且 M-mode 软件随后执行了一条非法指令，则 Trap 将在 M-mode 中捕获，而不是委托给 S-mode。相比之下，Trap 可以水平放置。使用相同的示例，如果 M 模式已将非法指令异常委托给 S 模式，并且 S 模式软件稍后执行非法指令，则 Trap 在 S 模式中发生。

委托中断导致中断在委托者特权级别被屏蔽。例如，如果通过设置 `mideleg[5]` 将监控定时器中断 (STI) 委托给 S 模式，则在 M 模式下执行时不会采用 `STI`。相比之下，如果 `mideleg[5]` 清晰，则可以在任何模式下进行 STI，并且无论当前模式如何，都会将控制权转移到 M 模式。

![机器异常委托寄存器 `medeleg`](../pic/Pic-3-10.jpg "机器异常委托寄存器`medeleg`")

`medeleg` 为第 39 页表 3.6 中所示的每个同步异常分配了一个位位置，位位置的索引等于 `mcause` 寄存器中返回的值（即，设置位 8 允许将用户模式环境调用委托给较低权限的 Trap 处理程序）。

![机器中断委托寄存器 mideleg](../pic/Pic-3-11.jpg "机器中断委托寄存器mideleg")

`mideleg` 保存各个中断的 Trap 委托位，位布局与 `mip` 寄存器中的位布局相匹配（即，`STIP` 中断委托控制位于位 5）。对于不能在低特权模式下发生的异常，相应的 `medeleg` 位应该是只读的零。特别是，`medeleg[11]` 是只读零。
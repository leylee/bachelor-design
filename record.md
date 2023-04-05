# 过程记录

## 20230301 D1 开发板虚拟内存映射错误

### 现象

在 D1 上启动时执行到映射虚拟内存的函数 `activateMapping()` 中的 `csrw satp` 指令后, 会在执行后续指令时异常卡死. 在 qemu 中执行却没有这个问题.

### 原因

根据 *The RISC-V Instruction Set Manual
Volume II: Privileged Architecture* 4.3.1 节, 叶子页表项的 accessed 和 dirty bit 有两种管理方式:

> - When a virtual page is accessed and the A bit is clear, or is written and the D bit is clear, a
page-fault exception is raised.
> - When a virtual page is accessed and the A bit is clear, or is written and the D bit is clear, the
implementation sets the corresponding bit(s) in the PTE. The PTE update must be atomic
with respect to other accesses to the PTE, and must atomically check that the PTE is valid
and grants sufficient permissions.
Updates of the A bit may be performed as a result of
speculation, but updates to the D bit must be exact (i.e., not speculative), and observed in
program order by the local hart. Furthermore, the PTE update must appear in the global
memory order no later than the explicit memory access, or any subsequent explicit memory
access to that virtual page by the local hart. The ordering on loads and stores provided by
FENCE instructions and the acquire/release bits on atomic instructions also orders the PTE
updates associated with those loads and stores as observed by remote harts.  
The PTE update is not required to be atomic with respect to the explicit memory access that
caused the update, and the sequence is interruptible. However, the hart must not perform
the explicit memory access before the PTE update is globally visible.

在 *玄铁 C906 用户手册* 5.2.2.1 中, 明确表示 A 位和 D 位为 0 时, 对该页表的访问会触发 Page Fault, 对应了 riscv 手册中的第一种管理方式.
>
> - D – Dirty
>   - D 位为 1 时，表明该页是否被改写。
>   - 1’b0: 当前页未被写/不可写；
>   - 1’b1: 当前页已经被写/可写。
>   - 此位在 C906 的硬件实现与 W 属性类似。当 D 位为 0 时，对此页面进行写操作会触发 Page
Fault (store) 异常，通过在异常服务程序中配置 D 位来维护该页面是否已经被改写/可写的定义。
该位复位为 0。
> - A – Accessed
>   - A 位为 1 时，表明该页被访问过。为 0 时表示没被访问过，对该页表的访问会触发 Page Fault
>   - (对应访问类型) 异常且将该域置为 1。该位复位为 0。

在 mapping.c 中的 `mapLinearSegment()` 函数中, 设置页表项的语句为

```c
*entry = ((vpn - KERNEL_PAGE_OFFSET) << 10) | segment.flags | VALID;
```

仅设置了 valid 位, 没有设置 accessed & dirty 位, 因此后续访存会触发 Page Fault.

猜测 QEMU 能够正常执行是使用了 risc-v 规范中的第二种管理方式.

### 解决方法

将 `mapLinearSegment()` 函数中, 设置页表项的语句改为

```c
*entry = ((vpn - KERNEL_PAGE_OFFSET) << 10) | segment.flags | VALID | ACCESSED | DIRTY;
```

修复后, QEMU 和 D1 均能正常执行.

## 20230302 `allocTid()` 失败

### 现象

第一次执行到 `allocTid()` 函数时, 无法找到未使用的 tid, kernel panic 输出 `"Alloc tid failed!"`

### 原因

CPU 线程池 `CPU.pool` 未初始化.

### 解决方法

在 `newThreadPool()` 函数中添加初始化代码:

```c
for (usize i = 0; i < MAX_THREAD; ++i) {
    pool.threads[i].occupied = 0;
}
```

## 内核中 .bss 段未初始化导致多个错误

### 现象

同 `allocTid()失败` 章节, 其本质原因就是 `.bss` 段未初始化.

### 解决办法

在 `initMemory()` 函数开头添加初始化 `.bss` 段的代码, for 循环清零

添加代码后再次出现异常, 经检查发现内核的栈空间开在了 `.bss.stack` 上, 也被初始化清零了, 而此时栈上已经有了在使用的数据. 解决办法是把栈空间开在 `.stack` 段上. 至此问题 解决.

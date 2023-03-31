# 环境搭建 (Archlinux)

## 安装 riscv 工具链

```bash
$ sudo pacman -S riscv64-linux-gnu-gcc riscv64-linux-gnu-gdb
```

## 安装 xfel

```bash
$ sudo pacman -S xfel libusb
```

## 安装 T-HeadDebugServer

从 [平头哥社区](https://occ.t-head.cn/community/download) 下载 `T-Head-DebugServer-linux-x86_64-V5.16.7-20230109.sh.tar.gz`

```bash
$ tar -xzf ~/Downloads/T-Head-DebugServer-linux-x86_64-V5.16.7-20230109.sh.tar.gz
$ ls
T-Head-DebugServer-linux-x86_64-V5.16.7-20230109.sh
$ sudo mkdir -p /usr/local/share/T-Head-DebugServer
$ sudo ./T-Head-DebugServer-linux-x86_64-V5.16.7-20230109.sh
```

安装过程中提示输入安装路径, 输入 `/usr/local/share/T-Head-DebugServer` 即可. 安装脚本运行结束后, 创建 `/usr/local/bin/T-HeadDebugServer`:

```bash
#!/bin/bash

# 进入 T-Head-DebugServer 的安装目录, 否则会产生找不到 .so 的错误
pushd /usr/local/share/T-Head-DebugServer > /dev/null
./DebugServerConsole.elf "$@"
# 返回上一个目录
popd > /dev/null
```

添加 execute 权限:

```bash
$ sudo chmod +x /usr/local/bin/T-HeadDebugServer
```

尝试执行, 可以获得如下结果:

```bash
$ T-HeadDebugServer 
+---                                                    ---+
|  T-Head Debugger Server (Build: Jan  9 2023, Linux)      |
   User   Layer Version : 5.16.07 
   Target Layer version : 2.0
|  Copyright (C) 2022 T-HEAD Semiconductor Co.,Ltd.        |
+---                                                    ---+
ERROR: No T-HEAD CKLINK has been connected to PC or T-HEAD CKLINK driver isn't installed correctly!
ERROR: Cklink connection failed.
```

将调试器与开发板连接并上电, 并连接调试器和电脑, 再次启动 DebugServer:
```bash
$ T-HeadDebugServer
+---                                                    ---+
|  T-Head Debugger Server (Build: Jan  9 2023, Linux)      |
   User   Layer Version : 5.16.07 
   Target Layer version : 2.0
|  Copyright (C) 2022 T-HEAD Semiconductor Co.,Ltd.        |
+---                                                    ---+
T-HEAD: CKLink_Lite_V2, App_ver 2.36, Bit_ver null, Clock 2526.316KHz,
       5-wire, With DDC, Cache Flush On, SN CKLink_Lite_V2-45C901C8D8.
+--  Debug Arch is CKHAD.  --+
+--  CPU 0  --+
T-HEAD Xuan Tie CPU Info:
        WORD[0]: 0x0910090d
        WORD[1]: 0x11002000
        WORD[2]: 0x260c0001
        WORD[3]: 0x30030066
        WORD[4]: 0x42180000
        WORD[5]: 0x50000000
        WORD[6]: 0x60000853
        MISA   : 0x8000000000b4112d
Target Chip Info:
        CPU Type is C906FDV, Endian=Little, Vlen=128, Version is R1S0P2.
        DCache size is 32K, 4-Way Set Associative, Line Size is 64Bytes, with no ECC.
        ICache size is 32K, 2-Way Set Associative, Line Size is 64Bytes, with no ECC.
        Target is 1 core.
        MMU has 256 JTLB items.
        PMP zone num is 8.
        HWBKPT number is 2, HWWP number is 2.
        MISA: (RV64IMAFDCVX, Imp M-mode, S-mode, U-mode)

GDB connection command for CPUs(CPU0):
        target remote 127.0.0.1:1025
        target remote 192.168.1.230:1025
        target remote 192.168.189.19:1025

****************  DebuggerServer Commands List **************
help/h
        Show help informations.
*************************************************************
DebuggerServer$ q
q
DebuggerServer$ 
DebuggerServer quit
```

DebugServer 已连接到调试器并读取开发板上的 CPU 型号.
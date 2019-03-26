```
title : How syzkaller works
author ： 张津（listennter）
date : 2019-03-25 
```

# How Syzkaller  Works

本文主要参考`Google syzkaller` 项目的说明文档中`How syzkaller works`文档译制而来，帮助大家快速理解`syzkaller` 工具的大体工作流程。下面就是对`szykaller`工具如何工作的简单说明。

您可能需要使用`syzkaller`去fuzz一些Linux kernel的外部接口(external interface),目前仅仅支持网络堆栈的外部fuzz(此处更新于2017-10-27)。

`syzkaller` 已经可以支持网络堆栈的外部fuzzing，这个功能主要通过`TUN/TAP`接口实现。它允许构建一个虚拟网络模拟内核接口从外部网络接受数据报文，这个过程与通过真实网络接口传递数据报文使用相同的API调用序列(驱动程序层除外)。在使用这个功能的时候您需要在kernel config中允许`CONFIG_TUN`参数.这一部分的具体代码参见:<https://github.com/google/syzkaller/blob/master/sys/linux/vnet.txt>

## Overview

syzkaller系统的过程结构如下图所示; 红色标签表示相应的配置选项。

![process_structure](F:\listennter\syzkaller\images\process_structure.png)

`syz-manager` 进程负责几个VM示例的运行(process starts)，监控(monitors)和重启(restarts)工作.同时会在VM实例中运行`sys-fuzzer`进程。`syz-manager`会维持语料库并且完成崩溃转储(crash storage)工作.与`syzkaller`进程不同的是，`syz-manger`模块运行在稳定的内核上不会被白噪音(white-noise) fuzzer加载。

`syz-fuzzer`运行在不稳定的VM实例中，引导整个fuzzing进程(包括生成输入,输入变异和输入最小化)，并且通过RPC将触发新的输入到`szykaller`进程。同时可以瞬时触发`szy-executor`进程

每个`syz-executor`进程执行一个输入(syscalls的序列)，它听从`szy-fuzzer`的指令，并且将执行结果返回给`szy-fuzzer` 进程。它被设计的尽可能简单，使用C++语言编写并编译成为静态库，使用共享内存进行通信。

## Syscall descriptions

`szykaller`使用系统调用接口(syscall interface)的声明描述去操作程序，下面是一个样例:

```c
open(file filename, flags flags[open_flags], mode flags[open_mode]) fd
read(fd fd, buf buffer[out], count len[buf]) len[buf]
close(fd fd)
open_mode = S_IRUSR, S_IWUSR, S_IXUSR, S_IRGRP, S_IWGRP, S_IXGRP, S_IROTH, S_IWOTH, S_IXOTH
```

更多相关的描述可以在:<https://github.com/google/syzkaller/tree/master/sys/linux>中查看。`syz-fuzzer`通过这样的系统调用描述生成调用序列，并使用`syz-executor`去执行。

## Crash reports

当`syzkaller`捕获一次崩溃，他会将相关的信息保存在`workdir/crashes`中。在这个目录下会给每个不通过类型的崩溃创建一个子目录，基本样例如下:

```
 - crashes/
   - 6e512290efa36515a7a27e53623304d20d1c3e
     - description
     - log0
     - report0
     - log1
     - report1
     ...
   - 77c578906abe311d06227b9dc3bffa4c52676f
     - description
     - log0
     - report0
     ...
```

这些描述文件是通过一组正则运算式提取的，如果您使用了不同的内核架构进行测试，或者您看到了以前没有出现过的内核错误信息，那么您可能需要扩展这个集合。

下面提供三种特殊的崩溃类型:

- `no output from test machine`: 本次测试没有产生任何输出
- `lost connection to test machine`:与主机的ssh连接意外关闭
- `test machine is not executing programs`: 机器仍在运行(没有crash)，但是长时间没有运行测试程序。


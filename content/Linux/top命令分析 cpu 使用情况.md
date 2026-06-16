+++
date = '2026-03-23T20:45:00+08:00'
draft = false
title = 'top命令分析 cpu 使用情况'
+++

今天面试的时候，被面试关问了一个分析服务器 cpu 使用情况的问题，毫无疑问的祭了。现在悲伤的学习一下。

在 Linux 中，我们可以使用 `top` 命令查看当前服务器的 cpu 使用情况，以下是输出：

```bash
top - 19:46:52 up  1:44,  2 users,  load average: 0.00, 0.00, 0.00
Tasks:  32 total,   1 running,  31 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7746.1 total,   5931.0 free,    803.6 used,   1011.5 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   6777.0 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
     65 root      19  -1   72328  12412  11644 S   0.7   0.2   0:19.93 systemd-journal
```

### 一、看整体 CPU 使用率 

```bash
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

该行最关键，个字段含义是：

- **us**：用户态占用 CPU 的百分比

- **sy**：内核态占用 CPU 的百分比

- **ni**：被调整过优先级的进程占用 CPU 的百分比

- **id**：空闲 CPU 的百分比

- **wa**：等待 I/O 的时间百分比

- **hi**：硬中断占用 CPU 的百分比

- **si**：软中断占用 CPU 的百分比

- **st**：被虚拟机偷走的 CPU 时间百分比

### 二、单个进程的 CPU 占用

关键列是：

```bash
%CPU
```

### 三、load average

```bash
load average: 0.00, 0.00, 0.00
```

表名最近 1 分钟，5 分钟、15 分钟的平均负载都几乎为 0。

**因此得出结论，我们可以通过 %Cpu(s) 查看整体 CPU 使用率；通过 %CPU 查看单个进程的 CPU 占用；通过 load average 查看过去的历史时间 CPU 占用情况**

使用 `htop` 可以获取体验感更好的实时交互面板，长这个样子：

![image-20260323204357234](C:\Users\hanjie\AppData\Roaming\Typora\typora-user-images\image-20260323204357234.png)

上面的 `0[...]` 这种的是每一个核的使用情况。

---
title: XDP实验
date: 2024-09-04T19:31:06+08:00
lastmod: 2024-09-04T19:31:06+08:00
author: wantflying
authorlink: https://github.com/wantflying
cover: img/xdp1.png
categories:
  - eBPF
tags:
  - xdp
  - ebpf
draft: true
---

本实验基于 xdp 官方 demo 学习参考使用，仅作个人记录学习使用。
xdp 实践：

<!--more-->

### 实验环境搭建

```
#本实验基于xdp官方demo 学习参考使用，仅作个人记录学习使用
git clone https://github.com/wantflying/xdp-labs
#检查实验环境是否符合条件
./configure
#编译libbpf libxdp
make
#将编译后的文件install 到system wide
cd xdp-labs/lib/xdp-tools/
make install
```

---

### xdp 程序构建、部署、卸载示例

```
#进入basic01 编译文件，epf程序包含两个部分，用户空间程序以及内核空间程序
cd basic01
make
#可以通过llvm-objdump或者readelf看文件 这个时候c语言格式的ebpf源码文件会被编译成elf文件（可执行或者可链接的文件格式）
#这里可以看到有一行是r0=2，就是说r0寄存器的值是2，ebpf有11个寄存器，从0到10，r0一般是存放程序
返回值，r1到r6一般是上下文参数或者用户空间带进内核空间参数，r10一直存放堆栈指针位置
root@lima-learning-ebpf:/Users/beifeng/ebpf-demo/xdp-labs/basic01# llvm-objdump -S xdp_pass_kern.o

xdp_pass_kern.o:        file format elf64-bpf
Disassembly of section xdp:
0000000000000000 <xdp_prog_simple>:
;       return XDP_PASS;
       0:       b7 00 00 00 02 00 00 00 r0 = 2
       1:       95 00 00 00 00 00 00 00 exit

#接下来就是将elf文件attach到内核中，有三种方式：
1.iproute2
ip link set dev lo xdpgeneric obj xdp_pass_kern.o sec xdp
2.xdp-loader
root@lima-learning-ebpf:/Users/beifeng/ebpf-demo/xdp-labs/basic01# xdp-loader load -m skb lo xdp_pass_kern.o
root@lima-learning-ebpf:/Users/beifeng/ebpf-demo/xdp-labs/basic01# xdp-loader status lo
CURRENT XDP PROGRAM STATUS:

Interface        Prio  Program name      Mode     ID   Tag               Chain actions
--------------------------------------------------------------------------------------
lo                     xdp_prog_simple   skb      105  3b185187f1855c4c
3.c代码编程手工load
./xdp_pass_user --dev lo
./xdp_pass_user --dev lo --unload-all
```
---
### eBPF持久化机制-Map
- **类型**
BPF_MAP_TYPE_ARRAY
BPF_MAP_TYPE_PERCPU_ARRAY
- **Pinning机制实现Map共享**
Pinning机制对于eBPF程序来说没有区别，map创建使用其实还是跟之前一样，在内存中创建好map之后，如果load和查看都是在一个程序，那根据bpf程序直接直接拿到map的fd，如果不在那就在eBPF程序load之后，在目录上挂载一个目录指向这个map，这样其他程序也可以根据这个path，拿到这个map的fd。
有一点要注意就是如果eBPF程序卸载之后又去load，然后load新生成的map，会重新pinning，这个个用户空间查看程序拿的还是之前的eBPF对应的Map的fd，因此就两种思路：一种查看程序每次取数据时候，查看mapid是否变化，如果有，说明需要重新启动查看程序，然后复用。另外一种就是eBPF程序不生成新的map，而是使用旧的map
```
#这里我们其实可以看到bpf已经被挂载，是因为我们之前使用iproute挂载xdp自动创建
#如果没有这个，我们可以手动创建mount -t bpf bpf /sys/fs/bpf/
#到这里我其实也可以猜到这种共享是基于什么原理了,基于文件描述符实现信息共享：
root@lima-ebpfvm:/# mount |grep bpf
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)

```



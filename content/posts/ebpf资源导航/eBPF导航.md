---
title: eBPF实验环境搭建
date: 2024-08-26T13:47:53+08:00
lastmod: 2024-08-26T13:47:53+08:00
author: wantflying
authorlink: https://github.com/wantflying
cover: img/ebpf.png
categories:
  - eBPF
tags:
  - Linux
  - 资源
draft: false
---

ebpf实验环境搭建，需要**良好的网络质量**，以及bpftool使用

<!--more-->

### github链接
- https://github.com/iovisor/bcc  bcc
- https://github.com/lizrice/learning-ebpf learing-ebpf书籍配套案例学习
- https://isovalent.com/books/learning-ebpf/#form  可以获取《learning ebpf》 pdf副本，填写邮箱即可
- https://isovalent.com/resource-library/labs/   cilimu 线上实操实验
---
### 环境安装
```
#https://github.com/lima-vm/lima 使用lima搭建实验环境 
#本人使用的是arm64 mac，使用brew安装lima
brew install lima
#下载learing ebpf案例库
git clone --recurse-submodules https://github.com/lizrice/learning-ebpf
cd learning-ebpf
#使用lima 配置文件，也可以根据配置文件自己微调参数
limactl start learning-ebpf.yaml
limactl shell learning-ebpf

#执行案例python文件 测试环境
```

### bpftool常见操作
```
#查看当前所有的bpf程序
bpftool prog list
#查看bpf 字节码
bpftool prog dump xlated name  hello
#查看bpf汇编代码
bpftool prog dump jited name  hello

#为hello bpf程序添加事件
bpftool net attach xdp id 113 dev eth0
bpftool net list
#查看hello程序输出
cat /sys/kernel/debug/tracing/trace_pipe
或者
bpftool prog tracelog

#查看bpf map
bpftool map list
bpftool map dump name hello.bss

#取消程序事件绑定，并卸载程序
bpftool net detach xdp dev eth0
rm /sys/fs/bpf/heloo

```

### 专业名称
- ELF
	ELF全称是Executable and Linkable Format，意思就是说它是一种可执行文件的文件格式，可以代表一些x86服务器二进制文件、可执行文件、对象代码等


### XDP-LAB
```

git submodule add https://github.com/libbpf/libbpf/ lib/libbpf
git submodule add https://github.com/xdp-project/xdp-tools/ lib/xdp-tools
```
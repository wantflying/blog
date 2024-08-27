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

ebpf实验环境搭建，需要**良好的网络质量**

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

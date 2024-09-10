---
title: XDP实验
date: 2024-09-04T19:31:06+08:00
lastmod: 2024-09-04T19:31:06+08:00
author: wantflying
authorlink: https://github.com/wantflying
cover: img/xdp2.png
categories:
  - eBPF
tags:
  - xdp
  - ebpf
draft: false
---

本实验基于 xdp 官方 demo 学习理解参考使用，仅作个人记录学习使用。

<!--more-->

### 实验环境搭建
本实验基于xdp官方demo 学习参考使用，仅作个人记录学习使用

```shell
git clone https://github.com/xdp-project/xdp-tutorial/tree/master
#检查实验环境是否符合条件
./configure
#编译libbpf libxdp
make
#将编译后的文件install 到system wide
cd xdp-labs/lib/xdp-tools/
make install
#实验网络脚本设置
../testenv/testenv.sh alias
alias t='/Users/beifeng/ebpf-demo/xdp-tutorial/testenv/testenv.sh'
../testenv/testenv.sh setup --name env1

```

---

### xdp 程序构建、部署、卸载示例

```shell
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
- **类型** <br>
BPF_MAP_TYPE_ARRAY
BPF_MAP_TYPE_PERCPU_ARRAY
- **Pinning机制实现Map共享** <br>
Pinning机制对于eBPF程序来说没有区别，map创建使用其实还是跟之前一样，在内存中创建好map之后，如果load和查看都是在一个程序，那根据bpf程序直接直接拿到map的fd，如果不在那就在eBPF程序load之后，在目录上挂载一个目录指向这个map，这样其他程序也可以根据这个path，拿到这个map的fd。
有一点要注意就是如果eBPF程序卸载之后又去load，然后load新生成的map，会重新pinning，这个个用户空间查看程序拿的还是之前的eBPF对应的Map的fd，因此就两种思路：一种查看程序每次取数据时候，查看mapid是否变化，如果有，说明需要重新启动查看程序，然后复用。另外一种就是eBPF程序不生成新的map，而是使用旧的map
```shell
#这里我们其实可以看到bpf已经被挂载，是因为我们之前使用iproute挂载xdp自动创建
#如果没有这个，我们可以手动创建mount -t bpf bpf /sys/fs/bpf/
#到这里我其实也可以猜到这种共享是基于什么原理了,基于文件描述符实现信息共享：
root@lima-ebpfvm:/# mount |grep bpf
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)

#循环查看mapid，如果变更重新获取
while (1) {
		prev = record; /* struct copy */

		map_fd = open_bpf_map_file(pin_dir, "xdp_stats_map", &info);
		if (map_fd < 0) {
			return EXIT_FAIL_BPF;
		} else if (id != info.id) {
			printf("BPF map xdp_stats_map changed its ID, restarting\n");
			close(map_fd);
			return 0;
		}

		stats_collect(map_fd, map_type, &record);
		stats_print(&record, &prev);
		close(map_fd);
		sleep(interval);
	}
#load ebpf时不生成新的map，需要在open 和 load之间处理一下
obj = open_bpf_object(file, ifindex);  
......
err = bpf_map__reuse_fd(map, pinned_map_fd);
......
err = bpf_object__load(obj);
```

---
### 数据包处理
#### 数据包格式
- 数据包结构以及返回码
```c
struct xdp_md {
	__u32 data;/*数据包开始指针*/
	__u32 data_end;/*数据包结束指针，用来校验数据包格式，越界检查*/
	__u32 data_meta;/*xdp程序附带的元数据起始地址*/
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex 数据包设备id*/
	__u32 rx_queue_index;  /* rxq->queue_index  数据包队列索引*/
};

/*程序返回代码*/
enum xdp_action {
	XDP_ABORTED = 0,/*数据包丢弃，产生一个丢弃的tracepoint事件：xdp：xdp-exception*/
	XDP_DROP,/*数据包直接丢弃*/
	XDP_PASS,/*数据包正常传递内核网络协议栈*/
	XDP_TX,/*丢回原本接口重新传输数据包*/
	XDP_REDIRECT,/*数据包重定向到其他网卡*/
};
/*数据包头定义以及顺序*/

| 结构体           | 头文件 |
| ---             | --- |
| struct ethhdr   |  <linux/if_ether.h>|
| struct ipv6hdr  |  <linux/ipv6.h>   |
| struct iphdr    |  <linux/ip.h>   |
| struct icmp6hdr |  <linux/icmpv6.h> |
| struct icmphdr  |  <linux/icmp.h>   |


```
- 函数内联和循环标记
```c
__always_inline

#pragma unroll
```
- 常用代码或函数
```c
/*ebpf 打印日志  cat /sys/kernel/debug/tracing/trace_pipe */
bpf_trace_printk("execve called with %s\n", (char *)ctx->args[0]);

```
#### ICMPV6报文解析
1. icmpv6报文分析以及xdp执行过程分析
我们可以使用官方提供的testenv工具创建一个veth pair对，一端名称叫作env1放在根namespace，另外一端放在命名空间下。本次实验目的就是ping ipv6地址，就会发送ipv6报文，然后丢弃这个数据报文，并记录packets数量。我们xdp程序是在网络协议栈外面的，所以他的影响范围其实就是两个方向：**其他主机或者namespace请求过来的流量，或者本机器出去到其他节点或者namespace流量**，也就是说同主机同namespace流量并不会触发xdp程序。
2. 具体步骤
```c
/*当前环境 fc00:dead:cafe:1::1在根namepace，fc00:dead:cafe:1::2在env1 namesapce
root@lima-ebpfvm:packet01-parsing# t status
Currently selected environment: env1
  Namespace:      env1 (id: 0)
  Prefix:         fc00:dead:cafe:1::/64
  Iface:          env1@if2 UP fc00:dead:cafe:1::1/64 fe80::98ae:42ff:fe98:c746/64 

All existing environments:
  env1
  */
//icmpv6 可以理解组成部分 eth头+ipv6头+icmpv6头+数据
static __always_inline int parse_ethhdr(struct hdr_cursor *nh,
void *data_end,
struct ethhdr **ethhdr)

{
//计算eth头大小
int ethhdrsize =sizeof(**ethhdr);
//查看结束指针是否超过范围，超过则直接交给协议栈处理，说明不是ipv6报文
if(nh->pos + ethhdrsize > data_end){
return -1;
}
//更新ethhdr指针，更新nh指针，nh代表当前报文处理到哪个位置
*ethhdr = nh->pos;
nh->pos += ethhdrsize;
//返回下一层协议，用来处理上层IP层报文
return (*ethhdr)->h_proto; /* network-byte-order */
}
//同理ip报文也是类似处理，解析来进行验证，理论上在根namespace是可以ping通veth1，但是env1
//另外一端ping不通，因为前者不经过xdp程序，后者经过xdp程序
//可以正常ping
root@lima-ebpfvm:packet01-parsing# ping fc00:dead:cafe:1::1
PING fc00:dead:cafe:1::1(fc00:dead:cafe:1::1) 56 data bytes
64 bytes from fc00:dead:cafe:1::1: icmp_seq=1 ttl=64 time=0.248 ms
64 bytes from fc00:dead:cafe:1::1: icmp_seq=2 ttl=64 time=1.47 ms
//无法ping通，同时可以看到xdp——status收集到报文
root@lima-ebpfvm:packet01-parsing# ip netns exec  env1 ping fc00:dead:cafe:1::1
PING fc00:dead:cafe:1::1(fc00:dead:cafe:1::1) 56 data bytes
^C
--- fc00:dead:cafe:1::1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1032ms

root@lima-ebpfvm:packet01-parsing# ./xdp_stats --dev env1

Collecting stats from BPF map
 - BPF map (bpf_map_type:6) id:8 name:xdp_stats_map key_size:4 value_size:16 max_entries:5
XDP-action  
XDP_ABORTED            0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.009177
XDP_DROP               2 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.009130
XDP_PASS               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.009119
XDP_TX                 0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.009112
XDP_REDIRECT           0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.009105
```

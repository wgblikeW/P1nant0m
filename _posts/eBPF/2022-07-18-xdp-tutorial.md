---
layout:     post
title:      "XDP-tutorial 学习如何编写 eBPF XDP 程序"
subtitle:   "跟着 xdp-tutorial repo 以及 Linux repo bpf/samples 从零开始学习编写 eBPF XDP 程序"
date:       2022-07-18 07:00:00
author:     "P1nant0m"
header-img: "img/in-post/bg2.jpg"
tags:
    - eBPF
    - XDP
    - tutorial
---

### XDP-Tutorial

> ⭐ 本文章仍然在不断的更新中，因此文章结构以及内容可能会时常发生变化。同时本领域也是作者刚刚涉及的领域，难免在文本内容中出现错误，还望指正，大家一同学习共勉。

XDP-Tutorial 是一个 Github 上的 repo，皆在指导人们如何遵循最基本的步骤，高效地实现为内核中的 XDP 系统进行编程。我会随着这个课程的步伐与大家一同探寻 Linux 内核世界的奥秘以及如何使用高性能的 XDP 程序对网络数据报进行处理。随着学习的深入，我会列出理解每一节课程的编程实现所需要前置知识，查漏补缺，帮助我们大家更好地理解其中的知识脉络，为以后单独开发相关的系统提供思路支撑。



#### XDP (eXpress Data Path) 简介

XDP 是 Linux 内核上游（Linux 内核原始版本非 Linux 分发版）的一部分，它为用户提供了将用户编写的包处理程序安装进入内核的通道，安装进入内核的包处理程序会在系统接收到包（还未对数据包进行任何处理）的时候触发执行，从而提供了一种高性能的方式允许用户自定义处理内核接收到数据包时的行为。



#### XDP 程序操作模式种类

原生模式  (Native XDP) 卸载模式 (Offloaded XDP) 通用模式 (Generic XDP)

> Native XDP：
>
> ​	该模式是默认模式。在这种模式下，XDP 的 BPF 程序 在网络驱动程序的早期接收路径之外直接运行。使用该模式需要硬件设备的支持。使用 `git grep -l XDP_SETUP_PROG drivers` 命令可以检查驱动程序是否支持此模式。
>
>
> Offloaded XDP：
>
> ​	在卸载模式下，XDP 的 BPF 程序直接卸载到网卡上，而不是在主机 CPU 上执行。因本就仅有相当低开销的 XDP 程序的执行从 CPU 上转移到网卡上，从而这种模式能够比原生 XDP 具有更高的性能。使用这种操作模式的 XDP 程序需要得到网卡的支持。使用 `git grep -l XDP_SETUP_PROG_HW drivers` 可以检查哪些网卡驱动程序支持 Offloaded 模式。
>
> Generic XDP：
>
> ​	对于没有提供 Offloaded 或 Native 模式支持的网卡设备来说，内核提供了一个使用 XDP 通用模式的选项。使用此模式不需要需要任何的驱动支持，因为其运行在一个相对于网络内核栈中相对靠后的位置。其存在主要的目的是为了开发人员在不支持上述两种模式的环境中进行 XDP 程序的开发与调试。在该模式下，XDP 程序的运行性能没有前两者那么高效，因此在生产环境中 



#### XDP 程序使用场景

- 缓解 DDoS 攻击，防火墙

  利用 XDP BPF 的特性我们可以自行定义驱动丢弃恶意网络包的行为逻辑。通过在对数据包进行处理的早期阶段令数据包处理器向网络驱动抛出 `XDP_DROP` 丢弃恶意数据包，我们能够仅付出极小代价的情况下将数据包丢弃，维持系统资源处于一个健康可用的状态。另外，利用 `XDP_TX` `XDP_PASS` `XDP_REDIRECT` 我们还能够自定义逻辑对流量进行清洗、控制流量的走向，实现对主机的保护。

  借助 XDP，我们可以在网卡或其驱动程序中以完全可编程的方式获得相同的功能，而这相比于昂贵的防火墙硬件设备来的非常便宜且快速。**<u>并且我们可以通过远程调用 API 更改规则来控制映射，然后将映射中的规则集动态传递给每台特定计算机中加载的 XDP 程序，这样就能够动态的控制集群中网络拓扑结构</u>**。

- 负载均衡

- 先于内核协议栈的包过滤/处理

- 监控

#### XDP Hook

使用 LLVM 对内核态的 XDP 程序进行编译后，我们可以使用两种方式将 ELF 文件中的 BPF 字节码加载进入内核并挂载至指定的网络设备当中。

- 编写 C 用户态程序，使用 BPF Loader 将编译后的 XDP ELF 文件加载进入内核之中

- 通过 `iproute2 ip` 挂载 （该方式所使用的 BPF Loader 并不是通过 Libbpf 库实现的，这意味着当我们开始使用 BPF Maps 的时候可能会出现不兼容的情况），下面将给出一段示例展示如何使用 `ip` 工具将编译后的 ELF 文件加载进入内核之中

  ```bash
  # 向 lo 网络设备上以 xdpgeneric 方式挂载 xdp_pass_kern.o ELF 文件中 section 为 xdp 的字节码
  ip link set dev lo xdpgeneric obj xdp_pass_kern.o sec xdp
  ```

  有了挂载的方式我们还需要有查看和卸载 XDP 程序的方式，下面将介绍使用该工具进行查看和卸载的方式，

  ```bash
  # 展示 lo 网络设备的相关信息
  ip link show dev lo
  # 方式 2
  bpftool net list dev lo
  
  # 从 lo 网络设备中卸载以 xdpgeneric 方式挂载的 XDP 程序
  ip link set dev lo xdpgeneric off
  ```

- Go 语言使用 libbpfgo 库加载 ELF 文件中的 XDP 程序字节码，这个方式是本人认为最优雅的方式

  ```go
  // 加载 eBPF 程序编译后的 ELF 文件 	
  bpfModule, err := bpf.NewModuleFromFile("main.bpf.o")
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 加载其中的 eBPF 对象
  err = bpfModule.BPFLoadObject()
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 提取其中的内核态程序 "target" 是自编写的 XDP 程序函数名
  xdpProg, err := bpfModule.GetProgram("target")
  	if xdpProg == nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 指定需要挂载的网络设备 lo
  _, err = xdpProg.AttachXDP("lo")
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  ```

在上述 Go 程序代码我们可以发现 `bpfModule.GetProgram("target")` 指定了我们想要从 ELF 文件中加载的 eBPF 程序对象，这意味着我们可以在一个 ELF 文件可以包含多个 XDP 程序，我们可以从中选择我们所需要的进行加载。这种能力是 libbpfgo 封装 libbpf 得来的，而 libbpf 是通过封装系统调用得到的。

在 libbpf 中内部使用 `struct bpf_object` `struct bpf_program` `struct bpf_map` 三类结构体去管理和抽象 eBPF 程序，用户需要使用 libbpf 对外暴露的 API 接口去操作相关的数据结构。



> XDP 程序操作模式种类：原生模式  (Native XDP) 卸载模式 (Offloaded XDP) 通用模式 (Generic XDP)
>
> Native XDP：
>
> ​	该模式是默认模式。在这种模式下，XDP 的 BPF 程序 在网络驱动程序的早期接收路径之外直接运行。使用该模式需要硬件设备的支持。使用 `git grep -l XDP_SETUP_PROG drivers` 命令可以检查驱动程序是否支持此模式。
>
> Offloaded XDP：
>
> ​	在卸载模式下，XDP 的 BPF 程序直接卸载到网卡上，而不是在主机 CPU 上执行。因本就仅有相当低开销的 XDP 程序的执行从 CPU 上转移到网卡上，从而这种模式能够比原生 XDP 具有更高的性能。使用这种操作模式的 XDP 程序需要得到网卡的支持。使用 `git grep -l XDP_SETUP_PROG_HW drivers` 可以检查哪些网卡驱动程序支持 Offloaded 模式。
>
> Generic XDP：
>
> ​	对于没有提供 Offloaded 或 Native 模式支持的网卡设备来说，内核提供了一个使用 XDP 通用模式的选项。使用此模式不需要需要任何的驱动支持，因为其运行在一个相对于网络内核栈中相对靠后的位置。其存在主要的目的是为了开发人员在不支持上述两种模式的环境中进行 XDP 程序的开发与调试。在该模式下，XDP 程序的运行性能没有前两者那么高效，因此在生产环境中推荐使用前两种模式。



#### References

1. [Github repo](https://github.com/xdp-project/xdp-tutorial) for Learning XDP Programming 
2. [Cilium BPF reference guide](https://cilium.readthedocs.io/en/latest/bpf/) for building industrial application
3. General Introduction to XDP in [the academic paper](https://github.com/xdp-project/xdp-paper/blob/master/xdp-the-express-data-path.pdf) or [the presentation](https://github.com/xdp-project/xdp-paper/blob/master/xdp-presentation.pdf).  
4. [Linux 内核观测技术 BPF](https://item.jd.com/12939760.html)

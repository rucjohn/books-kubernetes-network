# 网络虚拟化基石：network namespace

在本书的开篇就介绍 network namespace，是因为它足够重要，毫不夸张地说，它是整个 Linux 网络虚拟化技术的基石。因此，我们将花较多的篇幅介绍。

顾名思义，Linux 的 namespace（命名空间）的作用就是 ”隔离内核资源“。在 Linux 的世界里，文件系统挂载点、主机名、POSIX 进程间通信消息队列、进程 PID 数字空间、IP 地址、user ID 数字空间等全局系统资源被 namespace 分割，送到一个个抽象的独立空间里。而隔离上述系统资源的 namespace 分别是 Mount namespace、UTS namespace、IPC namespace、PID namespace、network namespace 和 user namespace。对进程来说，要想使用 namespace 里面的资源，首选要 ”进入“（具体操作方法，下文会介绍）到这个 namespace，而且还无法跨 namespace 访问资源。Linux 的 namespace 给里面的进程千万了两个错觉：

1. 它是系统里唯一的进程。
2. 它独享系统的所有资源。

> 默认情况下，Linux 进程处在和宿主机相同的 namespace，即初始的根 namespace 里，默认享有全局系统资源。

Linux 内核自 2.4.19 版本接纳第一个 namespace: Mount namespace（用于隔离文件系统挂载点）起，到 3.8 版本的 user namespace（用于隔离用户权限），总共实现了上文提到的 6 种不同类型的 namespace。尽管 Linux 的 namespace 隔离技术很早便存在于内核中，而且它就是为 Linux 的容器技术而设计的，但它一直鲜为人知。直到 Docker 引领的容器技术革命爆发，它才进入普罗大众的视线 --- Docker 容器作为一项轻量级的虚拟化技术，它的隔离能力来自 Linux 内核的 namespace 技术。

说到 network namespace，它在 Linux 内核 2.6 版本引入，作用是隔离 Linux 系统的设备，以及 Ip 地址、端口、路由表、防火墙规则等网络资源。因此，每个网络 namespace 里都有自己的网络设备（如 IP 地址、路由表、端口范围、`/proc/net` 目录等）。从网络的角度看，network namespace 使得容器非常有用，一直直观的例子就是：由于每个容器都有自己的（虚拟）网络设备，并且容器里的进程可以放心地绑定在端口上而不必担心冲突，这就使得在一个主机上同时运行多个监听 80 端口的 WEB 服务变有可能，如图 1-1 所示。



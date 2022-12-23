# 配置 network namespace


当 namespace 里面的进程涉及网络通信时，namespace 里面的（虚拟）网络设备就必不可少了。

通过上文的阅读，我们已经知道，一个全新的 network namespace 会附带创建一个本地回环地址。除此之外，没有任何其他的网络设备。而且细心的读者应该已经发现，network namespace 自带的 lo 设备状态为 DOWN，因此，当尝试访问本地回环地址时，网络是不通的。下面的小测试就说明了这一点。
```bash
# 进入 netns1 network namespace，ping 127.0.0.1
ip netns exec netns1 ping 127.0.0.1
```
输出
```
connect: Network is unreachable
```

在我们的例子中，如果想要访问本地回环地址，首先需要把设备状态改为 UP。
```bash
ip netns exec netns1 ip link set dev lo up
```

然后，再次尝试 ping 127.0.0.1，发现能够 ping 通。
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.012ms
...
```

但是，仅有一个本地回环设备是没法与外界通信的。如果我们想与外界（比如主机上的网卡）进行通信，就需要在 namespace 里再创建一对虚拟的以太网卡，即所谓的 veth pair。

顾名思义，veth pair 总是成对出现且相互连接，它就像 Linux 的双向管道（pipe），报文从 veth pair 一端进去就会由另一端收到。关于 veth pair 更详细的介绍，参见 1.2 节。

下面的命令将创建一对虚拟以太网卡，然后把 veth pair 的一端放到 netns1 network namespace 中。

```bash
ip link add veth0 type veth peer name veth1
ip link set veth1 netns netns1
```

如上所示，我们创建了 veth0 和 veth1 这么一对虚拟的以太网卡。在默认情况，它们都在主机的根 network namespace 中，将其中一块虚拟网卡 veth1 通过 `ip link set` 命令移动到 netns1 network namespace。

那么 veth0 和 veth1 之间能直接通信吗？还不能，因为这两块网卡刚创建出来还都是 DOWN 状态，需要手动把状态设置成 UP。这个步骤的操作和上文对 lo 网卡的操作类似，只是多了一步绑定 IP 地址，如下所示：
```bash
ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
ifconfig veth0 10.1.1.2/24 up
```

上面两条命令首选进入 netns1 这个 network namespace，为 veth1 绑定 IP 地址 10.1.1.1/24，并把网卡状态设置成 UP，而仍在主机根 network namespace 中的网卡 veth0 被我们绑定了 IP 地址 10.1.1.2/24。这样一来，我们就可以 ping 通 veth pair 的任意一点了。

例如，在主机上 ping 10.1.1.1 
```bash 
ping 10.1.1.1
```
输出
```
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.022ms
...
```

同理，我们进入 netns1 network namespace 去 ping 主机上的虚拟网卡，如下所示
```bash
ip netns exec netns1 ping 10.1.1.2
```
输出
```
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.014ms
...
```

另外，不同 network namespace 之间的路由表和防火墙规则等也是隔离的，因此我们刚刚创建的 netns1 network namespace 没法和主机共享路由和防火墙，这一点通过下面的测试就能说明。
```bash
ip netns exec netns1 route
ip netns exec netns1 iptables -L
```

如上所示，我们进入 netns1 network namespace，分别输入 `route` 和 `iptables -L` 命令，期望查询路由表和 iptables 规则，却发现空空如也。这意味着从 netns1 network namespace 发包到因特网也是徒劳的，因为网络不通！

想连接因特网，有若干解决方法。例如：
- 可以在主机的根 network namespace 创建一个 Linux 网桥并绑定 veth pair 的一端到网桥上；
- 可以通过适当的 NAT（网络地址转换）规则并辅以 Linux 的 IP 转发功能（配置`net.ipv4.ip_forward=1`）；

{% hint style="info" %}
<mark style="color:blue;">**说明：**</mark>

用户可以随意将虚拟网络设备分配到自定义的 network namespace 里，而连接真实硬件的物理设备则只能放在系统的根 network namespace 中。并且任何一个网络设备最多只能存在于一个 network namespace 中。
{% endhint %}

进程可以通过 Linux 系统调用 `clone()`、`unshare()` 和 `setns` 进入 network namespace，下面会有代码示例。非 root 进程被分配到 network namespace 后只能访问和配置已经存在于该 network namespace 的设备。当然，root 进程可以在 network namespace 里创建新的网络设备。除此之外，network namespace 里的 root 进程还能把本 network namespace 的虚拟网络设备分配到其他 network namespace —— 这个操作路径可以从主机的根 network namespace 到用户自定义 network namespace，反之亦可。请查看下面这条命令：

```bash
ip netns exec netns1 ip link set veth1 netns 1
```

该怎么理解上面这条看似有点复杂的命令呢？分解成两个部分：
（1）`ip netns exec netns1`，进入 netns1 network namespace。
（2）`ip link set veth1 netns 1`，把 netns1 network namespace 下的 veth1 网卡挪到 PID 为 1 的进程（即 init 进程）所在的 network namespace。

通常 init 进程都在主机的根 network namespace 下运行，因此上面这条命令其实就是把 veth1 从 netns1 network namespace 移动到系统根 network namespace。有两种途径索引 network namespace：
- 名字（例如 netns1）
- 属于该 namespace 的进程 PID

对 namespace 的 root 用户而言，他们都可以把其 namespace 里的虚拟网络设备移动到其他 network namespace，甚至包括主机根 network namespace！这就带来了潜在的安全风险。如果用户希望屏蔽这一行为，则需要结合 PID namespace 和 Mount namespace 的隔离特性做到 network namespace 之间的完全不可达。


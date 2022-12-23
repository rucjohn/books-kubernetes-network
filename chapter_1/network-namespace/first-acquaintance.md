# 初识 network namespace

和其他 namespace 一样，network namespace 可以通过系统调用来创建，我们可以调用 Linux 的 `clone()`（其实是 UNIX 系统调用 `fork()` 的延伸）API 创建一个通用的 namespace，然后传入 `CLONE_NETWORK` 参数表面创建一个 network namespace.

高阶读者可以参考下文的 C 代码创建一个 network namespace。与其他 namespace 需要读者自己写 C 语言代码调用系统 API 才能创建不同，network namespace 的增删改查功能已经集成到 Linux 的 `ip` 工具的 `netns` 子命令中，因此大大降低了初学者的体验门槛。下面先介绍几条简单的网络 namespace 管理的命令。

## 创建

创建一个名为 netns1 的 network namespace，可以使用以下命令：
```bash
ip netns add netns1
```

当 `ip` 命令创建了一个 network namespace 时，系统会在 `/var/run/netns` 路径下面生成一个挂载点。挂载点的作用一方面是方便对 namespace 的管理，另一方面是使 namespace 即使没有进程运行也能继续存在。

## 进入

一个 network namespace 被创建出来后，可以使用 `ip netns exec` 命令进入，做一些网络查询配置的工作。
```bash
ip netns exec netns1 ip link list
```
输出
```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

如上所示，就是进入 netns1 这个 network namespace 查询网卡信息的命令。目前，我们没有任何配置，因此只有一块系统默认的本地回环设备 lo。

## 查看

想查看系统中有哪些 network namespace，可以使用以下命令：
```bash
ip netns list
```

## 删除
想删除 network namespace，可以通过以下命令实现：
```bash
ip netns delete netns1
```
{% hint style="info" %}
<mark style="color:blue;">**说明：**</mark>

注意，上面这条命令实际上并没有删除 netns1 这个 network namespace，它只是移除了挂载点（下文会解释）。只要里面还有进行运行着，network namespace 便会一直存在。
{% endhint %}




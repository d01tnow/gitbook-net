# SO_REUSEADDR-SO_REUSEPORT 的作用

套接字 5 元组: 协议, 本地地址, 本地端口, 远程h地址, 远程端口. 两个套接字的 5 元组中任一个不同, 就认为是不同的套接字. 默认情况下，任意两个socket都无法绑定到相同的(协议,本地IP地址和本地端口)。

## SO_REUSEADDR

SO_REUSEADDR 可以使 TCP套接字处于 TIME_WAIT 状态下的socket，不用等待 2 * MSL , 立即重复绑定使用。server程序总是应该在调用bind()之前设置SO_REUSEADDR套接字选项。TCP先调用close()的一方会进入TIME_WAIT状态. 但是, 由于没有在 TIME_WAIT 状态下等待 2 * MSL 时间, 有可能会收到前次该端口的发出数据的响应.

## SO_REUSEPORT

Linux 内核3.9加入了SO_REUSEPORT.

允许将多个 socket 绑定到相同的地址和端口, 前提是每个 socket 绑定前都设置了 SO_REUSEPORT 选项.
通过这个选项可以i实现如下功能:

* 阻止 port 劫持, 限制所有使用相同 ip 和 port 的 socket 都必须拥有相同的有效用户 id.
* linux内核在处理SO_REUSEPORT socket的集合时，进行了简单的负载均衡操作，即对于UDP socket，内核尝试平均的转发数据报. 对于TCP监听socket，内核尝试将新的客户连接请求(由accept返回)平均的交给共享同一地址和端口的socket(监听socket)。

但是, 不具备 SO_REUSEADDR 的快速重用 TIME_WAIT 状态的 socket 功能.

## 参考

1. [SO_REUSEADDR和SO_REUSEPORT作用](https://www.jianshu.com/p/141aa1c41f15)

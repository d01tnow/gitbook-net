# TCP 套接字选项 SO_LINGER 和 SO_LINGER2

本文描述 tcp 套接字选项 SO_LINGER 和 SO_LINGER2 各自的作用和区别.

## SO_LINGER

用于控制用户调用 close() 或 shutdown() (注意：虽然man page这么写，但是 shutdown 内核代码流程中并未用到该选项)关闭 socket 后内核的行为.
**注意：如果是调用exit()函数关闭socket，那么无论是否启用SO_LINGER选项，socket总会在后台执行linger等待；**
默认不启用, 即 l_onoff == 0
注释：l_linger的单位依赖于实现: 4.4BSD假设其单位是时钟滴答（百分之一秒），但Posix.1g规定单位为秒。

``` txt
       SO_LINGER
              Sets or gets the SO_LINGER option.  The argument is a linger
              structure.

                  struct linger {
                      int l_onoff;    /* linger active */
                      int l_linger;   /* how many seconds to linger for */
                  };

              When enabled, a close(2) or shutdown(2) will not return until
              all queued messages for the socket have been successfully sent
              or the linger timeout has been reached.  Otherwise, the call
              returns immediately and the closing is done in the background.
              When the socket is closed as part of exit(2), it always
              lingers in the background.
```

l_onoff | l_linger | close() 行为 | 发送队列 | 底层行为
--- | --- | --- | --- | ---
0 | 忽略 | 立即返回. 忽略返回值 | 保持,直至发送完成 | 内核接管套接字, 将数据发送至对端
!= 0 | 0 | 立即返回. | 放弃发送缓存内数据 | 如果有未读数据, 直接发送 RST 包, 自身立即复位, 不会进入 FIN_WAIT1, TIME_WAIT 状态
!= 0 | > 0 | 阻塞模式下等数据发送完成 or l_linger 超时 . 非阻塞模式, 立即返回, 如果数据没有发送完成并被e确认, 返回 EAGAIN(errno==11) | 在 l_linger 时间内保持, 尝试发送, 若超时则放弃 | 超时则发送 RST

### 图示

* Close 默认操作, 立即返回. 此种情况，close立即返回，如果send buffer中还有数据，close将会等到所有数据被发送完之后之后返回。由于我们并没有等待对方TCP发送的ACK信息，所以我们只能保证数据已经发送到对方，我们并不知道对方是否已经接受了数据。由于此种情况，TCP连接终止是按照正常的4次握手方式，需要经过TIME_WAIT。
![close默认操作](tcp-close-default.jpg)

* l_onoff != 0 && l_linger > 0 且在 l_linger 超时前收到了ACK. 在这种情况下，close会在接收到对方TCP的ACK信息之后才返回(l_linger消耗完之前)。但是这种ACK信息只能保证对方已经接收到数据，并不保证对方应用程序已经读取数据。
![linger启用且l_linger>0](tcp-close-linger-on-timeout-enough.jpg)

* l_onoff != 0 && l_linger > 0 但 l_linger 超时到的时候还没有收到 ACK. 这种情况，由于l_linger值太小，在send buffer中的数据都发送完之前，close就返回，此种情况终止TCP连接，更l_linger = 0类似，TCP连接终止不是按照正常的4步握手，所以，TCP连接不会进入TIME_WAIT状态，那么，client会向server发送一个RST信息.
![lingeri启用且l_linger太短](tcp-close-linger-on-timeout-too-short.jpg)

## SO_LINGER2

该选项是TCP层面的，用于设定孤儿套接字在FIN_WAIT2状态的生存时间，该选项可以用来替代系统级别的tcp_fin_timeout配置；在用于移植的代码中不应该使用该选项；另外，需要注意，不要混淆该选项与socket的SO_LINGER选项；

``` txt
       TCP_LINGER2 (since Linux 2.4)
              The lifetime of orphaned FIN_WAIT2 state sockets.  This option
              can be used to override the system-wide setting in the file
              /proc/sys/net/ipv4/tcp_fin_timeout for this socket.  This is
              not to be confused with the socket(7) level option SO_LINGER.
              This option should not be used in code intended to be
              portable.
```

## 源码分析

### SO_LINGER 源码

在调用close()系统调用时，如果引用计数已经为0，则会进行套接字关闭操作，我们从inet_release开始分析；前置步骤请移步[套接字之close系统调用](http://www.linuxtcpipstack.com/521.html)；如果启用了SO_LINGER选项，那么会将lingertime传入到传输层的关闭函数中，tcp为tcp_close；

``` c
/*
 *	The peer socket should always be NULL (or else). When we call this
 *	function we are destroying the object and from then on nobody
 *	should refer to it.
 */
int inet_release(struct socket *sock)
{
	struct sock *sk = sock->sk;

	if (sk) {
		long timeout;

		/* Applications forget to leave groups before exiting */
		/* 退出组播组 */
		ip_mc_drop_socket(sk);

		/* If linger is set, we don't return until the close
		 * is complete.  Otherwise we return immediately. The
		 * actually closing is done the same either way.
		 *
		 * If the close is due to the process exiting, we never
		 * linger..
		 */
		timeout = 0;

		/*
			设置了linger标记，进程未在退出，
			则设置lingertime延迟关闭时间
		*/
		if (sock_flag(sk, SOCK_LINGER) &&
		    !(current->flags & PF_EXITING))
			timeout = sk->sk_lingertime;
		sock->sk = NULL;

		/* 调用传输层的close函数 */
		sk->sk_prot->close(sk, timeout);
	}
	return 0;
}
```

tcp_close函数，在关闭socket销毁资源之前，调用sk_stream_wait_close函数等待数据发送完毕或者达到lingertime超时时间，然后才继续进入关闭socket销毁资源的流程；

```c
void tcp_close(struct sock *sk, long timeout)
{
       /* ... */

	/* If socket has been already reset (e.g. in tcp_reset()) - kill it. */
	/* CLOSE状态 */
	if (sk->sk_state == TCP_CLOSE)
		goto adjudge_to_death;

	/* As outlined in RFC 2525, section 2.17, we send a RST here because
	 * data was lost. To witness the awful effects of the old behavior of
	 * always doing a FIN, run an older 2.1.x kernel or 2.0.x, start a bulk
	 * GET in an FTP client, suspend the process, wait for the client to
	 * advertise a zero window, then kill -9 the FTP client, wheee...
	 * Note: timeout is always zero in such a case.
	 */
	/* 修复状态，断开连接 */
	if (unlikely(tcp_sk(sk)->repair)) {
		sk->sk_prot->disconnect(sk, 0);
	}
	/* 用户进程有数据未读 */
	else if (data_was_unread) {
		/* Unread data was tossed, zap the connection. */
		NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONCLOSE);

		/* 设置为close */
		tcp_set_state(sk, TCP_CLOSE);

		/* 发送rst */
		tcp_send_active_reset(sk, sk->sk_allocation);
	}
	/* lingertime==0，断开连接 */
	else if (sock_flag(sk, SOCK_LINGER) && !sk->sk_lingertime) {
		/* Check zero linger _after_ checking for unread data. */
		sk->sk_prot->disconnect(sk, 0);
		NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPABORTONDATA);
	}
	/* 关闭状态转移 */
	else if (tcp_close_state(sk)) {
		/* We FIN if the application ate all the data before
		 * zapping the connection.
		 */

		/* RED-PEN. Formally speaking, we have broken TCP state
		 * machine. State transitions:
		 *
		 * TCP_ESTABLISHED -> TCP_FIN_WAIT1
		 * TCP_SYN_RECV	-> TCP_FIN_WAIT1 (forget it, it's impossible)
		 * TCP_CLOSE_WAIT -> TCP_LAST_ACK
		 *
		 * are legal only when FIN has been sent (i.e. in window),
		 * rather than queued out of window. Purists blame.
		 *
		 * F.e. "RFC state" is ESTABLISHED,
		 * if Linux state is FIN-WAIT-1, but FIN is still not sent.
		 *
		 * The visible declinations are that sometimes
		 * we enter time-wait state, when it is not required really
		 * (harmless), do not send active resets, when they are
		 * required by specs (TCP_ESTABLISHED, TCP_CLOSE_WAIT, when
		 * they look as CLOSING or LAST_ACK for Linux)
		 * Probably, I missed some more holelets.
		 * 						--ANK
		 * XXX (TFO) - To start off we don't support SYN+ACK+FIN
		 * in a single packet! (May consider it later but will
		 * probably need API support or TCP_CORK SYN-ACK until
		 * data is written and socket is closed.)
		 */
		/* 发送fin */
		tcp_send_fin(sk);
	}

	/*  等待关闭，无数据发送或sk_lingertime超时 */
	sk_stream_wait_close(sk, timeout);

adjudge_to_death:
	/* socket关闭，释放资源 */
        /* ... */
}
```

下面的sk_stream_closing函数检查连接状态，当为TCPF_FIN_WAIT1 | TCPF_CLOSING | TCPF_LAST_ACK时，说明还有数据要发送，这时返回1，等待继续执行；sk_stream_wait_close在等待连接状态不为上述状态时，或者有信号要处理，或者超过lingertime，则返回；

``` c
/**
 * sk_stream_closing - Return 1 if we still have things to send in our buffers.
 * @sk: socket to verify
 */
static inline int sk_stream_closing(struct sock *sk)
{
	return (1 << sk->sk_state) &
	       (TCPF_FIN_WAIT1 | TCPF_CLOSING | TCPF_LAST_ACK);
}

void sk_stream_wait_close(struct sock *sk, long timeout)
{
	if (timeout) {
		DEFINE_WAIT_FUNC(wait, woken_wake_function);

		add_wait_queue(sk_sleep(sk), &wait);

		do {
			if (sk_wait_event(sk, &timeout, !sk_stream_closing(sk), &wait))
				break;
		} while (!signal_pending(current) && timeout);

		remove_wait_queue(sk_sleep(sk), &wait);
	}
}
```

### SO_LINGER2 源码分析

启动FIN_WAIT_2定时器两个相关逻辑差不多，所以只拿一个位置来说明；在tcp_close函数中，如果判断状态为FIN_WAIT2，则需要进一步判断linger2配置；如下所示，在linger2<0的情况下，关闭连接到CLOSE状态，并且发送rst；在linger2 >= 0的情况下，需判断该值与TIME_WAIT等待时间TCP_TIMEWAIT_LEN值的关系，如果linger2 > TCP_TIMEWAIT_LEN，则启动FIN_WAIT_2定时器，其超时时间为二者的差值；如果linger2<TCP_TIMEWAIT_LEN，则直接进入到TIME_WAIT状态，该TIME_WAIT的子状态是FIN_WAIT2，实际上就是由TIME_WAIT控制块进行了接管，统一交给TIME_WAIT控制块来处理；详细处理过程，后续补充；

``` c
void tcp_close(struct sock *sk, long timeout)
{
        /* ... */
	if (sk->sk_state == TCP_FIN_WAIT2) {
		struct tcp_sock *tp = tcp_sk(sk);
		/* linger2小于0，无需等待 */
		if (tp->linger2 < 0) {

			/* 转到CLOSE */
			tcp_set_state(sk, TCP_CLOSE);
			/* 发送rst */
			tcp_send_active_reset(sk, GFP_ATOMIC);
			__NET_INC_STATS(sock_net(sk),
					LINUX_MIB_TCPABORTONLINGER);
		} else {

			/* 获取FIN_WAIT_2超时时间 */
			const int tmo = tcp_fin_time(sk);

			/* FIN_WAIT_2超时时间> TIME_WAIT时间，加FIN_WAIT_2定时器 */
			if (tmo > TCP_TIMEWAIT_LEN) {
				inet_csk_reset_keepalive_timer(sk,
						tmo - TCP_TIMEWAIT_LEN);
			}
			/* 小于TIME_WAIT时间，则进入TIME_WAIT */
			else {
				tcp_time_wait(sk, TCP_FIN_WAIT2, tmo);
				goto out;
			}
		}
	}

        /* ... */
}
```

## 参考

1. [TCP套接字选项SO_LINGER与TCP_LINGER2](http://www.linuxtcpipstack.com/531.html)
2. [SO_LINGER 延时关闭 优雅关闭](https://www.cnblogs.com/my_life/articles/5174585.html)

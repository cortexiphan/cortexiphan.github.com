---

layout: post
categories: [Programming]
tags: [Programming]

---

# TIME_WAIT状态小探 #

这篇文章是对网上一些文章整理之后的总结，主要讨论TIME_WAIT状态的问题以及解决方法。

## TIME_WAIT状态以及它的作用 ##

![TCPstate](http://d1g3mdmxf8zbo9.cloudfront.net/images/tcp/tcp-state-diagram.png)
（引自vincent.bernat.im）

从TCP状态转移图中可以看到，一个TCP连接中，只有主动关闭，即先发送FIN包的那一端会最终进入TIME_WAIT状态。当然，如果两端同时关闭连接，即在收到对端FIN之前就发出FIN包，那么两端最终都会进入TIME_WAIT状态。

TIME_WAIT状态有两个作用：

### 防止上一个连接的延时包被新连接当做正常包接收 ###

一个TCP连接由一个4元组（source_ip、source_port、destination_ip、destination_port）唯一标识。一个旧连接关闭后，如果一个具有相同4元组的新连接马上建立起来，这时旧连接的一个包因为延时刚刚到达，并且它的序列号刚好在新连接的接收窗口中，那么新连接就会把这个包当做正常包接收，进而使数据出现混乱。虽然每次建立连接时使用的序列号都是随机产生的，但是序列号的长度只有32位，在高速网络中可能会很快出现序列号循环，这种情况下序列号刚好在接收窗口中的可能性就高很多。TIME_WAIT的2MSL过去之后，我们可以确信旧连接的延时包都已经在网络上消失，不可能再干扰到新连接了。
![delayed_segment](http://www.serverframework.com/asynchronousevents/assets_c/2011/01/TIME_WAIT-why-thumb-500x711-277.png)
（引自www.serverframework.com）

### 确保对端可靠地关闭当前连接 ###

这里又可以从两个方面考虑

- 为了可以重传最后的ACK。在对端发出的FIN包丢失或者本端发出的最后ACK丢失时，本端需要重传最后的ACK。对端发出FIN包之后一旦超时没有收到ACK，就会重传FIN包，本端需要能够响应重传的FIN包，所以需要留在TIME_WAIT状态一段时间。至于TIME_WAIT的时间为什么是2MSL，而不是MSL，甚至RTO的某个函数，我想原因应该是这样的：
- 确保在本端试图建立相同4元组的新连接时，对端的旧连接已经关闭。TIME_WAIT状态结束之后，本地就可以建立一个新的、具有相同4元组的连接，如果这时对端仍处在LAST_ACK状态，那么本地发过去的SYN包就会被对端抛弃，并且用RST响应，导致建立连接失败。
  ![last_ack_long](http://d1g3mdmxf8zbo9.cloudfront.net/images/tcp/last-ack.png)
  （引自vincent.bernat.im）

## TIME_WAIT对系统性能的影响 ##

TIME_WAIT对系统的影响主要有以下3个方面

### 导致系统无法新建一个与TIME_WAIT套接字相同4元组的连接 ###

当然，这也是设置TIME_WAIT状态的目的。对于发起主动连接的客户端来说，不能建立这样的连接也不是什么大问题，因为默认情况下系统会为套接字选择一个可用端口，除非我们再connect之前执行了bind。需要注意的是，如果客户端需要频繁发起并主动关闭连接，这可能就会出现问题，太多的端口处于TIME_WAIT状态，导致系统可选择的端口耗尽，那也会无法建立新连接。

### 导致程序无法立即重启 ###

很多系统的TCP实现了比RFC要求的更严格的标准，RFC只要求不能存在两个具有相同4元组的连接，而很多系统则要求本地ip和端口被某个socket占用之后，不能够再执行bind。所以，如果有TIME_WAIT的套接字占用了本地程序监听的ip和端口，那本地程序重启是将会bind失败。

### CPU和内存消耗 ###

保存TCP连接结构需要消耗内存消耗，查询可用端口时也有CPU消耗。但这些消耗都很小，与数据处理的消耗相比，可以忽略。

## 解决TIME_WAIT问题的方法 ##



### 什么时候才需要TIME_WAIT的问题？ ###

- 主动发起连接的一方容易受到TIME_WAIT的影响。如果TIME_WAIT状态的socket大量积累，那么系统能够选择的可用端口就会减少甚至消耗殆尽，这是本地就不能发起连接了。
- 被动接受连接的一方较少受到TIME_WAIT的影响。被动接受连接的服务器一般使用众所周知的地址和端口，即使有很多socket处于TIME_WAIT状态，这个地址和端口也只构成了TCP连接4元祖的一部分，只要客户端的的ip不同，或者使用的端口不同，新建连接就不会有问题。除非某个客户端的本地端口耗尽才会造成无法建立连接，但这也只是个别客户的问题，对其他客户没有影响。

### 通过添加资源来减轻TIME_WAIT的影响 ###

- 前面提到客户端可能存在本地端口耗尽的情况，我们可以扩大系统可以选择的本地端口范围来减轻这个问题，但是能够增加的两不多，只是聊胜于无。
- TIME_WAIT只是限制具有相同4元祖的的连接，我们只要增加4元祖中的任何一个，就可以减轻TIME_WAIT的问题，比如增加服务器的ip（让同一个域名对应多个ip），增加监听的端口（http建通80、81、82...），客户端使用多个ip，增加系统可选择的端口范围等。

### 缩短TIME_WAIT的时间 ###

按照一般的想法，既然TIME_WAIT的让我们长时间不能新建连接，那么可以考虑把TIME_WAIT的时间缩短，即修改MSL的时间。不过这很可能解决不了问题，当问题已经严重到port range里面已经没有可用端口时，TIME_WAIT的时间变短，很可能只会适得更多的短连接建立并且关闭，端口仍然会被TIME_WAIT沾满。

### 避免TIME_WAIT的出现 ###

- 按照陈浩的说法，既然TIME_WAIT是主动关闭连接那一边才出现的状态，那就让对端先关闭连接得了，这样TIME_WAIT的破问题就是对方的了。需要关闭连接时，服务器可以在应用层通知客户端主动关闭连接，这样TIME_WAIT状态就留给了客户端，而客户端一般不会受到TIME_WAIT的困扰，只要它不需要主动建立大量的短连接。
- 更激进一点的，服务器可以不进行优雅的四次挥手断开连接，而是直接发送RST包，这样当然就不会有TIME_WAIT状态了。但是，这种方法会导致缓冲区里的数据被丢弃，所以程序需要在应用层保证这种结果是可接受的，比如上面服务器通知客户端主动断开连接，但客户端超时不发送FIN包，那服务器就可以放松RST包以放弃连接。
- 设置socket的option：SO_LINGER。设置socket为不等待之后，socket在断开连接时直接跳过TIME_WAIT阶段，直接发送RST包。

### 设置SO_REUSEADDR ###

前面提到过，某些系统上一个ip和端口已经被一个socket占用时（比如TIME_WAIT或者FIN_WAIT_2状态），bind会失败，导致程序退出后马上重启的失败。设置socket的option为SO_REUSEADDR后，这样的bind就可以成功。
一些文章提到SO_REUSEADDR只是让程序可以在一个已经被占用的端口上bind的成功，而具有相同4元组的连接仍然不允许建立。但笔者在Debian上实验后发现，服务端设置了SO_REUSEADDR之后，并且正在listen的端口有socket处于TIME_WAIT状态，客户端用TIME_WAIT对应的端口连接服务器，服务器会直接重用TIME_WAIT的socket建立连接。

### 设置tw_reuse ###

tw_reuse默认是关闭的，打开之后，可以使新连接直接重用旧的TIME_WAIT状态的连接来建立（新旧连接具有相同的4元组），只要新连接的时间戳严格大于旧连接。这就要求两边的TCP都支持timestamp的选项。当然，TCP协议还是要保证TIME_WAIT状态所提供的两个功能得以实现：1）旧连接的延时包不能干扰新连接，这个通过时间戳来保证。2）本地就连接结束，新连接建立时，对方的旧连接已经关闭，这里假设本地建立新连接时，对方仍处于LAST_ACK状态，对方收到SYN包后，仍然返回已经发送过的FIN和ACK，而本地不认识这些包，直接返回RST，导致对方LAST_ACK的结束，这之后本地再发送SYN就可以成功建立连接了。

### 设置tw_recycle ###

tw_recycle比tw_reuse还要激进，tw_reuse只是在必要时（新旧连接具有相同的4元组）才重用TIME_WAIT状态的连接，而tw_recycle直接提前结束TIME_WAIT状态的连接。这是TCP协议会仍旧记录所有TIME_WAIT状态连接的时间戳，对于该4元组上的连接的数据包，如果其时间戳不能严格大于该记录的时间，则直接被丢弃。NAT背后的机器很可能时间不能同步，这时下会出现连接不上的情况。

## 总结 ##

虽然有时候TIME_WAIT会给我们带来一些麻烦，但这仍然是TCP协议中的一个非常必要的好东西。如果想享受TCP带来的可靠保证，又不想受到TIME_WAIT的影响，那最好的办法是让对方关闭连接；如果应用层可以确保不经过四次握手关闭也能保证程序逻辑正确，那直接通过RST包中断连接也是一个方法；实在不行，再考虑设置tw_reuse或者tw_recycle，按照RFC的建议，这两个参数只有在经过非常专业的分析之后才应该设置。

## 引用 ##

[1] [TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)

[2] [Coping with the TCP TIME-WAIT state on busy Linux servers](http://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux.html)

[3] [Bind before connect](https://idea.popcount.org/2014-04-03-bind-before-connect/)

# 几行go代码实现NAT内网穿透技术
### 一、我们为什么需要内网穿透？
比如有2个不同地域用户，他们都在公司或小区内网中，如果需要双方通信就必须借助第三方服务器进行中转。如语音通话、视频通话、文件传输等，他们会占用大量的服务器带宽！如果可以让在网络之间打个洞让他们之间互相通信，不再从服务器中转，他们的通信质量取决于他们自己的网络是不是很爽呀~尤其是用了5G网络以后！

NAT内网穿透，就是想让2个不同地域的内网用户穿越过对方网关，可以直接点对点通信。这个技术在很久很久以前就有了，NAPT端口的映射方式有四种（完全圆锥型NAT，地址限制圆锥型NAT，端口限制圆锥型NAT，对称型NAT）关于NAT穿透原理可以看一下这篇文章介绍https://cloud.tencent.com/developer/article/1005974 


### 二、P2P UDP打洞功能演示
费话不多讲，我们直接动手操作。我这里在本地模拟一下打洞过程。我们看到客户端第一次与服务端连接后，进行了关闭。之后没服务端啥事，都是客户端之间的通信~

![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579157797191-1579157797198.png)
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579157815898-1579157815909.png)
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579157831335-1579157831337.png)


### 三、技术思路：
1. 服务端开始监听的UDP端口
2. 客户端连接服务端UDP端口后发送消息
3. 服务端收到的客户端数量为2时，分别告诉对方另一方的IP地址与端口号。服务端任务完成。
4. 客户端读取来自服务器的消息，得到另一方客户端的IP地址与端口号，断开与服务器连接。
5. 2个客户端都尝试向对方发送数据


### 四、代码：
详细代码已上传至 https://github.com/xs25cn/p2p-demo

#### golang中主要用到的几个方法：
```golang
//服务端最核心几个方法
listen, err :=net.ListenUDP(network string, laddr *UDPAddr)
listen.LocalAddr()
listen.ReadFromUDP(b []byte)
listen.WriteToUDP(b []byte, addr *UDPAddr)

//客户端最核心几个方法
conn, err := net.DialUDP(network string, laddr, raddr *UDPAddr)
conn.Write([]byte(s))
conn.ReadFromUDP(data)
conn.Close()

```

#### 1，服务端核心代码 

```golang

	listen, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.IPv4zero, Port: *port})
	for {
		n, remoteAddr, err := listen.ReadFromUDP(data)
		clients = append(clients, *remoteAddr)
		if len(clients) == 2 {
			c1, err := listen.WriteToUDP([]byte(clients[0].String()), &clients[1])
			c2, err := listen.WriteToUDP([]byte(clients[1].String()), &clients[0])
		}
	}

```

#### 2，客户端核心代码
```golang
	//连接服务端发送消息，接收消息
	conn, err := net.DialUDP("udp", cAddr, &net.UDPAddr{IP: net.ParseIP(sIP), Port: sPort})
	conn.Write([]byte(s))
	conn.ReadFromUDP(data)
	conn.Close()
	
	//和另一客户端建立连接，收发消息
	conn2, err := net.DialUDP("udp", srcAddr, dstAddr)
	conn2.Write([]byte("......"))
	//给对方发数据
	go func() {
		for {
			time.Sleep(time.Second * 5)
			conn2.Write([]byte("xxx"))
		}
	}()
	//输出对方发来的数据
	for {
		data := make([]byte, 1024)
		conn2.ReadFromUDP(data)
	}
}

```

以上是一个简单的代码逻辑,这也就是P2P UDP打洞的最基本流程吧。是不是挺简单的。但真想将它做好不是那么简单。多数NAT设备内 部都有一个UDP转换的空闲状态计时器，如果在一段时间内没有 UDP数据通信，NAT设备会关掉由“打洞”操作打出来的“洞”。

#### 本DEMO详细代码已上传至 https://github.com/xs25cn/p2p-demo

### 五、其实要考虑的问题很多......
- [x] 如果想要真正的应用还有很多问题没有解决~
	- [ ] 这里的例子用的是 对称型NAT，不是所有网络都可能支持每种的！
	- [ ] UDP数据传输包的顺序问题，数据丢包问题
	- [ ] 空闲状态下的超时问题
	- [ ] TCP打洞
	- [ ] 如何实现web的P2P打洞技术
	- [ ] webRTC的封装使用
	
使用P2P技术做数据传输，写出一个高质量的系统还需要下一番苦功夫。无论是文件传输出，视频聊天，语音通话......5G时代的到来，做的好可以养活一家公司，做的不好就是一个DEOM而已~ 希望我写的简单demo能带你入门。
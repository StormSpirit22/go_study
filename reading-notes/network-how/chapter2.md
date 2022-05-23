## 第二章：用电信号传输 TCP/IP 数据

## 2.1 创建套接字

### 协议栈的内部结构

![协议栈内部结构](/Users/tianyou/Documents/Github/go_study/.go_study/assets/network-how/2.1-1.png)



Socket 库，其中包括解析器，解析器用来向 DNS 服务器发出查询。

浏览器、邮件等一般应用程序收发数据时用 TCP ; DNS 查询等收发较短的控制数据时用 UDP 。

在互联网上传送数据时，数据会被切分成一个一个的网络包，而将网络包发送给通信对象的操作就是由 IP 来负责的。此外，IP 中还包括ICMP 协议和 ARP 协议。ICMP 用于告知网络包传送过程中产生的错误以及各种控制消息，ARP 用于根据 IP 地址查询相应的以太网 MAC 地址。

网卡驱动程序负责控制网卡硬件，而最下面的网卡则负责完成实际的收发操作，也就是对网线中的信号执行发送和接收的操作。

### 套接字的实体就是通信控制信息

套接字中记录了用于控制通信操作的各种控制信息，比如通信对象的IP地址、端口号、通信操作的进行状态等。协议栈则需要根据这些信息判断下一步的行动，这就是套接字的作用。

Windows 中可以用 netstat 命令显示套接字内容。

![netstat](/Users/tianyou/Documents/Github/go_study/.go_study/assets/network-how/2.1-2.png)

看上图，比如第 8 行，它表示 PID 为 4 的程序正在使用 IP 地址为 10.10.1.16 的网卡与 IP 地址为 10.10.1.18 的对象进行通信。此外我们还可以看出，本机使用 1031 端口，对方使用 139 端口，而 139 端口是 Windows 文件服务器使用的端口，因此我们就能够看出这个套接字是连接到一台文件服务器的。

在 mac 上我也试了一下 netstat 命令：

![image-20220523154907140](/Users/tianyou/Documents/Github/go_study/.go_study/assets/network-how/2.1-3.png)

mac 下的 netstat 命令结果稍微有点不同，Recv-Q 和 Send-Q 是表示在 socket queue 中有多少数据等待接收或发送。其他的都差不多。

可参考 [Netstat: network analysis and troubleshooting, explained](https://acloudguru.com/blog/engineering/netstat-network-analysis-and-troubleshooting-explained)

### 调用socket时的操作

![消息收发操作](/Users/tianyou/Documents/Github/go_study/.go_study/assets/network-how/2.1-4.png)

创建套接字时，首先分配一个套接字所需的内存空间，然后向其中写入初始状态。

接下来，需要将表示这个套接字的描述符告知应用程序。收到描述符之后，应用程序在向协议栈进行收发数据委托时就需要提供这个描述符。由于套接字中记录了通信双方的信息以及通信处于怎样的状态，所以只要通过描述符确定了相应的套接字，协议栈就能够获取所有的相关信息，这样一来，应用程序就不需要每次都告诉协议栈应该和谁进行通信了。
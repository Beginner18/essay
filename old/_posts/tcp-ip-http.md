---
title: tcp-ip-http
date: 2019-06-28 19:39:05
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "网络" #分类
archives: "近期"
tags:   #标签
	- tcp/ip
	- http1.0-2
	- webSocket
	
---

1 socket
---
>- socket是一套完成TCP，UDP协议的接口。
>- socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议。
>- socket连接：socket连接就是所谓的长连接，理论上客户端和服务器端一旦建立起连接将不会主动断掉；但是由于各种环境因素可能会是连接断开，比如说：服务器端或客户端主机down了，网络故障，或者两者之间长时间没有数据传输，网络防火墙可能会断开该连接以释放网络资源。所以当一个socket连接中没有数据的传输，那么为了维持连接需要发送心跳消息~~具体心跳消息格式是开发者自己定义的
>- 套接字用于描述IP地址和端口，是一个通信链的句柄。应用程序通过套接字向网络发出请求或应答网络请求。注意的是套接字既不是程序也不是协议，只是操作系统提供给通信层的一组抽象API接口。
>
![socket简图](../pic/tcp-ip/socket在tcpip中的应用.jpg)
>- 服务端首先初始化Socketsocket()，然后与端口绑定bind()，再对端口进行监听listen()，接着调用accept()堵塞等待客户端连接。此时，若有一个客户端初始化了一个Socket，然后连接服务端connect()。若连接成功，此时客户端与服务端的连接就建立了。客户端发送请求write()，服务端接收请求并处理read()，然后将回应发送给客户端write()，客户端读取数据read()，最后关闭连接close()，一次交互结束。
![socket工作原理](../pic/tcp-ip/socket工作原理.jpg)
>- socket读写过程：
>
```
当客户端和服务端使用TCP协议进行通信时，客户端封装一个请求对象req，将其序列化成为字节数组，然后通过套接字socket将字节数组发送到服务端，服务端通过套接字socket读取到字节数组，再反序列化成为请求对象req后进行处理，处理完毕后生成一个响应对应的res，将响应对象res序列化成字节数组，然后通过套接字将字节数组发送给客户端，客户端通过套接字socket读取到字节数组，再反序列化成为响应对象。
平时使用的套接字其实只是一个引用（一个对象ID），这个套接字对象实际上是放在操作系统内核中。其内部有两个重要的缓冲结构，一个是读缓冲read buffer，一个是写缓存write buffer，他们都是有限大小的数组结构。
但对客户端的socket写入字节数组时，即序列化后的请求消息对象req，是将字节数组拷贝到内核区套接字对象的write buffer中，内核网络模块会有单独的线程负责不停地将write buffer的数据拷贝到网卡硬件，网卡硬件再将数据传送到网线，经过一系列路由器和交换机，最终送达服务器的网卡硬件中。
同样，服务端内核的网络模块也有单独的线程不停地接收到数据拷贝到套接字的read buffer中等待用户层来读取，最终服务端的用户进程通过socket引用的read()方法将read buffer中的数据拷贝到用户程序内部中，进行反序列化成请求对象req进行处理。然后服务端将处理后的相应对象，走一个相反的流程发送给客户端。
```
>- socket长链接：
>
```
短连接一般都是单项请求数据，服务端不能主动将数据推送给客户端，而长连接即可利用后端与前端的技术结合，实现服务端的推送功能，若数据库有更新，后端程序即可立即将数据推送出来，无需多次反复请求、建立连接、断开连接。
长连接实质是指建立socket套接字连接后不断是否适用都保持连接，其安全性较差。而短连接指的是建立socket套接字连接后发送接收数据后马上断开连接。
长连接在基于TCP的通讯中，一直保持连接，不管当前是否发送或接收数据。而短连接只是在有数据传输的时候才进行连接，服务端和客户端通信与传输数据完毕后就关闭连接，例如在HTTP、CMPP等。
```
>- 什么是心跳包为什么需要：
心跳包就是在客户端和服务端间定时通知对方自己状态的一个自己定义的命令字，按照一定的时间间隔发送，类似于心跳，所以叫做心跳包。网络中的接收和发送数据都是使用Socket进行实现。但是如果此套接字已经断开（比如一方断网了），那发送数据和接收数据的时候就一定会有问题。可是如何判断这个套接字是否还可以使用呢？这个就需要在系统中创建心跳机制。其实TCP中已经为我们实现了一个叫做心跳的机制。如果你设置了心跳，那TCP就会在一定的时间（比如你设置的是3秒钟）内发送你设置的次数的心跳（比如说2次），并且此信息不会影响你自己定义的协议。也可以自己定义，所谓“心跳”就是定时发送一个自定义的结构体（心跳包或心跳帧），让对方知道自己“在线”,以确保链接的有效性。
>
```
定义自己的协议
如果想要使传输的数据有意义，则必须使用到应用层协议比如Http、Mqtt、Dubbo等。基于TCP协议上自定义自己的应用层的协议需要解决的几个问题：
心跳包格式的定义及处理
报文头的定义，就是你发送数据的时候需要先发送报文头，报文里面能解析出你将要发送的数据长度
你发送数据包的格式，是json的还是其他序列化的方式
```
>- socket连接池：
>
```
什么是Socket连接池,池的概念可以联想到是一种资源的集合，所以Socket连接池，就是维护着一定数量Socket长连接的集合。它能自动检测Socket长连接的有效性，剔除无效的连接，补充连接池的长连接的数量。从代码层次上其实是人为实现这种功能的类，一般一个连接池包含下面几个属性：
空闲可使用的长连接队列
正在运行的通信的长连接队列
等待去获取一个空闲长连接的请求的队列
无效长连接的剔除功能
长连接资源池的数量配置
长连接资源的新建功能
场景： 一个请求过来，首先去资源池要求获取一个长连接资源，如果空闲队列里面有长连接，就获取到这个长连接Socket,并把这个Socket移到正在运行的长连接队列。如果空闲队列里面没有，且正在运行的队列长度小于配置的连接池资源的数量，就新建一个长连接到正在运行的队列去，如果正在运行的不下于配置的资源池长度，则这个请求进入到等待队列去。当一个正在运行的Socket完成了请求，就从正在运行的队列移到空闲的队列，并触发等待请求队列去获取空闲资源，如果有等待的情况。
```
[socket简介](https://www.jianshu.com/p/01b9a454de5a)
[socket简单理解](https://www.jianshu.com/p/9fbf9b78e2c0)
>- socket之family: 
>
```
    +---------------------------------------------
    |   family        |              说  明       
    |---------------- | --------------------------
    |   AF_INET       |             IPv4协议      
    |   AF_INET6      |             IPv6协议      
    |   AF_LOCAL      |             Unix域协议    
    |   AF_ROUTE      |             路由套接字    
    |   AF_KEY        |             秘钥套接字    
    +---------------------------------------------       
```
>- socket type:
>
```
    +---------------------------------------------
    |   type          |          说     明        
    |---------------- | --------------------------
    |  SOCK_STREAM    |         字节流套接字      
    |  SOCK_DGRAM     |         数据报套接字      
    |  SOCK_SEQPACKET |         有序分组套接字    
    |  SOCK_RAW       |         原始套接字(IP)        
    +---------------------------------------------
```
>- protocol协议：
>
```
    +---------------------------------------------
    |   protocol      |          说      明       
    |---------------- | --------------------------
    |   IPPROTO_TCP   |          TCP传输协议      
    |   IPPROTO_UDP   |          UDP传输协议      
    |   IPPROTO_SCTP  |          SCTP传输协议     
    +---------------------------------------------
```
>- socket连接：connect timeout表明IP不在， connect refused为端口不在；
>- 	connect函数：
>
```
#include <sys/socket.h>
int connect (int sockfd, const struct sockaddr *seraddr, socklen_t addrlen);
```
>- bind函数：绑定端口（一般服务端用，知名端口)
>
```
#include <sys/socket.h>
int bind (int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
//返回值情况：若成功则返回0，若失败则返回-1
```
>- listen函数(linux内核，backlog可以限制等待链接队列&已完成连接队列):
>
```
listen函数仅由TCP服务器调用，当被调用时，它完成两件事：
(1)当socket函数创建一个套接字时，被创建的套接字被假设为一个主动套接字，也就是说，它是一个将调用connect发起连接的客户套接字。listen函数把一个未连接的套接字转换成一个被动套接字，知识内核应接受指向该套接字的连接请求。根据TCP状态转换图，调用listen导致套接字从CLOSED状态转换到LISTEN状态。
(2)本函数的第二个参数规定了内核应该为相应的套接字排队的最大连接个数。
#include <sys/socket.h>
int listen (int sockfd, int backlog);
```
![tcp监听套接字两个队列](../pic/tcp-ip/tcp为监听套接字维护的两个队列.png)
![tcp listen](../pic/tcp-ip/tcp之listen.png)

>- accept函数（返回内核自动生成的全新描述符）： accept函数由TCP服务器调用，用于从已完成连接队列队列头返回下一个已完成连接。如果已完成连接为空，那么进程被投入睡眠。如果accept函数调用成功，其返回值是由内核自动生成的一个全新描述符，代表与所返回客户的TCP连接。accept函数的第一个参数为监听套接字描述符，称它的返回值为已连接套接字。一个服务器通常仅仅创建一个监听套接字，它在该服务器的生命期内一直存在。每个由服务器进城接受的客户连接，内核都会为它创建一个已连接套接字。当服务器完成对某个给定客户的服务时，相应的已连接套接字就被关闭。
>- closet函数：终止tcp连接,可以用shutdown只关闭读或者写
>
```
#include <unistd.h>
int close (int sockfd);
//返回值情况：若成功则为0，若出错则为-1
close一个TCP套接字的默认行为是把该套接字标记成已关闭，然后立即返回到调用进程。该套接字描述符不能再由调用进程使用，也就是说它不能再作为read函数或write函数的第一个参数。
```
[tcp套接字编程](https://www.jianshu.com/p/70f2c260067b)
>- 案例：
>- 
```
#include#include
#include
#include
#include
#include
#include
#include
#include
static bool stop = false;
/*SIGTERM 信号的处理函数，触发时结束主进程中的循环*/
static void handle_term(int sig)
{
    stop = true;
}
int main(int argc, char* argv[])
{
    signal(SIGTERM, handle_term);
    if(argc <= 3)
    {
        printf("usage: %s ip_address port_number backlog\n", basename(argv[0]));
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi(argv[2]);
    int backlog = atoi(argv[3]);
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    assert(sock >= 0);
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(sock, backlog);
    assert( ret != -1);
    /*循环等待连接，直到有SIGTERM信号将它中断*/
    while(!stop)
    {
        sleep(1);
    }
    /*关闭socket*/
    close(sock);
    return 0;
}
```
>- 
>
```
#include#include
#include
#include
#include
#include
#include
#include
#include
static bool stop = false;
/*SIGTERM 信号的处理函数，触发时结束主进程中的循环*/
static void handle_term(int sig)
{
    stop = true;
}
int main(int argc, char* argv[])
{
    signal(SIGTERM, handle_term);
    if(argc <= 3)
    {
        printf("usage: %s ip_address port_number count\n", basename(argv[0]));
        return 1;
    }
    const char* ip = argv[1];
    int port = atoi(argv[2]);
    int count= atoi(argv[3]);
    struct sockaddr_in server_address;
    bzero(&server_address, sizeof(server_address));
    server_address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_address.sin_addr);
    server_address.sin_port = htons(port);
    int sockfd[count];
    for(int i = 0; i < count; i++)
    {
        sockfd[i] = socket(AF_INET, SOCK_STREAM,0);
        assert(sockfd[i] >= 0);
        if(connect(sockfd[i],(struct sockaddr*)&server_address,sizeof(server_address)) < 0)
        {
            printf("connection failed\n");
        }
    }
    /*循环等待连接，直到有SIGTERM信号将它中断*/
    while(!stop)
    {
        sleep(1);
    }
    for(int j = 0; j < count; j++)
    {
        close(sockfd[j]);
    }
    return 0;
}
```

1.1 socketfd
---
>- 套接字描述符，socket函数成功时返回的一个小的非负值与文件描述符类似，成为套接字描述符socketfd
>- 得到socketfd需要执行协议族

2 tcp/ip 
---
数据链路层是解决同一网络内节点之间的通信，而网络层主要解决不同子网间的通信。例如在广域网之间通信时，必然会遇到路由（即两节点间可能有多条路径）选择问题
一般五层模型
>
```
OSI只是一个理论上的参考模型，也就是说实际应用中作为参考的。现行的tcp/ip协议族选用了5层架构，把 
会话层和表示层的功能整合在了应用层，这样有助于给开发者更多的选择。层次太多增加协议的复杂性也造成
效率的折损。在osi参考模型中，会话层的功能是会话控制和同步，表示层是解决两个系统间交换信息的语法
与语义问题，还有数据表示转化(转为主机无关编码)，加解密和压缩与解压缩功能。很明显这两层在实际应用
中很难保持统一性，应用通常会选择不同的加解密方式，不同的语义和时序，谈不上复用的东西作为协议的一
部分自然是没有意义的，所以这两层的功能交给应用开发者作为应用层的一部分功能开发是比较合适的。
```
![osi七层模型](../pic/tcp-ip/osi七层网络协议.jpg)
[OSI上三层模型](https://blog.csdn.net/Kaiwii/article/details/8224261)
[OSI七层模型](https://blog.csdn.net/yaopeng_2005/article/details/7064869)
[会话层表示层应用层](https://www.jianshu.com/p/801e650850b7)
[wireshark抓包工具](https://blog.csdn.net/ch853199769/article/details/78753963)

2.1 链路层
---
>- 最大传输单元MTU: 以太网和8 0 2 . 3对数据帧的长度都有一个限制，其最大值分别是1 5 0 0和1 4 9 2字节。链路层的这个特性称作M T U，最大传输单元。不同类型的网络大多数都有一个上限。
如果I P层有一个数据报要传，而且数据的长度比链路层的M T U还大，那么I P层就需要进行分片（ f r a g m e n t a t i o n），把数据报分成若干片，这样每一片都小于M T U。
>- 路径MTU：路径MTU：当在同一个网络上的两台主机互相进行通信时，该网络的M T U是非常重要的。但是如果两台主机之间的通信要通过多个网络，那么每个网络的链路层就可能有不同的M T U。重要的不是两台主机所在网络的M T U的值，重要的是两台通信主机路径中的最小M T U。它被称作路径M T U。
>- 两台主机之间的路径M T U不一定是个常数。它取决于当时所选择的路由。而选路不一定
是对称的（从A到B的路由可能与从B到A的路由不同），因此路径M T U在两个方向上不一定是
一致的。
DST:目的地mac地址
SRC:发送方mac地址
标示上层协议
校验合

![以太网帧](../pic/tcp-ip/IEEE分组以太网帧.jpeg)
[以太网帧，链路层](https://www.cnblogs.com/fadewalk/p/10539628.html)
>- arp协议：地址解析协议;功能上为链路层协议，层次封装上与ip协议平起平坐可以作为网络层协议；ARP（Address Resolution Protocol）即地址解析协议， 用于实现从 IP 地址到 MAC 地址的映射，即询问目标IP对应的MAC地址。
>
```
协议到底所属哪一层，可以从应用/功能来考虑，也可以从层次/包封装来考虑。
以ARP协议为例，它的功能最终是获取到MAC信息，服务于链路层，从这点考虑，ARP是链路层协议；但是从层次来看，ARP基于Ethernet协议，IP协议基于Ethernet协议，它们在Ethernet协议里面有独立的Type类型，前者是0x0806，后者是0x0800，既然ARP和IP协议"平起平坐"，那么IP是网络层，ARP难道就不是网络层？
```
![arp地址解析协议](../pic/tcp-ip/ARP协议.jpg)
>- 广播：大部分的广播包，它们有一个共同特征：二层封装时目的MAC是全f（ffff.ffff.ffff）或三层封装时目的IP是全1（255.255.255.255）。可以这样更方便的记住：目的地址最大的，就是广播。
[arp协议](https://zhuanlan.zhihu.com/p/28771785)
![arp请求](../pic/tcp-ip/arp请求.jpg)
![arp应答](../pic/tcp-ip/arp应答.jpg)


2.2 IP协议
---
>- IP分片： 若一IP数据报大小超过相应链路的MTU的时候，IPV4和IPV6都执行分片(fragmentation)，各片段到达目的地前通常不会被重组(re-assembling)。IPV4主机对其产生的数据报执行分片，IPV4路由器对其转发的数据也执行分片。然而IPV6只在数据产生的主机执行分片；IPV6路由器对其转发的数据不执行分片。
>- 另外，IP数据报分片后，只有第一片带有UDP首部或ICMP首部，其余的分片只有IP头部，到了端点后根据IP头部中的信息再网络层进行重组。而TCP报文段的每个分段中都有TCP首部，到了端点后根据TCP首部的信息在传输层进行重组。IP数据报分片后，只有到达目的地后才进行重组，而不是向其他网络协议，在下一站就要进行重组。
>- 最后一点，对IP分片的数据报来说，即使只丢失一片数据也要重新传整个数据报（既然有重传，说明运输层使用的是具有重传功能的协议，如TCP协议）。这是因为IP层本身没有超时重传机制------由更高层（比如TCP）来负责超时和重传。当来自TCP报文段的某一段（在IP数据报的某一片中）丢失后，TCP在超时后会重发整个TCP报文段，该报文段对应于一份IP数据报（可能有多个IP分片），没有办法只重传数据报中的一个数据分片。
![IPv4协议](../pic/tcp-ip/IPv4协议.jpeg)
version: ip协议版本
header lenght:ipv4头部中32位字的数量
type of service:8bit, 前三位为优先级，D T R位+两位保留位；D T R全部置位1表示低延迟 高吞吐 高可靠性；eg: 111 0 1 1 00；最高优先级，高吞吐 高可靠；保留位可用于拥塞标志，路由器转发分组时有能力可以设置，有些协议发现拥塞标志会通知发送方；发送方降低发送速率缓解拥塞；
identification:分片标识，每发送一个ip数据片计数+1
flag: 标识是否为原始数据报的最后一个分组
fragment Offset: 分片的起始字节在原始数据报中的偏移值/位置
TTL：经过的路由器上线
protocol：上层协议
header checksum:首部校验和(ipv6无)
![IP转发](../pic/tcp-ip/IP转发.jpg)
路由选择（ip转发）：从路由表中选择与目的地ip最匹配的下一跳ip，获取到mac地址，更改dst的mac地址，实现路由选择；ip地址的DST SRC均不变； P145页
>
```
源和目的mac地址产生变化
源和目的ip地址不变
转发的过程：
1、主机A查看查看自己的ARP表。如果主机A第一次给主机B发送数据包，那将无法从ARP映射表中找到对应的mac地址。
所以主机A将以FF:FF:FF:FF:FF:FF为目的mac地址，发送一个广播包。
2、这个时候，如果主机B在本网段内。则主机B会向主机A发送一个应答包。让主机A得到主机B的mac地址。这样主机A
将分别以自己和主机B的mac地址作为源和目的地址，以自己和主机B的ip地址作为源和目的地址发送数据包。交换机将通
过自身的mac地址表找到主机B对应的端口，并发送给主机B。
3、但是一般情况并非是这样，如果主机B不在本网段内。路由器的端口将通过把自己的mac地址发送给主机A，这个时候
主机A的操作和2基本相同。将目的mac地址替换为路由器端口的mac地址。并且发送出去，并且通过同样的道理发送到路
由器的端口。（注：如果是直连，则不需经过交换机直接发送）
4、路由器从端口中接收数据包之后，进行数据链路层帧头的拆包。并从网络层的ip数据报头中获取目的ip地址，根据路由
表进行转发。获得下一跳的ip地址后，如果路由器中没有下一跳的mac地址，则进行mac广播这里跟1中是相同的。但是区
别在于，这里就算知道了下一跳的ip地址，但是保留原ip报文中的ip信息。不进行替换。
5、路由器将源mac地址替换为转发出的端口mac地址，目的mac地址转换为下一跳mac地址。进行转发。
6、当下一跳收到数据包后，重复3、4、5的步骤。直到主机B收到对应的数据包
```



![IPv6协议](../pic/tcp-ip/IPv6协议.gif)

***flow label***: ipv6流标识???作用是啥？
===

下一头部：标识下一头部的协议，扩展类型，下一头部可以为ip分片 tcp。。。
![IPV6分片](../pic/tcp-ip/ipv6分片.jpg)
ipv6分片头部包括：下一个头部、保留位、分片偏移、M 1为包含更多分片 0表示为原始分片的最后一个分片、标识符

2.3 ARP协议地址解析协议
---
IP地址获取硬件地址：确定帧目的地接口
网络硬件：主地址由设备制造商定义，并储存在设备的永久性内存中
网络层地址到硬件层地址的动态映射
arp消息中的目的地以太网地址可不填写
广播的形式发送：DST为ff:ff:ff:ff广播地址；
基于链路层
OP： 1为arp请求、2为arp应答、3为RARP请求、4为RARP应答
![ARP协议](../pic/tcp-ip/arp帧地址格式.jpeg)

2.4 ICMP协议（IP层一部分）
---
负责传递需要注意的差错报文和控制报文
通常由IP层本身、上层的传输协议甚至某种情况下用户触发执行
不为IP网络提供可靠性
标明故障和配置信息
丢包（路由器缓冲区溢出）不会触发ICMP消息
可用于邻居发现、配置主机管理组播地址
ICMP报文在IP数据报内被封装传输
信息类请求操作系统自动处理
>- 请求/回显 ping
>- 路由发现
>- 邻居发现
>- 组播侦听
>- 。。。
差错报文传递给用户进程或者传输层协议：差错报文包括违规数据报
>- 重定向
>- 目的不可达
>- 端口不可达

![ICMP协议](../pic/tcp-ip/ICMP协议.jpg)

2.5 TCP协议
---
>- 名次解释：
>
>>- window: congestion window，receive window，sliding window 等等。window 是以 tcp segment 数量为单位，我们可以说当前 window 值由几个 tcp 包构成，而当我们说 window size 的时候，又是在说一个 window 所包含的字节数。window size 除了和 tcp segment 的数量有关之外，还和单个 tcp segment 的最大 size 有关，即 MSS 值。
>>- MTU（最大传输单元与整个网络有关): MTU：maximum transmission unit，最大传输单元，由硬件规定，如以太网的MTU为1500字节。
>>- MTU = MSS + TCP Header + IP Header.
>>- MSS 值，即 maximum segment size，最大分节大小，为TCP数据包每次传输的最大数据分段大小，一般由发送端向对端TCP通知对端在每个分节中能发送的最大TCP数据。MSS值为MTU值减去IPv4 Header（20 Byte）和TCP header（20 Byte）得到。
>>- transmission window: Transmission window 的大小是真正表示，发送方可以一次发送多少个尚未被 ack 的包。这个值是一般先取 CWND 和 RWND 的较小值，之后会动态的调整变化，slow start 阶段会变化，congestion avoidence 发生时也会变化
>>- MSS 是TCP选项中最经常出现，也是最早出现的选项。MSS选项占4byte。MSS是每一个TCP报文段中数据字段的最大长度，注意：只是数据部分的字段，不包括TCP的头部。TCP在三次握手中，每一方都会通告其期望收到的MSS（MSS只出现在SYN数据包中）如果一方不接受另一方的MSS值则定位默认值536byte。
MSS值太小或太大都是不合适。太小，例如MSS值只有1byte，那么为了传输这1byte数据，至少要消耗20字节IP头部+20字节TCP头部=40byte，这还不包括其二层头部所需要的开销，显然这种数据传输效率是很低的。MSS过大，导致数据包可以封装很大，那么在IP传输中分片的可能性就会增大，接受方在处理分片包所消耗的资源和处理时间都会增大，如果分片在传输中还发生了重传，那么其网络开销也会增大。因此合理的MSS是至关重要的。MSS的合理值应为保证数据包不分片的最大值。对于以太网MSS可以达到1460byte.
>>-  到了这里有一个问题自然就明了了，TCP分段的原因是MSS，IP分片的原因是MTU，由于一直有MSS<=MTU，很明显，分段后的每一段TCP报文段再加上IP首部后的长度不可能超过MTU，因此也就不需要在网络层进行IP分片了。因此TCP报文段很少会发生IP分片的情况。
[tcp分片Ip分片讲解](https://www.cnblogs.com/alan666/p/8311901.html)
[tcp滑动窗口单位为字节](https://blog.csdn.net/zmj_88888888/article/details/9091957)
![TCP协议](../pic/tcp-ip/tcp协议.jpeg)
四元标识：两方IP+端口
三次握手
![三次握手](../pic/tcp-ip/TCP三次握手.jpeg)
>- 交换数据：窗口大小 seq等
>- ack确认
四次挥手
![四次挥手](../pic/tcp-ip/四次挥手.jpeg)
>- timeWait:最后一个ack确保收到，未收到会重发；2个msl老数据消散
>
```
msl(maxinum segment lifetime)任何tcp实现必须选择一个MSL建议2min，berkeley的实现为30s;
MSL为任何IP数据包能够在因特网中存活的最长时间，该时间是有限的任何IP数据有跳限限制；
路由器错误会导致迷途分组：在多个路由器中来回循环，跳限达到IP数据消失；
可靠实现tcp全双工连接的终止：若被动关闭方未收到ack会重新发送（主动关闭方需要timewait)，等待主动关闭方再次发送ack；
允许老的重复分节在网络中消失：每次成功建立tcp连接保证来自该连接先前的化身的老的重复分组已经在网络中消失；timewait期间不会对相同IP相同端口新建连接；
```
>- 一方关闭，另一方先确认ack，仍可继续发送数据，待发送完数据再发送fin
> ![tcp状态转换图](../pic/tcp-ip/tcp状态转换图.jpg)
>- 窗口大小：两方交换窗口大小
seq：双方交换初始化的seq值
>- 滑动窗口：提升网络吞吐量；TCP滑动窗口协议主要为了解决在网络传输数据的过程中，发送方和接收方传输数据速率不一致的问题，从而保证数据传输的可靠性，达到流量控制的效果
>
```
发送方收到ack后将窗口值滑动至ack序列号位置；
接收方根据应用程序处理接收数据速度+缓存大小更新窗口
发送方收到窗口更新大小变成窗口大小
```
[滑动窗口](https://juejin.im/post/5c9f1dd651882567b4339bce)
>- 慢启动：慢启动就开始发送数据时的行为控制。慢启动的总体思路就是从一个很低的初始值开始，逐渐增加数据发送的速度，直到达到超时或者丢包为止。慢启动的实现思路如下
>
```
当主机开始发送数据时，如果立即将大量数据注入网络，就有可能引起网络拥塞。因此，较好的方法是由小到大逐渐增大发送窗口。
每个连接维护一个变量cwnd（congestion window）
当连接刚建立或者遇到丢包时，将cwnd设置为1，单位为MSS（maximum segment size）
每收到一个新的ack，cwnd加一
当发送数据时，能够发送的数据包数量为min(cwnd,awnd)，awnd为接收端的滑动窗口大小(reciver's advertised window)
这样，每过一个传输轮次，cwnd的值变为2，4，8，···，指数型增加。
可以预见，在没有出现超时或者丢包时，慢启动增长的速度是指数级的，所以慢启动实际上并没有那么“慢”，“慢”是慢在它的起点只有1个MSS。
```
>- 拥塞控制：拥塞窗口减半，接收2个ack可发送一个数据包，降低发送速率
>- 拥塞避免：以较小增量发送数据
>
```
前面提到，慢启动的目的是逐渐增加发送速度进行试探，直到出现网络拥塞，而真正出现拥塞时又该怎么做呢，就是“拥塞避免”所做的事情了。拥塞避免主要由两部分组成：
一个判断当前网路出现拥塞的机制
在出现拥塞时降低发送速度的机制
而拥塞避免的实现思路如下：
当出现超时时，将cwnd设置为当前值的一半（即当出现超时时就认为是出现了拥塞）
每收到一个新的ack，cwnd加1/cwnd（即当传输成功cwnd个数据包时，窗口大小加一，也就是随着RTT线性增加）
这里的两个变更cwnd的行为通常称为“乘法减小”和“加法增大”。
```
>- 慢启动&拥塞避免：
>
```
值得注意的是，慢启动和拥塞避免实际上是两个不同的算法，它们一个用于试探网络资源的上限，另一个用于资源使用率达到或者接近上限时的行为。在1988年的论文中给出了一个结合了慢启动和拥塞避免的算法，具体实现思路如下：
发送端维持两个变量：拥塞窗口cwnd（congestion window）和慢启动门限ssthresh（slow start threshold），通过这两个变量来决定当前应该执行慢启动还是拥塞避免算法。
发送数据时，能够发送的数据包数量为min(cwnd,awnd)
出现超时时，ssthresh更新为cwnd/2（但不能小于2），cwnd设置为1
每收到一个新的zck，发送端的行为是:
如果cwnd<ssthresh，cwnd+=1（慢启动阶段，窗口指数级别增加）
否则cwnd+=1/cwdn（拥塞避免阶段，窗口线性增加）
```
>- 快速重传：多次收到重复ack则重新发送数据
>
```
快速重传的目的就是让发送端尽快感知到丢包。TCP发送方在每发送一个分段时会启动一个计时器，如果相应的数据包确认没在特定时间内被送回，发送方就假设这个分段在网络上丢失了，需要重发。这也是TCP用来估计RTT的测量方法。
```
>- 快速恢复：在上文，知道只是报文段丢失，而不是网络出现拥塞后，发送方会调整ssthresh为原来的一半，然后继续进行拥塞避免算法，这个过程就叫快开恢复算法。
![tcp慢启动拥塞避免快速重传](../pic/tcp-ip/tcp control.jpg)
[tcp慢启动拥塞控制](https://juejin.im/post/5d2757356fb9a07ef7109ecc)



>- 重复确认：
>
```
重复确认基于以下过程：如果接收方接收到一个数据分段，就会将该分段的序列号加上数据字节长的值，作为分段确认的确认号，发送回发送方，表示期望发送方发送下一个序列号的分段。但是如果接收方提前收到更大的序列号的分段，或者说接收到无序到达的分段，接收方需要立即使用之前的确认号发送分段确认。此时如果发送方收到接收方相同确认号的分段确认超过1次，并且该对应序列号的分段超时计时器仍没超时的话，则这就是出现重复确认，需要进入快速重传。
快送重传就是基于以下机制：如果假设重复阈值为3，当发送方收到4次相同确认号的分段确认（第1次收到确认期望序列号，加3次重复的期望序列号确认）时，则可以认为继续发送更高序列号的分段将会被接受方丢弃，而且会无法有序送达。发送方应该忽略超时计时器的等待重发，立即重发重复分段确认中确认号对应序列号的分段。
```
>- 超时重传：超过ttl仍未收到ack重传
>- 延时确认：返回ack时，等一定时间有数据或者多个ack时只发送一个最新ack
>- Nagle算法：小数据包时，收到ack确认才可发送下一个小数据包；可以禁用
Nagle+延时确认会造成短时死锁
>- SACK: 接受到的多个ack范围区间，发送方接到sack可以判定丢包或者乱序数据，重发
>- 保活机制：不影响数据流的情况下探测对方；非正常关机：直接断网
>
```
为何需要保活机制：
通过保活判定对方是否在线
连接是否断掉
```
>- 保活机制的实现：
>
```
保活机制是由一个保活计时器实现的。当计时器被激发，连接一段将发送一个保活探测报文，另一端接收报文的同时会发送一个ACK作为响应。
保活时间：默认7200秒（2小时）
保活时间间隔：默认75秒
保活探测数：默认9次
保活时间内连接处于无数据传输状态，则可发起保活探测；保活时间
没有收到ack，间隔一定时间持续发送；保活时间间隔
若干次没有收到，认为主机不可达，中断连接；保活探测数
```
[保活机制](https://juejin.im/post/5d1b2e6fe51d45775c73dd3e)
>- tcp拆包粘包：
>
```
TCP是个“流”协议，所谓流，就是没有界限的一串数据。大家可以想想河里的流水，是连成一片的，其间并没有分界线。TCP底层并不了解上层业务数据的具体含义，它会根据TCP缓冲区的实际情况进行包的划分，所以在业务上认为，一个完整的包可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的TCP粘包和拆包问题。
```
![tcp拆包粘包](../pic/tcp-ip/tcp拆包粘包.png)
>- MSS(tcp最大传输段大小，即一个tcp包最大传输字段max segement size)，MTU(max transition unit最大传输单元，与整个网络状态有关）:
>
```
MSS最大传输大小的缩写，是TCP协议里面的一个概念。
MSS就是TCP数据包每次能够传输的最大数据分段。为了达到最佳的传输效能TCP协议在建立连接的时候通常要协商双方的MSS值，这个值TCP协议在实现的时候往往用MTU值代替（需要减去IP数据包包头的大小20Bytes和TCP数据段的包头20Bytes）, 通讯双方会根据双方提供的MSS值得最小值确定为这次连接的最大MSS值。而一般以太网MTU都为1500, 所以在以太网中, 往往TCP MSS为1460。
```
>- 若tcp的MSS大于MTU-40则在ip层会做ip分割，首个ip分节包含端口，之后不带端口，对于NAT协议（需要ip+端口+传输协议六元组）则数据无法到达
[tcp的MSS&网络MTU](https://blog.csdn.net/Superwen_go/article/details/78564368)
>
```
假设客户端分别发送了两个数据包D1和D2给服务端，由于服务端一次读取到的字节数是不确定的，故可能存在以下4种情况。
（1）服务端分两次读取到了两个独立的数据包，分别是D1和D2，没有粘包和拆包；
（2）服务端一次接收到了两个数据包，D1和D2粘合在一起，被称为TCP粘包；
（3）服务端分两次读取到了两个数据包，第一次读取到了完整的D1包和D2包的部分内容，第二次读取到了D2包的剩余内容，这被称为TCP拆包；
（4）服务端分两次读取到了两个数据包，第一次读取到了D1包的部分内容D1_1，第二次读取到了D1包的剩余内容D1_2和D2包的整包。
如果此时服务端TCP接收滑窗非常小，而数据包D1和D2比较大，很有可能会发生第五种可能，即服务端分多次才能将D1和D2包接收完全，期间发生多次拆包。
```
>- 拆包粘包发生原因：
>
```
问题产生的原因有三个，分别如下。
（1）应用程序write写入的字节大小大于套接口发送缓冲区大小；
（2）进行MSS大小的TCP分段；
（3）以太网帧的payload大于MTU进行IP分片。
```
>- 粘包问题解决策略：
>
```
由于底层的TCP无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题只能通过上层的应用协议栈设计来解决，根据业界的主流协议的解决方案，可以归纳如下。
（1）消息定长，例如每个报文的大小为固定长度200字节，如果不够，空位补空格；
（2）在包尾增加回车换行符进行分割，例如FTP协议；
（3）将消息分为消息头和消息体，消息头中包含表示消息总长度（或者消息体长度）的字段，通常设计思路为消息头的第一个字段使用int32来表示消息的总长度；
（4）更复杂的应用层协议。
```
[tcp拆包粘包](https://www.cnblogs.com/wade-luffy/p/6165671.html)

2.6 其他
---
>- DHCP自动配置：以广播的形式请求ip地址discovery，获得路由器应答offer，绑定ip地址获取应答request获取ack；
>- NAT地址转换：内网ip+port转换为公网ip+new port（一般一个公网ip可以利用NAT连接很多内网机器，ip跟端口组成映射，跨多层传输协议）
DNS：域名解析


[tcp/ip协议详解](https://juejin.im/post/5a069b6d51882509e5432656)
[一篇文章带你熟悉 TCP/IP 协议](https://juejin.im/post/5a069b6d51882509e5432656)
[一篇文章带你熟悉http协议](https://www.jianshu.com/p/6e9e4156ece3)
[网络接口](https://blog.csdn.net/wsclinux/article/details/55049708)
[tcp协议详解](https://blog.csdn.net/sinat_36629696/article/details/80740678)
[深入浅出tcp](https://blog.csdn.net/itermeng/article/details/77973279)
[tcp http与socket连接的区别](https://blog.csdn.net/min996358312/article/details/68969519)
[数字签名与验签](https://blog.csdn.net/rosetta/article/details/7712606)
[websocket](https://www.cnblogs.com/fuqiang88/p/5956363.html)

3 udp
---
![UDP协议](../pic/tcp-ip/UPD协议.jpeg)
>- 使用 UDP 需要有强大到不容置疑的理由，when in doubt, use TCP.
>- 广播：不能跨路由器，所有机器都接收解析到ip+port层；不感兴趣的主机也必须接受
>- 组播：感兴趣的主机订阅组播IP（可以限制源ip即发送组播消息的ip地址范围），路由器支持，组播消息只发送给感兴趣的主机；
>- UDP应用范围：
>
>>- 实时音视频是可以而且应该用 UDP 的，一方面因为它常常涉及到网络穿透，另外一方面它不需要重传。——我需要实时的看到你的图像跟声音，至于中间丢一帧什么的完全不重要。而为了重传往往会造成延迟与不同步，考虑一下，某一帧因为重传，导致0.5秒以后才到，那么整个音视频就延迟了0.5秒。考虑一下接收方看视频，如果使用 TCP 导致视频的中间延迟了0.5秒，只要我不按「快进」键，那么后续的视频全都会比发送方延迟0.5秒。这种延迟是累加的，随着持续丢帧，延迟会越来越大，达到数秒，甚至分钟级，这会严重影响实时音视频的用户体验。因此「实时音视频聊天」功能通常都会使用 UDP 实现。
>>- 网络真的非常非常可靠，以至于你完全不需要考虑 UDP 丢包问题的情况。典型的例子应该是专门为有线局域网设计的协议。
>>- 另外一个问题是 TCP 是纯粹的流式数据，所以制定传输协议的时候，接受方需要自行判定一个包的开始和结束，因为你完全可能接受到半个包或者两个包。——如果数据报的起止判定对你具体的程序会成为大问题，也可以考虑 UDP。

>- tcp与udp的区别：
>- ![tcp与udp的区别](../pic/tcp-ip/tcp与udp的区别.jpg)
![tcp与udp连接的区别](../pic/tcp-ip/tcp与udp连接.jpg)
>
```
从上图也能清晰的看出，TCP通信需要服务器端侦听listen、接收客户端连接请求accept，等待客户端connect建立连接后才能进行数据包的收发（recv/send）工作。而UDP则服务器和客户端的概念不明显，服务器端即接收端需要绑定端口，等待客户端的数据的到来。后续便可以进行数据的收发（recvfrom/sendto）工作。 
在前面讲解UDP时，提到了UDP保留了报文的边界，下面我们来谈谈TCP和UDP中报文的边界问题。在默认的阻塞模式下，TCP无边界，UDP有边界。 
对于TCP协议，客户端连续发送数据，只要服务端的这个函数的缓冲区足够大，会一次性接收过来，即客户端是分好几次发过来，是有边界的，而服务端却一次性接收过来，所以证明是无边界的； 
而对于UDP协议，客户端连续发送数据，即使服务端的这个函数的缓冲区足够大，也只会一次一次的接收，发送多少次接收多少次，即客户端分几次发送过来，服务端就必须按几次接收，从而证明，这种UDP的通讯模式是有边界的。
```
>- tcp优点：
> 
```
可靠，稳定
TCP的可靠体现在TCP在传递数据之前，会有三次握手来建立连接，而且在数据传递时，有确认、窗口、重传、拥塞控制机制，在数据传完后，还会断开连接用来节约系统资源。
```
>- tcp缺点： 慢，效率低，占用系统资源高，易被攻击 
>
```
TCP在传递数据之前，要先建连接，这会消耗时间，而且在数据传递时，确认机制、重传机制、拥塞控制机制等都会消耗大量的时间，而且要在每台设备上维护所有的传输连接，事实上，每个连接都会占用系统的CPU、内存等硬件资源。 
而且，因为TCP有确认机制、三次握手机制，这些也导致TCP容易被人利用，实现DOS、DDOS、CC等攻击。
```
>- udp优点：
>
```
快，比TCP稍安全 
UDP没有TCP的握手、确认、窗口、重传、拥塞控制等机制，UDP是一个无状态的传输协议，所以它在传递数据时非常快。没有TCP的这些机制，UDP较TCP被攻击者利用的漏洞就要少一些。但UDP也是无法避免攻击的，比如：UDP Flood攻击……
```
>- udp缺点：
>
```
不可靠，不稳定 
因为UDP没有TCP那些可靠的机制，在数据传递时，如果网络质量不好，就会很容易丢包
```
[tcp与udp](https://blog.csdn.net/neo233/article/details/79435425)

4 http协议 http1.0 http1.1 http2.0
---
[浏览器输入地址到获取资源](https://blog.csdn.net/qq_21993785/article/details/81188253)
[从浏览器输入地址到收到资源展示全流程](https://blog.csdn.net/guizaijianchic/article/details/78024982)

>- http1 建立连接 客户端请求  服务端发送响应并关闭连接 获取响应
>- http1.1 复用连接,建立连接: 请求1->响应1, 请求2->响应2, 其他顺序请求获取响应后才可再次发送请求； 由于复用连接，客户端需要区分每个响应：context-length或者分块传输(Chunked) 
服务端需提供context-length标明响应body的长度，客户端做区分(服务端必须处理完所有数据才可以发送响应)；  
http1.1 规定可以不使用Content-Length字段，而使用"分块传输编码"（Chunked Transfer Encoding）。只要请求或响应的头信息有Transfer-Encoding: chunked字段，就表明body将可能由数量未定的多个数据块组成。每个数据块之前会有一行包含一个16进制数值，表示这个块的长度；最后一个大小为0的块，就表示本次响应的数据发送完了。
>- http2 数据压缩 头跟数据分成不同帧 建立连接 发送请求1 2 3 4 获取请求2 3 1 4 请求返回可以无序
>- http头堵塞问题：服务端顺序处理请求，若某一请求处理时间较久，其后所有请求排队堵塞
>- http基于tcp传输层协议，存在tcp头堵塞问题，Tcp全双工 发送不堵塞接收 
>- http2.0采用二进制数据分帧更好的利用底层tcp，解决1.1的队首问题；
>- http2.0利用分帧机制，不再依赖多个TCP连接去实现多流并行了
>- http2.0客户端发起的流标识为奇数，服务端发起的流标识为偶数
>- http2.0可以支持优先级，首部压缩减少传输数量量，客户端服务端均可以乱序发送数据，之后通过流标识及帧进行数据重组；http2.0服务端支持数据推送


>> http2多路复用
>> 
```
Tcp全双工：
发送数据流头堵塞 按序到达
发送堵塞不影响接收
http:
请求
1 请求以数据帧形式发送
2 并发发送
3 若发送的数据帧出现tcp头堵塞则堵塞
返回
无需按接收到的请求顺序返回结果
完成的结果，按tcp协议写入到socket
发送头堵塞不影响接收端接收数据
server端处理请求方式：
单进程单线程 排队 对头堵塞
单进程多线程 
多进程单线程
多进程多线程
消息：
* 		流：流是连接中的一个虚拟信道，可以承载双向的消息；每个流都有一个唯一的整数标识符（1、2…N）；
* 		消息：是指逻辑上的 HTTP 消息，比如请求、响应等，由一或多个帧组成。
* 		帧：HTTP 2.0 通信的最小单位，每个帧包含帧首部，至少也会标识出当前帧所属的流，承载着特定类型的数据，如 HTTP 首部、负荷，等等
每帧由消息流号标识
```
>- ***http响应四种模型:***
  
>>- 单进程I/O模型: 
服务端开启一个进程，一个进程仅能处理一个请求，并且对请求顺序处理；
>>- 多进程I/O模型: 服务端并行开启多个进程，同样的一个进程只能处理一个请求，这样服务端就可以同时处理多个请求；
>>- 复用I/O模型: 服务端开启一个进程，但是呢，同时开启多个线程，一个线程响应一个请求，同样可以达到同时处理多个请求，线程间并发执行；
>>- 复用多线程I/O模型: 服务端并行开启多个进程，同时每个进程开启多个线程，这样服务端可以同时处理进程数M*每个进程的线程数N个请求。   
[http协议](https://blog.51cto.com/13570193/2108347)  
[http断开连接](https://blog.51cto.com/biketiger/1716962)  
[http2详解](https://blog.csdn.net/zqjflash/article/details/50179235)  
[http1.1 vs http2.0](https://www.cnblogs.com/XiongMaoMengNan/p/8425724.html)  
[http2.0定义的帧](http://www.blogjava.net/yongboy/archive/2015/03/20/423655.html)  
[http协议详解](https://www.jianshu.com/p/6e9e4156ece3)  



>- ***TLS(transport layer security) SSL(socket security layer)***
>- http明文传输不安全
>- TSL/SSL介于应用层http及传输层tcp之间的安全层，提供传输数据加密
>
![https协议](../pic/tcp-ip/https协议.jpeg)
>- TSL/SSL客户端与服务端的传输消息为对称加密，对称加密的加密密钥为非对称加密
>- 版本1: server存在公钥&私钥，cli端建立连接后https请求获取公钥，cli端产生随机密码通过公钥对随机密码加密并传输至server端，server通私钥解密获取密码；cli-server对传输消息通过密码加密处理传输
>- 版本1存在问题：窃听者截取cli获取公钥信息将窃听者自己的公钥发送给cli，cli通过窃听者公钥加密后密码传输，窃听者获取密码后，并将密码通过server公钥加密给server；
>
![https劫持](../pic/tcp-ip/https信息劫持攻击.jpeg)
>- 版本2:CA证书，CA 是 PKI 系统中通信双方信任的实体，被称为可信第三方（Trusted Third Party，简称TTP），CA 证书，就是 CA 颁发的证书
>
>![CA证书](../pic/tcp-ip/CA证书https.jpeg)

>>- Agree on the version of the protocol to use.
>>- Select cryptographic algorithms.
>>- Authenticate each other by exchanging and validating digital certificates.
>>- Use asymmetric encryption techniques to generate a shared secret key, which avoids the key distribution problem. SSL or TLS then uses the shared key for the symmetric encryption of messages, which is faster than asymmetric encryption.
>> ![SSL handshake](../pic/tcp-ip/SSL handshake.gif)
>>- step1: client发送"client hello" 消息为加密信息（SSL/TSL版本、客户端带优先级支持的加密套件、随机byte string)列表
>>- step2: server发送"server hello"消息包括密码套件（从cli发送的密码套件组选择的、sessionID、新的随机byte string、数字证书），server端可以请求客户端数字证书
>>- step3: cli认证数字证书
>>- step4: cli发送随机byte string（被证书中公钥加密）至server，用于后续计算出传输消息所用的密码
>>- step5: 若server要求客户端的数字证书，cli将随机数及cli的数字证书（若没有则"no digital certificate alert“代替）公钥加密后发送给server
>>- step6: server认证cli数字证书
>>- step7: cli发送"finished"消息，密钥加密
>>- step8: server发送"finished"消息，密钥加密
>>- step9: session内cli和server通过密码进行对称加密消息传输

[https TSL/SSL 协议](https://blog.csdn.net/freekiteyu/article/details/76423436) 
[https协议](https://juejin.im/post/5a030e326fb9a0450a66c8ea)
[ssl/tsl handshake](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_7.1.0/com.ibm.mq.doc/sy10660_.htm)

4.1 session&&cookie&&token
---
http协议为无状态协议，如果需要关联不同的http request需要在请求中存储用户数据，可以通过session cookie URL参数
>- URL参数不安全通过URL地址即可知悉参数；用户信息放置在URL parameter不安全
>- cookie&URL参数存储在cli端
>- ***cookie: 保存在本地终端的数据，服务器产生***  
cookie的组成有：名称(key)、值(value)、有效域(domain)、路径(域的路径，一般设置为全局:"\")、失效时间、安全标志(指定后，cookie只有在使用SSL连接时才发送到服务器(https))。下面是一个简单的js使用cookie的例子:
用户登录时产生cookie:   

```
document.cookie = "id="+result.data['id']+"; path=/";
document.cookie = "name="+result.data['name']+"; path=/";
document.cookie = "avatar="+result.data['avatar']+"; path=/";
使用到cookie时做如下解析：
var cookie = document.cookie;var cookieArr = cookie.split(";");var user_info = {};for(var i = 0; i < cookieArr.length; i++) {
    user_info[cookieArr[i].split("=")[0]] = cookieArr[i].split("=")[1];
}
$('#user_name').text(user_info[' name']);
$('#user_avatar').attr("src", user_info[' avatar']);
$('#user_id').val(user_info[' id']);
```

>- ***session为会话管理，cli与服务端交互服务端用session保存用户的信息，用户离开即销毁：***
当用户打开某个web应用时，便与web服务器产生一次session。服务器使用session把用户的信息临时保存在了服务器上，用户离开网站后session会被销毁。这种用户信息存储方式相对cookie来说更安全，可是session有一个缺陷：如果web服务器做了负载均衡，那么下一个操作请求到了另一台服务器的时候session会丢失。
>>
>>若server可以产生sessionID(关联用户数据)，cli请求时带sessionID即可;  
可以对session加过期时间（安全起见）;  
session用户离开页面过期(一段时间无请求则过期)，用户离开session销毁;

>-  ***cookie 和session的区别, cookie可以只保存sessionID***
>>
>>session无法跨机器问题：session外部保存，eg redis缓存  
1、cookie数据存放在客户的浏览器上，session数据放在服务器上。  
2、cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗
   考虑到安全应当使用session。  
3、session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能
   考虑到减轻服务器性能方面，应当使用COOKIE。  
4、单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。  
5、所以个人建议：将登陆信息等重要信息存放为SESSION、其他信息如果需要保留，可以放在COOKIE中  

>- token:  token的意思是“令牌”，是用户身份的验证方式，最简单的token组成:uid(用户唯一的身份标识)、time(当前时间的时间戳)、sign(签名，由token的前几位+盐以哈希算法压缩成一定长的十六进制字符串，可以防止恶意第三方拼接token请求服务器)。还可以把不变的参数也放进token，避免多次查库
>- ***JWT标准Token:***
>>
>>实施 Token 验证的方法挺多的，还有一些标准方法，比如 JWT，读作：jot ，表示：JSON Web Tokens 。JWT 标准的 Token 有三个部分：  
***header（头部）***:加密算法，加密设置alg为对应算法，否则alg为none

```
{
  "alg": "HS256"
}
意思是这个 JWT 用的算法是 HS256。上面的内容得用 base64url 的形式编码一下，所以就变成这样：
eyJhbGciOiJIUzI1NiJ9
```
 
>>
payload（数据）：具体内容为标准字段
>>

```
iss：Issuer，发行者
sub：Subject，主题
aud：Audience，观众
exp：Expiration time，过期时间
nbf：Not before
iat：Issued at，发行时间
jti：JWT ID
eg:
{
 "iss": "ninghao.net",
 "exp": "1438955445",
 "name": "wanghao",
 "admin": true
}
使用 base64url 编码：
eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ
```

>>
***signature（签名）***: JWT 的最后一部分，这部分内容有三个部分，先是用 Base64 编码的 header.payload ，再用加密算法加密一下，加密的时候要放进去一个 Secret ，这个相当于是一个密码，这个密码秘密地存储在服务端。  
>>

```
const encodedString = base64UrlEncode(header) + "." + base64UrlEncode(payload); 
HMACSHA256(encodedString, 'secret');
处理后的签名：
SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
最终token为：
eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ.SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

>>
***Token的签发***：密钥 加密算法 payload 生成签名，组成token   base64URL(head).base64URL(payload).signature;加密算法可选非对称加密  
>>
***Token的验证***：cli cookie传递的token，根据head payload 加密算法 解密密码 验证签名  
>>
***详细参考***: [JWT详解](https://ninghao.net/blog/2834)

>- Token ，如果指的是OAuth Token 或类似的机制的话，提供的是认证和授权，认证是针对用户，授权是针对App;其目的是让 某App有权利访问某用户的信息。这里的 Token是唯一的。不可以转移到其它 App上，也不可以转到其它 用户上

>- token就是令牌，比如你授权（登录）一个程序时，他就是个依据，判断你是否已经授权该软件；cookie就是写在客户端的一个txt文件，里面包括你登录信息之类的，这样你下次在登录某个网站，就会自动调用cookie自动登录用户名；session和cookie差不多，只是session是写在服务器端的文件，也需要在客户端写入cookie文件，但是文件里是你的浏览器编号.Session的状态是存储在服务器端，客户端只有session id；而Token的状态是存储在客户端。 
>- cookie kv形式储存在cli端，可以存储sessionID token等
>- session服务端生成服务端存储用户相关信息，根据cli传递的session进行信息验证
>- token由header payload signature组成，表示加密算法 具体信息 签名，服务端保存签名用密码，根据算法 密码对signature进行验证，header payload不可篡改，否则signature验证无法通过； 
>***auth2.0认证***[auth2.0认证](https://www.jianshu.com/p/243b85771be2)

[session cookie token](https://blog.csdn.net/jikeehuang/article/details/51488020)
[JWT Json WEB token](https://www.jianshu.com/p/af8360b83a9f)
[token原理及应用](https://blog.csdn.net/qq_32784541/article/details/79655146)
[appKey secret vs token](https://blog.csdn.net/xiaofei0859/article/details/50819954)
[auth2.0详解](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

4.2 跨域请求(CORS)
---
定义：当前发起请求的域与该请求指向的资源所在的域不一样。域指：协议 + 域名 + 端口号；  ​   eg：域名为aaa.cn的网站，它发起一个资源路径为aaa.cn/books/getBookInfo的 Ajax 请求，那么这个请求是同域的，因为资源路径的协议、域名以及端口号与当前域一致（例子中协议名默认为http，端口号默认为80）。但是，如果发起一个资源路径为bbb.com/pay/purchase的 Ajax 请求，那么这个请求就是跨域请求，因为域不一致，与此同时由于安全问题，这种请求会受到同源策略限制。
>- 服务端设置跨域请求相关设置：go-gin可以参考https://github.com/gin-contrib/cors
[跨域资源请求详解参考网站](https://www.jianshu.com/p/f880878c1398)

4.3 CSRF攻击
---
>- 浏览器缓存用户登陆过的所有cookie session信息
>- 服务端无法判定是否为用户浏览器发送的请求还是用户发起的请求
>
![跨域资源攻击](../pic/tcp-ip/CSRF攻击.jpg)
>- 攻击者利用用户缓存在浏览器的相关验证信息伪冒用户身份访问相关网站，篡改相关信息
>- ***防御方法：***

>>- (1).Cookie Hashing(所有表单都包含同一个伪随机值)：攻击者无法获取cookie则提交表单时无法包含此伪随机数；服务端进行hash随机数验证，验证通过做相应操作
>>- (2).验证码：用户提交需要填写随机字串
>>- (3).One-Time Tokens(不同的表单包含一个不同的伪随机值)

[CSRF攻击参考网站](https://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)

4.4 验签
---
>- 服务端之间传递参数需要信息验证：给出不同域名对应的ID&seceret
>- cli端请求server：请求参数中添加ID、随机序列、过期时间三项；参数按照字母升序排序组成para1=value&para2=value2；对组合字符串加密->encode base64 -> hash取[n:m]->作为参数signature=XX组合成请求参数之一；
>- server端根据请求参数中除signature外的其他参数按照相应规则进行处理获取signature值若与请求参数中signature不一致则验签失败
>- 拒绝未授权请求&冒充请求（除非appKey Secret 加密方法全部泄漏）

5 webSocket协议
---

5.1 简介
---
WebSocket是一种协议，与HTTP协议一样位于应用层，都是TCP/IP协议的子集。HTTP协议是单向通信协议，只有客户端发起HTTP请求，服务端才会返回数据。而WebSocket协议是双向通信协议，在建立连接之后，客户端和服务器都可以主动向对方发送或接受数据。WebSocket协议建立的前提需要借助HTTP协议，建立连接之后，持久连接的双向通信就与HTTP协议无关了。

WebSocket是应用层协议，是TCP/IP协议的子集，通过HTTP/1.1协议的101状态码进行握手。也就是说，WebSocket协议的建立需要先借助HTTP协议，在服务器返回101状态码之后，就可以进行websocket全双工双向通信了，就没有HTTP协议什么事情了。

WebSocket协议的目标是在一个独立的持久连接上提供全双工双向通信。客户端和服务器可以向对方主动发送和接受数据。在JS中创建WebSocket后，会有一个HTTP请求发向浏览器以发起请求。在取得服务器响应后，建立的连接会使用HTTP升级将HTTP协议转换为WebSocket协议。也就是说，使用标准的HTTP协议无法实现WebSocket，只有支持那些协议的专门浏览器才能正常工作。  

由于WebScoket使用了自定义协议，所以URL与HTTP协议略有不同。未加密的连接为ws://，而不是http://。加密的连接为wss://，而不是https://。

5.2 对比http
---

>- 前端通过浏览器与服务端建立tcp长连接，全双工通信
>- 通过http get建立连接 之后升级协议为webSocket
>- http协议请求型，惰性，客服端发起请求，服务端才回复请求

>>- ajax可以通过轮询实现后端推送消息给前端：一定时间后发起http请求，并得到response(不论是否有新的数据),导致后端服务器过忙，很多无用请求;***1个request对应一个response,轮询发起http请求***
>- 前端通过long poll实现服务端推送消息至前端：前端发起请求，服务端无数据时不发送response直到有新的数据发起response;***1个request对应一个response，阻塞在服务端response***
>- http流实现后端推送消息至前端：前端发起请求，服务端可以推送多次response;***一个request对应多个response***
>- ajax轮询 需要服务器有很快的处理速度和资源。（速度）
>- long poll 需要有很高的并发，也就是说同时接待客户的能力。（场地大小）
>- 传统轮询技术(Ajax短轮询)是客户端向服务器发起HTTP请求，无论数据是否更新，服务器都会传输数据。一个request对应一个response。  
服务器推送技术(Ajax长轮询)是短轮询的变种，是客户端向服务器发起HTTP请求，只有等待数据更新后才会传输数据，否则服务器保持连接状态。接着发起下一次HTTP请求，一个request对应一个response。  
服务器推送技术(HTTP流)，在客户端只发起一次HTTP请求，服务器保持连接状态，在数据更新之后，服务器会传输数据，否则保持连接状态。此时一个requset对应多个response。
无论是短轮询、长轮询，还是HTTP流，相同点在于都需要客户端先发起HTTP请求。

5.3 示例
---
***Js webSocketAPI***

```// 打开WebSocket, 传递的参数url没有同源策略的限制。
let websocket = new WebSocket(url)

// 监听open事件，在成功建立websocket时向url发送纯文本字符串数据(如果是对象则必须序列化处理)。
websocket.onopen = () => {
  if (websocket.readyState === WebSocket.OPEN) {
    websocket.send('hello world')
  }
}

// 监听message事件，在服务器响应时接受数据。返回的数据存储在事件对象中。
websocket.onmessage = e => {
  let data = e.data
  console.log(data)
}

// 监听error事件，在发生错误时触发，连接不能持续。
websocket.onerror = () => {
  console.log('websocket connecting error!!')
}

// 监听close事件，在连接关闭时触发。只有close事件的事件对象拥有额外的信息。可以通过这些信息来查看关闭状态
websocket.onclose = e => {
  let clean = e.wasClean // 是否已经关闭
  let code = e.code // 服务器返回的数值状态码。
  let reason = e.reason //服务器返回的消息。
}
```

![webSocekt请求](../pic/webSocket客户端请求.png)  
[看完让你彻底搞懂Websocket原理](https://www.cnblogs.com/fuqiang88/p/5956363.html)  
[webSocket协议](https://blog.csdn.net/qq_34803821/article/details/84657721)  
[webSocket协议解析](https://www.cnblogs.com/unclekeith/p/8087182.html)  
[golang中的webSocket实现](https://blog.csdn.net/xuduorui/article/details/76464576)  
[golang官方webSocket实现](https://godoc.org/golang.org/x/net/websocket#Codec)  
// 基于转为hijack（cli端负责关闭conn）的conn新建webSocket conn  
[golang webSocket github](https://github.com/gorilla/websocket)
[websocket介绍](https://juejin.im/post/5aae10adf265da239c7b2443)

6 GRPC
===

6.1 RPC
---

RPC 的主要功能目标是让构建分布式计算（应用）更容易，在提供强大的远程调用能力时不损失本地调用的语义简洁性。跨应用层&传输层；编解码、网络传输、同步||异步
>- GRPC基于http/2.0
>- server：实现相应的方法，并注册（default mutex生成handler），listen，accept
>- client: dial或者建立connect调用server的grpc方法
>同步&&异步（是否需要返回值）
>- cli&server单个连接(tcp conn)VS多个连接，单个连接单个缓冲区若数据量足够多缓冲区不够用，不能充分发挥网络优势；基于http/2的grpc单个连接对req请求的处理为并发处理，多个连接并无特殊优势；  
>- message queue为异步调用，sender向queue节点发送消息，consumer从queue节点获取消息并处理，不适合应用于需要返回值的同步请求中；消息暂存压力在queue节点
**参考：**  
[***RPC入门总结（一）RPC定义和原理***](https://blog.csdn.net/kingcat666/article/details/78577079)
[***深入浅出RPC-浅出篇***](https://blog.csdn.net/mindfloating/article/details/39473807)    
[***深入浅出RPC-深入篇***](https://blog.csdn.net/mindfloating/article/details/39474123)
[什么是RPC框架](https://blog.csdn.net/b1303110335/article/details/79557292)  
[分布式结构核心RPC原理](https://blog.csdn.net/dongnaosenlu/article/details/76998419)  
[RPC入门总结（一）RPC定义和原理](https://blog.csdn.net/kingcat666/article/details/78577079)  
[简介http1.0-grpc](https://pingcap.com/blog-cn/grpc/)  

6.2 net/rpc
---
golang远程rpc包
>- 可以基于tcp, udp或者http协议
>- 需要注册: 
>
```
// Server represents an RPC Server.
type Server struct {
	serviceMap sync.Map   // map[string]*service 存放注册的service
	reqLock    sync.Mutex // protects freeReq
	freeReq    *Request
	respLock   sync.Mutex // protects freeResp
	freeResp   *Response
}
type service struct {
	name   string                 // name of service
	rcvr   reflect.Value          // receiver of methods for the service
	typ    reflect.Type           // type of the receiver
	method map[string]*methodType // registered methods
}
```
>- serverConn: 可用于tcp
>
```
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),
		enc:    gob.NewEncoder(buf),
		encBuf: buf,
	}
	// 从连接conn中读取req对应的service.method arg res等，根据service.method从注册map中取对应的函数，执行函数调用
	server.ServeCodec(srv)
}
func (server *Server) Accept(lis net.Listener) {
	for {
		conn, err := lis.Accept()
		if err != nil {
			log.Print("rpc.Serve: accept:", err.Error())
			return
		}
		go server.ServeConn(conn)
	}
}
// ServeHTTP implements an http.Handler that answers RPC requests.
func (server *Server) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if req.Method != "CONNECT" {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.WriteHeader(http.StatusMethodNotAllowed)
		io.WriteString(w, "405 must CONNECT\n")
		return
	}
	conn, _, err := w.(http.Hijacker).Hijack()
	if err != nil {
		log.Print("rpc hijacking ", req.RemoteAddr, ": ", err.Error())
		return
	}
	io.WriteString(conn, "HTTP/1.0 "+connected+"\n\n")
	server.ServeConn(conn)
}
// 注册http handler
// HandleHTTP registers an HTTP handler for RPC messages on rpcPath,
// and a debugging handler on debugPath.
// It is still necessary to invoke http.Serve(), typically in a go statement.
func (server *Server) HandleHTTP(rpcPath, debugPath string) {
	http.Handle(rpcPath, server)
	http.Handle(debugPath, debugHTTP{server})
}
// HandleHTTP registers an HTTP handler for RPC messages to DefaultServer
// on DefaultRPCPath and a debugging handler on DefaultDebugPath.
// It is still necessary to invoke http.Serve(), typically in a go statement.
func HandleHTTP() {
	DefaultServer.HandleHTTP(DefaultRPCPath, DefaultDebugPath)
}
```

6.3 GRPC
---
适用于分布式系统、前后端、手机应用；基于http2.0的传输的双向流和完全集成的可插拔身份验证;
>- server: 类似net/rpc
>
```
// Server is a gRPC server to serve RPC requests.
type Server struct {
	opts options
	mu     sync.Mutex // guards following
	lis    map[net.Listener]bool
	conns  map[io.Closer]bool
	serve  bool
	drain  bool
	cv     *sync.Cond          // signaled when connections close for GracefulStop
	m      map[string]*service // service name -> service info
	events trace.EventLog
	quit               chan struct{}
	done               chan struct{}
	quitOnce           sync.Once
	doneOnce           sync.Once
	channelzRemoveOnce sync.Once
	serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop
	channelzID int64 // channelz unique identification number
	czData     *channelzData
}
```
>- protocol buffer协议
>- 负载均衡
>- 超时处理
>- 流量控制
>- tracing
>- 健康检查
>- 认证
>
>```
/// proto文件
syntax = "proto3";
package bistream;
service MapStream {
rpc GetMap(stream ReqMap)returns(stream ResMap);
}

message ReqMap {
    int32 seqNum = 1;
}
message ResMap {
    string resMessage = 2;
}
```
>- protoc -I packageOfProto/ xx/xx.proto --go_out=plugins=grpc:packageName
>- client端实现
>
>```
// client
package client
import (
	"google.golang.org/grpc"
	"context"
	"time"
	"work/grpc-eg/bistream"
	"github.com/prometheus/common/log"
	"io"
)
var streamConn *grpc.ClientConn
func init(){
	ctx := context.Background()
	newCtx, _ := context.WithTimeout(ctx, 1*time.Second)
	conn, err := grpc.DialContext(newCtx, "localhost:8080", grpc.WithInsecure())
	if err != nil {
		log.Errorf("cli err is: %+v", err)
		panic(err)
	}
	streamConn = conn
}
func GetStream(ctx context.Context){
	cliConn := streamConn
	cli := bistream.NewMapStreamClient(cliConn)
	cliStream, err := cli.GetMap(ctx)
	if err != nil {
		log.Errorf("failed to get map: %+v", err)
		return
	}
	len := 10
	notify := make(chan struct{})
	for i:=0;i<len;i++{
		if err := cliStream.Send(&bistream.ReqMap{
			SeqNum:int32(i+1),
		}); err != nil {
			log.Errorf("failed to send seqNum: %+v", i+1)
		}else{
			log.Infof("sent seqNum: %+v", i+1)
		}
	}
	// cli关闭发送端
	cliStream.CloseSend()
	go func(){
		for {
			res, err := cliStream.Recv()
			if err == io.EOF {
				log.Infof("end cli from res")
				close(notify)
				break
			}
			if err != nil {
				log.Errorf("failed to receive res")
				continue
			}
			if res == nil {
				log.Infof("res is nil")
				continue
			}
			log.Infof("res is: %+v", res.ResMessage)
		}
	}()
	<-notify
}
```
>- server端实现
>
>```
package streameg
import (
	"work/grpc-eg/bistream"
	"io"
	"github.com/prometheus/common/log"
	"time"
)
type StreamEg struct {
	Description map[int32]string
}
func GetStreamEg()*StreamEg{
	return &StreamEg{
		Description:map[int32]string{
			1:"num1",
			2:"num2",
			3:"num3",
			4:"num4",
			5:"num5",
			6:"num6",
			7:"num7",
			8:"num8",
		},
	}
}
// GetMap grpc流
func (s *StreamEg)GetMap(reqStream bistream.MapStream_GetMapServer) error{
	for {
		// 必须并行处理 防止cli发送完req关闭conn，但server端未来得及发送res
		req, err := reqStream.Recv()
		if err == io.EOF {
			log.Infof("end of res from cli")
			return nil
		}
		if err != nil {
			log.Errorf("server err is: %+v", err)
			return err
		}
		time.Sleep(200*time.Millisecond)
		if req == nil {
			if err := reqStream.Send(nil); err != nil {
				return err
			}
		}else{
			if err := reqStream.Send(&bistream.ResMap{
				ResMessage:s.Description[req.SeqNum],
			}); err != nil {
				return err
			}
		}
	}
	return nil
}
```
>- main函数
>
>```
package main
import (
	"work/grpc-eg/server"
	"os"
	"os/signal"
	"syscall"
)
func main(){
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		server.RegistryStreamEg()
	}()
<-sigChan
}
```

[grpc proxy&bridge](http://cwiki.guazi.com/pages/viewpage.action?pageId=67820086)  
[grpc stream介绍](https://segmentfault.com/a/1190000016503114)  
[grpc stream实例](https://colobu.com/2017/04/06/dive-into-gRPC-streaming/)  
[grpc 官方案例](https://www.grpc.io/docs/tutorials/)

7 并发编程 select I/O复用
===
8 前后端通信
===

>- 前端作为通信的cli端，若http协议只能通过请求获取response
>- 服务端之间通信: 服务端均可作为通信的server端存在
> 
>>- 服务端A B
>>- B推送消息给A，A作为server端，B作为cli端调用接口(http rpc)发送消息给A； B主动发起请求，获取response
>>- A需要从B获取消息: B作为server端提供响应接口，A作为cli端调用B(http rpc)； A主动发起请求获取response

9 其他
===
>- IP地址是逻辑地址用来标识网络中的一个主机，ip地址唯一，计算机间通信通过ip
>- IP分公有ip及私有ip，公有ip全球可以访问，私有地址(以192.168.开头)不能直接在internet网络中应用
>- IP地址=网络地址+主机地址
>- 网络地址就是判断网络中的计算机在不在同一网络,在就能通,不在就不能通
>- 网络地址: IP地址&子网掩码，二进制计算获取网络地址
>- 域名internet中计算机或者计算机组的名称,域名便于人类记忆
>- DNS(Domain name server): 域名服务器，存有域名跟ip的对应表，可以将域名解析为ip；
>- chrom浏览器输入google.com：1 请求dns服务器解析google.com获取ip地址 2 与该ip建立连接
>- 用网线直接连接的计算机或是通过HUB（集线器）或普通交换机间接的计算机之间要能够相互通,计算机必须要在同一网络；即网络地址必须相同
>- 网络A的IP地址范围为“192.168.1.1~192. 168.1.254”，网络B的IP地址范围为“192.168.2.1~192.168.2.254”，两者的子网掩码为255.255.255.0
>- 网络A与B在不同的网络中不能直接通信，两者必须通过网关通信：如果网络A中的主机发现数据包的目的主机不在本地网络中，就把数据包转发给它自己的网关，再由网关转发给网络B的网关，网络B的网关再转发给网络B的某个主机
>- 网关的IP地址是具有路由功能的设备的IP地址： 路由器、启用了路由协议的服务器（实质上相当于一台路由器）、代理服务器（也相当于一台路由器）。
>- 一台主机可以有多个网关,主机找不到可用网关则使用默认网关
>- 自动设置就是利用DHCP（Dynamic Host Configuration Protocol, 动态主机配置协议）服务器来自动给网络中的电脑分配IP地址、子网掩码和默认网关 。这样做的好处是一旦网络的默认网关发生了变化时，只要更改了DHCP服务器中默认网关的设置，那么网络中所有的电脑均获得了新的默认网关的IP地址。这种方法适用于网络规模较大、TCP/IP参数有可能变动的网络。另外一种自动获得网关的办法是通过安装代理服务器软件（如MS Proxy）的客户端程序来自动获得，其原理和方法和DHCP有相似之处。  

[IP地址，子网掩码，默认网关，DNS服务器详解](https://www.cnblogs.com/JuneWang/p/3917697.html)

10 网络相关tools
===
10.1 nginx
---
[nginx简介](https://blog.csdn.net/tjiyu/article/details/53027619)
[nginx配置](https://blog.csdn.net/zzq900503/article/details/70049348)
[nginx介绍](https://juejin.im/post/5821c24e570c350060bef4c3)
10.2 代理服务器
---
>- 正向代理（翻墙 vpn）：cli请求server经过代理服务器，proxy作为cli请求server,server不知道真正的cli是谁；一对一，多对一(多个cli一个代理一个server)
![正向代理](../pic/tcp-ip/正向代理.png)
>- 反向代理：proxy作为server，cli请求proxy， proxy转发至不同的server，负载均衡的实现方式；cli不知道真正的server；也存在多对多
![反向代理](../pic/tcp-ip/反向代理.png)
>- 正向代理需要你主动设置代理服务器ip或者域名进行访问，由设置的服务器ip或者域名去获取访问内容并返回；而反向代理不需要你做任何设置，直接访问服务器真实ip或者域名，但是服务器内部会自动根据访问内容进行跳转及内容返回，你不知道它最终访问的是哪些机器
>- 正向代理是代理客户端，为客户端收发请求，使真实客户端对服务器不可见；而反向代理是代理服务器端，为服务器收发请求，使真实服务器对客户端不可见。

![正向代理反向代理](../pic/tcp-ip/正向代理反向代理.png)
>- 正向代理中，proxy和client同属一个LAN，对server透明； 反向代理中，proxy和server同属一个LAN，对client透明。 实际上proxy在两种代理中做的事都是代为收发请求和响应，不过从结构上来看正好左右互换了下，所以把前者那种代理方式叫做正向代理，后者叫做反向代理
>正向代理的典型用途是为在防火墙内的局域网客户端提供访问Internet的途径。正向代理还可以使用缓冲特性减少网络使用率。反向代理的典型用途是将 防火墙后面的服务器提供给Internet用户访问。反向代理还可以为后端的多台服务器提供负载平衡，或为后端较慢的服务器提供缓冲服务

[正向代理&反向代理](https://www.jianshu.com/p/208c02c9dd1d)
>- go实现http反向代理
>- 提供一个服务s,其handler对满足一定url的请求，做代理转发，将请求转发至真实的ip(服务上)获得结果，将结果返回至最初的请求，即代理结果为handler结果
>
```
package main
import (
	"context"
	"net/http"
	"net/http/httputil"
	"net/url"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"golang.guazi-corp.com/tools/alice/logx"
	logx_metrics "golang.guazi-corp.com/tools/alice/logx/hook/metrics"
	"golang.guazi-corp.com/tools/alice/logx/provider/zap"
	"golang.guazi-corp.com/tools/alice/metrics"
	"golang.guazi-corp.com/tools/alice/metrics/provider"
	"golang.guazi-corp.com/tools/alice/metrics_providers/provider/prometheus"
	"golang.guazi-corp.com/tools/alice/recover/sentry"
)
var (
	wg sync.WaitGroup
	//countonce sync.Once
	//queue     chan int
)
type handle struct {
}
func (h *handle) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path == "/hello" {
		//count := counter()
		//if count > 30 && count < 60 {
		//	w.WriteHeader(400)
		//	return
		//}
		w.WriteHeader(200)
		w.Write([]byte("ok"))
		return
	}
	if r.URL.Path == "/check" {
		logx.Info(context.Background(), "check", nil)
		w.WriteHeader(200)
		w.Write([]byte("ok"))
		return
	}
	toHost := r.URL.Query().Get("to")
	if toHost == "" {
		w.WriteHeader(400)
		w.Write([]byte("no target addr"))
		return
	}
	hSplit := strings.Split(toHost, ":")
	if len(hSplit) > 2 {
		w.WriteHeader(400)
		w.Write([]byte("error to host"))
		return
	} else if len(hSplit) == 2 {
		if hSplit[1] != "6060" {
			w.WriteHeader(400)
			w.Write([]byte("error to host port, only 6060 except"))
			return
		}
	} else {
		toHost += ":6060"
	}
	logx.Info(context.Background(), "to host: "+toHost, nil)
	u, _ := url.Parse("http://" + toHost + r.URL.Path + "/?" + r.URL.RawQuery)
	u.Path = ""
	// proxy代理，将现有请求转发至u上，并获取结果
	proxy := httputil.NewSingleHostReverseProxy(u)
	if err := metrics.Counter("pprof_proxy_dst", 1, map[string]string{
		"to":   toHost,
		"path": r.URL.Path,
	}); err != nil {
		logx.Warn(context.Background(), "add metrics error: "+err.Error(), nil)
	}
	proxy.ServeHTTP(w, r)
}
func main() {
	//prepare log
	logx.InitNamespaceProject(os.Getenv("GROUP"), os.Getenv("PROJECT"))
	logx.AddProvider(zap.NewProvider())
	logx.AddHook(logx_metrics.NewHook(nil))
	logx.SetPanicHandler(sentry.Handler)
	//prepare metrics
	err := provider.Register(prometheus.NewProvider().StartServer(9998))
	if err != nil {
		panic(err)
	}
	logx.Info(context.Background(), "start server ......", nil)
	srv := &http.Server{
		Addr:    ":80",
		Handler: &handle{},
	}
	srv.SetKeepAlivesEnabled(false)
	wg.Add(1)
	go func() {
		defer wg.Done()
		srv.ListenAndServe()
	}()
	sigChan := make(chan os.Signal, 1)
	defer close(sigChan)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	<-sigChan
	srv.Close()
	wg.Wait()
}
```

10.3 cli与server交互
---
>- redis&mysql client与server交互是否为tcp：不同机器上为tcp socket，同一机器为unix domain socket

11 DNS详解
===
>- 根域名提供TLD(top level domain)的名称+ip(避免循环查询，名称也是一个域名)，顶级域名包括:com net
>- 顶级域名提供SLD(second level domain）二级域名的地址及服务器；
>- 二级域名可以查询到对应的主机
>- www.baid.com. : com为顶级域名由公司专门管理，baidu为二级域名，www为主机名，递归查询
>- 每台主机（集群保留根域名的ip地址）
>- 内置根域名服务ip: 根据内置的根域名服务器IP地址，DNS服务器向所有这些IP地址发出查询请求，询问math.stackexchange.com的顶级域名服务器com.的NS记录。最先回复的根域名服务器将被缓存，以后只向这台服务器发请求
[根域名详解](https://www.ruanyifeng.com/blog/2018/05/root-domain.html)
[DNS域名解析](https://www.ruanyifeng.com/blog/2016/06/dns.html)
>- CDN负载均衡需要分布在各地的缓存服务器、内容分发服务器、核心web服务器、全局负载均衡（Global Server Load Balance, GSLB）
>- 用户请求到GSLB(请求域名的ip地址配置为GSLB相当于二级域名，DNS查询)、GSLB返回缓存服务器（GSLB与各服务器保持通讯，查询服务器状态、负载均衡等等）、用户请求缓存服务器若缓存未命中从核心服务器获取；内容分发负责更新删除缓存等
[DNS负载均衡，CDN分发](https://www.jianshu.com/p/05eb8365ab5f)



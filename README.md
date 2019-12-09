# mdns
base on https://opensource.apple.com/tarballs/mDNSResponder/

```
mDNSResponder是苹果的Bonjour项目的一部分。 Bonjour是法语“你好”的意思。
Bonjour软件源自正IETF零配置网络工作。零配置工作有三个要求：
1.分配IP地址（即使没有分配DHCP服务器的IP地址）
2.提供名称到地址的转换（即使没有DNS服务器）
3.在网络上发现相关的网络服务（同样没有其他的基础协议支持）

也就是说不需要以上DNS等相关的服务的支持，通过零配置直接完成这些任务。厉害吧？
对于1，通过自分配的本地链接地址实现。
对于2，通过多播（mDNS）发送类似DNS的查询来满足。
对于3，通过DNS Service Dicsovery（DNS-SD）满足。
自分配的本地链接地址自1998年, 在Windows '98和Mac OS 8.5中shouci出现。当然在其他平台上也支持。
```
# mDNSResponder
```
mDNSResponder项目实现了上面的2和3。其中，m是Multicast缩写。即使用户没有设置传统的DNS服务器，用户也能够使用服务名称(比如www.sohu.com)，而不是点分十进制IP地址(suhu的IP是122.13.86.100)来标识主机。它还为用户无需事先了解服务的细节，就可以发现网络上正在播发什么服务，也无需配置机器。

选择名称“ mDNS”，是因为该协议类似于常规DNS。不过，mDNS和DNS的主要区别在于：
mDNS查询是通过多播发送到所有本地主机，而不是通过单播发送到特定的已知服务器。本地链接上的每个主机都运行一个mDNSResponder，该DNS响应器不断监听那些多播查询，如果mDNSResponder接收到自己关注的查询，则做出响应。

mDNS协议使用与单播DNS相同的数据包格式，相同的名称结构和相同的DNS记录类型。这部分的主要区别在于：
1.查询被发送到不同的UDP端口（5353，而不是53），并通过多播发送到地址224.0.0.251。
2.所有“ mDNS”名称都以“ .local”结尾。当用户键入“ yourcomputer.local”时。进入他们的Web浏览器时，就会显示“ .local”。.local告诉主机OS应该使用本地多播查找该名称，而不是通过将该名称发送给互联网的DNS服务。这有助于用户区分特定名称是广域网的（例如“ www.apple.com”）还是仅仅在本地（例如“ yourcomputer.local”）。
mDNSResponder源代码
    +------------------+
    |   Application    |
    +------------------+
    |    mDNS Core     |
    +------------------+
    | Platform Support |
    +------------------+

由于Apple认为公开这部分代码比较好，可以供其他开发人员使用。这部分代码可以兼容不同种类的OS。
典型的mDNS程序包含三个组件：

“ mDNS Core”层在所有应用程序和所有操作系统都是相同的。
“Platform Support”层提供了特定于每个平台的必要支持例程，例如，调用哪个例程以发送UDP数据包，调用哪个例程以加入多播组等等。
“Application”层可以执行特定应用程序想要执行的任何操作。 它调用“ mDNS Core”层提供的例程来执行所需的功能:
*发布服务，
*浏览特定服务类型的命名实例
*将命名实例解析为特定的IP地址和端口号，
```

# 移植
```
Apple当前为Mac OS 9，Mac OS X，Microsoft Windows，VxWorks以及POSIX平台（例如Linux，Solaris，FreeBSD等）提供“平台支持”层。
注意，OS X已经提供了mDNS 对应的系统调用，因此不需要移植这份代码。

如果每个应用程序将mDNSResponder代码链接到应用内部中，那么最终将遇到如下图所示的情况：

  +------------------+    +------------------+    +------------------+
  |   Application 1  |    |   Application 2  |    |   Application 3  |
  +------------------+    +------------------+    +------------------+
  |    mDNS Core     |    |    mDNS Core     |    |    mDNS Core     |
  +------------------+    +------------------+    +------------------+
  | Platform Support |    | Platform Support |    | Platform Support |
 
这样效率不高。其实OS X提供了一种通用的系统服务，可以通过“ /usr/include/dns_sd.h”API对其进行访问就ok了。所以实际上每个应用都访问daemon程序，只用一份就好了：

                                   -------------------
                                  /                   \
  +---------+    +------------------+    +---------+   \  +---------+
  |  App 1  |<-->|    daemon.c      |<-->|  App 2  |    ->|  App 3  |
  +---------+    +------------------+    +---------+      +---------+
                 |    mDNS Core     |
                 +------------------+
                 | Platform Support |
                 +------------------+
如果希望在内部使用，不用第三方的服务，可以参照上图来移植。在 mDNSPosix目录有具体的说明。
```

# 在旧版本的编译器移植
```
有的编译器太旧，不支持双斜杠的注释：//，那么就要手动改了。比如：
打开BBEdit：

打开 “Find” 对话选择 “Use Grep”
搜索  : ([^:])//(.*)
替换: \1/*\2 */
将mDNSResponder代码拖到多文件搜索界面
选择"Replace All" 替换所有。

对于面向更多命令行，可以进入代码目录，执行下面的命令：
find mDNSResponder ( -name *.c* -or -name *.h ) -exec sed -i .orig -e ‘s,^//(.),/\1 /,’ -e '//*/!s,([^:])//(.),\1/*\2 */,’ {} ;
```
# mDNSPosix
```
mDNSPosix是Apple的mDNS到Posix平台的移植。
目录：
## mDNSCore-包含核心mDNS代码的目录。该代码是用纯ANSI C编写的，并被证明具有很高的可移植性。每个平台都需要此核心协议引擎代码。
## mDNSShared-一个包含有用代码的目录，该代码不是主协议引擎本身的核心。
## mDNSPosix-特定于Posix平台的文件：Linux，Solaris，FreeBSD，NetBSD，OpenBSD等。
## Clients -客户端代码示例，显示如何将API用于守护程序提供的服务。
```
# 编译代码
```
在mDNSPosix目录，make对应的系统名称：
make os=linux
之后在mDNSPosix/build/prod目录会生成下面的文件：
.
├── libdns_sd.so
├── libnss_mdns-0.2.so
├── mDNSClientPosix
├── mdnsd
├── mDNSIdentify
├── mDNSNetMonitor
├── mDNSProxyResponderPosix
└── mDNSResponderPosix
通用平台使用（例如在台式计算机上）：
-mdnsd
-libmdns
-nss_mdns（有关nss_mdns，请参见nss_ReadMe.txt）
专用平台使用：
-mDNSClientPosix
-mDNSResponderPosix
-mDNSProxyResponderPosix
测试工具：
-dns-sd命令行工具（来自“Client”文件夹）
-mDNSNetMonitor
-mDNSIdentify
```

# 运行说明
```
------------
                                                   +--------------------+
                                                   | Client Application |
   +----------------+                              +--------------------+
   |  uds_daemon.c  | <--- Unix Domain Socket ---> |      libmdns       |
   +----------------+                              +--------------------+
   |    mDNSCore    |
   +----------------+
   |  mDNSPosix.c   |
   +----------------+

mdnsd分为三个部分。

## mDNSCore是主要的协议引擎
## mDNSPosix.c提供了在Posix OS上的对应移植
## uds_daemon.c将Unix域套接字接口，导出到mDNSCore提供的服务

客户端应用程序与libmdns链接，libmdns实现dns_sd.h头文件中定义的功能，并实现IPC协议，该协议用于通过Unix Domain Socket接口与守护程序进行通信。

严格来说，nss_mdns只是mdnsd的另一个客户端，就像其他任何客户端一样与libmdns链接。只不过它在多播DNS的正常运行中起着核心作用，因此它与其他必要的系统支持组件一起编译安装。
```
# 小型嵌入式系统客户端
```
有时候需要在极小的系统下支持mdns，但是系统资源有限，这时候可以取消uds_daemon和libmdns层。直接调用core的接口。下面是一些简要的说明，如果没有这样的需求就直接跳过不用看了。

    +--------------------+
    | Client Application |
    +--------------------+
    |      mDNSCore      |
    +--------------------+
    |    mDNSPosix.c     |
    +--------------------+

不过，这样工作量会比较大。
1.应用程序调用mDNS_Init，后者调用平台（mDNSPlatformInit）。

mDNSPlatformInit获取接口列表（get_ifi_info），并向核心注册每个接口（mDNS_RegisterInterface）。它还为每个接口创建一个多播套接字（SetupSocket）。

3.然后，应用程序反复调用select（）来处理文件描述符事件。在每次调用select（）之前，应用程序都会调用mDNSPosixGetFDSet（）来给mDNSPosix.ca机会将其自己的文件描述符添加到集合中，然后在select（）返回之后， 调用mDNSPosixProcessFDSet（）来使 接收并处理数据包。

4.当核心需要发送UDP数据包时， 调用mDNSPlatformSendUDP。该例程查找与内核请求的源地址相对应的接口，并使用为该接口创建的UDP套接字发送数据报。如果套接字是流发送侧控制的，则丢弃数据包。

5.当SocketDataReady运行时，它使用复杂的例程“ recvfrom_flags”来接收数据包。
另外，如果决定在平台中使用线程，注意要实现mDNSPlatformLock（）和mDNSPlatformUnlock（）调用。
```
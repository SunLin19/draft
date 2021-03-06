
案例一

同一局域网一用户通过tcp协议连接不上我们的服务器，其它用户都是正常的，服务器端没有做任何限制，telnet服务器的端口也不通，从服务器端抓包来看已经收到用户的syn数据包，但是服务器端并没有响应这个数据包，发出ack包，也就没有完成tcp的三次握手，当然连接建立不起来，为什么会发生这种情况？
我们知道在tcp网络程序当中，当我们建立一个连接的情况下，主动断开一方会进入TIME- WAIT状态，而TIME- WAIT状态需要占用机器一个文件描述符，消耗服务器资源，为了提高TCP的性能，“RFC1323 – TCP Extensions for High Performance”提出了 一个机制（http://tools.ietf.org/html/rfc1323#page-29）来替代TIME- WAIT状态的功能。linux实现了它。这个机制通过记录来自每台主机的每个连接的分组时间戳来实现，要求来自同一主机的同一连接的分组所携带的时间戳要比之前记录的时间戳新，以便“防止回绕的序号PAWS机制“（http://tools.ietf.org/html/rfc1323#page-17）丢弃接收属于旧连接的延时分组。这依赖于来自每个主机的每个 TCP连接分组所携带的时间戳要单调递增才能实现。然而经过NAT的连接，其分组携带时间戳每个用户都不同的，也就是说同一个ip ，携带的时间戳不会单调递增。服务器端对同一个ip 过来的包的timestamp做一个验证，导致这些连接分组被认为是属于旧连接的延时分组而被丢弃。在linux上可以通过如下netstat -s|grep timestamp查看具体有多少被drop的包。解决办法就是
echo 0 > /proc/sys/net/ipv4/tcp_timestamps

##案例二

同一局域网的所有用户死活连接不上我们的服务器端（udp协议的服务器端），局域网没有任何限制，而且用户到部分其他常用的站点，部分能够打开，部分打不开。为什么呢？
以太网(Ethernet)数据帧的长度必须在46-1500字节之间,这是由以太网的物理特性决定的，这个1500字节被称为链路层的MTU(最大传输单元).但这并不是指链路层的长度被限制在1500字节,其实这这个MTU指的是链路层的数据区.并不包括链路层的首部和尾部的18个字节.所以,事实上,这个1500字节就是网络层IP数据报的长度限制.因为IP数据报的首部为20字节,所以IP数据报的数据区长度最大为1480字节.而这个1480字节就是用来放TCP传来的TCP报文段或UDP传来的UDP数据报的.又因为UDP数据报的首部8字节,所以UDP数据报的数据区最大长度为1472字节.这个1472字节就是我们可以使用的字节数。当我们发送的UDP数据大于1472的时候会怎样呢？这也就是说IP数据报大于1500字节,大于MTU.这个时候发送方IP层就需要分片(fragmentation).把数据报分成若干片,使每一片都小于MTU.而接收方IP层则需要进行数据报的重组.这样就会多做许多事情,而更严重的是,由于UDP的特性,当某一片数据传送中丢失时,收方便无法重组数据报.将导致丢弃整个UDP数据报。一些电信运营商为了封堵网络共享，在送的路由器上把mtu给改了，导致用户出现这种情况，这个时候，通过修改路由器的mtu进行问题解决，一般adsl的mtu为1492，如果无法确定的通过ping -f -l mtu数据区大小 网关得到mtu的值，再设置一个值就是了，基本可以完美解决这个问题。

##案例三

用户使用一个linux网关连接不上目标服务器，但是从linux网关服务器上telnet目标服务器端口是通的，抓包的结果如下：

从数据来看，三次握手是成功的，客户端也发送了数据给我们服务器端，服务器端也确认了该数据，这个时候应该服务器端返回给客户端才对，不返回也应该发出fin包结束这个连接，后面也看到都是客户端在重试，重新发数据包，到服务器端，服务器端也确认了这个数据，可以确定的是服务器端发给客户端的数据包过大，需要分片，从代理服务器到服务器端中间的某个网络设备配置异常或者因为从代理服务器到目标服务器的“路径”为了响应各种各样的事件（负载均衡、拥塞、断电等等）而被动态地修改——这可能导致路径最大传输单元在传输过程中发生改变——有时甚至是反复的改变。其结果是，在主机寻找新的可以安全工作的最大传输单元的同时，更多的分组被丢失掉了，导致连接失败。为了确定是中间网络设备设置异常，还是其它原因引起的，找一台正常的机器，抓包分析结果如下：

从结构来看，服务器端返回的数据包大小确实是过大，超过一个分组，数据包需要切片，因而可以确定是中间网络设备设置异常引起的。而这个代理服务器既有udp协议又有tcp协议，如果更改通过如下命令
ip link set eth0 mtu 1400
设置mtu的话，有可能出现案例2的问题，导致连接异常，我们知道tcp连接的三次握手过程当中可以协商一个mss（Maximum Segment Size，最大分段大小），连接建立后，以两端最小mss进行数据传输，充分利用这样一个特性就能解决这个连接异常问题，iptables能够修改tcp的包头，利用iptables的这个特性，轻松的解决这个问题，具体设置命令如下：
iptables -t mangle -A POSTROUTING -o eth1 -p -m tcp –tcp-flags SYN,RST SYN -j TCPMSS –set-mss 1400
执行完上述命令后，客户端顺利的连接上服务器端，抓包的结果如下：


##案例四
一用户反馈到我们服务器连接非常缓慢，叫用户ping到服务器延迟很低，服务器也没有任何负载，服务器很正常，其它用户连接没有任何问题，而且用户在同网段装的一个虚拟机，虚拟机采用桥接方式上网，虚拟机里面访问没有任何问题，为什么会出现这样一个情况呢？
TCP协议在工作时，如果发送端的TCP协议软件每传输一个数据分组后，必须等待接收端的确认才能够发送下一个分组，由于网络传输的时延，将有大量时间被用于等待确认，导致传输效率低下，为此TCP在进行数据传输时使用了滑动窗口机制，而这个滑动窗口是动态的增长的，也就是说我们常见的采用下载工具下载的时候，看到下载的速度是慢慢的增长，而不是一下子把整个带宽给撑满了。同时tcp还有流控，拥塞控制，出错重发等机制，当在窗口增大的过程中，由于中间某个路由设备出现异常丢掉tcp的分组，导致不能连接当中的窗口不能确认导致，一直持续的小窗口，从而导致整个速率就非常之慢，因此我们在服务器端关闭滑动窗口机制，采用固定的窗口大小（linux默认为64K）进行数据传输，配置方法如下：
echo 0 > /proc/sys/net/ipv4/tcp_window_scaling
linux默认窗口为64K，窗口的大小也可调，通过如下参数进行窗口大小的调整
echo 4096 65536 65536 >/proc/sys/net/ipv4/tcp_rmem
echo 4096 65536 65536 >/proc/sys/net/ipv4/tcp_wmem

##案例五
我们的一个同事的问题，非用户的问题，他的两台服务器之间传输数据，a服务器到b服务器拉取数据，速度可以到达1M每秒，而b服务器到a服务器拉取数据只有可怜的50K每秒，说明一下，这个两个机器都是光纤网络环境，中间的带宽都是正常的，没有跑满。为什么存在这个情况？
首先我就抓包进行数据分析，抓包出来的内容如下：


从上面看到tcp的窗口很小只有14，而且不能动态的往上增长，想到上面的那个案例，调整tcp的窗口使用固定的窗口大小
echo 0 > /proc/sys/net/ipv4/tcp_window_scaling
再抓包看到的内容如下：

而从实际的下载速度来看，比前面的不指定窗口的大小要快那么一丁点，还是没有达到我们想要的要求，具体数据如下

而从我们二次抓包的数据看到同一分组多次传输，具体如下图

这个时候想到的中间网络质量有问题，而从另外一台到这台服务器拉取数据正常，这个时候就想确认一下两边是否是对称路由，也就是走的路由的路是否一致，排查网络设备问题，最后用tracert确认路径是一致的也排除中间网络设备的问题，这时候可以确认这台机器有问题，我ifconfig运行了一下，看到这台机器进来这一侧，有错误包，而且还丢包。具体图如下

最后运行一下ethtool eth1看到的内容如下

网卡变成100M，且跟交换机的自动协商功能也关闭了，这个时候通过命令 ethtool -s eth1 speed 1000 autoneg on把网卡设置为1000M并打开自动协商，再测试的速度如下


速度恢复正常，多次ifconfig也没有发现网卡有错误包跟丢包了。
案例六
一个做游戏开发的朋友，他们的一个用户，前几天用户登录游戏还好好的，今天更新一个补丁出去，用户登录不了游戏，而游戏并没有更新网络连接这个模块，他打日志，发现socket连接读取数据出错，报内核缓冲器已满，从打日志的数据来看，客户端接收到的数据不完整，导致连接不上游戏。这是为什么呢？
由于我没有用户的一个服务器端环境，但是我可以提供一个代理给用户，可以抓取通过代理服务器中转到游戏服务器的数据包，我提供一个vpn给用户，用户通过我的vpn连接游戏服务器，在我vpn服务器上进行抓包分析，从抓包的情况来看，具体内容如下

从上面数据来看，连接过程已经传输了好几K的数据，中间也没有丢失分组，而连接的最后被发送了一个rest包，正是因为这个rest包导致连接异常，客户端看到的数据包不完整，这个rest数据包，我的代理服务器不可能发送，这个时候需要确认是否是客户端发送的，还有一种情况是中间网络设备发送的，如果是中间网络设备发送的，这个问题基本没有办法解决，而这个时候需要到用户机器上进行抓包分析，装一个wireshark，进行抓包一看，看到这个rest包有客户端发出的，而对方的游戏，我问了一下，不可能发出这么底层的数据包，这个时候最容易发送这个数据包的就是一些安全软件，这个用户机器上我看了一下装了2个安全软件，一个是某某杀毒软件，一个是某某安全软件，先关闭杀毒软件，还是无法连接，最后关闭某某安全软件，就彻底解决了，用户也连接上游戏服务器了。看来软件冲突也是问题之一啊！
以上就是我在最近工作当中遇到的一下通讯协议的问题排查过程以及解决方法，中间牵涉到一部分内核的调整，在网络上看到很多人未经考虑直接就拿着别人的一些参数直接用上了，而出了问题也不知道怎么去排查。


http://blog.netzhou.net/?p=180

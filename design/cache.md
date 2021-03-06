缓存机制在我们的实际研发工作中，被极其广泛地应用，通过这些缓存机制来提升系统交互的效率。简单的总结来说，就是在两个环节或者系统之间，会引入一个cache/buffer做为提升整体效率的角色。

而 有趣的是，这种缓存机制令人惊奇并且优美的遵循着“几何分形”的规律，也就是几何分形学中的“自相似性”：从整体上看遵循某种组成规律或者特性，同时从每 一个局部看，仍然遵循某种组成的规律或者特性。我们的这些系统，从整体上看遵循了缓存机制，每一个组成的局部也遵循缓存机制。

等同类比的一个概念，我们常常说的“空间换时间”，牺牲一部分空间代价，来换取整体效率的提升。

例如A和B两者之间的数据交换，为了提升整体的效率，引入角色C，而C被用于当做热点数据的存储，或者是某种中间处理的机制。

我们先从web前端层面开始，看看有哪些比较关键的缓存机制？它们又是怎样协调工作的呢？

一、前端Cache机制

1. 域名转为IP地址（域名服务器DNS缓存）

我们知道域名其实只是一个别名，真实的服务器请求地址，实际上是一个IP地址。获得IP地址的方式，就是查询DNS映射表。虽然这是一个非常简单的查询， 但如果每次用户访问一个url都去查询DNS一次，未免显得太频繁，会产生一个可怕的访问量级。DNS服务器会告诉你，你别老是经常过来，万一我挂了，我们就无法愉快地玩耍了。

各个浏览器的缓存时间，会有一定的差别。例如，在chrome浏览器中查看dns的缓存时间的方式是：chrome://net-internals/#dns。

浏览器一般会在本地会建立一个DNS缓存，在一段比较长的时间里，都是使用本地的缓存映射。例如，在Win7系统的cmd里，可以通过“ipconfig /flushdns ”的方式来立刻刷新本地DNS。

优点：域名映射为IP非常快。

成本：消耗一定的浏览器空间来存储映射关系

2. 访问服务器，获取静态内容（地理位置分布式服务CDN）

可能有人会觉得，这个CDN不是缓存。其实，CDN的原理就是将离你很远的东西，放在离你很近的地方，通过这种方式提高用户的访问速度。从这个角 度，它也可以理解为牺牲空间成本换取了时间，本质上也是一种特殊的中间cache。腾讯、阿里等这些大的一线互联网公司一般倾向于自己建立CDN系统，中 小型企业也经常使用第三方的CDN服务。

优点：解决用户离服务器太远的时候，网络路由中跳来跳去的严重耗时。

成本：全国各地部署多套静态存储服务，管理成本比较高，发布新文件的时候，需要等待全国节点的更新等。

3. 浏览器本地缓存（无网络交互类型）

在前端优化原则中，其中一条就是尽量消灭请求，以达到降低服务器压力和提升用户体验的效果。静态文件，例如Js、html、css、图片等内容，很多内容可以1次请求，然后未来就直接访问本地，不再请求web服务器。

常用的实现方法是通过Http协议头中的expire和max-age来控制，这两者的使用方法和区别，我这里就不赘叙了。还有一种HTML5中很热的方式，则是localStorage，尤其在移动端也被做为一个强大的缓存，甚至当做一种本地存储来广泛使用。

优点：减少网络传输，加快页面内容展示速度，提升用户体验。

成本：占用客户端的部分内存和磁盘，影响实时性。

4. 浏览器和web服务协议缓存（有网络交互类型）

浏览器的本地缓存是存在过期时间的，一旦过期，就必须重新向服务器请求。这个时候，会有两种情形：

服务器的文件或者内容没有更新，可以继续使用浏览器本地缓存。

服务器的文件或者内容已经更新，需要重新请求，通过网络传输新的文件或者内容。

这里的协商方式也可以通过Http协议来控制，Last-Modified和Etag，这个时候请求服务器，如果是内容没有发生变更的情况，服务器会返回 304 Not Modified。这样的话，就不需要每次访问服务器都通过网络传输一个比较大的文件或者数据包，只要简单的http应答就可以达到相同的请求文件效果。

下图中的例子，是腾讯的自建CDN（imgcache.gtime.cn）：

优点：减少频繁的网络大数据包传输，节约带宽，提升用户体验。

成本：增加了服务器处理的步骤，消耗更多的CPU资源。

5. 浏览器中间代理

上面的几种cache机制，实际上都是非常常见。但是，在移动互联网时代，流量昂贵是很多用户心中深深的痛。于是，又出现了一种新型的中间cache， 也就是在浏览器和web服务器再架设一个中间代理。这个代理服务器会帮助手机浏览器去请求web页面，然后将web页面进行处理和压缩（例如压缩文件和图片），使页面变小，然后再传输给手机端的浏览器。

部分手机浏览器（例如Chrome）号称可以节省流量，提升访问速度，实际上就是上述做法。但是，也分为两种情况：

    用户的网络和手机配置都比较差，因为页面被压缩变小，加载和传输速度变快，并且节约了流量。

    用户的网络和手机配置都比较好，本身直连速度已经很快了，反而因为设置了中间代理，加载速度变慢，也可节约流量。

优点：节约用户流量，大部分情况下提升了加载速度。

成本：需要架设中间代理服务器，对各种文件进行压缩，有比较高的服务器维护成本。

6. 预加载缓存机制

这种加载方式主要流行在移动端，为了解决手机网速慢和浏览器加载性能问题，浏览器会判断页面的关联内容，进行“预加载”。 也就是说，在用户浏览A页面的时候，就提前下载并且加载B页面的内容。给用户的体验就是，B页面一瞬间就出现了，中间没有任何延迟的感觉，从而带来更好的 极佳的用户体验。

这种实现机制，往往由浏览器来实现，当然，手机页面本身，也可以通过JS来自身实现。而这种机制也存在一些问题，浏览器需要预判用户的浏览行为，在一些场景下，这个预判算法本身不一定准确，如果不准确则带来一定的流量、内存和系统资源的浪费。

优点：给用户带来极佳的页面展示体验。

缺点：预判实现比较复杂，占据一定的内存和手机系统资源，可能产生流量和资源浪费。

前端的cache当然不仅仅如此简单，如果细致到每一个小环节和组成部分，我们会发现实际上是无处不在的，例如浏览器的渲染行为、网络网卡的传输环节，小环节和小环节之间也有无数这种类型的cache角色。

这个就如同几何分形学中的自相似性：从整体上看符合某种组成规律或者特性，同时，从局部看，仍然符合某种组成的规律或者特性。

几何分形的现象在我们生活中，也是非常常见的，例如：

人体中的几何分形例子，例如：人体有1个头部+4肢，局部上看人的手指也是1个手指头+4个手指；人体无论整体或者局部，都大致遵循黄金分割点0.618的比例来生长（五官按照这个比例越多，越好看）。

例如下图中的叶子，每个局部都和主干组成结构相似。

二、Web系统和几何分形学

1. Web系统中的缓存机制

看完上面的前端cache，我们会感觉到缓存机制在前端中的确无处不在，那么它在其他地方和环节，是否也无处不在？

可以看看这张图：

实际上，每一个环节本身是可以又再次被放大的，放大以后，我们又看见了更多缓存机制的“特性”存在。从一个整体来看，符合该规律，从组成部分来看，仍然符合该规律。

每一个组成缓存机制的“成员”的内部，又存在着更多的缓存机制。

Apache内部的一些“缓存机制”：

    url映射缓存mod_cache（有mode_disk_cache和mod_mem_cache，后者官方已不推荐）

    缓存热点文件打开描述符mod_file_cache（对于静态文件的情况，减少打开文件中open行为的耗时）

    启动的时候，通过prefork模式设置的StartServers服务进程池，牺牲内存空间。

MySQL内的一些“缓存机制”：

    数据库的索引，牺牲磁盘空间（组合索引等会占据很大的磁盘空间）

    innodb_buffer_pool_size，热点数据的缓存，牺牲内存空间

    innodb_flush_method写入磁盘的机制，可以配置成缓冲写入的方式

    query_cache_size查询缓存，牺牲内存空间

    thread_cache_size数据库连接池的缓存个数，牺牲内存空间

2. 接近硬件层面的“空间换时间”

那我们再来看更细小的一个环节，计算机写的操作。我们会发现，在内存和物理磁盘之间，还有一个磁盘缓冲区（页高速缓存）的存在，这个是内存和磁盘之间的“缓存”。当然，读取的操作也是同理。

下图是“放大”MySQL中的写入磁盘：

实际上，更进一步看，CPU和内存之间也存在缓存机制（常用指令会存在放在寄存器中，因为CPU访问寄存器会远快于访问内存，中间为了缓冲它们之间差距，设置了多级高速缓存）。

例如下图是Intel i7 920的各级缓存大小：

这个时候，我们可以看出来，计算机系统从大的系统层面看，是遵循“缓存机制”的规律的，同时，在每个局部成员的层面，同样遵循该规律。

3. 现实世界中的“缓存机制”

我们现在喝水通常使用的是杯子，杯子实际上也扮演着一个特殊的Cache角色。举个例子：一个人离饮水机比较远，他渴了，他有如下两种“喝水”的方式：

    不用杯子，每次渴了直接去饮水机喝（这个比较霸气侧漏，不要在意细节）。结果：频繁跑动，耗费体力。

    使用杯子，渴了先喝杯子（Cache）上的水，如果杯子没有，带上杯子去装水，再喝。结果：比较少跑动，节省体力。

这样看不直观，简化为一个流程图如下：

这虽然是个人尽皆知的道理，但是，这个方法本身是“进化”出来的。百万年前的原始人类和其他大自然的动物一样的，喝水遵循了第一种方式，只是随着人类的发展，“进化”出第二种喝水的方式。

这里也存在一个缓存机制，就是用杯子的空间获取喝水效率的时间。

还有一个更为典型的例子，就是坐车/运输，假设我们从深圳去广州，我们会去坐客运车。而客运车（假设上面有40个 座位）实际上相当于一个40个座位的“队列”。遵循着网络传输的相同的规律“队列满或者超时则发送”。客车本身的40个位置，就像一个“发送缓冲区”。使 用和不使用这个大的缓冲区，客车也可以有两者运作方式：

    车站发现来一个人，用只能容纳一个人的小车，不等待直接送一个人去广州。

    车站发现来一个人，先放进客车buffer中，等待人满或者达到班车约定时间（队列超时）再出发。

显而易见，第一种是太浪费资源了。

除此之外，还有很多各种各样的例子，如江河上的大坝、我们桌面上的一些东西（它们占据宝贵的桌面空间）、我们公司附近小店里的商品、离我们近的东西等等。

看到这里，很多人会渐渐发觉，计算机的一些原理，竟然在现实世界里有无处不在的“映射和影子”。

几何分形学是个非常有趣的东西，某些规律，实际上还贯穿在整个宏观和微观世界中。

例如“绕转”的现象：

4. 现实世界和计算机“缓存机制”原理的关系，为什么遵循“几何分形”？

实际上，计算机的原理来源于数学，而数学是日常生活现象和规律的高度抽象，源于生活，高于生活。

同时，不仅仅“缓存机制”，还有很多其他技术的原理，也能找到这种遵循“几何分形学”的样子。

http://mp.weixin.qq.com/s?__biz=MjM5ODIzNDQ3Mw==&mid=208702835&idx=1&sn=3cd8e5b9e5d1f9e49cf590ea8e521449&scene=0#rd

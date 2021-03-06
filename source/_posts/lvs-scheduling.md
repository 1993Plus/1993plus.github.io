---
title: LVS 的调度算法
date: 2017-10-27 17:40:16
category: LVS
tags:
	- LVS
	- LVS Scheduling
---

**LVS 连接调度算法**

IPVS 在内核中的负载均衡调度是以连接为粒度的。在 HTTP 协议（非持久）中，每个对象从 WEB 服务器上获取都需要建立一个 TCP 连接，同一用户的不同请求会被调度到不同的服务器上，所以这种细粒度的调度在一定程度上可以避免单个用户访问的突发性引起服务器间的负载不均衡。

在内核中的连接调度算法上，IPVS 已经实现了一下八种调度算法：

* **轮叫调度（Round-Robin Scheduling）**

轮叫调度（Round-Robin Scheduling）算法就是以轮叫的方式依次将请求调度到不同的服务器。优点是简洁，它不需要记录当前所有连接的状态，所以它是一种无状态调度。

在系统实现时，还引入了一个额外的条件，当服务器的权值为零时，表示该服务器不可用而不被调度。这样做的目的是将服务器切出服务（如屏蔽服务器故障和系统维护），同时与其它加权算法保持一致。

轮叫调度算法假设所有服务器处理性能均相同，不管服务器的当前连接数和响应速度。该算法相对简单，不适用于服务器组处理性能不一的情况，而且当请求服务时间变化较大时，轮叫调度算法容易导致服务器间的负载不均衡。

虽然 Round-Robin DNS 方法也是以轮叫调度的方式将一个域名解析到多个 IP 地址，但轮叫 DNS 方法的调度粒度是基于每个域名服务器的，域名服务器对域名解析的缓存会妨碍轮叫解析域名生效，这会导致服务器间负载的严重不均衡。IPVS 轮叫调度算法的粒度是基于每个连接的，同一用户的不同连接会被调度到不同的服务器上，所以这种粒度的轮叫调度要比 DNS 的轮叫调度优越很多。

* **加权轮叫调度（Weighted Round-Robin Scheduling）**

加权轮叫调度（Weighted Round-Robin Scheduling）算法可以解决服务器间新能不易的情况，它用相应的权值表示服务器的处理性能，服务器的确实权值为 1 。假设服务器 A 的权值为 1 ，B 的权值为 2 ，则表示服务器 B 的处理性能是 A 的两倍。加权轮叫调度算法是按权值的高低和轮叫方式分配请求到各服务器。权值高的服务器先收到连接，权值高的服务器比权值低的服务器处理更多的连接，相同权值的服务器处理相同数目的连接数。

例如，有三个服务器 A、B、C，权值分别为 4、3、2，则在一个调度周期内调度序列为 AABABCABC。加权轮叫调度算法还是比较简单和高效。当请求的服务器时间变化很大，单独的加权轮叫调度算法依然会导致服务器间的负载不均衡。

当服务器权值为零时，该服务器将不被调度；当所有服务器的权值为零，则没有任何服务器可用，算法返回 NULL，所有的新连接都会被丢掉。加权轮叫调度也不需要记录当前所有连接的状态，所以它也是一种无状态调度。

* **最小连接调度（Least-Connection Scheduling）**

最小连接调度（Least-Connection Scheduling）算法是吧新的连接请求分配到当前连接数最小的服务器。最小连接调度是一种动态调度算法，它通过服务器当前所活跃的连接数来估计服务器的负载情况。调度器需要记录各个服务器已建立的连接数目，当一个请求被调度到某台服务器，其连接数加 1；当连接终止或超时，其连接数减 1。

在系统实现时，也引入的当服务器的权值为零时，表示该服务器不可用而不被调度。

当各个服务器有相同的处理性能时，最小连接调度算法能把负载变化大的请求分布平滑到各个服务器上，所有处理视角比较长的请求不可能被发送到同一台服务器上。但是，当各个服务器的处理能力不同时，该算法并不理想，因为 TCP 连接处理请求后会进入 TIME_WAIT 状态，TCP 的 TIME_WAIT 一般为 2 分钟，此时连接还占用服务器的资源，所以会出现这种情形，性能高的服务器已处理所收到的连接，连接处于 TIME_WAIT 状态，而性能低的服务器已经忙于处理所收到的连接，还不断地收到新的连接请求。

* **加权最小连接调度（Weighted Least-Connection Scheduling）**

加权最小连接调度（Weighted Least-Connection Scheduling）算法是最小连接调度的超集，各个服务器用相应的权值表示其处理性能。服务器的缺省权值为 1，系统管理员可以动态的设置服务器的权值。加权最小连接调度在调度新连接是尽可能使服务器的已建立连接数和其权值成比例。

* **基于局部性的最少链接调度（Locality-Based Least Connection Schduling）**

基于局部性的最少链接调度（Locality-Based Least Connection Scheduling，简称 LBLC）算法是针对请求报文的目标 IP 地址的负载均衡调度，目前主要用于 Cache 集群系统，因为在 Cache 集群中客户请求报文的目标  IP 地址是变化的。假设任何后端服务器都可以处理任一请求，算法的设计目标是在服务器的负载基本平衡情况下，将相同目标 IP 地址的请求调度到同一台服务器，来提高各个服务器的访问局部性和主存 Cache 命中率。

此外，对目标节点 IP 要进行周期性的垃圾回收（Garbage Collection），将过期的目标 IP 地址到服务器关联性进行回收。过期的关联项是指哪些当前时间（实现时采用系统时钟节拍数 jiffies）减去最近使用时间超过设定过期时间的关联项，系统缺省的设定过期时间为 24 小时。

* **带复制的基于局部性最少链接调度（Locality-Based Least Connection with Replication Scheduling）**

带复制的基于局部性最少链接调度（Locality-Based Least Connection with Replication Scheduling，简称 LBLCR）算法也是针对目标 IP 地址的负载均衡，目前主要用户 Cache 集群系统。它与 LBLC 算法的不同之处是它要维护从一个目标 IP 地址到一组服务器的映射，而 LBLC 算法维护从一个目标 IP 地址到一台服务器的映射。对于一个“热门”站点的服务请求，一台 Cache 服务器可能会忙不过来处理这些请求。这时，LBLC 调度算法会从所有的 Cache 服务器中按“最小连接”原则选出一台服务器，映射该“热门”站点到 Cache 服务器（服务器集合），当该“热门”站点的请求负载增加时，会增加集合里的 Cache 服务器，来处理不对增长的负载；当该“热门”站点的请求负载降低时，会减少集合里 Cache 服务器的数目。这样，该“热门”站点的映像不太可能出现在所有 Cache 服务器上，从而提高 Cache 集群系统的使用效率。

LBLCR 算法先根据请求的目标 IP 地址找出该目标 IP 地址对应的服务器组；按“最小连接” 原则从该服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器；若该服务器超载，则按“最小连接”原则从整个集群中选出一台服务器，将该服务器加入到服务器组中，将请求发送带该服务器。同时，当服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的程度。

此外，对目标节点 IP 要进行周期性的垃圾回收（Garbage Collection），将过期的目标 IP 地址到服务器关联性进行回收。过期的关联项是指哪些当前时间（实现时采用系统时钟节拍数 jiffies）减去最近使用时间超过设定过期时间的关联项，系统缺省的设定过期时间为 24 小时。

* **目标地址散列调度（Destination Hashting Scheduling）**

目标地址散列调度（Destination Hashing Scheduling）算法也是针对目标 IP 地址的负载均衡，但它是一种静态映射算法，通过一个散列（Hash）函数将一个目标 IP 地址映射到一台服务器。

目标地址散列调度算法先根据请求的目标 IP 地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且为超载，将请求发送到该服务器，否则返回空。

* **源地址散列调度（Source Hashing Scheduling）**

源地址散列调度（Source Hashing Scheduling）算法正好与目标地址散列调度算法相反，它根据请求的源 IP 地址，作为散列键（Hash Key）从静态分配的散列表中找出对应的服务器，若该服务器是可用的且为超载，将请求发送到该服务器，否则返回空。它采用的散列函数与目标地址散列调度算法相同。它的算法流程与目标地址散列调度算法基本相似，除了将请求的目标 IP 地址换成请求的源 IP 地址。

在实际应用中，源地址散列调度和目标地址散列调度可以结合使用在防火墙集群中，他们可以保证整合系统的唯一出入口 。



**动态反馈负载均衡算法**

动态反馈负载均衡算法考虑到实时负载和响应情况，不对调整服务器间处理请求的比例，来避免有些服务器超载依然收到大量请求，从而提高整合系统的吞吐量。该算法在负载调度器上运行了一个 Monitor Daemon 进程，Monitor Daemon 来监视和收集各个服务器的负载信息。Monitor Daemon 可根据多个负载信息算出一个综合负载值。Monitor Daemon 将各个服务器的综合负载值和当前权值算出一组新的权值，若新权值和当前权值的差值大于设定的阀值，Monitor Daemon 将改服务器的权值设置到内核中的 IPVS 调度中，而在内核中连接调度一般采用加权轮叫调度算法或者加权最小连接调度算法

**连接调度**

当客户通过 TCP 连接访问网络时，服务所需的实际和所要消耗的计算资源是千差万别的，它依赖于很多因素。例如，它依赖于请求的服务类型、当前网络带宽的情况、以及当前服务器资源利用情况。一些负载比较重的请求需要进行计算密集的查询、数据库访问、很长响应数据流；而负载比较轻的请求往往只需要读一个 HTML 页面或者进行很简单的计算。

请求处理时间的千差万别可能会导致服务器利用的倾斜（Skew），即服务器间的负载不平衡。例如，有一个 WEB 页面有 A、B、C 和 D 文件，其中 D 是大图像文件，浏览器需要连理四个连接来取这些文件。当多个用户通过浏览器同时访问该页面是，最极端的情况是所有 D 文件的请求被发送到同一台服务器，所以说，有可能存在这样的情况，有些服务器已经超负荷运行，而其他服务器基本是闲置的。同时，有些服务器已经忙不过来，有很长的请求列队，还不对地收到新的请求。反过来说，这会导致客户长时间的等待，觉得系统的服务质量差。

* 简单的连接调度

简单的连接调度可能会使得服务器倾斜的发生。在上面的例子中，若采用轮叫调度算法，且集群中正好有四台服务器，必有一台服务器总是收到 D 文件请求。这种调度策略会导致整合系统资源的低利用率，因为有些资源被用尽导致客户端的长时间等待，而其他资源空闲着。

* 实际 TCP/IP 流量的特征

网络流量是程波浪型发生的，在一段较长时间的小流量后，会有一端大流量的访问，然后是小流量，这样跟波浪一样周期性的发生。这就需要一个动态反馈机制，利用服务器组的状态来应对访问流的自相似性。



摘抄自章文嵩博士的 《Linux服务器集群系统(四)》原文链接： http://www.linuxvirtualserver.org/zh/lvs4.html



END!
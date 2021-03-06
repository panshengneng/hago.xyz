---
title: 负载均衡-序
---

## 序：
- 对Web站点扩展一开始不宜过早，除非是基于高可用性和就近部署的考虑。但对于架构师而言，在架构设计之初就要有扩展的计划，关键是要清楚何时进行扩展。这里先介绍的是水平扩展，所谓的扩展是通过扩展规模来提升承载能力的本领。这种本领往体现在增加物理服务器或集群节点，这种本领发挥强，可提升的承载空间越大，但往往也受到其它的约束比如单机的限制、成本等。

## 12.1 一些思考
- 对于web站点的水平扩展，负载均衡是一种常见的手段。生活中典型的例子就是项目外包。

## 12.2 HTTP重定向
- Http重定向可以将http请求进行转移，一般用于自动跳转，这种重定向由Http定义并由Http代理(如：浏览器)和Web服务器共同实现。正因为http重定向具备请求转移和自动跳转的本领，除了满足各种自动跳转外，还可以实现Web负载均衡达到web扩展的目的。

- 镜像下载就是Http重定向的典型案例。镜像下载的目的就是实现负载均衡，一般也达到了就近访问的目的，加快用户下载速度，避免一定的带宽浪费。通过不同的地域来源来转移请求只是实现负载均衡的一种策略，但有些时候不一定合理。

- 我们需要权衡转移请求的开销和实际处理请求的开销，前者对后者越小，那么重定向的意义就越大。

- Http重定向受到主站点性能的制约，不过它的调度具有一定的灵活性，可以通过Web程序实现调度策略。

## 12.3 DNS负载均衡
- DNS负责域名解析，若一个域名解析可以对应多个IP地址，这时DNS服务器便充当了负载均衡调度器，将请求分散到多个服务器上，常见的策略是对多个A记录进行RR(轮询)。A记录的功能就是将域名映射到指定的IP地址。

- 和重定向相比，DNS负载均衡完全节省了主站点或者说DNS服务器充当了主站点的职能，为了提高此时DNS服务器的可用性，可以同时使用多台DNS服务器。DNS服务器可以根据用户IP进行智能解析，使其找到可用A记录中离用户最近的一台服务器。当监测到某台实际服务器出现故障后，可以使用DNS动态协议来迅速修改DNS记录，不过还是有一定的延迟    

- DNS负载均衡工作在DNS层，或多或少具有一定的局限性，比如实际服务器实时负载健康监测难等，DNS记录缓存更新延迟等。

- 所谓“均衡”不是指每台实际服务器承担的工作量或者说负载是一样的，实际应该是能者多劳。

## 12.4 反向代理负载均衡
- 反向代理服务器的核心工作就是转发Http请求，工作在HTTP层，因此也称为七层负载均衡。反向代理服务器是转发请求不是转移，前面的都是转移。

- 在这种调度模式下，任何对实际服务器的请求都必须经过调度器，调度器必须等待实际服务器的响应并反馈给用户。

- 调度器可以按权重分配任务给后端服务器，因为后端服务器的能力可能不一样。按权重分配的配置在Nginx中使用weight参数定义，很多反向代理服务器还支持权重分配的RR调度策略。

- 反向代理服务器本身的并发处理能力很重要，当反向代理服务器的吞吐率达到上限时，添加再多的后端服务器也无济于事。反向代理服务器转发操作本身具有一定的开销如创建线程、与后端服务器建立TCP连接、接收后端服务器返回的结果、分析HTTP头信息、用户态和内核态的切换等。

- 大多反向代理服务器自身可以监测到后端服务器的健康状况如系统负载、响应时间、是否可用、TCP连接数、流量等。

- 在实际应用中，我们可以备用一定数量的后端服务器，一旦某些后端服务器出现故障可以使用备用服务器进行替换提供服务。

- 使用RR调度策略时即使是同一用户对同一内容的多次请求有可能是被转发到不同的后端服务器上，这样就会产生后端服务器本地Session不同步及缓存利用率下降的问题。

- 解决Session一致的问题可以使用在一定Session周期内使同一个用户的请求始终转发到同一台后端服务器，要设计持续性调度算法，比如对用户的IP进行hash计算或使用Cookie持久性算法。

- 当然，在后端服务器本地上保存Session和缓存并不是一个明智的选择，这样会导致为了保证Session一致等情况无法做到让后端服务器承担不同的权重。应该使用分布式的Session(多台Session服务器)和分布式缓存。

- 反向代理服务器由于本身开销大等原因存在扩展性差，性能受到极限等限制。

## 12.5 IP负载均衡
- 网络地址转换(NAT)负载均衡工作在传输层，对数据包中的IP地址和端口进行修改，从而达到转发的目的，称为四层负载均衡。

- NAT是使用用户处在内部网络却可以与互联网进行通信。而反向NAT就是将实际服务器放于内部网络，将用户的数据包转发给内部的后端服务器。

- 在Linux中修改IP数据包可通过Netfilter，Netfilter工作在内核中，我们可以通过iptables在用户态对内核态的Netfilter的过滤表进行操作如删除、插入和修改等，iptables最常用的应用场合就是防火墙了。

- 路由器的工作就是存储转发，修改数据包的MAC地址。而这里说的路由器就是要修改数据包的来源地址和端口或目标地址和端口。

- NAT服务器必须作为实际服务器的网关，否则数据包被转发后将一去不返。

- IPVS不仅可以实现基于NAT的负载均衡，还可以实现直接路由和IP隧道等负载均衡。IPVS的管理工具是ipvsadm，也称为LVS。

- RR是静态调度策略，LVS除了支持RR外，还支持许多动态调度策略如最小连接、带权重的最小连接、最短期望时间延迟等。

- NAT服务器将会制约集群扩展，它不仅要转发用户的请求给实际服务器，也要将实际服务器的响应转发给用户，显然它将会是一个瓶颈。解决这个瓶颈的简单办法是提高 NAT服务器的带宽或将基于NAT的集群和DNS的RR混合使用或使用直接路由等负载均衡方式。

## 12.6 直接路由
- 这种方式工作在数据链路层。它修改数据包的目标MAC地址，并没有修改目标IP，然后发给实际的服务器，实际服务器的响应数据直接发回给用户，而不用经过调度器。但实际服务器必须接入外网，而且不能将调度器作为默认网关，要给实际服务器添加和调度器IP地址相同的IP别名。

- 一个物理网卡拥有一个IP地址，但可以为它配置更多个IP，这里配置的IP就是IP别名。一个网卡接口最多可以设置256个IP别名。

- 使用直接路由转发无需修改数据包的目的端口，因为这种转发工作在数据链路层，它对上层端口无能为力。
​
- 越是响应数据包远远超过请求数据包的服务就越应该降低调度器转移请求的开销，也就越能提高整体扩展能力，最终也就越依赖于WAN的出口带宽。

- LVS－DR非常适合搭建可扩展有负载均衡系统，不论是web服务器还是文件服务器以及视频服务器，都有不错的出色表现。

## 12.7 IP隧道​
- 基于IP隧道的负载均衡系统也可以使用LVS来实现，称为LVS－TUN，与LVS－DR不同的是，实际服务器和调度器可以不在同一个WAN网段，调度器通过IP隧-道技术来转发请求到实际服务器，所以实际服务器必须有合法的IP地址。

- 基于IP隧道的请求转发机制是将调度器收到的IP数据包封装在一个新的IP数据包中，转交给实际服务器，然后实际服务器的响应数据包可以直接到达用户端。

- 要使用IP隧道则所有的服务器必须支持IP隧道协议。基于IP隧道的独特方式，可以将实际服务器部署在不同的地域并根据就近原则转移请求，比如一些CDN服务器就是基于IP隧道技术实现的。

## 12.8 考虑可用性
- 在负载均衡系统中多台服务器分散开销的同时，本身也提高了实际服务器的可用性。为避免大量的请求造成的血崩效应，一般将实际服务器的数量大于实际数目。为实现调度器的故障平滑转移，保证调度器的可用性，可以使用heartbeat解决这个问题，即准备备用调度器，做到出故障时进行替换。
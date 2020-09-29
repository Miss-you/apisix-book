# Apache APISIX 高性能实践

[TOC]

>个人介绍：yousa（厉辉）Apache APISIX PMC，云原生社区管委会成员，专注于高性能网络服务器研发，是 Apache APISIX 开源项目核心成员，目前关注云原生 Ingress、API 网关、Envoy 等相关领域，喜欢开源，乐于分享，GItHub：https://github.com/Miss-you
## 前言

Apache APISIX 是一个动态、实时、高性能的 API 网关，基于 Nginx/Openresty 网络库和 etcd 实现，提供负载均衡、动态上游、灰度发布、服务熔断、身份认证、可观测性等七层流量管理功能，可以用来处理网站、移动设备和 IoT 的流量，通常使用 Apache APISIX 来处理传统的南北向流量。本文就七层网关演进过程中遇到性能的问题进行展开，从经典性能问题 nginx 配置加载说起 ，介绍 Apache APISIX 各种性能优化手段，探讨 Apache APISIX 容器化部署最佳实践，主要从如下几点介绍：

* Apache APISIX 介绍
* 高性能之道
* 高性能之术
* 总结与展望
## 高性能 API 网关 APISIX

### Apache APISIX 发展历程

Apache APISIX 其中文名字念作 API 6，一方面是因为 APISIX 是 6 月 6 日正式开源的，另一方面创始人也是期望该开源项目可以 666，项目 7 月进入 CNCF 全景图，2019 年 10 月正式捐献给 Apache 开源基金会，并且已经有多家用户（比如贝壳找房、思必驰）将其落地生产系统。随后 Apache APISIX 便向前一路狂奔，一个月发一个 release 版本，功能不断完善和丰富，新增支持 Skywalking 探针等重大特性，9 个月的时间从 Apache 基金会光速毕业。更重要的是，Apache APISIX 的高性能（轻松秒杀竞品 Kong ）、友好的开发者体验、丰富的插件能力吸引越来越多的开发者加入，当前已有超过 108 名贡献者。

![图片](https://uploader.shimo.im/f/kHYkHXZhczY4ovwh.png!thumbnail)

### Apache APISIX 功能视图

Apache APISIX 作为一个全平台的高性能七层网关，提供多协议卸载、全动态加载、动态服务发现、精细化路由、服务治理（Trace、限流、重试、熔断、超时控制等）、丰富的负载均衡算法、高可扩展插件框架等功能，可用于 API Gateway、云原生 Ingress 或 Layer 7 负载均衡器等场景。

![图片](https://uploader.shimo.im/f/oHlDXD8avWJRKdWn.png!thumbnail)

常见的功能比如：

* 全平台：无论是裸机还是 Kubernetes，Intel 或者 Arm64 芯片，Apache APISIX 都可以运行；其底层平台可以支持 Openresty 或者 Tengine
* 多协议支持：核心功能比如 HTTP 1.1、HTTPS、HTTP 2 代理支持，也支持 gRPC 协议代理和转换，基于 client id 的对于 MQTT 协议的代理功能；若基于 tengine 平台，则可以利用 tengine 的能力支持 Dubbo 协议的代理功能
* 动态加载：无需重启服务即可动态加载路由以及框架插件，另一方面也支持动态路由，包括最基本的动态 Upstream、心跳探测（包括主动健康检查、被动熔断）、常见负载均衡（一致性哈希和有权重轮询）等
* 丰富的路由匹配能力：最基本的比如全路径匹配和前缀匹配，也可以基于 Nginx 内置变量（比如 args、host 域名、请求 header 和 cookies），甚至自定义 lua 代码定义匹配机制
* 安全防护：IP 黑白名单、常见鉴权（LDAP、JWT、BASIC、HMAC、OAUTH2 等）、限流限频（限流限连接限并发）
* 运维友好：分布式追踪（Skywalking、Zipkin）、Prometheus、无状态节点
* 高度可扩展：自定义插件、自定义负载均衡算法插件、自定义路由

![图片](https://uploader.shimo.im/f/V0YVlSWpgdL26Fym.png!thumbnail)

### Apache APISIX 架构解析

Apache APISIX 开源架构如图，其主要分为数据面和控制面。

* 数据面：以 Nginx 的网络库为基础，（弃用 Nginx 的路由匹配、静态配置和 C 模块），使用 Lua 和 Nginx 动态控制请求流量，通过插件机制来实现各种流量处理和分发的功能：限流限速、日志记录、安全检测、故障注入等，同时支持用户编写自定义插件来对数据面进行扩充。
* 控制面：使用 etcd 来存储和同步网关的配置数据，管理员通过 admin API 或者 dashboard 可以在毫秒级别内通知到所有的数据面节点，同时 etcd 集群也保证了系统的高可用。
* 智能面：在智能面，开发者可以使用 DAG（有向无环图）对插件进行编排；另一方面，单纯的接入层本身没有价值，分析接入层的流量并反哺业务才有价值，通过决策树对请求流量进行实时分析和处理

![图片](https://uploader.shimo.im/f/zbKjBKu9rEvSAWwP.png!thumbnail)

## 高性能之道

众所周知，Nginx 是一款轻量级 Web 服务器、反向代理服务器，由于它的内存占用少、启动极快、高并发能力强，故其在互联网项目中得到广泛应用。而 Apache APISIX 是基于 Nginx 的一款高性能云原生 API 网关，它既具有 Nginx 的高性能，同时解决了 Nginx 动态加载的性能问题；在性能方面，对比同样是基于 Nginx 的另一个 API 网关 —— Kong，APISIX 也具有极大的优势。那么，为什么 Apache APISIX 性能如此之高？Apache ASPISIX 的高性能之道主要有以下三点：

* 动态路由
* JIT 加速
* Apache APISIX vs Kong （竞品对比）
### 动态路由

动态路由，指的是程序可以在运行时、在不重启进程的情况下，去修改参数、配置，乃至修改自身的代码。具体到 Nginx 和 OpenResty 领域，动态路由是指修改上游 Upstream 地址、SSL 证书、限流限速阈值，而不用重启网关服务。传统的开源 Nginx 对于这几类操作无法动态地完成，需要 reload Nginx 才可以将相应配置生效。频繁的 reload Nginx ，会带来流量抖动、 Nginx 内存和 cpu 利用率低下以及多次高负载场景 reload 业务造成雪崩的问题。

开源版本的 Nginx 并不支持动态路由特性，而商业版本的 Nginx Plus 提供了部分动态的能力，可以用 REST API 来完成部分配置的更新，但这最多算是一个不够彻底的改良。

而 OpenResty 的杀手锏之一就是动态路由的能力，Nginx 的这些问题都不存在。你可能纳闷儿，为什么基于 Nginx 的 OpenResty 却可以支持动态呢？原因也很简单，Nginx 的逻辑是通过 C 模块来完成的，而 OpenResty 是通过脚本语言 Lua 来完成的——脚本语言的一大优势，便是运行时可以去做动态地改变。

以 Apache APISIX 的动态路由为例，APISIX 的动态路由主要分为三部分，healthchecker/balancer 和 address cache，healthchecker 包括被动熔断和主动健康检查，熔断是指由终端的请求触发，进而分析上游的返回值来作为健康与否的判断条件。如果没有终端请求，那么上游是否健康就无从得知了；主动健康检查基于 ngx.timer 定时去轮询指定的上游接口，来检测健康状态。balancer 是负载均衡器，会从可用的后端节点选一个合适的服务，常见负载均衡算法有一致性哈希和有权重轮询。addr cache 则是由于 apisix 最终需要的后端地址是 ip 和端口，如果是域名的话需要每次解析域名，所以为了更高的性能，我们会将域名解析结果进行缓存，加速访问速度。

![图片](https://uploader.shimo.im/f/iYd8L04P6W3pHXXX.png!thumbnail)

更进一步，Apache APISIX 将热更新做到了极致，不仅可以支持动态加载路由，还支持热更新插件，又或者路由匹配中嵌入 lua 匹配逻辑， 无需重启服务，就可以持续更新配置和插件。

（局限性待补充）

### JIT 加速

Apache APISIX 是基于 Openresty 平台使用 Lua 语言编写的，我们都知道，LUA 是一门解释语言，类似于 Python，需要解释器解释执行，其灵活性好，但性能相较于静态编译语言比如 C/C++/Golang 会差许多，但为什么基于 LuaJIT 的 Apache APISIX 转发性能如此高？LuaJIT 与 LUA 又有什么区别？

JIT 是 just in time 的缩写，是动态编译的一种形式，是一种优化虚拟机运行的技术。程序运行通常有两种方式，一种是静态编译，一种是动态解释，而 JIT 混合了这二者，Java 和 .Net 也均采用了这样的加速方式，当然也包括 LuaJIT。所以，标准 Lua 和 LuaJIT 是两回事儿，另一方面，LuaJIT 只是兼容了 Lua 5.1 的语法。

![图片](https://uploader.shimo.im/f/mFD7Ese0daMDDeUo.png!thumbnail)

解释执行，执行效率低、代码暴露；静态编译，无法动态热更新、平台兼容性差；而 JIT 动态编译，兼具二者优点，但同时有一定局限性，入门容易，精通难。

JIT 通常分为 Method JIT 和 Trace JIT，Lua JIT 这里采用的是 Trace JIT。LuaJIT 的解释器会在执行字节码的同时，记录一些运行时的统计信息，比如每个 Lua 函数调用入口的实际运行次数，还有每个 Lua 循环的实际执行次数。当这些次数超过某个随机的阈值时，便认为这段代码足够热，这时便会触发 JIT 编译器开始工作。JIT 编译器会从热函数的入口或者热循环的某个位置开始，尝试编译对应的 Lua 代码路径，将其最终转换成对应目标系统的机器码，加速执行。

Apache APISIX 的很多性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式。当然不仅仅是 Apache APISIX 是如此，大家最常用的 Nginx Ingress 的很多优化案例也是如此，比如，https://github.com/kubernetes/ingress-nginx/pull/5665

此外，出于性能方面的考虑，LuaJIT 还扩展了 table 的相关函数：table.new 和 table.clear。这是两个在性能优化方面非常重要的函数，在 OpenResty 的 lua-resty 库中会被频繁使用。

所以，写出正确的 LuaJIT 代码的门槛并不高，但要写出高效的 LuaJIT 代码绝非易事。

（局限性待补充）

### Apache APISIX vs Kong

有对比才更有说服力，Apache APISIX 和 Kong 都是基于 Openresty/LuaJIT 实现的高性能 API 网关，我们从三个方面简要对比他们的差异：转发性能、路由匹配算法以及配置生效时间。

* 在开启 prometheus/鉴权/限流插件情况下，Apache APISIX 单核转发 QPS 为 15000，转发时延 0.7 ms，而 Kong 的单核转发能力为 1500 左右，转发时延 2 ms。
* Apache APISIX 的路由复杂度是 O(k)，只和 uri 的长度有关，和路由数量无关；kong 的路由时间复杂度是 O(n)，随着路由数量线性增长。
* Apache APISIX 的配置下发只要 1 毫秒就能达到所有网关节点，使用的是 etcd 的 watch；Kong 网关是定期轮询数据库，一般需要 5 秒才能获取到最新配置
>除此之外，Apache APISIX 的 IP 匹配时间复杂度是 O(1)，不会随着大量 IP 判断而导致 cpu 资源跑满；kong 的最新版本也换用了 Apache APISIX 的 IP 匹配库；

当然 Apache APISIX 相比于 Kong 做了更多的优化，比如 Apache APISIX 为了降低 Runtime GC 带来的卡顿，自身做了内存池 apisix.core.table 的封装方便多种对象高效地复用。类比一下，如果说对于 C/C++开发者来说 libevent/muduo/libco 是高性能开发的圣经的话，那么 Apache APISIX 便是 Openresty/Nginx 开发者的高性能开发的标杆，一切性能优化尽在 Apache APISIX 的代码中。Talk is cheap，show you the code.

（应该再展开一下性能优化细节或者具体列表）

![图片](https://uploader.shimo.im/f/OYSV87xUdzPwyd3T.png!thumbnail)

（性能压测结果 APISIX vs Envoy vs Kong，以后补充跟 Envoy 的对比结果）

## 高性能之术

### 调优工具

俗语有云：“工欲善其事，必先利其器。”个人认为，程序员高性能编程往往需要解决大量性能问题，定位性能问题也需要一件“利器”。 如同医生给病人看病，需要依靠专业的医学工具（比如 X 光片、听诊器等）进行诊断，最后依据医学工具的检验结果快速精准地定位出病因所在。性能调优工具（比如 perf / gprof 等）之于性能调优就像 X 光之于病人一样，它可以一针见血地指出程序的性能瓶颈。

但是常用的性能调优工具 perf 等，在呈现内容上只能单一地列出调用栈或者非层次化的时间分布，不够直观。所以通常做法是 perf 或者 systemtap 等动态追踪工具结合火焰图，快速找到性能问题根因。

前面我有提到 LuaJIT 可以通过 JIT 热代码加速代码执行，但 LuaJIT 中 JIT 编译器的实现还不完善，有一些 Lua 原语它还无法进行 JIT 编译，因为这些原语实现起来比较困难，也就是所谓的 NYI（是 Not yet Implemented 的缩写），当 JIT 编译器在当前代码路径上遇到它不支持的操作时，便会退回到解释器模式，性能便会大打折扣。所以，我们也需要尽可能少的让 Lua JIT 遇到 NYI 操作，便需要扫描工具。在 Lua JIT 中可以使用自带的 jit.dump 和 jit.v 模块，来扫描 NYI 代码。它们都可以打印出 JIT 编译器工作的过程，我们便可以通过日志查看关键路径的代码是否存在 NYI，也就是 JIT 加速器无法加速的部分，有针对性的优化代码。

![图片](https://uploader.shimo.im/f/10M7U4m5qu1OP6tV.png!thumbnail)

### Ingress 架构图

![图片](https://uploader.shimo.im/f/rSM1Pme2eCVtLzxF.png!thumbnail)

上图是 Apache APISIX 在 Kubernetes 上常见的部署架构图，Master 节点上部署 ETCD、API server 以及 Controller 等，Apache APISIX 的 Node 机与业务 Node 建议分开，整个网络入口由负载均衡+Apache APISIX 提供服务。

管控面即左边虚线箭头部分，管控面主要考虑可扩展性以及运维友好性，Apache APISIX 管控面主要有 Dashboard、Admin API、ETCD 配置中心，Dashboard 作为 svc 部署、Admin API 与 Dashboard 部署在一起、Etcd 分发配置。

数据面即右边橙色和绿色实线部分，数据面主要考虑性能、数据面高可用，整个数据面其依赖于 负载均衡、Apache APISIX 集群化、以及 service 的 cluster ip，客户端发起请求，负载均衡收到请求后将请求转到其中一个 APISIX 节点，APISIX 匹配路由，获取 upstream 后端服务信息，通过服务配置的 Cluster ip 转到对应 pod 节点中；高可用则依赖于各个接入层级的高可用机制，比如集群化、心跳探测、被动熔断、会话保持等。

### 容器化部署最佳实践

前面讲了 Apache APISIX 高性能之道，但如果不对其运行环境进行一些参数调优，就会无法充分发挥出高性能的优势。Apache APISIX 在容器化中调优技巧主要有如下几点：

#### Apache APISIX 配置优化

提高连接池大小、单 worker 最大连接数量

（还没写完）

#### Node 配置调优

（还没写完）

比如提高连接队列大小、进程可用 FD 上限、开启 TIME_WAIT 快速回收

#### nginx worker 数量和绑核

因为 APISIX 运行在容器里面，调用 API 获取系统内存、CPU，取到的是宿主机的资源大小，若 nginx.conf 中 worker 数量配置是 auto，那么 worker 数量很容易会过多。而 APISIX worker 数量会根据 cgroup 信息进行计算，pod 分配几个 cpu 则启动几个 worker。

另一个比较重要的是 nginx 绑核，但由于容器本身的限制以及安全性问题，在普通容器中的进程调用绑核接口并不会生效。若想要成功调用绑核接口，则需要 Apache APISIX 运行在特权容器中，获取更高的执行权限。但启用特权容器后，一旦容器被入侵，那么入侵者就可以非常方便的获取 Node 机的 root 权限，比较危险；另一方面，特权容器中的进程便可以修改 Node 机的一些系统配置（比如 sysctl），会导致容器之间会互相影响，同时一旦出现跟调参相关的问题，也会非常难以排查。故非常不建议在生产环境中为了绑核而使用特权容器运行 Apache APISIX。

## 总结

如果你还在被 Nginx 或者 Nginx Ingress 的 reload 性能问题所折磨，又或者对 Kong 的转发能力并不满意，欢迎大家使用 Apache APISIX 将其作为接入网关或者 Ingress。

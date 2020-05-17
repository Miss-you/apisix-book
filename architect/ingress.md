# Awesome-ingress
A curated list of awesome ingress.

### 目录

- [Awesome-ingress](#awesome-ingress)
    - [目录](#%e7%9b%ae%e5%bd%95)
    - [Ingress总览](#ingress%e6%80%bb%e8%a7%88)
    - [开源的原因](#%e5%bc%80%e6%ba%90%e7%9a%84%e5%8e%9f%e5%9b%a0)

### Ingress总览

|              | Kubernetes Ingress                                     | Istio Ingress                                                  | Ambassador                                         | Kong Ingress                                                             | APISIX ingress                                                  | Traefik                                             | NGINX Ingress                                  | HAproxy                                                                 |
|--------------|--------------------------------------------------------|----------------------------------------------------------------|----------------------------------------------------|--------------------------------------------------------------------------|-----------------------------------------------------------------|-----------------------------------------------------|------------------------------------------------|-------------------------------------------------------------------------|
| 协议         | http/https，http2，  grpc                              | http/https，http2，  grpc，tcp，tcp+tls，  mongo，mysql，redis | http/https，http2，  grpc，tcp，tcp+tls            | http/https，http2，grpc                                                  | http/https，http2，grpc，tcp/udp，  tcp+tls，tcp+sni，  Dubbo   | http/https，http2，  grpc，tcp，tcp+tls             | http/https，http2，  grpc，tcp/udp             | http/https，http2，  grpc，tcp，tcp+tls                                 |
| 基础平台     | nginx/openresty                                        | envoy                                                          | envoy                                              | openresty                                                                | openresty/tengine                                               | traefik                                             | nginx/nginx plus                               | haproxy                                                                 |
| 路由匹配     | host，path                                             | host，path，  method，headers                                  | host，path，  method，headers                      | path，method，host，header                                               | path，method，  host，header，  nginx变量，args变量  自定义函数 | host，path，  headers，query，  path prefix，method | host，path                                     | host，path                                                              |
| 命名空间支持 | 共用或指定命名空间                                     | 共用或指定命名空间                                             | 共用或指定命名空间                                 | 指定命名空间                                                             | 共用或指定命名空间                                              | 共用或指定命名空间                                  | -                                              | 共用或指定命名空间                                                      |
| 部署策略     | 金丝雀部署，ab部署                                     | 金丝雀部署、蓝绿部署  灰度部署、根据header  白名单             | 金丝雀部署、蓝绿部署  灰度部署、根据header  白名单 | 金丝雀部署、蓝绿部署                                                     | ab部署、灰度发布、  金丝雀部署                                  | 金丝雀部署、蓝绿部署  灰度部署                      | -                                              | 蓝绿部署  灰度部署                                                      |
| upstream探测 | 重试、超时                                             | 重试、超时、  心跳探测、熔断                                   | 重试、超时、  心跳探测、熔断                       | 心跳探测、熔断                                                           | 重试、超时、   心跳探测、熔断                                   | 重试、超时、   心跳探测、熔断                       | 重试、超时、心跳探测                           | 心跳探测                                                                |
| 负载均衡算法 | RR，会话保持，  最小连接，一致性hash  EWMA             | RR，会话保持，  一致性hash，  maglev负载均衡                   | RR，会话保持，  一致性hash，  maglev负载均衡       | WRR  会话保持                                                            | 一致性hash，  WRR                                               | WRR，动态RR  会话保持                               | RR，会话保持，  最小连接，最短时间，一致性hash | RR，static-RR  最小连接，源ip，  uri，uri param，  uri header  会话保持 |
| 鉴权方式     | basic-auth  oauth                                      | basic, external auth,   Oauth, OpenID                          | basic, external auth,   Oauth, OpenID              | basic, Key, HMAC,   LDAP, Oauth 2.0,   PASETO,   OpenID Connect          | key-auth,   OpenID Connect                                      | basic auth  auth-url  external auth                 | -                                              | basic-auth  Oauth  Auth TLS                                             |
| 动态路由     | -                                                      | +                                                              | +                                                  | +                                                                        | +                                                               | +                                                   | -                                              | +                                                                       |
| JWT          | -                                                      | +                                                              | +                                                  | +                                                                        | +                                                               | +                                                   | +                                              | +                                                                       |
| DDOS防护能力 | limit-conn，  limit-count，  limit-req，  ip-whitelist | limit-req，  ip-whitelist                                      | limit-req，  ip-whitelist                          | limit-conn，  limit-count，  limit-req，  ip-whitelist，  response limit | limit-conn，  limit-count，  limit-req，  ip-whitelist          | limit-conn，  limit-req，  ip-whitelist             | rate-limit                                     | limit-conn，  limit-req，  ip-whitelist                                 |
| 全链路跟踪   | +                                                      | +                                                              | +                                                  | +                                                                        | +                                                               | +                                                   | -                                              | +                                                                       |
| 协议转换     | -                                                      | grpc，mongo，mysql  redis                                      | grpc，mongo，mysql  redis                          | -                                                                        | grpc，Dubbo                                                     | grpc                                                | -                                              | -                                                                       |
| 文档完善程度 | 文档丰富                                               | 快速入门、详细接口文档、细节文档丰富                           |                                                    | 快速入门、详细接口文档、细节文档丰富                                     | 稀少                                                            | 快速入门、详细接口文档、细节文档丰富                | 较少                                           | 较少                                                                    |

* Kubernetes Ingress：即 Kubernetes 推荐默认使用的 Nginx Ingress。它的主要优点为简单、易接入。缺点是Nginx reload耗时长的问题根本无法解决。另外，虽然可用插件很多，但插件扩展能力非常弱。
* Istio Ingress 和 Ambassador Ingress 都是基于非常流行的 Envoy。说实话，我认为这两个 Ingress 没有什么缺点，可能唯一的缺点是他们基于 Envoy 平台，如果大家对这个平台都不是很熟悉，上手门槛会比较高。
* Kong：其本身就是一个 API 网关，它也算是开创了先河，将 API 网关引入到 Kubernetes 中当Ingress。另外相对边缘网关，Kong 在鉴权、限流、灰度部署等方面做得非常好。Kong Ingress 还有一个很大的优点：提供了一些 API、服务的定义，可以抽象成 Kubernetes 的 CRD，通过K8S Ingress 配置便可完成同步状态至 Kong 集群。缺点就是部署特别困难，另外在高可用方面，与 APISIX 相比也是相形见绌。
* APISIX Ingress：它具有非常强大的路由能力、灵活的插件拓展能力，在性能上表现也非常优秀。同时，它的缺点也非常明显，尽管APISIX开源后有非常多的功能，但是缺少落地案例，没有相关的文档指引大家如何使用这些功能。
* Traefik ：基于 Golang 的 Ingress，它本身是一个微服务网关，在 Ingress 的场景应用比较多。他的主要平台基于 Golang，自身支持的协议也非常多，总体来说是没有什么缺点。如果大家熟悉 Golang 的话，也推荐一用。
* Nginx Ingress：主要优点是在于它完全支持 TCP 和 UDP 协议，但是缺失了鉴权方式、流量调度等其他功能。
* HAproxy：是一个久负盛名的负载均衡器。它主要优点是具有非常强大的负载均衡能力，其他方面并不占优势。

### 开源的原因

其实上述的表格其实是在11-12月份我依据自己对一些用过的网关的经验以及基于[Comparing Ingress controllers for Kubernetes](https://medium.com/flant-com/comparing-ingress-controllers-for-kubernetes-9b397483b46b)这篇文章整理的（因为比如traefik/ambassador等网关，我也没有足够的时间来对从未接触过的这些Ingress做功能以及性能等方面的评估。

另一方面，开源社区的Ingress他们的发展速度非常快，我相信半年后，我的表格可能就跟不上潮流了。

最后也是因为当我发了这张图后，有很多同事或者开源社区的朋友私下找我索要这张表格，足以说明我整理的内容的价值。

所以，为了让大家在选型Ingress的路上少走弯路，让该表格能够持续更新，我将这张表格开源出来了，希望大家会喜欢。

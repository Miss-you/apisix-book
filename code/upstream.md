Table of Contents
=================

* [前言(为什么会写这篇文章)](#%E5%89%8D%E8%A8%80%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E5%86%99%E8%BF%99%E7%AF%87%E6%96%87%E7%AB%A0)
* [APISIX upstream](#apisix-upstream)
  * [何为upstream](#%E4%BD%95%E4%B8%BAupstream)
  * [APISIX下upstream的设计及功能介绍](#apisix%E4%B8%8Bupstream%E7%9A%84%E8%AE%BE%E8%AE%A1%E5%8F%8A%E5%8A%9F%E8%83%BD%E4%BB%8B%E7%BB%8D)
    * [配置参数](#%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0)
    * [Consumer](#consumer)
  * [Upstream相关源码解读](#upstream%E7%9B%B8%E5%85%B3%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB)
    * [Upstream init](#upstream-init)
    * [Fetch upstream](#fetch-upstream)


# 前言(为什么会写这篇文章)

> 本人之前是个API网关小白, 因近期在做API网关相关的开发工作，所以找到了业界最火最牛逼的Apache APISIX, 研读了比较感兴趣的小部分的源码及核心组件的设计思路，受益颇多。 每当遇到疑惑时，@院生老师和@厉辉老师总是会耐心的为我们解惑，在此我深表感谢。 某人曾说过：“高效学习的关键在于知识的输出与分享”, 然从院生老师那了解到厉辉(yousa)老师正在写一本关于apisix的书籍，故加入到了这个开源项目，那么接下来将把之前学习到的东西陆续的输出到这里。
---

# APISIX upstream

## 相关概念

- Downstream（下游）: 下游主机连接到apisix，发送请求并接收响应，即发送请求的主机。

- Upstream（上游）: 上游主机接收来自apisix的连接和请求，并返回响应，即接受请求的主机。

---

## APISIX下upstream的设计及功能介绍

Upstream 是虚拟主机抽象，对给定的多个服务节点按照配置规则进行负载均衡。Upstream 的地址信息可以直接配置到 `Route`（或 `Service`) 上，当 Upstream 有重复时，就需要用“引用”方式避免重复了。

![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstream_example.png)

如上图所示，通过创建 Upstream 对象，在 `Route` 用 ID 方式引用，就可以确保只维护一个对象的值了。

Upstream 的配置可以被直接绑定在指定 `Route` 中，也可以被绑定在 `Service` 中，不过 `Route` 中的配置
优先级更高。这里的优先级行为与 `Plugin` 非常相似

### 配置参数

APISIX 的 Upstream 除了基本的复杂均衡算法选择外，还支持对上游做主被动健康检查、重试等逻辑，具体看下面表格。

|名字    |可选|说明|
|-------         |-----|------|
|type            |必填|`roundrobin` 支持权重的负载，`chash` 一致性哈希，两者是二选一的|
|nodes           |与 `k8s_deployment_info`、 `service_name` 三选一|哈希表，内部元素的 key 是上游机器地址列表，格式为`地址 + Port`，其中地址部分可以是 IP 也可以是域名，比如 `192.168.1.100:80`、`foo.com:80` 等。value 则是节点的权重。当权重值为 `0` 代表该上游节点失效，不会被选中，可以用于暂时摘除节点的情况。|
|service_name    |与 `nodes`、 `k8s_deployment_info` 三选一 |用于设置上游服务名，并配合注册中心使用，详细可参考[集成服务发现注册中心](discovery.md) |
|k8s_deployment_info|与 `nodes`、 `service_name` 三选一|哈希表|字段包括 `namespace`、`deploy_name`、`service_name`、`port`、`backend_type`，其中 `port` 字段为数值，`backend_type` 为 `pod` 或 `service`，其他为字符串 |
|key             |可选|在 `type` 等于 `chash` 是必选项。 `key` 需要配合 `hash_on` 来使用，通过 `hash_on` 和 `key` 来查找对应的 node `id`|
|hash_on         |可选|`hash_on` 支持的类型有 `vars`（Nginx内置变量），`header`（自定义header），`cookie`，`consumer`，默认值为 `vars`|
|checks          |可选|配置健康检查的参数，详细可参考[health-check](../health-check.md)|
|retries         |可选|使用底层的 Nginx 重试机制将请求传递给下一个上游，默认 APISIX 会启用重试机制，根据配置的后端节点个数设置重试次数，如果此参数显式被设置将会覆盖系统默认设置的重试次数。|
|enable_websocket|可选| 是否启用 `websocket`（布尔值），默认不启用|
|labels          |可选| 用于标识属性的键值对。 |
|pass_host            |可选|`pass` 透传客户端请求的 host, `node` 不透传客户端请求的 host, 使用 upstream node 配置的 host, `rewrite` 使用 `upstream_host` 配置的值重写 host 。|
|upstream_host    |可选|只在 `pass_host` 配置为 `rewrite` 时有效。|

`hash_on` 比较复杂，这里专门说明下：

1. 设为 `vars` 时，`key` 为必传参数，目前支持的 Nginx 内置变量有 `uri, server_name, server_addr, request_uri, remote_port, remote_addr, query_string, host, hostname, arg_***`，其中 `arg_***` 是来自URL的请求参数，[Nginx 变量列表](http://nginx.org/en/docs/varindex.html)
1. 设为 `header` 时, `key` 为必传参数，其值为自定义的 header name, 即 "http_`key`"
1. 设为 `cookie` 时, `key` 为必传参数，其值为自定义的 cookie name，即 "cookie_`key`"
1. 设为 `consumer` 时，`key` 不需要设置。此时哈希算法采用的 `key` 为认证通过的 `consumer_id`。
1. 如果指定的 `hash_on` 和 `key` 获取不到值时，就是用默认值：`remote_addr`。

创建上游对象用例：

```json
curl http://127.0.0.1:9080/apisix/admin/upstreams/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "type": "roundrobin",
    "k8s_deployment_info": {
        "namespace": "test-namespace",
        "deploy_name": "test-deploy-name",
        "service_name": "test-service-name",
        "backend_type": "pod",
        "port": 8080
    }
}'

curl http://127.0.0.1:9080/apisix/admin/upstreams/2 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "type": "chash",
    "key": "remote_addr",
    "enable_websocket": true,
    "nodes": {
        "127.0.0.1:80": 1,
        "foo.com:80": 2
    }
}'
```

上游对象创建后，均可以被具体 `Route` 或 `Service` 引用，例如：

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "upstream_id": 2
}'
```

为了方便使用，也可以直接把上游地址直接绑到某个 `Route` 或 `Service` ，例如：

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "plugins": {
        "limit-count": {
            "count": 2,
            "time_window": 60,
            "rejected_code": 503,
            "key": "remote_addr"
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "39.97.63.215:80": 1
        }
    }
}'
```

下面是一个配置了健康检查的示例：

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "plugins": {
        "limit-count": {
            "count": 2,
            "time_window": 60,
            "rejected_code": 503,
            "key": "remote_addr"
        }
    },
    "upstream": {
         "nodes": {
            "39.97.63.215:80": 1
        }
        "type": "roundrobin",
        "retries": 2,
        "checks": {
            "active": {
                "http_path": "/status",
                "host": "foo.com",
                "healthy": {
                    "interval": 2,
                    "successes": 1
                },
                "unhealthy": {
                    "interval": 1,
                    "http_failures": 2
                }
            }
        }
    }
}'
```

更多细节可以参考[健康检查的文档](../health-check.md)。

下面是几个使用不同`hash_on`类型的配置示例：

### Consumer

创建一个consumer对象:

```shell
curl http://127.0.0.1:9080/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "username": "jack",
    "plugins": {
    "key-auth": {
           "key": "auth-jack"
        }
    }
}'
```

新建路由，打开`key-auth`插件认证，`upstream`的`hash_on`类型为`consumer`：

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "key-auth": {}
    },
    "upstream": {
        "nodes": {
            "127.0.0.1:1980": 1,
            "127.0.0.1:1981": 1
        },
        "type": "chash",
        "hash_on": "consumer"
    },
    "uri": "/server_port"
}'
```

测试请求，认证通过后的`consumer_id`将作为负载均衡哈希算法的哈希值：

```shell
curl http://127.0.0.1:9080/server_port -H "apikey: auth-jack"
```

---

## Upstream相关源码解读

### Upstream init

 在讲fetch_upstream核心代码之前，我们先从源头开始，也就是在_M.http_init_worker(init.lua)阶段对upstream的初始化工作做一个简单的了解:
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstream_init.png)

http_init_worker函数还对router、service等均做了init工作, 背后的工作流大同小异,可选项可配置是否使用ngx.timer做watch&update工作。比如router.init_worker()会返回一个user_routes对象, 当请求过来的时候会先调用router.match(),  在match函数内部会判断缓存的版本是否与user_routes的版本一致，若一致则往下走真正的match流程； 若不一致，会重建[radixtree](https://github.com/api7/lua-resty-radixtree). 感慨一下, radixtree的设计思路和代码真的很牛逼！真心建议大家去研读下,本人也基于此写了一个golang版本的高性能网关路由(基于cgo). 重建radixtree的时需要用到user_routes.values这个结构. 那么关于upstream的初始化就讲到这里了.

### Fetch upstream

我们先来看一下http_access_phase的大致工作流:
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/fetch_upstream.png)

我们可以看到当matched_route.value.upstream_id不为空时会返回本地内存的upstream对象, 核心代码如下:
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstreams_get.png)
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/config_get.png)

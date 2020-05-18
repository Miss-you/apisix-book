# 常见问题

## APISIX 和其他的 API 网关有什么不同之处？

高性能、云原生、大牛云集。

## APISIX 的性能怎么样？

可以参考Apache APISIX压测性能数据：[benchmark](https://github.com/apache/incubator-apisix/blob/master/doc/benchmark-cn.md)

## APISIX 是否有控制台界面？

是的，在 0.6 版本中我们内置了 dashboard，你可以通过 web 界面来操作 APISIX 了。

## 个人如何开发插件？

[如何开发插件](doc/plugin-develop-cn.md)

## 我们为什么选择 etcd 作为配置中心？

对于配置中心，配置存储只是最基本功能，APISIX 还需要下面几个特性：

1. 集群支持
2. 事务
3. 历史版本管理
4. 变化通知
5. 高性能

APISIX 需要一个配置中心，上面提到的很多功能是传统关系型数据库和KV数据库是无法提供的。与 etcd 同类软件还有 Consul、ZooKeeper等，更详细比较可以参考这里：[etcd why](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/why.md#comparison-chart)。

## 为什么在用 Luarocks 安装 APISIX 依赖时会遇到超时，很慢或者不成功的情况？

遇到 luarocks 慢的问题，有以下两种可能：

1. luarocks 安装所使用的服务器不能访问
2. 你所在的网络到 github 服务器之间有地方对 `git` 协议进行封锁

针对第一个问题，你可以使用 https_proxy 或者使用 `--server` 选项来指定一个你可以访问或者访问更快的
luarocks 服务。 运行 `luarocks config rocks_servers` 命令（这个命令在 luarocks 3.0 版本后开始支持）
可以查看有哪些可用服务。

如果使用代理仍然解决不了这个问题，那可以在安装的过程中添加 `--verbose` 选项来查看具体是慢在什么地方。排除前面的
第一种情况，只可能是第二种，`git` 协议被封。这个时候可以执行 `git config --global url."https://".insteadOf git://` 命令使用 `https` 协议替代。

## 如何通过APISIX支持A/B测试？

比如，根据入参`arg_id`分组：

1. A组：arg_id <= 1000
2. B组：arg_id > 1000

可以这么做：
```shell
curl -i http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "vars": [
        ["arg_id", "<=", "1000"]
    ],
    "plugins": {
        "redirect": {
            "uri": "/test?group_id=1"
        }
    }
}'

curl -i http://127.0.0.1:9080/apisix/admin/routes/2 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/index.html",
    "vars": [
        ["arg_id", ">", "1000"]
    ],
    "plugins": {
        "redirect": {
            "uri": "/test?group_id=2"
        }
    }
}'
```

更多的 lua-resty-radixtree 匹配操作，可查看操作列表：
https://github.com/iresty/lua-resty-radixtree#operator-list

## 如何修改日志等级

默认的APISIX日志等级为`warn`，如果需要查看`core.log.info`的打印结果需要将日志等级调整为`info`。

具体步骤：

1、修改conf/config.yaml中的nginx log配置参数`error_log_level: "warn"`为`error_log_level: "info"`。

2、重启APISIX：`apisix restart`或者是`apisix stop`+`apisix start`

之后便可以在logs/error.log中查看到info的日志了。

## 如何加载自己编写的插件

Apache APISIX 的插件支持热加载，如果你的 APISIX 节点打开了 Admin API，那么对于新增/删除/修改插件等场景，均可以通过调用 HTTP 接口的方式热加载插件，不需要重启服务。

```shell
curl http://127.0.0.1:9080/apisix/admin/plugins/reload -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT
```

如果你的 APISIX 节点并没有打开 Admin API，那么你可以通过手动 reload APISIX 的方式加载插件。

```shell
apisix reload
```

## 项目中 doc/apisix-plugin-design.graffle 文件是什么文件？

它是一个绘图工具， APISIX 开源项目中的很多图都是使用它绘制的，为了图方便就直接放在github上了。

## Apache APISIX 单元测试是使用的什么工具？

test-nginx

## 用 RPM 包方式安装 APISIX 会同时安装 Dashboard 吗

rpm包安装的apisix自动会有dashboard；另一方面1.2版本之后，apisix核心和dashboard是一起发版本的。所以从luarocks和rpm途径安装的apisix是有dashboard

## apisix 命令作用

- `make deps` 是安装lua库相关的依赖
- `make install` 会被废弃掉，你可以忽略它

## 怎么用 Apache APISIX 实现 http 到 https 的跳转

我写了文档和测试案例给大家参考下：https://github.com/apache/incubator-apisix/pull/1595

## apisix start的单例模式

## allow admin的问题这个很多人提到了，我今天看下

## apisix route匹配机制，多条路由规则，有优先顺序吗？

完全匹配 > 前缀匹配（深度优先）> 优先级（深度一样时）

## 如何升级apisix版本？

如何通过rpm包升级apisix
如何通过luarocks升级apisix
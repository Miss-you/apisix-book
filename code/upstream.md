# 前言(为什么会写这篇文章)
本人之前是个API网关小白, 因近期在做API网关相关的开发工作，所以找到了业界最火最牛逼的Apisix, 研读了比较感兴趣的小部分的源码及核心组件的设计思路，受益颇多。 每当遇到疑惑时，@院生老师和@厉辉老师总是会耐心的为我们解惑，在此我深表感谢。 某人曾说过：“高效学习的关键在于知识的输出与分享”, 然从院生老师那了解到厉辉(yousa)老师正在写一本关于apisix的书籍，故加入到了这个开源项目，那么接下来将把之前学习到的东西陆续的输出到这里。

---
# 主题
浅谈upstream的初始化及fetch流

---

## Upstream init
 在讲fetch_upstream核心代码之前，我们先从源头开始，也就是在_M.http_init_worker(init.lua)阶段对upstream的初始化工作做一个简单的了解:

![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstream_init.png)

http_init_worker函数还对router、service等均做了init工作, 背后的工作流大同小异,可选项可配置是否使用ngx.timer做watch&update工作。比如router.init_worker()会返回一个user_routes对象, 当请求过来的时候会先调用router.match(),  在match函数内部会判断缓存的版本是否与user_routes的版本一致，若一致则往下走真正的match流程； 若不一致，会重建[radixtree](https://github.com/api7/lua-resty-radixtree). 感慨一下, radixtree的设计思路和代码真的很牛逼！真心建议大家去研读下,本人也基于此写了一个golang版本的高性能网关路由(基于cgo), 重建radixtree的时需要用到user_routes.values这个结构. 那么关于upstream的初始化就讲到这里了.


## Fetch upstream
我们先来看一下http_access_phase的大致工作流:
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/fetch_upstream.png)

我们可以看到当matched_route.value.upstream_id不为空时会返回本地内存的upstream对象, 核心代码如下:
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstreams_get.png)
![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/config_get.png)

## 总结
 以上就是关于upstream的初始化工作及获取流程的一个简单的解读啦，欢迎大家提出宝贵建议。
# 前言(为什么会写这篇文章)
本人之前是个API网关小白, 因近期在做API网关相关的开发工作，所以找到了业界最火最牛逼的Apisix, 研读了比较感兴趣的小部分的源码及核心组件的设计思路，受益颇多。 每当遇到疑惑时，@院生老师和@厉辉老师总是会耐心的为我们解惑，在此我深表感谢。 某人曾说过：“高效学习的关键在于知识的输出与分享”, 然从院生老师那了解到厉辉(yousa)老师正在写一本关于apisix的书籍，故加入到了这个开源项目，那么接下来将把之前学习到的东西陆续的输出到这里。

---
## 这篇文章主要为大家讲解Apisix是如何获取upstream等信息的。
在讲fetch_upstream核心代码之前，我们先从源头开始，也就是在_M.http_init_worker(init.lua)阶段对upstream的初始化工作做一个简单的了解:

![image](https://raw.githubusercontent.com/rockXiaofeng/apisix-book/upstream_learning/code/images/upstream.png)
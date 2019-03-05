# Serverless往事（五）：注册中心

[上一次][4]讲到了我对未来项目架构的一些设想，提到了几个服务中心。
由于一些现实需求，实现一个注册中心被提上了议程。今天就聊聊注册中心。

## 需求变更

需求变更是一种见怪不怪的常态了，我们回顾一下之前的设计。此前的计划是MT的IVF（嗯，就是我们的内部HR管理系统）和Serverless服务（招聘系统）一一对应；IVF单向调取Serverless的api（安全策略）。

![dispatch][5]

当时我们实现大致是这样的：

* 配置文件写入唯一域名，IVF根据tenant拼接出相应的URL调取cloudfront （${tenant}.XXX.com）

* couldfront分发不同前缀的api至对应的lambda

* 通过couldfront时，添加tenant配置信息到api head里

这个实现只能说是权宜之计，开发重心并不在serverless这块，所以我也没想着花费很多人力去实现过于复杂的架构。当然不和谐的端倪还是开始显现了：

* 新增lambda服务后必须重新部署所有的cloudlfront，不然无法实现新的api分发

* tenant信息被放在了cloudfront配置里，很难扩展

* IVF代码里要写死各个lambda的api前缀才能调用，匪夷所思

还有一个问题我当时是没有考虑到的，cloudfront的域名不唯一该怎么办呢？

是的，需求就是来的这么突然——我们需要支持多个IVF到同一个serverless应用的调用，域名不再唯一。我的第一直觉是，我们**可能**要维护一张巨大的表单——tenant和它的URL。

## 新的思路

// TODO 解释维护tenan表单的难度

IVF的实现成本远高于serverless，我想的还是在serverless里做文章。我们原先的api调用方法如下所示：

```javascript
Post(`https://${tenant}/XXX.com/${service-prefix}/${method}`, data)
```

这里`service-prefix`用于cloudfront分发服务。我后来仔细想了一下，我们并不是非得使用cloudfront的这个功能。假如IVF能直达lambda服务，调用方法就可以简便许多：

```javascript
Post(`https://${service}/${method}`, {...data, tenant})
```

新的设计思路：

1. 实现一个服务中心，用于存储各lambda的endpoint、api前缀以及其他有用信息

2. 服务启动后，lambda主动注册信息到该服务中心
   
3. IVF从注册中心实时拉取服务列表，并驻于内存中

4. IVF直接通过lambda的endpoint调用该服务的api

![Register][6]

这样就是实现了一个简单的服务注册和服务发现的方案。


[4]: https://www.jianshu.com/p/9d6bae2d2d76
[5]: ./draw.io/dispatch.png
[6]: ./draw.io/Register.png
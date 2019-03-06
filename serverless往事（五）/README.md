# Serverless往事（五）：注册中心

[上一次][4]讲到了我对未来项目架构的一些设想，提到了几个服务中心。最近由于一些现实的需求，实现一个注册中心竟被提上了议程。今天就聊聊注册中心。

## 需求变更

需求变更是一种见怪不怪的常态了，我们回顾一下之前的设计。此前的计划是MT的IVF（嗯，就是我们的内部HR管理系统）和Serverless服务（招聘系统）一一对应；IVF单向调取Serverless服务的api（安全策略）。

![dispatch][5]

当时我们的实现是这样的：

* IVF从配置文件里获取cloudfront域名，并根据tenant拼接出相应的URL（${tenant}.XXX.com）

* couldfront分发不同前缀的api至对应的lambda

* 请求通过couldfront时，添加tenant配置到api header里

这个实现只能说是权宜之计，由于开发重心并不在serverless，所以我也没想着花很多人力去实现过于复杂的架构。当然，有些端倪还是能感受到的：

* 新增lambda服务后必须重新部署所有的cloudlfront，不然无法实现新的api分发

* tenant信息被放在了cloudfront配置里，很难扩展

* IVF代码要写死各个lambda的api前缀才能调用，匪夷所思

还有一个问题我当时是没有考虑到的，cloudfront的域名不唯一该怎么办呢？

是的，需求就是来得这么突然——我们需要支持多个IVF到同一个serverless应用的调用，域名不再唯一。我的第一直觉是：**可能**要维护一张巨大的表单——tenant和它的URL。

## 新思路

Tenant数量巨大，说实在这个表单很难维护，写入数据库可能是唯一的选择。怎么增删改查URL，怎么同步跨VPC数据库，这些都是很头疼的事。还有一件更难受的事情是，IVF的实现成本过高，正面临着重构甚至重写的可能；我不想花费太多人力在这里。几度思索后，最后决定在serverless里做文章。

再看一下原先的api调用方法：

```javascript
Post(`https://${tenant}/XXX.com/${service-prefix}/${method}`, data)
```

这里`service-prefix`用于cloudfront分发服务。我后来仔细一想，其实我们并不是非得使用cloudfront这个功能的。假如IVF能直达lambda服务，调用方法就可以简便许多：

```javascript
Post(`https://${service}/${method}`, {...data, tenant})
```

新的设计思路：

1. 实现一个服务注册中心，用于存储各lambda的endpoint、api前缀以及一些其他配置

2. 服务启动后，lambda主动到该中心注册信息
   
3. IVF从注册中心实时拉取服务列表，并驻于内存中（redis）

4. IVF通过lambda的endpoint调用该服务的api

> **注：** endpoint是Web服务入口的URL，映射到地址. 

![Register][6]

## 实现

到此为止，我们勾勒出了一个简单的服务注册和服务发现的方案。下一步是具体实现。设计方案很多，最后根据成本和可扩展性的考量，我选了这个方案[Service Discovery Using Key Value Store][7]，其实就是用lambda+dynamodb造个轮子。代码很简单，只提供`/register`和`/discovery`两个API。

### 服务注册

我们使用的都是lambda function，因此并不需要特地做心跳。秉着一切从简的原则，只需在部署完成后，调用`/health`检测，顺便注册自身的endpoint即可。

![Service Registration][8]

### 服务发现

服务发现的话还需要顾及Serverless自己的Graphql服务。

![Graphql Service][9]

其实从上图可以看出，IVF和Graphql除了分属不同VPC，它们都只是单纯的服务消费者；而lambda则被设计为纯粹的服务提供者。我们把上图抽象处理：

![Service Discovery][10]

* ①、③用于拉取初始参数。这里兼容了以前的配置中心——SSM:parameter store存储了Registry本身的endpoint和一些参数
  
* ②是Provider注册endpoint和一些API信息

* ④是Consumer定时拉取上述endpoint，并在内存里维护一张service-endpoint表单

* ⑤是Consumer最后向Provider发起如下请求

    ```javascript
    Post(`https://${service-api}/${method}`, data)
    ```

### SideCar

上图事实上还有个sidecar没有说，下面我来单独介绍一下。

![SideCar][11]

上图摩托车的边车就是Sidecar。[SideCar pattern][12]是一个很经典的微服务设计模式，是一种将应用功能从应用本身剥离出来作为单独进程的方式。在Service Mesh大火后，SideCar被用来专门处理日志采集、服务监控、熔断控制、消息代理等等这类与主线业务关联不大的功能。

我们的Sidecar自然没那么强大（也没必要），主要功能就是拉取配置信息、维护service-endpoint表单、API组装等一些基础功能。不过好处也是显而易见的：

* 可重用
* 隔离/解耦
* 单一职责

SideCar也有它自己的问题。但确实值得采用，它很好得解决了在几乎不侵入原有代码的基础上，实现服务发现这个功能。

## 小结

最近厂里好多组都把应用放在了lambda上，从实践经验来看lambda就是一个容器，和那些在跑在EC2上的容器并没有本质区别。以前还有人担心 All in AWS全家桶会导致尾大不掉；至少从我现在的代码来看几乎没有AWS的侵入。洋葱架构的实现，为后续替换controller或是持久层留足了余地。AWS全家桶毕竟只是一套工具，应该说绝大多数问题其实是设计造成的，并非来自工具本身。


### 相关播客

* [Serverless往事（一）：从零搭建一个web应用][1]
* [Serverless往事（二）：前后端分离][2]
* [Serverless往事（三）：微服务启航][3]
* [Serverless往事 （四）：空谈架构][4]


[1]: https://www.jianshu.com/p/cd5e87b6b3b4
[2]: https://www.jianshu.com/p/221664ea1367
[3]: https://www.jianshu.com/p/13b316f521cd
[4]: https://www.jianshu.com/p/9d6bae2d2d76
[5]: ./draw.io/dispatch.png
[6]: ./draw.io/Register.png
[7]: https://docs.aws.amazon.com/aws-technical-content/latest/microservices-on-aws/service-discovery.html#service-discovery-using-key-value-store
[8]: ./draw.io/service-registration.png
[9]: ./draw.io/graphql.png
[10]: ./draw.io/Service-Discovery.png
[11]: ./draw.io/sidecar.jpg
[12]: https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar
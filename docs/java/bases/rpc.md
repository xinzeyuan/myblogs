# URL
  - https://zhuanlan.zhihu.com/p/66311537

通俗易懂 RPC、REST、Dubbo、HTTP、RMI 的区别与联系

RPC、REST、Dubbo、HTTP、RMI。接触分布式系统之后，这些概念天天在耳边打转。单拎出来一个，多多少少也能讲一讲它是什么，但是对于这些名词之间的区别和联系却一直没有一个系统性的认识。这个周末，终于是下定决心把它们彻底搞清楚。

RPC 与 REST
同事跟你讲RPC与REST的时候，他心里想的应该是“API设计风格”。这样讲没错，但是不准确。我们先来看这两种“API设计风格”有什么区别：

如果我开了一个小餐馆，想设计一个订餐的API：
![](https://pic1.zhimg.com/80/v2-94f2d3a1a96ce10d0b40fa7d17a78d00_720w.jpg)

两种风格的API区别，总结一下其实非常简单：

RPC面向过程，只发送 GET 和 POST 请求。GET用来查询信息，其他情况下一律用POST。请求参数是动词，直接描述动作本身。
RESTful面向资源，使用 POST、DELETE、PUT、GET 请求，分别对应增、删、改、查操作。请求参数是名词，这个名词就是“增删改查”想要操作的对象。
前面提到，这样对比RPC与REST并不完全准确，原因在于RPC不仅仅是一种API设计风格，它的概念比这要广得多。PRC全称是Remote Procedure Call，即远程过程调用。我发送了一个RPC请求比如 POST /removeItem?itemId=456，实际上是调用了服务端的一个方法 removeItem(int itemId)。在我本地电脑上可以调用一个远在服务端的方法，所以叫远程过程调用。这个"远"的概念也不一定是跨越网络的，同一台主机的两个进程之间相互交流也完全可以是RPC。

# RPC 与 Dubbo
上一小节中提到，RPC虽然经常被当作API设计风格进行讨论，但它更多的是一种通信方式或概念（RPC不是协议！）。而Dubbo是实现了RPC概念的框架，使得调用远程服务就像调用本地方法一样，用过Dubbo的人肯定明白。所以RPC与Dubbo是实现与被实现的关系。我们常说Dubbo是阿里巴巴开源的RPC框架，就是这个道理。

这里需要提到的一点，就是Dubbo框架与Dubbo协议，一开始把我给弄混了。如果只说Dubbo一词，其实是有歧义的，因为Dubbo可以是框架也可以是协议。Dubbo框架默认使用的是Dubbo协议，这个协议是阿里巴巴自己实现的一种应用层协议，传输层还是TCP。所以Dubbo协议与HTTP、FTP，SMTP这些应用层协议是并列的概念。除了默认的Dubbo协议，Dubbo框架还支持RMI、Hessian、HTTP等协议。

# RPC 与 HTTP
之前看到过一个问题说，有了HTTP为什么还要使用RPC？

我猜这个人说的RPC指RPC风格的API设计，又把HTTP与RESTful的概念弄混了。

RPC是一种编程模型的概念，而HTTP是一种应用层协议。我们完全可以基于HTTP协议实现RPC通信，这两者并不矛盾。

# Dubbo、RMI、HTTP
如果将Dubbo与RMI、HTTP并列在一起，那么这个Dubbo就是指Dubbo框架默认的Dubbo协议。这个别再弄混了。

提一句RMI吧。这是一种古老的协议，在JDK1.1中就被实现了，我不知道那个时候RPC概念有没有被提出，但是RMI其实就是一种RPC的实现，比Dubbo要早得多。RMI的通信方式是把Java对象序列化为二进制格式，接收方收到以后再进行反序列化，所以局限性就是用RMI通信的系统都必须是用Java语言编写的。
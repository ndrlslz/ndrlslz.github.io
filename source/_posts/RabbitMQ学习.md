title: RabbitMQ学习
date: 2016-04-25 21:12:51
tags:
---

### AMQP

#### 好处
先谈谈不用amqp的情况，以Java为例，通常使用JMS API来做message client，和一个支持JMS的message broker来传输。 那如果我们需要跨语言的平台来进行交互，比如Java message client和Ruby message client，那就需要一个支持Java OpenWire协议和Ruby STOMP协议的message broker，然后我们找到了支持这种的ActiveMQ。但是如果我们需要在C#和Ruby之间传输呢，就需要一个支持C#的MSMQ和Ruby的STOMP协议的message broker。
![amqp_ruby_C#图](/uploads/amqp_ruby_csharp.png)

<!-- more -->

即使你找到了，在你需要进行平台的切换的时候，又需要为每个message client改变源码，消息库，配置等。 所以可以看出来，厂商之间的封闭性加大了producer和consumer的耦合。


所以有了AMQP(高级消息队列协议)，它提供了一个跨平台的标准的消息协议。你可以使用任意的支持AMQP的message client library和支持AMQP的message broker。比如之间的Java和Ruby的通信，你甚至可以用Qpid的client library来连接Rabbitmq的broker。
![amqp_java_ruby图](/uploads/amqp_java_ruby.png)

这样就对producer和consumer之间进行了解耦，producer和consumer之间无需关心对方的实现方式，producer只需要把消息发给broker，consumer只需要从broker取回消息就好。

----------------

#### 相关概念
![amqp_model图](/uploads/amqp_model.png)

**Virtual Host** 
用来对mq-server进行权限隔离，每个virtual host都有自己的exchange和queue，意味着如果你有多个application需要用自己的mq，可以用同一个mq-server，通过virtual host来隔离他们


**Exchange**
接收producer发来的消息，并转发给binding在exchange上的queue。有4种type的exchange

属性:
>* 持久性: Duration设置可以对exchange持久化
>* 自动删除: exchange所bind的queue删除完后，会自动删除exchange(默认不会自动删除)
>* 惰性: 不会自动创建，需要声明exchange，否则报错


**Queue**
消息队列，用于存储exchange转发过来的消息。

属性:
>* 持久性: Duration设置可以对queue持久化
>* 自动删除: queue会在consumer停止后删除自己。
>* 惰性: 不会自动创建，需要声明queue，否则报错


**Binding**
bingding是将exchange和queue进行绑定，绑定的关键字为route_key。当exchange接收到消息后，它会提取消息的route_key，通过route_key和exchange type将消息路由到相应的queue。

**Channel**
信道，client与broker创建连接后，还需要创建channel才可以通信。

---------------

#### Exchange type
**direct**

![direct图](/uploads/rabbitmq-direct.png)
queue通过route_key与exchange进行绑定，当消息发送到exchange后，检测消息中的route_key，然后发送到用相同route_key进行绑定的queue中，如上图，如果消息中的route_key为orange，则发到Q1 queue，如果是black或green，则发到Q2。
如果exchange绑定的多个queue的route_key相同，则会给每个queue都转发消息。

**fanout**

![fanout图](/uploads/rabbitmq-fanout.png)
这种方式，exchange不会管route_key，它会把消息广播到每个和它连接的queue中。

**topic**

![topic图](/uploads/rabbitmq-topic.png)
这种方式和direct差不多，不同之处在于它支持通配符，"*"匹配单个单词，"#"匹配多个单词，标记之间需要用"."来分割。

---------------

#### 可靠性
**ack**

当consumer从queue取得消息后，但这时consumer宕掉了，这时消息就没了。 所以rabbitmq可以开启autoAck，即当consumer处理完消息后，会发回给queue ack，这时queue才删除掉消息。
如果有多个consumer订阅同一个queue，那消息只会被某一个consumer收到，queue默认采用round robin(轮询)的策略来选择consumer。现在consumerA和consumerB订阅同一个queue，queue先把messageA发给consumerA，consumerA处理完后，返回ack表示处理完成，queue删掉messageA，messageB来了，queue把它分给consumerB，但这时consumerB宕掉了，断开了channel，queue检测到后重新把messageB发给consumerA。

**持久化**
rabbitmq默认是不持久化exchange和queue的，如果server重启了，那这些都会消失。
rabbitmq提供了持久化机制，需要将exchange和queue的durable设置为true，并将producer发送消息是的dilivery mode设置为2。
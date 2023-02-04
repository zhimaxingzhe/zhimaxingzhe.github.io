---
layout: post
title:  "消息中间件基础知识-从RabbitMQ、RocketMQ、Kafka到Pulsar"
categories: MQ
tags:  MQ
author: zhimaxingzhe
link: https://zhimaxingzhe.github.io/
---

* content
{:toc}
> by zhimaxingzhe from [消息中间件基础知识-从RabbitMQ、RocketMQ、Kafka到Pulsar](https://zhimaxingzhe.github.io/2023/01/26/消息中间件基础知识-从RabbitMQ、RocketMQ、Kafka到Pulsar/) 欢迎分享链接，转载请注明出处，尊重版权，若急用请联系授权。 [https://zhimaxingzhe.github.io](https://zhimaxingzhe.github.io)
本文梳理笔者的MQ知识，从消息中间件的基础知识讲起，在有了基础知识后，对市面上各主流的消息中间件进行详细的解析，包括 RabbitMQ、RocketMQ、Kafka、Pulsar，最后再横向对比这几款主流的消息中间件。





# 前言
本文梳理笔者的MQ知识，从消息中间件的基础知识讲起，在有了基础知识后，对市面上各主流的消息中间件进行详细的解析，包括 RabbitMQ、RocketMQ、Kafka、Pulsar，最后再横向对比这几款主流的消息中间件。

介绍MQ的文章网上千千万，最好的学习途径还是官方文档，文中介绍的这几款MQ都在努力推广自己，所以文档在权威性、全面性、专业性、时效性都是无人能及其左右，现在的官网文档甚至自己做竞品比对，比如RocketMQ就自己放了比对表格在首页。所以要学好哪一款MQ，就去看它的官网吧，地址放在文末参考资料中了。

最好的学习方法是带着问题去寻找答案，以费曼学习法为标准，产出可教学的资料，所以本文多是个人的所学梳理和所想记录，个人只是有限，难免有所疏漏，文中有错误和疏漏请不吝赐教，感谢🫶！若有帮助，请一键三连吧🤝。文中许多图片内容是随笔摘抄，若有冒犯，侵删🤝。

![发展历史](https://zhimaxingzhe.github.io/image/mq/前言-发展历史.png)

消息中间件的发展已经有近40年历史，早在上个世纪80年代就诞生了第一款消息队列 The Information Bus。

到90年代 IBM、Oracle、Microsoft 纷纷推出自家的MQ，但都是收费且闭源的产品，主要面向高端的企业用户，这些MQ一般都采用高端硬件，软硬件一体机交付，需要采购专门的维护服务，MQ本身的架构是单机的架构，用户的自主性较差。

进入新世纪后，随着技术成熟，人们开始讨论MQ的协议，诞生了JMS、AMPQ 两大协议标准，随之分别有 ActiveMQ、RabbitMQ的具体实现，并且是开源共建的，这使得这两款MQ在当时迅速流行开来，MQ的使用门槛也随之降低，越来越多系统融入了MQ作为基础能力。

再后来PC互联网、移动互联网的爆发式发展，由于传统的消息队列无法承受亿级用户的访问流量和海量数据传输，诞生了互联网消息中间件，核心能力是全面采用分布式架构、具备很强的横向扩展能力，开源典型代表有 Kafka、RocketMQ、Pulsar。Kafka 的诞生还将消息中间件从Messaging领域延伸到了 Streaming 领域，从分布式应用的异步解耦场景延伸到大数据领域的流存储和流计算场景。Pulsar 更是在 Kafka 之后集大家之成，在企业级应用上做得更好，存储和计算分离的设计使得拓展更加轻松。

如今，IoT、云计算、云原生引领了新的技术趋势。面向IoT的场景，消息队列开始从云内服务端应用通信，延伸到边缘机房和物联网终端设备，支持MQTT等物联网标准协议也成了各大消息队列的标配，我们看到Pulsar、Kafka、RocketMQ 都在努力跟随时代步伐，拓展自己在各种使用场景下的能力。

# 1、消息中间件的定义
在早些年MQ一直被叫做消息队列，就可以定义为传递消息的容器，随着时代的发展，MQ 都在努力拓展出来越来越多的功能，越来越多需求加在MQ纸上，消息中间件的能力越来越强，应用的场景也越来越多，如果非要用一个定义来概括只能是抽象出来一些概念，概括为跨服务之间传递信息的软件。

# 2、用途
## 异步处理
可以把接口请求根据业务的时效性程度，将不紧急的处理逻辑生成消息、事件放到MQ当中，再由专门的系统处理该消息、事件；如日志上报、归档事件、数据推送、数据分析、触发策略、变更推荐、添加积分、发送通知消息等。

## 削峰填谷
作为系统内部的一个消息池，抵抗洪峰，对后端服务起到保护作用。流量洪峰进来的时候，会转换为消息落到MQ当中，后端服务可以根据自己的处理能力来，流量不会直接冲击到后端服务，特别是落库、IO等操作。

## 服务解耦
减少系统、模块之间直接对接带来的耦合，交互统一按MQ中消息的协议，按需生产和消费，耦合程度大大降低。

## 发布订阅
系统产生的行为不需要通过接口等方式来通知到相关服务，只需要发布一次消息，订阅者都能消费到消息，执行服务自身的本职工作。

当然，一切收益都是有代价的，对于系统架构本身来说，会引入新组件，带来系统复杂度的提升，整体系统的可靠性也会是挑战，增加消息中间件的运维成本，还会带来整体系统一致性的问题。所以需要权衡自身系统是否有必要引入MQ，能解决什么痛点，投入产出能否让组织满意，对于本身流量不大的系统来说，保持简单架构是皆大欢喜的事情，毕竟，越简单越稳定，越耐用。

# 3、消息模型
## 队列模型
![消息模型-队列](https://zhimaxingzhe.github.io/image/mq/消息模型-队列.jpg)

一种是消息队列，生产者往队列写消息，消费者从这个队列消费消息，当然生产者可以是多个，消费者也可以是多个，但是一条消息只能被消费一次，具体怎么做的，这就涉及到具体的使用需求和每一款消息中间件的实现了，后面第二部分的时候会涉及到。这是最早的消息模型，这也是为什么消息队列MQ这个名字也一直有人在用吧。


## 订阅模型
![消息模型-发布订阅](https://zhimaxingzhe.github.io/image/mq/消息模型-发布订阅.jpg)

后来上个世纪80年代有人提出发布订阅模式，就是topic模式，生产者发布的消息，消息中间件会把消息投递给每一个订阅者，这个投递的过程有可能是推也可能是拉，支持哪一种也要看每一款的具体实现。

# 4、消息协议
从消息的生命周期来看，消息的生产、存储、消费的整个过程来规范步骤要达到的标准。比如金融级别的消息协议会要求保证消息生产过程中不丢、不重复，存储过程中也能有持久性、一致性的要求，在消费过程中保证消息正确被消费，如不重复、不错位等。 
![消息协议](https://zhimaxingzhe.github.io/image/mq/消息协议.png)
常见的消息协议：  
![消息协议-目录](https://zhimaxingzhe.github.io/image/mq/消息协议-目录.png)

接下来举例 AMPQ 协议的生产、消费过程标准。

## AMQP 协议  
高级消息队列协议（Advanced Message Queuing Protocol），一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同的开发语言等条件的限制。

生产消息
![AMQP-生产消息](https://zhimaxingzhe.github.io/image/mq/AMQP-生产消息.png)
消费消息
![AMQP-消费消息](https://zhimaxingzhe.github.io/image/mq/AMQP-消费消息.png)

## MQTT 协议
MQTT(消息队列遥测传输)是ISO 标准(ISO/IEC PRF 20922)下基于发布/订阅范式的消息协议。它工作在 TCP/IP协议族上，是为硬件性能低下的远程设备以及网络状况糟糕的情况下而设计的发布/订阅型消息协议。

MQTT协议是轻量、简单、开放和易于实现的，这些特点使它适用范围非常广泛。在很多情况下，包括受限的环境中，如：机器与机器（M2M）通信和物联网（IoT）。其在，通过卫星链路通信传感器、偶尔拨号的医疗设备、智能家居、及一些小型化设备中已广泛使用。  

## 其他协议
另外还有STOMP、OpenMessaging等，这里不做展开。当前市面上主流的消息中间件多是有自定义的协议发展起来的，如Kafka 在最开始并不算是一个消息中间件，而是用于日志记录系统的一部分，所以并不是基于某种中间件消息协议来做的，而是基于TCP/IP，根据自定义的消息格式，来传递日志消息，为满足对于消息丢失是有一定容忍度的；在后来逐步发展到可以支持正好一次（Exactly Once）语义，实际上是通过 At Least Once + 幂等性 = Exactly Once 。

将服务器的ACK设置为-1，可以保证Procedure到broker不会丢失数据即 At Least Once；相对的，服务器级别设置为0，可以保证生产者发送消息只会发一次，即At Most Once语义但是，一些非常重要的消息，如交易数据，下游消费者要求消息不重不漏，即 Exactly Once，精准一次，在0.11版本之前，Kafka是无能为力的，只能通过设置ACK=-1，然后业务消费者自己去重。 

0.11版本之后，Kafka引入了幂等性概念，procedure 无论向 broker 发送多少次消息，broker只会持久化一条：At Least Once + 幂等性 = Exactly Once。要启用幂等性，只需要将 procedure 参数中的 enable.idempotence 设置为 true 即可，Kafka 的幂等性实现其实就是将原来在下游做的去重放在了数据上游。开启幂等性的procedure在初始化的时候会分配一个PID，发往同一个 partition 的消息会带一个 Sequence Number，而 broker 端会对<PID, Partition, SeqNumber>做缓存，当相同主键消息提交时，broker 只会持久化一条。  

基于这个理解我们看下 Kafka 的消息报文格式定义，
协议概要：

![AMQP-消费消息](https://zhimaxingzhe.github.io/image/mq/kafka消息协议.jpg)

再展开看Message的定义：
![AMQP-消费消息](https://zhimaxingzhe.github.io/image/mq/kafka通讯协议.jpg)

基于TCP/IP协议，通过定义消息格式，在请求和响应中做可靠性保证。且随着发展在修改协议，比如Timestamp是为了增加时间索引，在 0.10.0 版本后增加的，用于根据时间戳快速查找特定消息的位移值，优化 Kafka 读取历史消息缓慢的问题。

Streaming、Eventing 场景下目前还没有看到有公认消息协议的出现。

往下的篇幅将展开介绍 RabbitMQ、RocketMQ、Kafka、Pulsar 这四款主流消息中间件的基础知识。


# 5、RabbitMQ
基于Erlang语言开发实现，单机性能表现不错，横向拓展能力较弱，可用于吞吐量在万级的系统当中。

## 消息模式
RabbitMQ 支持简单模式、工作队列模式、发布/订阅模式、路由模式、主题模式和RPC模式。 


![RabbitMQ简单模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ简单模式.png)
<center>简单模式</center>  

![RabbitMQ简单模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ队列模式.png)
<center>队列模式</center> 

![RabbitMQ发布订阅模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ发布订阅模式.png)
<center>发布订阅模式</center> 

![RabbitMQ路由模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ路由模式.png)
<center>路由模式</center> 

![RabbitMQ主题模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ主题模式.png)
<center>主题模式</center> 

![RabbitMQ RPC模式](https://zhimaxingzhe.github.io/image/mq/RabbitMQ-RPC模式.png)
<center>RPC模式</center> 

以上所有模式实际上都及基于消息队列来实现的，发布订阅模式和主体模式，也是通过队列来实现的，对交换器绑定后再通过路由规则来分发消息到队列中，也就是BindingKey和RoutingKey，由于RoutingKey不能重复，也就意味着队列收到的消息不能一样，而每条消息只会发送给订阅列表里的一个消费者，从而就是没有消费者组的概念，无法做到真正的发布订阅。带着这个理解看RabbitMQ 架构就会比较清晰了。

## RabbitMQ 架构
![RabbitMQ架构](https://zhimaxingzhe.github.io/image/mq/RabbitMQ架构.png)

上图是单机的架构，那么集群架构是怎么样的呢？

![RabbitMQ高可用架构](https://zhimaxingzhe.github.io/image/mq/RabbitMQ高可用.png)

HA-Proxy一款提供高可用性、负载均衡以及基于TCP和HTTP应用的代理软件，主要是做负载均衡的7层，也可以做4层负载均衡。

Keepalived 是集群管理中保证集群高可用的一个服务软件，其功能类似于heartbeat，用来防止单点故障。

虽然是高可用方案，但总体来说横向扩展能力较弱。

RabbitMQ 控制台的介绍查看官网文档，也可以查看 [Part 3: The RabbitMQ Management Interface](https://www.cloudamqp.com/blog/part3-rabbitmq-for-beginners_the-management-interface.html)。

RabbitMQ就介绍到这里，更多信息可查看官网[RabbitMQ](https://www.rabbitmq.com/) ，接下里介绍 RocketMQ。

# RocketMQ
## 基础概念
![RocketMQ基础概念](https://zhimaxingzhe.github.io/image/mq/RocketMQ基础概念.png)
### Tag
Tag（标签）可以看作子主题，它是消息的第二级类型，用于为用户提供额外的灵活性。使用标签，同一业务模块不同目的的消息就可以用相同 Topic 而不同的 Tag 来标识。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 Tag 。标签有助于保持你的代码干净和连贯，并且还可以为 RocketMQ 提供的查询系统提供帮助。
### Group
RocketMQ中，订阅者的概念是通过消费组（Consumer Group）来体现的。每个消费组都消费主题中一份完整的消息，不同消费组之间消费进度彼此不受影响，也就是说，一条消息被Consumer Group1消费过，也会再给Consumer Group2消费。消费组中包含多个消费者，同一个组内的消费者是竞争消费的关系，每个消费者负责消费组内的一部分消息。默认情况，如果一条消息被消费者Consumer1消费了，那同组的其他消费者就不会再收到这条消息。
### Offset
在Topic的消费过程中，由于消息需要被不同的组进行多次消费，所以消费完的消息并不会立即被删除，这就需要RocketMQ为每个消费组在每个队列上维护一个消费位置（Consumer Offset），这个位置之前的消息都被消费过，之后的消息都没有被消费过，每成功消费一条消息，消费位置就加一。也可以这么说，Queue 是一个长度无限的数组，Offset 就是下标。

## RocketMQ 架构

![RocketMQ架构](https://zhimaxingzhe.github.io/image/mq/RocketMQ架构.png)
RabbitMQ类似有生产阶段、存储阶段、消费阶段，相较RabbitMQ的架构，增加了NameServer集群，横向拓展能力较好。参考的Kafka做的设计，故也同样拥有NIO、PageCache、顺序读写、零拷贝的技能，单机的吞吐量在十万级，横向拓展能力较强，官方声明集群下能承载万亿级吞吐。  

存储阶段，可以通过配置可靠性优先的 Broker 参数来避免因为宕机丢消息，简单说就是可靠性优先的场景都应该使用同步。  

1、消息只要持久化到CommitLog（日志文件）中，即使Broker宕机，未消费的消息也能重新恢复再消费。 

2、Broker的刷盘机制：同步刷盘和异步刷盘，不管哪种刷盘都可以保证消息一定存储在pagecache中（内存中），但是同步刷盘更可靠，它是Producer发送消息后等数据持久化到磁盘之后再返回响应给Producer。 

Broker通过主从模式来保证高可用，Broker支持Master和Slave同步复制、Master和Slave异步复制模式，生产者的消息都是发送给Master，但是消费既可以从Master消费，也可以从Slave消费。同步复制模式可以保证即使Master宕机，消息肯定在Slave中有备份，保证了消息不会丢失。  

Consumer 的配置文件中，并不需要设置是从 Master 读还是从 Slave读，当 Master 不可用或者繁忙的时候， Consumer 的读请求会被自动切换到从 Slave。有了自动切换 Consumer 这种机制，当一个 Master 角色的机器出现故障后，Consumer 仍然可以从 Slave 读取消息，不影响 Consumer 读取消息，这就实现了读的高可用。    

如何达到发送端写的高可用性呢？在创建 Topic 的时候，把 Topic 的多个Message Queue 创建在多个 Broker 组上（相同 Broker 名称，不同 brokerId机器组成 Broker 组），这样当 Broker 组的 Master 不可用后，其他组Master 仍然可用， Producer 仍然可以发送消息。

此架构下的RocketMQ 不支持把Slave自动转成 Master ，如果机器资源不足，需要把 Slave 转成 Master ，则要手动停止 Slave 色的 Broker ，更改配置文件，用新的配置文件启动 Broker。由此，在高可用场景下此问题变得棘手，故需要引入分布式算法的实现，追求CAP，但实践情况是不能同事满足CA的，在互联网场景下较多是在时间BASE理论，优先满足AP，尽可能去满足C。RocketMQ 引入的是实现Raft算法的Dledger，拥有了选举能力，主从切换，架构拓扑图是这样的：
![RocketMQ高可用](https://zhimaxingzhe.github.io/image/mq/RocketMQ高可用.png)

分布式算法中比较常常听到的是 Paxos 算法，但是由于 Paxos 算法难于理解，且实现比较困难，所以不太受业界欢迎。然后出现新的分布式算法 Raft，其比 Paxos 更容易懂与实现，到如今在实际中运用的也已经很成熟，不同的语言都有对其的实现。Dledger 就是其中一个 Java 语言的实现，其将算法方面的内容全部抽象掉，这样开发人员只需要关系业务即可，大大降低使用难度。

## 事务消息

![RocketMQ事务消息](https://zhimaxingzhe.github.io/image/mq/RocketMQ事务消息.png)

1.生产者将消息发送至Apache RocketMQ服务端。

2.Apache RocketMQ服务端将消息持久化成功之后，向生产者返回Ack确认消息已经发送成功，此时消息被标记为"暂不能投递"，这种状态下的消息即为半事务消息。

3.生产者开始执行本地事务逻辑。

4.生产者根据本地事务执行结果向服务端提交二次确认结果（Commit或是Rollback），服务端收到确认结果后处理逻辑如下：

    二次确认结果为Commit：服务端将半事务消息标记为可投递，并投递给消费者。

    二次确认结果为Rollback：服务端将回滚事务，不会将半事务消息投递给消费者。

5.在断网或者是生产者应用重启的特殊情况下，若服务端未收到发送者提交的二次确认结果，或服务端收到的二次确认结果为Unknown未知状态，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。 说明 服务端回查的间隔时间和最大回查次数，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits/)。

6.生产者收到消息回查后，需要检查对应消息的本地事务执行的最终结果。

7.生产者根据检查到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤4对半事务消息进行处理。

事务消息生命周期
![RocketMQ事务消息生命周期](https://zhimaxingzhe.github.io/image/mq/RocketMQ事务消息生命周期.png)

初始化：半事务消息被生产者构建并完成初始化，待发送到服务端的状态。

事务待提交：半事务消息被发送到服务端，和普通消息不同，并不会直接被服务端持久化，而是会被单独存储到事务存储系统中，等待第二阶段本地事务返回执行结果后再提交。此时消息对下游消费者不可见。

消息回滚：第二阶段如果事务执行结果明确为回滚，服务端会将半事务消息回滚，该事务消息流程终止。

提交待消费：第二阶段如果事务执行结果明确为提交，服务端会将半事务消息重新存储到普通存储系统中，此时消息对下游消费者可见，等待被消费者获取并消费。

消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见消费重试。

消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。

消息删除：Apache RocketMQ按照消息保存机制滚动清理最早的消息数据，将消息从物理文件中删除。更多信息，请参见消息存储和[清理机制](https://rocketmq.apache.org/zh/docs/featureBehavior/11messagestorepolicy/)。

## RocketMQ 新发展

![RocketMQ新发展](https://zhimaxingzhe.github.io/image/mq/RocketMQ新发展.png)

在过去“分”往往是技术实现的妥协，而现在“合”才是用户的真正需求。RocketMQ 5.0基于统一Commitlog扩展多元化索引，包括时间索引、百万队列索引、事务索引、KV索引、批量索引、逻辑队列等技术。在场景上同时支撑了RabbitMQ、Kafka、MQTT、边缘轻量计算等产品能力，努力实现“消息、事件、流”的扩展支持，云原生是主流。


更多信息可查看官网 [Apache RocketMQ](https://rocketmq.apache.org/zh/)。


# Kafka

Kafka 是一个分布式系统，由通过高性能TCP 网络协议进行通信的服务器和客户端组成。它可以部署在本地和云环境中的裸机硬件、虚拟机和容器上。

服务器：Kafka 作为一个或多个服务器集群运行，可以跨越多个数据中心或云区域。其中一些服务器形成存储层，称为代理。其他服务器运行 Kafka Connect以事件流的形式持续导入和导出数据，以将 Kafka 与您现有的系统（例如关系数据库以及其他 Kafka 集群）集成。为了让您实现关键任务用例，Kafka 集群具有高度可扩展性和容错性：如果其中任何一台服务器发生故障，其他服务器将接管它们的工作以确保连续运行而不会丢失任何数据。

客户端：它们允许您编写分布式应用程序和微服务，即使在出现网络问题或机器故障的情况下，也能以容错的方式并行、大规模地读取、写入和处理事件流。Kafka 附带了一些这样的客户端，这些客户端由 Kafka 社区提供的 数十个客户端进行了扩充：客户端可用于 Java 和 Scala，包括更高级别的 Kafka Streams库，用于 Go、Python、C/C++ 和许多其他编程语言以及 REST API。


## 架构
![Kafka 架构](https://zhimaxingzhe.github.io/image/mq/kafka架构.png)
与前面两个MQ类似有生产阶段、存储阶段、消费阶段，相比RocketMQ 这里的注册中心是用的Zookeeper，Kafka的诸多事件都依赖于ZK，元数据管理、各个角色的注册、心跳、选举、状态维护，这里的角色包括 Boker、 Topic、 Partition、 消费者组等。  

所以这里也会带来ZK watch 事件压力过大的问题，大量的ZK 节点事件阻塞在队列中, 导致自旋锁, 导致CPU 上升, 由于大量数量事件对象导致占用了大量的内存。

图中的Controller 是 Kakfa 服务端 Broker 的概念，Broker 集群有多台，但只有一台 Broker 可以扮演控制器的角色；某台Broker一旦成为Controller，它用于以下权力：完成对集群成员管理、主题维护和分区的管理，如集群broker信息、Topic维护、Partition维护、分区选举ISR、同步元信息给其他Broker等。

## 存储
![Kafka 存储](https://zhimaxingzhe.github.io/image/mq/kafka存储.png)

topic是逻辑上的概念，而partition是物理上的概念，即一个topic划分为多个partition，每个partition对应一个log文件。 

![Kafka 存储](https://zhimaxingzhe.github.io/image/mq/kafka存储2.png)


.log文件：存储消息数据的文件。    
.index文件：索引文件，记录一条消息在log文件中的位置。  
.snapshot文件：记载着生产者最新的offset。  
.timeindex时间索引文件： 
当前日志分段文件中建立索引的消息的时间戳，是在 0.10.0 版本后增加的，用于根据时间戳快速查找特定消息的位移值，优化 Kafka 读取历史消息缓慢的问题。为了保证时间戳的单调递增，可以将log.message.timestamp.type设置成logApendTime，而CreateTime不能保证是消息写入时间。  

![Kafka 存储](https://zhimaxingzhe.github.io/image/mq/kafka存储3.png)
上图是三个 Broker、两个 topic、两个 partition 的Broker 的存储情况，可以延伸想象一下百万级topic的存储情况会很复杂。

## Rebalnce 问题
为了解决强依赖 Zookeeper 进行 Rebalance 带来的问题，Kafka 引入了 Coordinator 机制。  
首先，触发 Rebalance （再均衡）操作的场景目前分为以下几种：消费者组内消费者数量发生变化，包括：    
- 有新消费者加入
- 有消费者宕机下线，包括真正宕机，或者长时间GC、网络延迟导致消费者未在超时时间内向 GroupCoordinator 发送心跳，也会被认为下线。  
- 有消费者主动退出消费者组（发送 LeaveGroupRequest 请求） 比如客户端调用了 unsubscrible() 方法取消对某些主题的订阅
- 消费者组对应的 GroupCoordinator 节点发生了变化。  
- 消费者组订阅的主题发生变化（增减）或者主题分区数量发生了变化。  
- 节点扩容 

更多信息可查看 Kafka 官网 [Apache Kafka](https://kafka.apache.org/)

# Pulsar
![Pulsar 特点](https://zhimaxingzhe.github.io/image/mq/Pulsar特点.png)

在最高层，一个 Pulsar 实例由一个或多个 Pulsar 集群组成。一个实例中的集群可以在它们之间复制数据。

在 Pulsar 集群中：

一个或多个 broker 处理和负载平衡来自生产者的传入消息，将消息分派给消费者，与 Pulsar 配置存储通信以处理各种协调任务，将消息存储在 BookKeeper 实例（又名 bookies）中，依赖于特定集群的 ZooKeeper 集群用于某些任务等等。
由一个或多个 bookie 组成的 BookKeeper 集群处理消息的持久存储。
特定于该集群的 ZooKeeper 集群处理 Pulsar 集群之间的协调任务。
下图展示了一个 Pulsar 集群：
![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar架构.png)

![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar架构2.png)

Pulsar 用 Apache BookKeeper 作为持久化存储，Broker 持有 BookKeeper client，把未确认的消息发送到 BookKeeper 进行保存。
BookKeeper 是一个分布式的 WAL（Write Ahead Log）系统，Pulsar 使用 BookKeeper 有下面几个便利：
- 可以为 topic 创建多个 ledgers：ledger 是一个只追加的数据结构，并且只有一个 writer，这个 writer 负责多个 BookKeeper 存储节点（就是 Bookies）的写入。Ledger 的条目会被复制到多个 bookies；
- Broker 可以创建、关闭和删除 Ledger，也可以追加内容到 Ledger；
- Ledger 被关闭后，只能以只读状态打开，除非要明确地写数据或者是因为 writer挂掉导致的关闭；
- Ledger 只能有 writer 这一个进程写入，这样写入不会有冲突，所以写入效率很高。如果 writer 挂了，Ledger 会启动恢复进程来确定 Ledger 最终状态和最后提交的日志，保证之后所有 Ledger 进程读取到相同的内容；  
- 除了保存消息数据外，还会保存 cursors，也就是消费端订阅消费的位置。这样所有 cursors 消费完一个 Ledger 的消息后这个 Ledger 就可以被删除，这样可以实现 ledgers 的定期翻滚从头写。

![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar分层和分片.png)

**节点对等**
从架构图可以看出，broker 节点不保存数据，所有 broker 节点都是对等的。如果一个 broker 宕机了，不会丢失任何数据，只需要把它服务的 topic 迁移到一个新的 broker 上就行。  

broker 的 topic 拥有多个逻辑分区，同时每个分区又有多个 segment。  

writer 写数据时，首先会选择 Bookies，比如图中的 segment1。选择了 Bookie1、Bookie2、Bookie4，然后并发地写下去。这样这 3 个节点并没有主从关系，协调完全依赖于 writer，因此它们也是对等的。

**扩展和扩容**
在遇到双十一等大流量的场景时，必须增加 consumer。
这时因为 broker 不存储任何数据，可以方便的增加 broker。broker 集群会有一个或多个 broker 做消息负载均衡。当新的broker 加入后，流量会自动从压力大的 broker 上迁移过来。  

对于 BookKeeper，如果对存储要求变高，比如之前存储 2 个副本现在需要存储 4 个副本，这时可以单独扩展 bookies 而不用考虑 broker。因为节点对等，之前节点的 segment 又堆放整齐，加入新节点并不用搬移数据。writer 会感知新的节点并优先选择使用。

**容错机制**
对于 broker，因为不保存任何数据，如果节点宕机了就相当于客户端断开，重新连接其他的 broker 就可以了。
对于 BookKeeper，保存了多份副本并且这些副本都是对等的。因为没有主从关系，所以当一个节点宕机后，不用立即恢复。后台有一个线程会检查宕机节点的数据备份进行恢复。

![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar分层和分片2.png)

在遇到双十一等大流量的场景时，必须增加 consumer。
这时因为 broker 不存储任何数据，可以方便的增加 broker。broker 集群会有一个或多个 broker 做消息负载均衡。当新的broker 加入后，流量会自动从压力大的 broker 上迁移过来。  

对于 BookKeeper，如果对存储要求变高，比如之前存储 2 个副本现在需要存储 4 个副本，这时可以单独扩展 bookies 而不用考虑 broker。因为节点对等，之前节点的 segment 又堆放整齐，加入新节点并不用搬移数据。writer 会感知新的节点并优先选择使用。


![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar统一的消息存储.png)

Pulsar 可以使用多租户来管理大集群。Pulsar 的租户可以跨集群分布，每个租户都可以有单独的认证和授权机制。租户也是存储配额、消息 TTL 和隔离策略的管理单元。

![Pulsar 架构](https://zhimaxingzhe.github.io/image/mq/Pulsar统一的消息存储2.png)

在和其他组件或者生态对接方面，Pulsar 可以支持很多种消息协议，对于存量系统的MQ首次接入、切换MQ都很方便。

更多信息可查看 Pulsar 官网 [Apache Pulsar](https://pulsar.apache.org/)

# 对比

![对比](https://zhimaxingzhe.github.io/image/mq/对比.png)

此图摘抄自[《面渣逆袭：RocketMQ二十三问》](https://www.cnblogs.com/three-fighter/p/16114162.html)
这个图没有Pulsar的信息，从网上看到的压测报告来看，Pulsar吞吐量大概是Kafka的两倍左右，延迟表现比Kafka低不少，Pulsar 的 I/O 隔离显著优于 Kafka。比较详实的Pulsar和Kafka的比对可以查阅StreamNative的文章[Pulsar和Kafka基准测试：Pulsar性能精准解析（完整版）](https://zhuanlan.zhihu.com/p/435922031)，StreamNative 作为 Apache Pulsar 的商业化公司，数据和结果还是比较可靠的。

# 进阶

说实话介绍MQ的文章网上千千万，最好的学习途径还是官方文档，文中介绍的这几款MQ都在努力推广自己，所以文档在权威性、全面性、专业性、时效性都是无人能及其左右，现在的官网文档甚至自己做竞品比对，比如RocketMQ就自己放了比对表格在首页。所以要学好哪一款MQ，就去看它的官网吧。  

常言道，最好的学习方法是带着问题去寻找答案，在路上捡拾更多果实，增加经验值，快速升级。很多人推荐费曼学习法，以教代学，按可以教别人的标准来学习，最终产出教学内容为目的来学习一个知识，能让自己高效学习。在我看来这很像绩效考核用的OKR工具，为项目设定关键成果，实现成功应该做什么？怎么做？而我写这篇文章是在实践费曼学习法。  

所以，在这里我给出几个问题，读者可以根据自己的兴趣爱好带着问题去寻找答案吧。  

    如何保证消息的可用性/可靠性/不丢失呢？
    如何处理消息重复的问题呢？
    顺序消息如何实现？
    怎么处理消息积压？
    怎么实现分布式消息事务的？半消息？
    如何实现消息过滤？

如果自己平时想到的问题太多，不知道先看哪一个，那么自己想清楚为什么要学这些知识点，哪个问题对于当前的自己收益最大。

# 参考资料
官网文档地址：  
[RabbitMQ 官网文档](https://www.rabbitmq.com/documentation.html)  
[Apache RocketMQ 官网文档](https://rocketmq.apache.org/zh/docs/)  
[Apache Kafka 官网文档](https://Kafka.apache.org/documentation/)  
[Apache Pulsar 官网文档](https://pulsar.apache.org/docs/)  

互联网各文档（只列出了部分，若有疏漏，请指出）：  
[技术盘点：消息中间件的过去、现在和未来](https://mp.weixin.qq.com/s/fqPDodfS4JK5attdgK4eqg)
[Kafka通讯协议指南](https://colobu.com/2017/01/26/A-Guide-To-The-Kafka-Protocol/)  
[新一代消息队列 Pulsar](https://blog.csdn.net/Tencent_TEG/article/details/122551505)  
[zkclient大量节点事件导致CPU飙升](https://www.cnblogs.com/Benjious/p/15086283.html)  
[Part 3: The RabbitMQ Management Interface](https://www.cloudamqp.com/blog/part3-rabbitmq-for-beginners_the-management-interface.html)  
[kafka学习十一-事务消息](https://blog.csdn.net/qq_35930102/article/details/108185765)  
[面渣逆袭：RocketMQ二十三问](https://www.cnblogs.com/three-fighter/p/16114162.html)  
[Pulsar和Kafka基准测试：Pulsar性能精准解析（完整版）](https://zhuanlan.zhihu.com/p/435922031)
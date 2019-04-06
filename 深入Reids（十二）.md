# Redis的发布与订阅

> 在前面的服务器中也略微提到了发布和订阅。所谓的发布和订阅也就相当于我们日常中的关注。主要是由publish，subscribe，psubscribe等组成。

在Redis有两种订阅的方式，一种是直接订阅（subscribe）可以订阅一个或多个频道，另外一种是模式订阅（psubscribe）一个模式可以订阅多个频道。Redis中被订阅的频道可以发布消息（publish）。

接下来就围绕这些内容进行分析，在进入正题之前先从一张图中读懂什么是订阅和模式匹配。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406201747692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

如果频道1发布消息（publish "new.1" "message"），那就会被传到客户端A和C。
## 频道的订阅和退订

> redisServer里的属性 dict *pubsub_channels字典保存频道订阅关系

### 频道的订阅（subscribe）

例如在一个客户端（例如client3）执行  subscribe new2 ，当然也可以选择多个频道。 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406203511400.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

* 如果选的频道有其他订阅者，那就将此客户端加到链表尾部。
* 如果没有任何订阅者，你那就肯定不存在于channels字典里，那就先在字典里创建一个键，并将键的值设置为空，然后在插入到链表尾。

### 频道的退订（unsubscribe）

> 用于解除服务器和客户端的订阅关系

 程序先根据被退定的频道名字来在channels字典里找，如果找到就删除此客户的订阅信息。如果这个频道只有仅此一个订阅者，那么这个订阅者解除订阅后，程序也会把channels中对应的键也删除。

## 模式的订阅和退订

> redisServer中的list *pubsub_patterns保存的是模式订阅关系，是一个链表，每个节点都由redisClient *client（订阅模式的客户端）和robj *pattern（被订阅的模式）。

![模式订阅关系节点](https://img-blog.csdnimg.cn/20190406205724122.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70) 


### 模式的订阅（psubscribe）

和它的定义一样，当有一个新的模式订阅节点被添加时，它就会加到链表的表尾里。

流程：

1. 新建一个pubsubPattern结构，pattern是被订阅的模式，client是订阅模式的客户端。
2. 将pubsubPattern叫入到pubsub_patterns链表的表尾。如下图。

如果当前是客户端2（client2）执行  psubscribe "sport.*"   ，那就如图中一样。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406210441968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)


### 模式的退订（punsubscribe）

> 当一个客户端要退订一个模式或一些模式的话，服务端会在pubsub_patterns中找并删除（当前客户端）订阅结构。

假设当前是client2，比如我们要退订在上一个中的"sport.*" ，它就会从链表中将这个节点删除。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406211541622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

## 消息的发布（publish <channel> <message>）

> 将message发送给channel的所有订阅者，至少有一个模式（pattern）和channel匹配，就将message发送。

### 将消息发个频道订阅者

因为订阅关系都在一个pubsub_channels字典里存着，在channels字典里找出指定的（channel）频道，它是一个链表就像频道的订阅刚开始的那张图，然后它就把消息根据链表一个一个的发送下去。

### 将消息发给模式订阅者

> 在模式订阅中也有一个保存模式订阅信息的链表那就是pubsub_patterns记录了所有的模式订阅关系。

当消息要publish的时候遍历链表找到那些和channel频道相匹配的模式，并将消息发送给订阅了这些模式的客户端。

## 查看订阅信息

> 可以使用pubsub channels，pubsub numsub，pubsub numpat来查看订阅信息

### pubsub channels [pattern]

* 如果不给定参数，那就返回服务器当前被订阅的所有频道。
* 如果给定pattern参数，那就返回和pattern模式匹配的频道。

以文章开头第一张手绘图为例，

pubsub channels ,会打印"new1","new2"，如果有更多的话就全部打印出来。

pubsub channels "new.[12]",会打印"new1"  "new2"。

### pubsub numsub [channel-1,channel-2...-n]

> 可以接受多个频道作为参数,也可以从名字看出是统计订阅了这些模式（channel-*）的个数

以频道的订阅那张图为例：

输入pubsub numsub "news1" "news2"

会回复 "news1" "2" "news2" "1"

### pubsub numpat 

> 返回服务器当前被订阅模式的数量（通俗的说就是服务器有多少个模式被客户端订阅）

以上边的模式的订阅图为例

输入pubsub numpat

回复 "2"

以上边的模式退订图为例

输入pubsub numpat

回复 "1"





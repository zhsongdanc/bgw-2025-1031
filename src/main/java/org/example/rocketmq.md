***\*1.RocketMQ生产者是如何发送消息的\****

***\*2.Broker是如何持久化接收到的消息到磁盘上\****

***\*3.基于DLedger技术的Broker主从同步原理\****

***\*4.消费者进行消息拉取和消费的过程\****

***\*5.消费者从Master或Slave上拉取消息的策略\****

***\*6.基于mmap内存映射实现磁盘文件的高性能读写\****

***\*7.Producer基于队列的消息分发机制\****

***\*8.Broker如何实现高并发消息数据写入\****

***\*9.RocketMQ读写队列的运作原理\****

***\*10.Consumer拉取消息的流程原理分析\****

***\*11.ConsumeQueue如何实现高性能消息读取\****

***\*12.CommitLog基于内存的高并发写入优化\****

***\*13.Broker数据丢失场景以及解决方案\****

***\*14.PageCache内存高并发读写问题和处理\****

***\*15.ConsumeQueue异步写入失败的恢复机制\****

***\*16.Broker写入与读取流程性能优化总结\****

***\*17.RocketMQ读写分离主从漂移设计\****

***\*18.RocketMQ为什么采取惰性读写分离模式\****

***\*19.Broker数据与服务是否都实现高可用了\****

***\*20.Broker基于Raft协议的主从架构设计\****

***\*21.Raft协议的Leader选举算法介绍\****

***\*22.Broker基于DLedger的数据写入流程\****

***\*23.Broker基于Raft协议的主从切换机制\****

***\*24.Consumer消息拉取的挂起机制分析\****

***\*25.Consumer的处理队列与并发消费\****

***\*26.Consumer处理成功后的消费进度管理\****

***\*27.Consumer消息重复消费原理剖析\****

***\*28.Consumer处理失败时的延迟消费机制\****

***\*30.基于RocketMQ全链路的消息零丢失方案总结\****

**
**

简历关键词：深入理解RocketMQ和Kafka的运行原理，并能够根据不同场景选择合适的方案，有效利用消息中间件以提升系统性能。

**
**

***\*1.RocketMQ生产者是如何发送消息的\****

***\*(1)Topic、MessageQueue和Broker之间的关系\****

比如现在有一个Topic，已经为它指定创建了4个MessageQueue，那么这个Topic的数据在Broker集群中是如何分布的呢？由于每个Topic的数据都是分布式存储在多个Broker中的，而为了决定这个Topic的哪些数据放在这个Broker上、哪些数据放在那个Broker上，所以RocketMQ引入了MessageQueue。



一个MessageQueue本质上就是一个数据分片。假设某个Topic有1万条数据，而且这个Topic有4个MessageQueue，那么可以认为会在每个MessageQueue中放入2500条数据。当然这不是绝对的，有可能有的MessageQueue数据多，有的数据少，这要根据消息写入MessageQueue的策略来决定。如果假设每个MessageQueue会平均分配这个Topic的数据，那么每个Broker就会有两个MessageQueue。



通过将一个Topic的数据拆分为多个MessageQueue数据分片，然后在每个Broker上都会存储一部分MessageQueue数据分片，这样就可以实现Topic数据的分布式存储。

**
**

***\*(2)生产者发送消息时写入哪个MessageQueue\****

由于生产者会和NameServer进行通信来获取Topic的路由数据，所以生产者可以从NameServer中得知：一个Topic有几个MessageQueue、每个MessageQueue在哪台Broker机器上。



假设消息写入MessageQueue的策略是：生产者会均匀把消息写入每个MessageQueue。也就是生产者发送20条数据出去，4个MessageQueue都会各自写入5条数据。那么通过这个策略，就可以让生产者把写入请求分散给多个Broker，可以让每个Broker都均匀分摊写入请求压力。如果单个Broker可以抗每秒7万并发，那么两个Broker就可以抗每秒14万并发，这样就实现了RocketMQ集群抗下每秒10万+超高并发。另外通过这个策略，可以让一个Topic中的数据分散在多个MessageQueue中，进而分散在多个Broker机器，从而实现RocketMQ集群分布式存储海量的消息数据。

**
**

***\*(3)如果某个Broker出现故障了怎么办\****

如果某个Broker临时出现故障了，比如Master Broker挂了，那么需要等待其他Slave Broker自动热切换为Master Broker，此时对这一组Broker来说就没有Master Broker可以写入了。



如果按照前面的策略来均匀把数据写入各个Broker上的MessageQueue，那么会导致在一段时间内，每次访问到这个挂掉的Master Broker都会访问失败。对于这个问题，通常来说可以在Producer中开启一个开关，就是sendLatencyFaultEnable。



一旦打开了这个开关，那么会有一个自动容错机制。如果某次访问一个Broker发现网络延迟有500ms，然后还无法访问，那么就会自动回避访问这个Broker一段时间。比如在接下来3000ms内，就不会访问这个Broker了。



这样就可以避免一个Broker故障后，短时间内生产者频繁发送消息到这个故障的Broker上去，出现较多次数的异常。通过sendLatencyFaultEnable开关，生产者会自动回避一段时间不去访问故障的Broker，过段时间再去进行访问。因为过一段时间后，这个故障的Master Broker就已经恢复好了，它的Slave Broker已切换为Master可以正常工作了。

**
**

***\*2.Broker是如何持久化接收到的消息到磁盘上\****

***\*(1)Broker收到的消息会顺序写入CommitLog\****

当生产者的消息发送到一个Broker上时，Broker会对这个消息做什么事情？首先Broker会把这个消息顺序写入磁盘上的一个日志文件CommitLog，也就是直接追加写入这个日志文件的末尾。一个CommitLog日志文件限定最多1GB，如果一个CommitLog日志文件写满了1GB，就会创建另一个新的CommitLog日志文件。所以，磁盘上会有很多个CommitLog日志文件。

**
**

***\*(2)MessageQueue在ConsumeQueue目录下的物理存储位置\****

ConsumeQueue中存储的不只是消息在CommitLog中的offset偏移量，还会包含消息的长度、Tag、HashCode。ConsumeQueue中的一条数据是20字节，每个ConsumeQueue文件会保存30万条数据，所以每个文件大概是5.72MB。



Topic的每个MessageQueue都对应了Broker机器上的多个ConsumeQueue文件，ConsumeQueue文件中保存了对应MessageQueue的所有消息在CommitLog文件中的物理位置(即offset偏移量)。

**
**

***\*(3)如何让消息写入CommitLog文件时的性能几乎等于往内存写入消息时的性能\****

RocketMQ中的Broker会基于OS操作系统的PageCache和顺序写来提升往CommitLog文件写入消息的性能。首先，Broker会以顺序的方式将消息写入到CommitLog文件，也就是每次写入时就是在文件末尾追加一条数据即可，对文件进行顺序写的性能要比随机写的性能高得多。



然后，数据写入CommitLog文件时，不是直接写入底层的物理磁盘文件，而是先进入OS的PageCache内存缓存，然后再由OS的后台线程选一个时间，通过异步的方式将PageCache内存缓冲中的数据刷入到CommitLog磁盘文件。

**
**

***\*3.基于DLedger技术的Broker主从同步原理\****

***\*(1)Broker的高可用架构原理\****

Producer发送消息到Broker后，Broker首先会将消息写入到CommitLog文件中，然后会将这条消息在这个CommitLog文件中的文件偏移量offset，写入到这条消息所属的MessageQueue对应的ConsumeQueue文件中。



如果要让Broker实现高可用，那么必须要有一组Broker：一个Leader Broker写入数据，两个Follower Broker备份数据。当Leader Broker接收到写入请求写入数据后，直接把数据同步给其他的Follower Broker。这样，一条数据就会在三个Broker上有三份副本。此时如果Leader Broker宕机，那么就让其他Follower Broker自动切换为新的Leader Broker，继续接收写入请求。

**
**

***\*(2)基于DLedger技术替换Broker的CommitLog\****

DLedger技术也有一个CommitLog机制。把数据交给DLedger，DLedger就会将数据写入CommitLog文件里。所以，如果基于DLedger技术来实现Broker高可用架构，实际上就是由DLedger来管理CommitLog，替换掉原来由Broker自己来管理CommitLog。同时，使用DLedger来管理CommitLog后，Broker还是可以基于DLedger管理的CommitLog去构建出各个ConsumeQueue文件的。

**
**

***\*(3)DLedger如何基于Raft协议选举Leader\****

既然会使用DLedger替换各个Broker上的CommitLog管理组件，那么每个Broker上都会有一个DLedger组件。如果我们配置了一组Broker，比如有3台机器，那么DLedger会如何从3台机器里选举出一个Leader呢？DLedger是基于Raft协议来进行Leader Broker选举的，那么Raft协议中是如何进行多台机器的Leader选举的呢？



这需要通过三台机器互相投票，然后选出一个Broker作为Leader。简单来说，三台Broker机器启动时，都会给自己投票选自己作为Leader，然后把这个投票发送给其他Broker。



比如Broker01、Broker02、Broker03各自先投票给自己，然后再把自己的投票发送给其他Broker。在第一轮选举中，每个Broker收到所有投票后发现，每个Broker都在投票给自己，所以第一轮选举是失败的。



接着每个Broker都会进入一个随机时间的休眠，比如Broker01休眠3毫秒，Broker02休眠5毫秒，Broker03休眠4毫秒。假设Broker01先苏醒过来，那么当它苏醒过来后，就会继续尝试投票给自己，并且将自己的投票发送给其他Broker。



接着Broker03休眠4毫秒后苏醒，它发现Broker01已经发来了一个投票是投给Broker01的。此时因为它自己还没有开始投票，所以会尊重别人的选择，直接把票投给Broker01了，同时把自己的投票发送给其他Broker。



接着Broker02休眠5毫秒后苏醒，它发现Broker01投票给Broker01，Broker03也投票给Broker01，而此时它自己还没有开始投票，于是也会把票投给Broker01，并且把自己的投票发送给给其他Broker。



最后，所有Broker都会收到三张投票，都是投给Broker01的，那么Broker01就会当选成为Leader。其实只要有(3 / 2) + 1个Broker投票给某个Broker，那么就会选举该Broker为Leader，这个半数加1就是大多数的意思。



以上就是Raft协议中选举Leader算法的简单描述。Raft协议确保可以选出Broker成为Leader的核心设计就是：当一轮选举选不出Leader时，就让各Broker进行随机休眠，先苏醒过来的Broker会投票给自己，其他Broker苏醒后会收到发来的投票，然后根据这些投票也把票投给那个Broker。这样，依靠这个随机休眠的机制，基本上可以快速选举出一个Leader。

**
**

***\*(4)DLedger如何基于Raft协议进行多副本同步\****

Leader Broker收到数据写入请求后，会由DLedger把数据同步给其他Follower Broker。其中，数据同步会分为两个阶段：一是uncommitted阶段，二是commited阶段。



首先Leader Broker上的DLedger会将数据标记为uncommitted状态，然后通过自己的DLedgerServer把uncommitted状态的数据发送给Follower Broker的DLedgerServer。接着Follower Broker的DLedger的DLedgerServer收到uncommitted状态的数据后，会返回一个ACK给Leader Broker的DLedger的DLedgerServer。



如果Leader Broker收到超过半数的Follower Broker返回ACK，那么就将数据标记为committed状态。然后Leader Broker的DLedger的DLedgerServer就会发送commited状态的数据给Follower Broker的DLedger的DLedgerServer，让它们也把数据标记为comitted状态。



以上就是DLedger基于Raft协议实现的Broker多副本同步机制。

**
**

***\*4.消费者进行消息拉取和消费的过程\****

***\*(1)什么是消费者组\****

一般情况下，一条消息进入Broker后，库存系统和营销系统作为两个消费者组，每个组都会拉取到这条消息。也就是说，这个订单支付成功的消息，库存系统会获取到一条，营销系统也会获取到一条，它们俩都会获取到这条消息。一般情况下，库存系统的两台机器中只有一台机器会获取到这条消息，营销系统也是同理。

**
**

***\*(2)集群模式消费 vs 广播模式消费\****

对于一个消费者组而言，它获取到一条消息后，如果消费者组内部有多台机器，到底是只有一台机器可以获取到这个消息，还是每台机器都可以获取到这个消息？这就是集群模式和广播模式的区别。



默认情况下都是集群模式：即一个消费者组获取到一条消息，只会交给组内的一台机器去处理，不是每台机器都可以获取到这条消息的。如果修改为广播模式，那么对于消费者组获取到的一条消息，组内每台机器都可以获取到这条消息。但是相对而言，广播模式用的很少，基本上都是使用集群模式来进行消费的。

**
**

***\*(3)MessageQueue和ConsumeQueue以及CommitLog之间的关系\****

在创建Topic时，需要设置Topic有多少个MessageQueue。Topic中的多个MessageQueue会分散在多个Broker上，一个Broker上的一个MessageQueue会有多个ConsumeQueue文件。但在一个Broker的运行过程中，一个MessageQueue只会对应一个ConsumeQueue文件。



对于Broker而言，存储在一个Broker上的所有Topic的所有MessageQueue数据都会写入一个统一的CommitLog文件，一个Broker收到的所有消息都会往CommitLog文件里面写。



对于Topic的各个MessageQueue而言，则是通过各个ConsumeQueue文件来存储属于MessageQueue的消息在CommitLog文件中的物理地址(即offset偏移量)。

**
**

***\*(4)MessageQueue与消费者的关系\****

大致可以认为一个Topic的多个MessageQueue会均匀分摊给消费者组内的多个机器去消费。这里的一个原则是：一个MessageQueue只能被一个消费者机器去处理，但是一台消费者机器可以负责多个MessageQueue的消息处理。

**
**

***\*(5)Push消费模式 vs Pull消费模式\****

***\**\*一般选择Push消费模式：\*\**\***既然一个消费者组内的多台机器会分别负责一部分MessageQueue的消费的，那么每台机器都必须要连接到对应的Broker，尝试消费里面MessageQueue对应的消息。于是就涉及到两种消费模式了，一个是Push模式、一个是Pull模式。这两个消费模式本质上是一样的，都是消费者机器主动发送请求到Broker机器去拉取一批消息来处理。



Push消费模式是基于Pull消费模式来实现的，只不过它的名字叫做Push而已。在Push模式下，Broker会尽可能实时把新消息交给消费者进行处理，它的消息时效性会更好。



一般我们使用RocketMQ时，消费模式通常都选择Push模式，因为Pull模式的代码写起来更加复杂和繁琐，而且Push模式底层本身就是基于Pull模式来实现的，只不过时效性更好而已。

**
**

***\**\*Push消费模式的实现思路：\*\**\***当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费，那么就马上返回一批消息给消费者处理。消费者处理完之后，会接着发送请求到Broker机器去拉取下一批消息。



所以，消费者机器在Push模式下处理完一批消息，会马上发起请求拉取下一批消息，消息处理的时效性非常好，看起来就像Broker一直不停地推送消息到消费机器一样。



此外，Push模式下有一个请求挂起和长轮询的机制：当拉取消息的请求发送到Broker，Broker却发现没有新的消息可以处理时，就会让处理请求的线程挂起，默认是挂起15秒。然后在挂起期间，Broker会有一个后台线程，每隔一会就检查一下是否有新的消息。如果有新的消息，就主动唤醒被挂起的请求处理线程，然后把消息返回给消费者。



可见，常见的Push消费模式，本质也是消费者不断发送请求到Broker去拉取一批一批的消息。

**
**

***\*(6)Broker如何读取消息返回给消费者\****

Broker在收到消费者的拉取请求后，是如何将消息读取出来，然后返回给消费者的？这涉及到ConsumeQueue和CommitLog。



假设一个消费者发送了拉取请求到Broker，表示它要拉取MessageQueue0中的消息，然后它之前都没拉取过消息，所以就从这个MessageQueue0中的第一条消息开始拉取。



于是，Broker就会找到MessageQueue0对应的ConsumeQueue0，从里面找到第一条消息的offset。接着Broker就需要根据ConsumeQueue0中找到的第一条消息的offset，去CommitLog中根据这个offset读取出这条消息的数据，然后把这条消息的数据返回给消费者。



所以消费者在消费消息时，本质就是：首先根据要消费的MessageQueue以及开始消费的位置offset，去找到对应的ConsumeQueue。然后在ConsumeQueue中读取要消费的消息在CommitLog中的offset偏移量。接着到CommitLog中根据offset读取出完整的消息数据，最后将完整的消息数据返回给消费者。

**
**

***\*(7)消费者处理消息、进行ACK响应和提交消费进度\****

消费者拉取到一批消息后，就会将这批消息传入注册的回调函数，当消费者处理完这批消息后，消费者就会提交目前的一个消费进度到Broker上，然后Broker就会存储消费者的消费进度。



比如现在对ConsumeQueue0的消费进度就是在offset=1的位置，那么Broker会记录下一个ConsumeOffset来标记该消费者的消费进度。这样下次这个消费者组只要再次拉取这个ConsumeQueue的消息，就可以从Broker记录的消费位置开始继续拉取，不用重头开始拉取了。

**
**

***\*5.消费者从Master或Slave上拉取消息的策略\****

***\*(1)消费者什么时候会从Slave Broker上拉取消息\****

Broker在实现高可用架构时会有主从之分。消费者可以从Master Broker上拉取消息，也可以从Slave Broker上拉取消息，具体要看Master Broker的机器负载。



刚开始消费者都是连接到Master Broker机器去拉取消息的，然后如果Master Broker机器觉得自己负载比较高，就会告诉消费者下次可以去Slave Broker拉取消息。

**
**

***\*(2)CommitLog会基于PageCache提升写性能\****

当Broker收到一个消息写入请求时，首先会把消息写入到OS的PageCache，然后OS会有后台线程过一段时间后异步把OS的PageCache中的消息刷入CommitLog磁盘文件中。



依靠这个将消息写入CommitLog时先进入OS的PageCache而不是直接写入磁盘的机制，才可以实现Broker写CommitLog文件的性能是内存写级别的，才可以实现Broker超高的消息写入吞吐量。

**
**

***\*(3)ConsumeQueue会基于PageCache提升性能\****

当消费者发送大量请求给Broker高并发读取消息时，Broker的ConsumeQueue文件的读操作就会变得非常频繁，而且会极大影响消费者拉取消息的性能和吞吐量。因此，ConsumeQueue同样也会基于OS的PageCache来进行优化。即向Broker的ConsumeQueue文件写入消息时，会先写入OS的PageCache。而且OS自己也有一个优化机制，就是读取一个磁盘文件的某数据时会自动把整个磁盘文件的数据缓存到OS的PageCache中。



由于ConsumeQueue文件主要用来存放消息的offset，所以每个ConsumeQueue文件是很小的，30万条消息的offset也就5.72MB而已。因此ConsumeQueue文件不会占用多少空间，它们整体的数据量很小，完全可以被缓存在PageCache中。



这样，当消费者拉取消息时，Broker就可以直接到OS的PageCache里读取ConsumeQueue文件里的内容，其读取性能与读内存时的性能是一样的，从而保证了消费消息时的高性能以及高吞吐。

**
**

***\*(4)CommitLog基于PageCache + 磁盘来一起读\****

当消费者拉取消息时，首先会读OS的PageCache里的少量ConsumeQueue数据，这时的性能是极高的，然后会根据读取到的offset去CommitLog文件里读取完整的消息数据。那么从CommitLog文件里读取完整的消息数据时，既会从OS的PageCache里读取，也会从磁盘里读取。



由于CommitLog文件是用来存放消息的完整数据的，所以它的数据量会很大。毕竟一个CommitLog文件就有1GB，所以整体可能多达几个TB。这么多的CommitLog数据，不可能都放在OS的PageCache里。因为OS的PageCache用的也是机器的内存，一般也就几十个GB而已。何况Broker自身的JVM也要用一些内存，那么留给OS的PageCache的内存就只有一部分罢了，比如10GB~20GB。所以是无法把CommitLog的全部数据都放在OS的PageCache里来提升消息者拉取时的性能的。



也就是说，CommitLog主要是利用OS的PageCache来提升消息的写入性能。当Broker不停写入消息时，会先往OS的PageCache里写，这里可能会累积10GB~20GB的数据。之后OS会自动把PageCache里比较旧的一些数据刷入到CommitLog文件，以腾出空间给新写入的消息。



因此有这样的结论：当消费者向Broker拉取消息时，可以轻松从OS的PageCache里读取到少量ConsumeQueue文件里的offset，这时候的性能是极高的。但当Broker去CommitLog文件里读取完整消息数据时，那么就会有两种可能。



第一种可能：如果读取的是刚刚写入CommitLog文件的消息，那么这些消息大概率还停留在OS的PageCache中。此时Broker可以直接从OS的PageCache里读取完整的消息数据，这时是内存读取，性能会很高。



第二种可能：如果读取的是较早之前写入CommitLog文件的消息，那么这些消息可能早就被刷入磁盘了，已经不在OS的PageCache里了。此时Broker只能从CommitLog文件里读取完整的消息数据了，这时的性能是比较差的。

**
**

***\*(5)何时从PageCache读以及何时从磁盘读\****

如果消费者一直在快速地拉取和消费消息，紧紧的跟上生产者往Broker写入消息的速度，那么消费者每次拉取时几乎都是在拉取最近刚写入CommitLog的消息，这些消息的数据几乎都可以从OS的PageCache里读取到。



如果Broker的负载很高导致消费者拉取消息的速度很慢，或者消费者拉取到一批消息后处理的性能很低导致处理速度很慢，那么都会导致消费者拉取消息的速度跟不上生产者写入消息的速度。



比如生产者都已经写入10万条消息了，结果消费者才拉取2万条消息进行消费。此时可能有5万条最新的消息是在OS的PageCache里，有3万条还没拉取去消费的消息只在磁盘里的CommitLog文件了。那么当消费者再拉取消息时，必然大概率需要从磁盘里的CommitLog文件中读取消息。接着，之前在OS的PageCache里的5万条消息可能又被刷入磁盘了，取而代之的是更加新的几万条消息在OS的PageCache里。当消费者再次拉取时，又会从磁盘里的CommitLog文件中读取那5万条消息，从而形成恶性循环。

**
**

***\*(6)Master Broker什么时候会让消费者从Slave Broker拉取消息\****

假设Broker已经写入了10万条消息，但是消费者仅仅拉取了2万条消息进行消费。那么下次消费者拉取消息时，会从第2万零1条数据开始继续往后拉取，此时Broker还有8万条消息是没有被拉取。



然后Broker知道最多还可以往OS的PageCache里放入多少条消息，比如最多也只能放5万条消息。这时候消费者过来拉取消息，Broker发现该消费者还有8万条消息没有拉取，而这8万是大于内存最多存放的5万。因此Broker便知肯定有3万条消息目前是在磁盘上的，而不在OS的PageCache内存里。于是，在这种情况下，Broker就会告诉消费者，这次会给它从磁盘里读取3万条消息，但下次消费者要去Slave Broker拉取消息了。



其实这个问题的本质就是：将消费者当前没有拉取的消息数量和Broker最多可以存放在OS的PageCache内存里的消息数量进行对比，如果消费者没拉取的消息总大小超过了最大能使用的PageCache内存大小，那么说明后续Broker会频繁从磁盘中加载数据，于是此时Broker就会通知消费者下次要从Slave Broker加载数据了。

**
**

***\*6.基于mmap内存映射实现磁盘文件的高性能读写\****

***\*(1)mmap是Broker读写磁盘文件的核心技术\****

Broker中大量使用了mmap技术去实现CommitLog这种大磁盘文件的高性能读写优化。Broker对磁盘文件的写入主要是通过直接写入OS的PageCache来实现性能优化的。因为直接写入OS的PageCache的性能与写入内存一样，之后OS内核中的线程会异步把PageCache中的数据刷入到磁盘文件，这个过程中就涉及到了mmap技术。

**
**

***\*(2)传统文件IO操作的多次数据拷贝问题\****

如果RocketMQ没有使用mmap技术，而是使用普通的文件IO操作去进行磁盘文件的读写，那么会存在多次数据拷贝的性能问题。假设有个程序需要对磁盘文件发起IO操作，需要读取文件里的数据到程序，那么会经过以下一个顺序：首先从磁盘上把数据读取到内核IO缓冲区，然后从内核IO缓存区里读取到用户进程私有空间，程序才能拿到该文件里的数据。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWEdI38MmP28peoibYBPmy9ibejQYf6eslCen28nHsSXcp2DV3XDMuEHDqg/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了读取磁盘文件里的数据，发生了两次数据拷贝。这就是普通IO操作的一个弊端，必然涉及到两次数据拷贝操作，这对磁盘读写性能是有影响的。如果要将一些数据写入到磁盘文件里去，也是一样的过程。必须先把数据写入到用户进程私有空间，然后从用户进程私有空间再进入内核IO缓冲区，最后进入磁盘文件里。在数据进入磁盘文件的过程中，同样发生了两次数据拷贝。这就是普通IO的问题：有两次数据拷贝。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWE2WYOHfZuP7g2b5UUwoQl6fdfaFYsZr9RuuMeZV21icIk1aGk2vLBo7A/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

***\*(3)RocketMQ如何基于mmap技术 + PageCache技术进行文件读写优化\****

RocketMQ底层对CommitLog、ConsumeQueue之类的磁盘文件的读写操作，基本上都会采用mmap技术来实现。具体到代码层面，第一步就是基于JDK NIO包下的MappedByteBuffer的map()方法：将一个磁盘文件(比如一个CommitLog文件，或者是一个ConsumeQueue文件)映射到内存里来。

**
**

***\**\*关于内存映射：\*\**\***可能有人会误以为是直接把那些磁盘文件里的数据读取到内存中，但这并不完全正确。因为刚开始建立映射时，并没有任何的数据拷贝操作，其实磁盘文件还是停留在那里，只不过map()方法把物理上的磁盘文件的一些地址和用户进程私有空间的一些虚拟内存地址进行了一个映射。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWE639q9NAnxb7YKIk9pZCRaSM4UCl2S9HzsRsyUkKGO3iaRnAtzib0wZ8A/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个地址映射的过程，就是JDK NIO包下的MappedByteBuffer.map()方法做的事情，其底层就是基于mmap技术实现的。另外这个mmap技术在进行文件映射时，一般有大小限制，在1.5GB~2GB之间。所以RocketMQ才让CommitLog单个文件在1GB、ConsumeQueue文件在5.72MB，不会太大。这样限制了RocketMQ底层文件的大小后，就可以在进行文件读写时，很方便的进行内存映射了。

**
**

***\**\*关于PageCache：\*\**\***实际上PageCache在这里就是对应于虚拟内存。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWEjtjQbY3ibP6RfywDOW25oxlHO0DNVVsd3amhZoysSTMWicKLsKeibvAibw/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

***\*(4)基于mmap技术 + PageCache技术来实现高性能的文件读写\****

第二步就可以对这个已经映射到内存里的磁盘文件进行读写操作了。比如程序要写入消息数据到CommitLog文件：首先程序把一个CommitLog文件通过MappedByteBuffer的map()方法映射其地址到程序的虚拟内存地址。接着程序就可以对这个MappedByteBuffer执行写入操作了，写入时消息数据会直接进入PageCache。然后过一段时间后，由OS的线程异步刷入磁盘中。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWEWJHBArLNP51XmU15iaoUZJGtdnAInib3xhzz36w9fcibtEvvXTToaia3qA/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上图可以看出，只有一次数据拷贝的过程，也就是从PageCache里拷贝到磁盘文件。这个就是使用mmap技术后，相比于普通IO的一个性能优化。



接着如果要从磁盘文件里读取数据：那么就会判断一下，当前要读取的数据是否在PageCache里，如果在则可以直接从PageCache里读取。比如刚写入CommitLog的数据还在PageCache里，此时消费者来消费肯定是从PageCache里读取数据的。但如果PageCache里没有要的数据，那么此时就会从磁盘文件里加载数据到PageCache中。而且PageCache技术在加载数据时**，**还会将需要加载的数据块的临近的其他数据块也一起加载到PageCache里。可见在读取数据时，其实也只发生了一次拷贝，而不是两次拷贝，所以这个性能相比于普通IO又提高了。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/DXGTicJyJ8QABwTc840DhtjibvlY4dyPWElE39UPB7gRdMKTKWE6ibzkkYnOI2Kl5xrgf6dgUjThDSN6V6zAN0iaMQ/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

***\*(5)内存预映射机制 + 文件预热机制\****

下面是Broker针对上述磁盘文件高性能读写机制做的一些优化：



优化一.内存预映射机制：Broker会针对磁盘上的各种CommitLog、ConsumeQueue文件预先分配好MappedFile，也就是提前对一些可能接下来要读写的磁盘文件，提前使用MappedByteBuffer执行map()方法完成映射，这样后续读写文件时，就可以直接执行了。



优化二.文件预热：在提前对一些文件完成映射之后，因为映射不会直接将数据加载到内存里来，那么后续在读取CommitLog、ConsumeQueue时，其实有可能会频繁的从磁盘里加载数据到内存中去。所以在执行完map()方法之后，会进行madvise系统调用，就是提前尽可能多的把磁盘文件加载到内存里去。



通过上述优化，才真正能实现这么一个效果：就是写磁盘文件时都是进入PageCache的，保证写入高性能。同时尽可能多的通过map() + madvise的映射后预热机制，把磁盘文件里的数据尽可能多的加载到PageCache里来，后续对ConsumeQueue、CommitLog进行读取时，才能尽可能从内存里读取数据。

**
**

***\*(6)总结\****

Broker在读写磁盘时，会大量把mmap技术和PageCache技术结合起来使用。通过mmap技术减少数据拷贝次数，然后利用PageCache技术实现尽可能优先读写内存，而不是读写物理磁盘。

**
**

***\*7.Producer基于队列的消息分发机制\****

***\*(1)消息是如何发送到Broker去的\****

Producer发送消息时需要指定一个Topic，需要知道Topic里有哪些Queue，以及这些Queue分别分布在哪些Broker上。因此，Producer发送消息到Broker的流程如下：



首先，Producer会从NameServer中拉取Topic的路由信息，然后缓存本地的Topic缓存中，并每隔30s进行刷新。然后，Producer会对获取到的Topic的Queues进行负载均衡，选择其中的一个Queue并找出对应的Broker主节点。最后，Producer才会与这个Broker的主节点进行通信并发送消息过去。

**
**

***\*(2)消息发送失败会如何处理\****

如果Producer往Broker的主节点写入消息失败，那么就会执行重试——重新选择一个Queue和Broker主节点。此外，故障的Broker在一段时间内也不会再次被选中。这就是重试机制 + 故障退避机制。



Broker故障的延迟感知机制：Broker故障后，虽然NameServer会通过心跳 + 定时任务可以感知并摘除掉它，但该故障的Broker信息无需实时通知Producer。通过Producer端的重试机制 + 故障退避机制，就可以让Broker故障时也能做到高可用。

**
**

***\*(3)Producer基于Hash的有序消息分发\****



按照Key进行Hash，比如让orderId相同的消息都进入同一个Queue里，以保证它们的顺序性。



一个Topic会有很多个Queue，这些Queue会落到各个Broker上。默认情况下，Producer往一个Topic写入的数据，会均匀地分散到各个Broker的各个Queue里。所以各个Broker的各个Queue里都会有一些Producer的数据。那么发送到Topic里的数据，是否是有顺序的？其实一个Queue就代表了一个队列，当消息进入到同一个Queue时，这些同一个Queue里的消息便是有顺序的，但是不同的Queue之间的消息则是没有顺序的。



如果需要让某一类数据有一定的特殊顺序性，比如同样orderId对应的多条消息(下单->支付->发券)是有顺序的，那么唯一的选择就是让同样orderId对应的所有消息都进入同一个Queue，并保证它们在同一个Queue里是有顺序的。具体的做法就是：根据消息的某个字段值对应的Hash值，对Queue的数量进行取模，按取模结果来选择Queue。从而实现同样的字段值对应同样的Queue，由此来实现顺序性。

**
**

***\*8.Broker如何实现高并发消息数据写入\****

写消息的方式有两种：一是随机写，二是顺序写。写消息的落地有两种：一是内存，二是磁盘。

**
**

***\*(1)数据持久化\****

一般来说，如果要持久化保存写入的消息数据，消息必须要落地到磁盘。如果消息只落地到内存，为了避免内存里的数据丢失，此时就需要设计一套避免内存里数据不丢失的机制，而且这套机制一般都会基于Write Ahead Log(预写日志)，也是需要写磁盘的。所以写入RocketMQ的消息数据，都会写入到磁盘里来保证持久化存储。

**
**

***\*(2)磁盘的随机写和顺序写\****

磁盘随机写：往磁盘文件里写的数据，其格式都是用户自定义的。用户每次写入数据都需要找到磁盘文件里的某个位置(在磁盘文件里随机寻址)，才能在那个位置里插入最新的数据。磁盘顺序写：每次写入数据都是在一个文件末尾去进行追加，绝对不会随机在文件里寻址来找到一个中间的位置进行插入。磁盘文件随机写的性能：几十ms到几百ms。磁盘文件顺序写的性能：约等同于在内存里随机写数据，毫秒级，低于1ms或几ms。

**
**

***\*(3)内存的随机写和顺序写\****

如果把数据往内存里写，也分顺序写和随机写。内存随机写：内存也是有地址空间的，在内存里随机写需要在内存地址里随机寻址，然后再去插入数据。内存顺序写：在一块连续的内存空间里，顺序地追加数据，避免了内存地址的随机寻址。

**
**

***\*(4)RocketMQ的消息写入\****

首先会将所有消息数据都顺序写入到CommitLog磁盘文件。1个CommitLog的大小为1GB，写满一个CommitLog就切换下一个CommitLog。这时CommitLog便会包含很多种数据，所以需要区分一条消息到底是属于哪个Topic哪个Queue的。



然后RocketMQ会有一个后台线程负责对CommitLog里的数据进行转发。该后台线程会监听CommitLog里的最新数据写入，负责把消息写入到所属的Topic对应的Queue里面。这个Queue本身也是属于一个磁盘文件，而Queue里面的内容则是每条数据在CommitLog磁盘文件里的offset偏移量。



RocketMQ便是通过以上的方式来实现消息数据的高并发写入和存储。

**
**

***\*9.RocketMQ读写队列的运作原理\****

多个Consumer会组成一个消费者组来消费同一个Topic，该Topic里的ReadQueue会均匀分配给同一个消费者组里的各个Consumer。



创建一个Topic时会默认4个WriteQueue和4个ReadQueue。其中WriteQueue会和磁盘里的文件对应起来，属于物理概念，而ReadQueue则属于虚拟概念。通常来说WriteQueue和ReadQueue会一一对应，WriteQueue用于Producer写消息时进行路由，ReadQueue用于Consumer读消息时进行路由。



WriteQueue其实只是对于Producer而言的，实际上只是逻辑上的一个名字而已。在物理上，WriteQueue对应的是ConsumeQueue。RocketMQ设计出WriteQueue和ReadQueue的原因是为了保持灵活性，方便扩容和缩容。



如果创建Topic时配置了4个WriteQueue、8个ReadQueue：由于WriteQueue和ReadQueue是一一对应的，当这8个ReadQueue均匀下发到4个Consumer时，可能会导致某Consumer分到的ReadQueue是完全没有数据的。



如果创建Topic时配置了8个WriteQueue、4个ReadQueue：由于WriteQueue和ReadQueue是一一对应的，可能会导致只有4个WriteQueue有对应的ReadQueue会进行下发到Consumer，剩余4个WriteQueue的数据没法给到Consumer进行消费。

**
**

***\*10.Consumer拉取消息的流程原理分析\****

基于RocketMQ编写Consumer代码时，用户一般会定义一个ConsumerListener回调监听函数，并在函数里添加处理消息的具体逻辑。这样，当Consumer拉取到消息后，会调用用户定义的ConsumerListener回调监听函数，从而实现对消息的处理。



当Consumer分配到Topic的某些ReadQueue之后，会有一个专门的线程处理ReadQueue里的消息。这个线程叫做PullMessageService，它会和对应的Broker建立好网络连接，然后不停地循环拉取ReadQueue里的消息。Consumer和某Broker建立好连接后，便可以感知分配给该Consumer的ReadQueue所映射的WriteQueue里最新的数据。Broker会根据最新数据的offset偏移量，从CommitLog中获取完整的消息数据，最后通过网络发送给Consumer。



Consumer的PullMessageService线程拉取到消息之后，会将消息写入一个叫ProcessQueue的内存数据结构中，这个ProcessQueue数据结构的作用其实是用来对消息进行中转用的。完成中转后，Consumer便能够开辟多个ConsumeMessageThread线程(即消息消费线程)，来对消息进行消费。ConsumeMessageThread线程从ProcessQueue读取到消息后，便会调用用户实现的consumeMessage()方法。这个consumeMessage()方法就是用户自定义ConsumerListener回调监听函数里的方法，里面便是用户对消息的业务逻辑处理。



如果执行用户的业务逻辑处理成功了，便会返回SUCCESS标记给ConsumeMessageThread线程。如果执行用户的业务逻辑处理失败了，便会返回RECONSUME_LATER标记给ConsumeMessageThread线程。



当ConsumeMessageThread线程收到执行用户的业务逻辑处理的SUCCESS标记后，Consumer便会上报某消息处理成功到Broker的WriteQueue，这样下次该消息就不会重复分发给Consumer去消费了。



当ConsumeMessageThread线程收到执行用户的业务逻辑处理的RECONSUME_LATER标记后，Consumer便会上报某消息处理失败到Broker的WriteQueue，这样下次该消息就会继续分发给Consumer去消费。

**
**

***\*11.ConsumeQueue如何实现高性能消息读取\****

***\*(1)ConsumeQueue的物理存储结构设计\****

首先，一个Topic是可以给多个ConsumerGroup去进行消费的。然后，不同的ConsumerGroup对一个ConsumeQueue的消费进度是不一样的。有的ConsumerGroup可能已经消费这个ConsumeQueue的500条消息，有的ConsumerGroup可能只消费100条消息。所以ConsumeQueue需要能够随时根据要消费的消息序号，定位到某条消息在磁盘文件里的不同位置，然后从该位置去进行读取。针对这样的读取需求，ConsumeQueue磁盘文件应该如何设计才能支持高效的磁盘位置定位以及读取？



首先会有一个目录来存放ConsumeQueue磁盘文件，比如~/topicName/queueId/多个磁盘文件，可见每个Topic都会有属于自己的目录。然后每个ConsumeQueue磁盘文件里都会存储一条一条的消息数据，每条消息数据在磁盘文件里存储的内容是：8个字节的offset + 4个字节的消息大小 + 8个字节的Tag的Hash值。



按上述方式设计ConsumeQueue消息内容的好处是定长，可以让ConsumeQueue的每条消息在ConsumeQueue磁盘文件里存储的大小是固定的20字节。这样每个ConsumeQueue磁盘文件最多30万条数据，只占5.72MB大小。当消息定长和磁盘文件固定大小后，就可以方便快速地根据逻辑偏移量在ConsumeQueue里定位和读取某条消息，从而快速地获取物理偏移量，然后再从CommitLog里定位和读取到具体的消息。

**
**

***\*(2)从ConsumeQueue高效读取消息\****

ConsumerGroup读取的消息，都会有一个自己的逻辑offset。逻辑offset可以认为是逻辑上的偏移量，指的是Queue里的第几条消息。ConsumerGroup里的一个Consumer会负责读取某个Queue里的消息。当Consumer从Queue里读取消息时，它会知道需要读取的是这个Queue在逻辑上的某个offset，也就是第几条消息。



因为消息是定长的，而且ConsumeQueue磁盘文件也是固定大小的，以及会最多存放30万条消息。所以当ConsumerGroup的一个Consumer需要读取ConsumeQueue里的某逻辑偏移量offset位置的消息时，就会根据该消息的逻辑偏移量offset，去定位到ConsumeQueue磁盘文件中的位置((offset - 1) * 20字节)作为起始位置，然后再连续读取20字节即可，这样就可以快速地将消息读取出来。



这就是ConsumeQueue的高效率、高性能的读取方式，无需遍历读取磁盘文件里一条一条的消息来进行查找，类似于根据索引去数组定位元素。

**
**

***\*(3)从CommitLog高效读取消息\****

一个CommitLog的大小就是1G。CommitLog的文件名就是这个文件里的第一条消息在整个CommitLog所有文件组成的文件组里的一个总的物理偏移量。也就是Broker会将CommitLog文件里第一条消息的这个总物理偏移量作为该CommitLog的文件名。



当Broker从ConsumeQueue里将消息在CommitLog的偏移量offset + 大小size读取出来后，就可以在所有的CommitLog文件里根据偏移量offset进行二分查找，便能确定该消息是存在于哪个CommitLog文件里。然后用偏移量offset对CommitLog文件名进行减法运算，便能知道消息在该CommitLog文件里的起始位置。最后从该CommitLog文件的对应起始位置开始读取size大小的内容，便能获取到该消息。

**
**

***\*(4)两次随机磁盘读\****

Broker读取一条消息时只需要两次随机磁盘读。第一次的随机磁盘读是针对ConsumeQueue文件，根据逻辑偏移量计算出具体位置后，再进行随机定位的读取。第二次的随机磁盘读是针对CommitLog文件，根据读出的物理偏移量利用二分查找定位具体位置，再进行随机定位的读取。

**
**

***\*12.CommitLog基于内存的高并发写入优化\****

***\*(1)Broker写入性能优化\****

往CommitLog写入消息时会基于磁盘顺序写来提升写入的性能，往ConsumeQueue写入消息时会基于异步转发写入机制来提升写入的性能。

**
**

***\*(2)Broker读取性能优化\****

从ConsumeQueue中读取消息时会基于定长消息 + 定长文件实现消息的一次定位和读取，从CommitLog中读取消息时会基于文件名(第一条消息的总物理偏移量) + 消息偏移量实现消息的一次定位和读取。

**
**

***\*(3)基于内存来提升读取和写入的性能\****

无论是写入还是读取，此时最大的问题就是需要对物理磁盘文件进行顺序写或随机读。所以优化的方向是：基于内存来进一步提升Broker的整体写入性能和读取性能。



实际上，RocketMQ会基于内存来提升CommitLog的写入性能。虽然磁盘顺序写已经比磁盘随机写好很多，但也比内存写差。具体上，Broker会基于MappedFile这种mapping文件内存映射机制，实现把消息数据先写入到内存、再从内存刷到磁盘。MappedFile可以把磁盘文件映射成一个内存空间，这里的内存是操作系统的PageCache。



所以当Broker往CommitLog写入消息时，会先写入到CommitLog磁盘文件映射的操作系统的PageCache中，而PageCache里的消息会在后续通过异步刷盘写入到CommitLog磁盘文件里。

**
**

***\*(4)Broker写入性能优化的总结\****

往CommitLog写入消息时，会通过顺序写内存 + 异步顺序刷磁盘来提升性能。往ConsumeQueue写入消息时，会基于异步转发的写入机制来提升性能。

**
**

***\*13.Broker数据丢失场景以及解决方案\****

***\*(1)Broker数据丢失场景分析\****

***\**\*第一种情况：\*\**\***由于RocketMQ是用Java开发的中间件系统，Broker启动后就是一个JVM进程。所以如果Broker这个JVM进程突然崩溃了，那么此时仅仅是JVM进程没了，其写到PageCache里的数据由于是OS管理的，因此数据不会丢失。这种情况发生的概率还是比较高的。

**
**

***\**\*第二种情况：\*\**\***Broker这个JVM进程所在的服务器故障宕机了，此时就可能导致Broker写入到PageCache里的数据丢失。这种情况发生的概率还是非常低的。

**
**

***\*(2)解决方案\****

将异步刷盘改成同步刷盘：Broker将消息顺序写入PageCache后，就等操作系统将PageCache的数据同步到磁盘文件后再返回响应给Producer。

**
**

***\*14.PageCache内存高并发读写问题和处理\****

***\*(1)PageCache内存高并发读写问题分析\****

如果可以忽略在小概率下的一点点数据丢失问题，将顺序写磁盘文件换成顺序写内存，那么就可以明显提升性能和吞吐量。而且即便是丢失数据，也是默认丢失500ms内的数据。



当Producer和Consumer都在高并发地往Broker写消息和读消息时：PageCache的内存数据可能会出现一个经典的问题，就是RocketMQ里的Broker Busy异常，也就是Broker过于繁忙，这会导致一些操作阻塞甚至失败。



Broker Busy就是在高并发的读写情况下，出现的竞争同一块PageCache数据太频繁太激烈的问题。为了解决这个问题，RocketMQ提供了一个叫TransientStorePoolEnabled的机制。

**
**

***\*(2)基于JVM堆外内存的内存读写分离机制\****

TransientStorePoolEnabled机制，可以理解为瞬时存储池启用机制。如果对Broker的读写压力真的大到出现Broker Busy异常，那么通过开启瞬时存储池，就可以实现内存级别的读写分离模式。



一般而言，在一个服务器上部署一个Java系统后，这个系统会作为一个JVM进程运行在操作系统上。其使用的内存会分为三种：第一种是JVM Heap内存(即JVM管理的堆内存)，第二种是OffHeap内存(即JVM堆外的内存)，第三种是PageCache内存(即由操作系统管理的页缓存)。



所以如果开启了TransientStorePoolEnabled，那么Broker在写消息时，就会先把消息写到JVM OffHeap堆外内存里。然后会有一个额外的后台线程，每隔一段时间定时把JVM OffHeap堆外内存里的数据写入到PageCache中。这样高并发写消息便往JVM OffHeap堆外内存里写，高并发读消息便从操作系统的PageCache中读，从而实现内存级别的读写分离。



所以，RocketMQ为了解决高并发场景下对PageCache竞争读写导致的Broker Busy问题，引入了JVM OffHeap做了缓存分离，实现了内存级别的读写分离，解决了对一块内存空间的写和读出现频繁竞争的问题。

**
**

***\*(3)JVM OffHeap + PageCache的数据丢失问题\****

系统设计里，凡事皆有利弊，没有什么方案是十全十美的。为了解决一个问题，往往会引入新的问题。RocketMQ为了解决高并发场景下对PageCache竞争读写导致的Broker Busy问题，引入了JVM OffHeap做了缓存分离，实现了内存级别的读写分离，解决了对一块内存空间的写和读出现频繁竞争的问题。但这会大大提高数据丢失的风险，数据丢失的情况主要有两种。

**
**

***\**\*情况一：Broker JVM进程关闭\*\**\***。比如Broker崩溃宕机、JVM进程意外退出，或者正常关闭Broker JVM进程进行重启等。由于OffHeap堆外内存是由JVM管理的，所以OffHeap堆外内存的数据此时会丢失，而OS管理的PageCache则不会丢失。

**
**

***\**\*情况二：Broker所在的服务器宕机\*\**\***。此时OffHeap堆外内存和PageCache里的数据都会丢失。



因此没有一个技术方案是完美的，只能抓住当前场景里的主要矛盾。如果是金融级的数据绝对不能丢失，可能就要牺牲性能和吞吐量，让数据的每一次写入都直接刷盘到磁盘文件。如果是大部分的普通情况，数据允许丢一点点，也就在服务器宕机的极端场景下才会丢几百毫秒的数据，保持默认即可。



RocketMQ默认就是只写PageCache + 异步刷盘，如果出现高并发竞争PageCache的问题，那么可以开启写JVM OffHeap。容忍一定的JVM崩溃也丢失一点数据，但实现了利用缓存分离抗高并发读写。

**
**

***\*15.ConsumeQueue异步写入失败的恢复机制\****

***\*(1)ConsumeQueue异步写入消息的两个线程\****

后台线程一：将写入到PageCache的数据异步刷盘到CommitLog磁盘文件里。后台线程二：监听CommitLog磁盘文件的最新数据写入到ConsumeQueue磁盘文件里。

**
**

***\*(2)ConsumeQueue异步写入失败有两种情况\****

***\**\*情况一：\*\**\***消息进入到PageCache时，Broker对应的JVM进程就宕机了，对应上述的两个后台线程便停止工作。此时只要Broker对应的JVM进程重启后，便会继续让上述两个线程处理PageCache里的消息。

**
**

***\**\*情况二：\*\**\***消息进入到CommitLog磁盘文件时，Broker对应的JVM进程就宕机了，对应上述的两个后台线程便停止工作。此时只要Broker对应的JVM进程重启后，负责监听CommitLog磁盘文件的后台线程也会继续处理里面的新增消息。而且Broker重启时也会对比CommitLog的最新数据和ConsumeQueue的最新数据，保证被中断的消息写入能正常恢复。

**
**

***\*16.Broker写入与读取流程性能优化总结\****

***\*(1)写入流程的优化\****

***\**\*默认先写入OS的PageCache后就直接返回成功了，优化成内存级的顺序写：\*\**\***这里会基于MappedFile机制来实现，将磁盘文件映射为一块OS的PageCache内存，让写文件等同于写内存。其中的亮点是基于OS的PageCache来写入数据，Broker的JVM进程崩溃(高概率)是不会导致PageCache的数据丢失的。只有服务器崩溃的小概率极端场景才会导致几百毫秒内写入的数据会丢失，所以丢数据的概率是很低的。

**
**

***\**\*对ConsumeQueue文件和IndexFile文件的写入，是通过异步来进行写入的：\*\**\***虽然将消息写入这两种文件时是异步写的，但只要数据还在CommitLog中没有丢失，那么即便异步写入失败也没影响。

**
**

***\*(2)存储结构的优化\****

***\**\*ConsumeQueue文件的存储结构是为了能够实现高性能的读取而设计的：\*\**\***在ConsumeQueue文件里存储的每条消息都是定长的20字节，每个ConsumeQueue文件满了是30万条消息，约5.72MB。此外，一个Topic目录会有多个MessageQueue目录，一个MessageQueue目录会有多个ConsumeQueue磁盘文件。

**
**

***\**\*CommitLog文件默认满了就是1GB：\*\**\***CommitLog的物理存储结构核心就是其文件名，每一条消息都会有一个在所有CommitLog里的总的物理偏移量，每个文件的名称就是文件里第一条消息在所有CommitLog里的总物理偏移量。

**
**

***\*(3)读取流程的优化\****

***\**\*根据消息的逻辑偏移量offset来定位哪个磁盘文件的哪个物理位置：\*\**\***通过第一次定位，就能找到这条消息在CommitLog里的物理偏移量offset。然后再通过第二次定位，也就是根据物理偏移量利用二分查找去对应的CommitLog中便能读取出该消息。

**
**

***\**\*高并发对PageCache进行读写竞争时可能会出现Broker Busy问题：\*\**\***此时可以通过开启TransientStorePoolEnabled，也就是启用JVM OffHeap堆外内存，来实现内存级的读写分离。

**
**

***\*17.RocketMQ读写分离主从漂移设计\****

***\*(1)优先从Broker主节点消费消息\****

RocketMQ是不倾向让Producer和Consumer进行读写分离的，而是倾向让写和读都由主节点来负责。从节点则用于进行数据复制和同步来实现热备份，如果主节点挂了才会选择从节点进行数据读取。所以Consumer默认下会消费Broker主节点的ConsumeQueue里的消息。

**
**

***\*(2)读写分离主从漂移的规则\****

如果Broker主节点过于繁忙，比如积压了大量写和读的消息超过本地内存的40%，那么当Consumer向Broker主节点发起一次拉取消息的请求后，Broker主节点会通知Consumer下一次去该Broker的某个从节点拉取消息。



而当Consumer向Broker从节点拉取消息一段时间后，从节点发现自己本地消息积压小于本地内存30%，拉取消息很顺利，那么Broker从节点会通知Consumer下一次回到Broker主节点去拉取消息。

**
**

***\*18.RocketMQ为什么采取惰性读写分离模式\****

***\*(1)什么是惰性读写分离模式\****

惰性读写分离其实就是上面说的读写分离主从漂移。这种漂移指的是：主从机器对外提供一个完整的服务，客户端有时候访问主、有时候访问从。惰性读写分离不属于彻底的读写分离，从节点的数据更多时候是用于备用。在以下两种情况下，Consumer才会选择到从节点去读取消费数据。



情况一：如果主节点过于繁忙，积压没有消费的消息太多，已经超过本地内存40%。此时主节点可能出现大量读写线程并发运行，机器运行效率可能已经降低，来不及处理这么多的请求，那么主节点就会让消费请求漂移到从节点去读取消费数据。如果在从节点消费得非常好，消息的积压数量很快下降到从节点本地内存的30%以内，就又会让Consumer漂移回主节点消费。



情况二：如果主节点崩溃了，那么Consumer也只能到从节点去读取数据进行消费。

**
**

***\*(2)MQ要实现真正的读写分离比较麻烦\****

RocketMQ作为一个MQ，一个Topic对应多个Queue，可以认为支持去从节点读取数据进行消费。Kafka作为一个MQ，一个Topic对应多个Partition，不同节点组成Leader和Follower主从结构进行数据复制，不支持去从节点读取数据进行进行。



MQ作为一个特殊的中间件系统，它要维护每个Consumer对一个Queue/Partition的消费进度。如果要实现真正的读写分离，那么维护这个消费进度就会非常麻烦。比如在从节点上进行读取消费时，一旦这个从节点宕机，此时主节点和其他从节点是不知道该从节点的消费进度的，消费进度要进行集中式保存就比较麻烦。所以考虑到消费进度的维护和保存，通常各个MQ都会让消费者在主节点进行读和写，这样就可以简单地对消费进度进行集中式维护和存储。

**
**

***\*(3)Broker从节点每隔10秒同步消费进度\****

由于Consumer向RocketMQ的Broker主节点进行消费时，有时候会漂移到Broker从节点进行消费。所以Broker从节点会每隔10s去Broker主节点进行元数据同步，比如同步给主节点最新的消费进度。由此可见，RocketMQ采用惰性读写分离，主要是为了避免维护不同消费者组去不同从节点消费时产生的复杂的消费进度。

**
**

***\*19.Broker数据与服务是否都实现高可用了\****

***\*(1)RocketMQ4.5.0之前\****

Broker主节点崩溃后，是没有高可用主从切换机制的，从节点只用于热备份，保证大部分的数据不会丢失而已。由于Broker提供的服务就两个：一个是写数据、一个是读数据。所以此时主节点崩溃后，只能靠从节点提供有限的数据和服务了，即只能提供读数据服务而不能提供写数据服务。对应的Producer都会出现写数据失败，但是Consumer可以继续去从节点读取数据进行有限的消费，数据消费完就没了。此外主节点崩溃后，从节点可能存在有些最新的数据没来得及同步过来，出现数据丢失的问题，所以数据和服务没有实现高可用。

**
**

***\*(2)RocketMQ4.5.0之后\****

实现了主从同步 + 主从切换的高可用机制，保证数据和服务都是高可用的。注意：在RocketMQ4.5.0以前的老版本，只是实现了单纯的主从复制，只能做到大部分数据不丢失，效果不是特别好。某个Broker分组内的主节点挂掉后，从节点是没法接管主节点的工作的。

**
**

***\*20.Broker基于Raft协议的主从架构设计\****

如果基于Raft协议，那么一组Broker最少需要三台机器。

**
**

***\**\*当这3台Broker启动后：\*\**\***会基于Raft协议进行Leader选举，选举出的Leader便会成为Broker主节点。

**
**

***\**\*当Producer往Broker主节点发起写请求时：\*\**\***Broker主节点首先会将新消息先写入到OS的PageCache中，接着将新消息同步Push到其余两台从节点。基于Raft协议，只要Broker主节点发现过半数Broker节点(包括它自己)写入新消息成功，那么Broker主节点就可以返回写入成功。而Broker从节点收到主节点的Push新消息请求后，也是首先写入OS的PageCache，然后就直接返回写入成功给Broker主节点。

**
**

***\**\*当Broker主节点宕机后：\*\**\***剩余的两台Broker从节点便会根据Raft协议进行Leader选举，选举其中一台Broker作为新的主节点。这样一个Broker主节点 + 一个Broker从节点，依然可以满足Raft协议，继续提供写服务和保证数据及服务的高可用。

**
**

***\*21.Raft协议的Leader选举算法介绍\****

***\**\*说明一：\*\**\***各个节点在启动时都是Follower。

**
**

***\**\*说明二：\*\**\***每个Follower都会给自己设置一个150ms~300ms之间的一个随机时间，可以理解为一个随机的倒计时时间。也就是说，有的Follower可能倒计时是150ms、有的Follower可能倒计时是200ms，每个Follower的倒计时一般不一样。这时必然会存在一个Follower，它的倒计时是最小的，它会最先到达倒计时的时间。

**
**

***\**\*说明三：\*\**\***第一个完成倒计时的Follower会把自己的身份转变为Candidate，变成一个Leader候选者，会开始竞选Leader。于是它会给自己投票想成为Leader，此外它还会发送请求给其他节点表示它完成了一轮投票，希望对方也投票给自己。

**
**

***\**\*说明四：\*\**\***其他还处于倒计时中的Follower节点收到这个请求后，如果发现自己还没给其他Candidate投过票，那么就把它自己的票投给这个Candidate，并发送请求给其他节点进行通知。如果发现自己已经给其他Candidate投过票，那么就忽略这个Candidate发送过来的请求。

**
**

***\**\*说明五：\*\**\***当某个Candidate发现自己的得票数已超半数quorum，那么它就成为Leader了，这时它会向其他节点发送Leader心跳。那些节点收到这个Leader的心跳后，就会重置自己的倒计时，比如原来的倒计时还剩10ms，收到Leader心跳时就重置为200ms。Follower节点通过Leader的心跳去不断重置自己的倒计时，不让倒计时到达，以此来维持Leader的地位，否则倒计时一到达，它就会从Follower转变为Candidate发起新一轮的Leader选举。

**
**

***\*22.Broker基于DLedger的数据写入流程\****

在Raft协议下，RocketMQ的Leader可以对外提供读和写服务，Follower则一般不对外提供服务，仅仅进行数据复制和同步，以及在Leader故障时完成Leader重新选举继续对外服务。



DLedger是一个实现了Raft协议的框架，它实现了Leader如何选举、数据如何复制、主从如何切换等功能。当Broker拿到一条消息准备写入时，就会切换为基于DLedger来进行写入，不过DLedger里写的不叫消息，而叫日志。



Broker节点的DLedger也是先往PageCache里写日志，然后会有后台线程进行异步刷盘将日志写入磁盘。而Leader节点的DLedger在往PageCache写完日志后，会异步复制日志到其他Follower节点，然后Leader节点会同步阻塞等待这些Follower节点写入日志的结果。当Leader节点发现过半Follower节点写入消息成功后，才会向Producer返回写入成功的响应，代表这条消息写入成功。

**
**

***\*23.Broker基于Raft协议的主从切换机制\****

Broker基于Raft协议的主从切换机制如下：

**
**

***\**\*说明一：\*\**\***Broker组内的各个节点一开始启动时都是Follower状态，都会判断自己是否有收到Leader心跳包。由于刚开始启动时没有Leader，所以各个节点不会收到心跳包，于是都会等待随机倒计时结束，准备切换成Candidate状态。

**
**

***\**\*说明二：\*\**\***其中的一个节点必然会优先结束随机倒计时并切换成Candidate状态。该节点切换成Candidate状态后，就会发起一轮新的选举，也就是给自己进行投票，并且把该投票也发送给其他节点。

**
**

***\**\*说明三：\*\**\***其他节点收到该投票后，就会判断自己是否已给某节点投票。此时这些节点并没有给某节点投过票，并且都还处于Follower状态，其倒计时还没有结束，于是这些节点便会把票投给第一个结束随机倒计时的节点。否则，就忽略该投票。

**
**

***\**\*说明四：\*\**\***第一个结束随机倒计时的节点收到其他节点的投票信息后，会判断投自己的票是否已超半数。如果是，则把Candidate状态切换成Leader状态。

**
**

***\**\*说明五：\*\**\***第一个结束随机倒计时的节点把状态切换成Leader后，就会定时给其他节点发送HeartBeat心跳。其他节点收到心跳后，就会重置倒计时。所以只要Leader正常运行，定时发送心跳过来重置倒计时，那么这些节点的倒计时永远不会结束，从而这些节点会一直维持着Follower状态。

**
**

***\**\*说明六：\*\**\***这样，处于Leader状态的节点，和一直维持Follower状态的那些节点，就会正常工作。Producer的消息会往Leader节点进行写入，然后会被复制到Follower节点。Consumer消费消息也会从Leader节点进行读取，然后其Leader节点的元数据也会定时同步到Follower节点上。之所以Leader节点能一直维持其Leader地位，是因为Leader节点会一直定时给Follower节点发送HeartBeat心跳，然后让这些Follower节点一直在重置自己的随机倒计时，让倒计时永远无法结束。否则，一旦Follower节点的随机倒计时结束了，它就会将自己的状态切换成Candidate，从而发起一轮新的Leader选举。

**
**

***\**\*说明七：\*\**\***假设此时Leader节点崩溃了，比如Broker JVM进程进行了正常的重启。那么该Leader节点就无法给Follower节点定时发送HeartBeat心跳了。于是那些Follower节点便会判断出没有收到心跳包，从而会等待其倒计时结束，切换成Candidate状态。其中必定会有一个Follower节点先结束倒计时切换成Candidate状态，然后发起新的Leader选举。当它发现有过半数节点给自己投票了之后，便会切换成Leader状态，完成主从切换，恢复工作。

**
**

***\**\*说明八：\*\**\***由于新的Leader之前是有完整的消息数据和元数据，所以新Leader只要切换成功，Consumer和Producer继续往新Leader读写即可。新的Leader会继续给其他Follower节点同步数据、定时发送Leader心跳包让Follower节点无法切换成Candidate状态。注意：基于Raft协议的Broker集群，每一组Broker至少需要部署3个节点。

**
**

***\*24.Consumer消息拉取的挂起机制分析\****

Consumer拉取消息时会有两种机制：长轮询机制(Long Polling)和短轮询机制(Short Polling)。Consumer如果没有开启长轮询机制(Long Polling)，那么就会使用短轮询机制(Short Polling)去拉取消息。

**
**

***\*(1)短轮询机制(Short Polling)\****

短轮询指的是短时间(默认1秒)挂起去进行消息拉取，这个1秒可以由shortPollingMillis参数进行控制。当Consumer发起请求去Leader节点拉取消息时，默认会采用短轮询机制。如果Leader节点上处理该请求的线程时，发现没有消息就会挂起1秒，挂起过程中并不会有响应返回给Consumer。1秒后该线程会苏醒，然后再去检查Leader节点是否有消息了，如果还是没有消息，就返回Not Found Message给Consumer。

**
**

***\*(2)长轮询机制(Long Polling)\****

长轮询其实指的就是长时间挂起去进行消息拉取。在开启了长轮询机制的情况下，当Consumer发起请求去Leader节点拉取消息时，如果Leader节点上处理该请求的线程发现没有消息，那么就会直接挂起，挂起过程中并不会有响应返回给Consumer。同时，Leader节点中会有一个长轮询后台线程，每隔5秒去检查Leader节点是否有新的消息进来。如果检查到有新消息则唤醒挂起的线程，并判断该消息是否是Consumer所感兴趣的。如果不是Consumer感兴趣的，则再判断是否长轮询超时，如果超时则返回Not Found Message给Consumer。



当Consumer采用Push模式去拉取消息时，那么会：挂起 + 每隔5秒检查 + 超时时间为15秒，15秒都没拉到消息就超时返回。当Consumer采用Pull模式去拉取消息时，那么会：挂起 + 每隔5秒检查 + 超时时间为20秒，20秒都没拉到消息就超时返回。

**
**

***\*(3)总结Consumer的Push模式和Pull模式\****

实际上，这两个消费模式本质是一样的，都是消费者机器主动发送请求到Broker机器去拉取一批消息来进行处理。Push消费模式底层也是基于消费者Pull模式来实现的，只不过它的名字叫做Push而已。意思是Broker会尽可能实时的把新消息交给消费者机器来进行处理，它的消息时效性会更好。一般使用RocketMQ时，消费模式通常都是选择Push模式，因为Pull模式的代码写起来更加的复杂和繁琐，而且Push模式底层本身就是基于消息拉取的方式来实现的，只不过时效性更好而已。



Push模式的实现思路：当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费就马上返回一批消息到消费机器去处理，处理完之后会接着立刻发送请求到Broker机器去拉取下一批消息。所以消费机器在Push模式下会处理完一批消息，马上发起请求拉取下一批消息，消息处理的时效性非常好，看起来就像Broker一直不停的推送消息到消费机器一样。



此外，Push模式下有一个请求挂起和长轮询的机制：当拉取消息的请求发送到Broker，结果发现没有新的消息给处理时，就会让请求线程挂起，默认是挂起15秒。然后在这个期间，Broker会有一个后台线程，每隔5秒就去检查一下是否有新的消息。另外在这个挂起过程中，如果有新的消息到达了会主动唤醒挂起的线程，然后把消息返回给消费者。

**
**

***\*25.Consumer的处理队列与并发消费\****

***\*(1)PullMessageService线程和ProcessQueue\****

Consumer中负责拉取消息的线程只有一个，就是PullMessageService线程。Consumer从Broker拉取到消息后，会有一个ProcessQueue处理队列，用于进行消息中转。



Consumer的PullMessageService线程拉取到消息后，会将消息写入一个叫ProcessQueue的内存数据结构中，这个ProcessQueue数据结构的作用其实是用来对消息进行中转用的。



由于Consumer负责消费的会是Broker中的某几个ConsumeQueue里的消息，所以Consumer拉取到的ConsumeQueue数据都会写到其内存的某几个ProcessQueue里面。也就是Consumer从Broker中拉取了几个ConsumeQueue的数据，就会对应有几个ProcessQueue，可以理解ProcessQueue和ConsumeQueue之间存在一一对应的映射关系。

**
**

***\*(2)ConsumeMessageThread消息消费线程\****

Consumer把拉取到的消息写入ProcessQueue完成中转后，就会提交消费任务到一个线程池里。通过这个线程池，就可以开辟多个ConsumeMessageThread线程(即消息消费线程)，来对消息进行并发消费。线程池里的每个线程处理消费消息完毕后，就会回调用户自己写代码实现的回调监听处理函数，处理具体业务。

**
**

***\*(3)消费者并发消费总结\****

***\**\*说明一：\*\**\***Consumer在启动时会往Broker注册，会通过RebalanceService组件获取Topic路由信息和ConsumerGroup信息。然后RebalanceService组件会通过负载均衡算法实现Queue到Consumer的分配，确定自己要拉取哪些Queue。

**
**

***\**\*说明二：\*\**\***Consumer在拉取Queue的消息时会有长轮询和短轮询两种模式，默认采用短轮询拉取消息。

**
**

***\**\*说明三：\*\**\***当Consumer拉取到消息后，就会写入在内存中和ConsumeQueue一一对应的ProcessQueue队列，并提交任务到线程池，由线程池里的线程并发地从ProcessQueue获取消息进行处理。

**
**

***\**\*说明四：\*\**\***这些线程从ProcessQueue获取到消息后，就会回调用户实现的回调监听处理函数listener.consumeMessage()。当回调监听处理函数执行完毕后，便会返回SUCCESS给线程，线程便会删除ProcessQueue里的该消息，这样线程又可以继续从ProcessQueue里获取下一条消息进行处理。

**
**

***\*26.Consumer处理成功后的消费进度管理\****

***\*(1)消息被处理完后要提交消费进度\****

当线程池里的线程从ProcessQueue获取到某消息，并回调用户实现的回调监听处理函数listener.consumeMessage()，然后执行成功返回线程SUCCESS后，就可以将该消息从ProcessQueue中删掉了。



当消息从ProcessQueue中删掉后，Consumer需要向Broker的Leader节点提交消息对应的ConsumeQueue的消费进度。因为Broker的Leader节点需要维护和管理：每个ConsumeQueue被各个ConsumeGroup消费的进度。

**
**

***\*(2)消费进度先存本地内存再异步提交\****

当回调监听处理函数返回SUCCESS后，Consumer本地的内存里会存储该Consumer对ConsumeQueue的消费进度。然后Consumer端会有一个后台线程，异步提交这个消费进度到Broker的Leader节点。即Consumer会先将消费进度提交到自己的本地内存里，接着有一个后台线程异步提交消费进度到Leader节点。



Broker的Leader节点收到Consumer提交的消费进度后，也会先存放到自己的内存中。然后Broker也会有一个后台线程将消费进度异步刷入磁盘文件里。

**
**

***\*27.Consumer消息重复消费原理剖析\****

***\*(1)消息被重复消费的消费端原因\****

由于一条消息被消费后，消费进度不管在Consumer端还是在Broker端，都会先进入内存。所以当消费进度还在内存时机器崩溃了或者系统重启，那么就会导致消息重复消费。

**
**

***\*(2)造成消费端重复消费消息的场景\****

主要就是如下两个情景造成消费被重复消息：情景一是Consumer端消费完消息后，消费进度还没进入内存或已经写入内存但还没提交给Broker，机器宕机或系统重启。情景二是Broker端收到Consumer提交的消费进度还没写入内存或刚写入内存，还没刷入磁盘，机器宕机或系统重启。对于Consumer消息重复消费的问题，在Consumer端需要实现一套严格的分布式锁和幂等性保障机制来进行处理。

**
**

***\*28.Consumer处理失败时的延迟消费机制\****

***\*(1)处理失败返回RECONSUME_LATER\****

Consumer端线程池里的线程从ProcessQueue获取到某消息后：如果在回调用户实现的回调监听处理函数listener.consumeMessage()时，消费失败返回了RECONSUME_LATER。那么Consumer也会把ProcessQueue里的这条消息进行删除，然后返回一个处理消息失败的ACK给Broker。

**
**

***\*(2)消息进入RETRY_Topic并检查延迟时间\****

Broker收到这个处理消息失败的ACK后，会对该消息的Topic进行改写，改写成RETRY_Topic_%。该消息也就成为了延迟消息，接着将消息写入到RETRY_Topic_%对应的CommitLog和ConsumeQueue中。然后Broker端会有一个延迟消息的后台线程对改写Topic的ConsumeQueue进行检查，检查里面的消息是否达到延迟时间。其中延迟时间会有多个，并且可以进行配置。

**
**

***\*(3)消息到达延迟时间再次进入原Topic重新消费\****

如果达到延迟时间，就会把该消息取出来再次进行改写Topic，改写为原来的Topic。这样该消息会被写入到原Topic对应的CommitLog对应的CommitLog和ConsumeQueue中，从而让该消息被Consumer在后续的消费中拉取到，进行重新消费。

**
**

***\*29.ConsumerGroup变动时的重平衡机制\****

每当一个ConsumerGroup中少了一个Consumer(机器宕机或重启)、或者多了一个Consumer(新增机器)时，就需要重新分配Topic的那些Queue给Consumer，而这部分工作会由Consumer端的RebalanceService组件完成。RebalanceService组件会每隔20秒去Broker拉取最新的Topic路由信息 + ConsumerGroup信息。



当某个Consumer宕机后，Broker是知道该宕机的Consumer对其负责的ConsumeQueue的消费进度的。所以在最多20秒后，其他Consumer就会进行重新的负载均衡，将宕机Consumer负责的ConsumeQueue分配好。



当ConsumerGroup新增一个Consumer时，由于新增的Consumer会往Broker进行注册，所以Broker能知道新增Consumer。新老Consumer都会每隔20秒拉取最新的Topic路由信息 + ConsumerGroup信息。这样新老Consumer都可以通过RebalanceService重平衡组件重新分配ConsumeQueue。

**
**

***\*30.基于RocketMQ全链路的消息零丢失方案总结\****

***\*(1)对全链路消息零丢失方案进行总结\****

***\**\*发送消息到RocketMQ的零丢失：\*\**\***方案一：同步发送消息 + 反复重试。方案二：事务消息机制。两者都有保证消息发送零丢失的效果，但是经过分析，事务消息方案整体会更好一些。

**
**

***\**\*RocketMQ收到消息之后的零丢失：\*\**\***开启同步刷盘策略 + 主从架构同步机制。只要让一个Broker收到消息后同步写入磁盘，同时同步复制给其他Broker，然后再返回响应给生产者表示写入成功，那么就可以保证RocketMQ自己不会丢失消息。

**
**

***\**\*消费消息的零丢失：\*\**\***RocketMQ的消费者可以保证处理完消息后，才会提交消息的offset给Broker，所以只要注意避免采用多线程异步处理消息时提前提交offset即可。如果想要保证在一条消息基于RocketMQ流转时绝对不会丢失，那么可以采取上述一整套方案。

**
**

***\*(2)消息零丢失方案的优势与劣势\****

优势就是：可以让系统的数据都是正确的，不会丢失数据。劣势就是：会让消息在流转链路中的性能大幅度下降，让消息生产和消费的吞吐量大幅度下降。

**
**

***\*(3)消息零丢失方案会导致吞吐量大幅度下降\****

在发送消息到RocketMQ的环节中，如果生产者仅仅只是简单的把消息发送到RocketMQ。那么不过就是一次普通的网络请求罢了，生产者发送请求到RocketMQ然后接收返回的响应。这个性能自然很高，吞吐量也是很高的。



如果生产者改成了基于事务消息的机制之后，那么此时实现原理如下图示，会涉及到half消息、commit or rollback、写入内部Topic、回调机制等诸多复杂的环节。生产者光是成功发送一条消息，至少要half + commit两次请求。所以当生产者一旦上了如此复杂的方案之后，势必会导致生产者发送消息的性能大幅度下降，从而导致发送消息到RocketMQ的吞吐量大幅度下降。



当Broker收到消息后，一样会让性能大幅度下降。首先RocketMQ的一台Broker机器收到消息后，会直接把消息刷入磁盘，这个性能就远远低于直接写入OS PageCache的性能。写入OS的PageCache相当于是写内存，可能仅仅需要0.1ms，但是写入磁盘文件可能就需要10ms。接着这台Broker还需要把消息复制给其他Broker完成多副本的冗余。这个过程涉及到两台Broker机器之间的网络通信 + 另外一台Broker机器需要写数据到自己本地磁盘，同样会比较慢。在Broker完成了上述两个步骤后，接着才能返回响应告诉生产者这次消息写入已经成功。



由此可见，写入一条消息需要强制同步刷磁盘，而且还需要同步复制消息给其他Broker机器。这两个步骤可能就让原本只要10ms完成的变成100ms完成了。所以也势必会导致性能和吞吐量大幅下降。



当消费者拿到消息之后，比如开启一个子线程去处理这批消息，然后就直接返回CONSUME_SUCCESS状态，接着就可以去处理下一批消息了。如果这样的话，该消费者消费消息的速度会很快，吞吐量也会很高。但为了保证数据不丢失，消费者必须在处理完一批消息后再返回CONSUME_SUCCESS状态。那么消费者处理消息的速度就会降低，吞吐量自然会下降。

**
**

***\*(4)消息零丢失方案到底适合什么场景\****

所以如果系统一定要使用消息零丢失方案，那么必然导致从头到尾的性能下降以及吞吐量下降，因此一般不要轻易在一个业务里使用如此重的一套方案。一般来说，与金钱、交易以及核心数据相关的系统和核心链路，可以使用这套消息零丢失方案。



对于非常核心的场景和少数核心链路的系统，才会建议使用这套复杂的消息零丢失方案。而对于其他大部分非核心的场景和系统，其实即使丢失一些数据，也不会导致太大的问题。此时可以不采取这套方案，或者可以在某些地方做一些简化。



比如可以把事务消息方案退化成同步发送消息 + 反复重试的方案。如果发送消息失败，就重试几次，但是大部分时候可能不需要重试，那么也不会轻易的丢失消息的。最多在该方案里，可能会出现一些数据不一致的问题，因为生产者可能在发送消息前宕机导致没法重试但本地事务已执行。



比如可以把Broker的刷盘策略改为异步刷盘 + 但使用一套主从架构。这样即使一台机器挂了，OS PageCache里的数据丢失了，其他机器上还有数据。不过大部分时候Broker不会随便宕机，那么异步刷盘策略下性能还是很高的。

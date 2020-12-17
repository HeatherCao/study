# Zookeeper是什么？
    ZooKeeper 是一个开源的分布式协调服务。
    
    它是一个为分布式应用提供一致性服务的软件，分布式应用程序可以基于 Zookeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。
    
# Zookeeper提供了什么？

    文件系统和通知机制
    
1.文件系统

    Zookeeper 提供一个多层级的节点命名空间（节点称为 znode）。
    与文件系统不同的是，这些节点都可以设置关联的数据，而文件系统中只有文件节点可以存放数据而目录节点不行。
    
    Zookeeper 为了保证高吞吐和低延迟，在内存中维护了这个树状的目录结构，这种特性使得 Zookeeper 不能用于存放大量的数据，每个节点的存放数据上限为1M。
    
2.Zookeeper 怎么保证主从节点的状态同步？

    Zookeeper 的核心是原子广播机制，这个机制保证了各个 server 之间的同步。实现这个机制的协议叫做 Zab 协议。
    Zab 协议有两种模式，它们分别是恢复模式和广播模式。
    
恢复模式：

    当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数 server 完成了和 leader 的状态同步以后，恢复模式就结束了。
    状态同步保证了 leader 和 server 具有相同的系统状态。
    
广播模式：

    一旦 leader 已经和多数的 follower 进行了状态同步后，它就可以开始广播消息了，即进入广播状态。
    这时候当一个 server 加入 ZooKeeper 服务中，它会在恢复模式下启动，发现 leader，并和 leader 进行状态同步。待到同步结束，它也参与消息广播。ZooKeeper 服务一直维持在 Broadcast 状态，直到 leader 崩溃了或者 leader 失去了大部分的 followers 支持。
    
# zookeeper的Znode有哪几种类型？

    1.临时节点（EPHEMERAL）：临时创建的，会话结束节点自动被删除，也可以手动删除，临时节点不能拥有子节点
    2.临时顺序节点（EPHEMERAL_SEQUENTIAL）：具有临时节点特征，但是它会有序列号，分布式锁中会用到该类型节点
    3.持久节点（PERSISTENT）：创建后永久存在，除非主动删除
    4.持久顺序节点（PERSISTENT_SEQUENTIAL）：该节点创建后持久存在，相对于持久节点它会在节点名称后面自动增加一个10位数字的序列号，这个计数对于此节点的父节点是唯一，如果这个序列号大于2^32-1就会溢出。
    
创建了一个持久节点/module1，且其数据为”module1”

    create /module1 module1
    
创建了一个临时节点 /module1/app1，数据为”app1”

    create -e /module1/app1 app1
    
create加上-s参数，可以创建顺序节点

# Zookeeper Watcher 机制
Zookeeper 允许客户端向服务端的某个 Znode 注册一个 Watcher 监听，当服务端的一些指定事件触发了这个 Watcher，
服务端会向指定客户端发送一个事件通知来实现分布式的通知功能，然后客户端根据 Watcher 通知状态和事件类型做出业务上的改变。

Watcher 机制，总的来说可以分为三个过程：

    客户端注册 Watcher、服务器处理 Watcher 和客户端回调 Watcher
    
Watcher 机制主要包括客户端线程、客户端 WatchManager 和 ZooKeeper 服务器三部分。
 
    - ZooKeeper ：部署在远程主机上的 ZooKeeper 集群，当然，也可能是单机的。 
    - Client ：分布在各处的 ZooKeeper 的 jar 包程序，被引用在各个独立应用程序中。 
    - WatchManager ：一个接口，用于管理各个监听器，只有一个方法 materialize()，返回一个 Watcher 的 set。
    

具体流程上：

    1.客户端在向 ZooKeeper 服务器注册 Watcher 的同时，会将 Watcher 对象存储在客户端的 WatchManager 中。 
    2.当ZooKeeper 服务器触发 Watcher 事件后，会向客户端发送通知
    3.客户端线程从 WatchManager 的实现类中取出对应的 Watcher 对象来执行回调逻辑。   
    
客户端注册 Watcher 实现：

    1. 创建一个 new ZooKeeper（） 客户端对象实例时，可以传入一个 Watcher .
       new ZooKeeper(String connectString,int sessionTimeout, Watcher watcher)  
       这个Watcher 将作为整个 ZooKeeper 回话期间的默认 Watcher，会一直被保存在客户端 ZKWatchManager 的 defaultWatcher 中。 
       
       另外，ZooKeeper 客户端也可以通过 getData、 getChildren 和 exist 三个接口来向 ZooKeeper 服务器注册 Watcher
       
    2.客户端会对当前客户端请求 request 进行标记， 将其设置为 “使用Watcher” 监听。
    同时会封装一个 Watcher 的注册信息，WatchRegistration 对象。 用于暂时保存数据节点的路径 和 Watcher 的对应关系。
    
    3.接着watchRegistration 又会被封装到 Packet 中去， 然后放入发送队列中等待客户端发送。随后，ZooKeeper 客户端就会向服务端发送这个请求，同时等待请求的返回。
    
    （在 ZooKeeper 中 Packet 可以看做是一个最小的通信协议单元，用于进行客户端与服务端之间的网络传输，任何需要传输的对象都需要包装成一个 Packet 对象。）
    
    4.接收来自服务端的响应， finishPacket 方法会从 Packet 中取出对象的 Watcher 并注册到 ZKWatchManager 中进行管理。
    
    5.请求返回，完成注册。

客户端回调 Watcher：

    客户端 SendThread 线程（IO线程，负责客户端和服务器端的数据通信, 也包括事件信息的传输）
    接收事件通知，交由 EventThread 线程（事件处理线程，要在客户端回调注册的 Watchers 进行通知处理）回调 Watcher。
服务端处理 Watcher 实现
    
    1.服务端接收 Watcher 并存储
    接收到客户端请求，处理请求判断是否需要注册 Watcher，需要的话将数据节点的节点路径和 ServerCnxn
    （ServerCnxn 代表一个客户端和服务端的连接，实现了 Watcher 的 process 接口，此时可以看成一个 Watcher 对象）存储在WatcherManager 的 WatchTable 和 watch2Paths 中去。  
    
    WatchManager 是 ZK 服务端 Watcher 的管理者，其内部管理的 watchTable 和 watch2Paths 两个存储结构，分别用两个维度对 Watcher 进行存储。
    watchTable 从数据节点路径的粒度来托管 Watcher。
    watch2Paths 从 Watcher 的粒度来控制事件触发需要触发的数据节点。
    
    2.watcher触发
        ①封装 WatchedEvent，将通知状态、事件类型以及节点路径封装成一个 WatchedEvent 对象
        ②查询Watcher
            没找到；说明没有客户端在该数据节点上注册过 Watcher
            找到；提取并从 WatchTable 和 Watch2Paths 中删除对应 Watcher（从这里可以看出 Watcher 在服务端是一次性的，触发一次就失效了）
            
    3.调用 process 方法来触发 Watcher
        这里 process 主要就是通过 ServerCnxn 对应的 TCP 连接发送 Watcher 事件通知。
    
Watcher 特性总结
    
    1.
    无论是服务端还是客户端，一旦一个 Watcher 被触发 ，Zookeeper 都会将其从相应的存储中移除。
    这样的设计有效的减轻了服务端的压力，不然对于更新非常频繁的节点，服务端会不断的向客户端发送事件通知，无论对于网络还是服务端的压力都非常大。
    
    2.客户端串行执行
    客户端 Watcher 回调的过程是一个串行同步的过程。
    
    3.轻量
    Watcher 通知非常简单，只会告诉客户端发生了事件，而不会说明事件的具体内容。
    给客户端的通知里只会告诉你通知状态（KeeperState），事件类型（EventType）和路径（Path）。不会告诉你原始数据和更新过后的数据！
    客户端向服务端注册 Watcher 的时候，并不会把客户端真实的 Watcher 对象实体传递到服务端，仅仅是在客户端请求中使用 boolean 类型属性进行了标记。
    
    4.watcher event 异步发送 watcher 的通知事件从 server 发送到 client 是异步的，这就存在一个问题，不同的客户端和服务器之间通过 socket 进行通信，由于网络延迟或其他因素导致客户端在不通的时刻监听到事件，由于 Zookeeper 本身提供了 ordering guarantee，
    即客户端监听事件后，才会感知它所监视 znode发生了变化。所以我们使用 Zookeeper 不能期望能够监控到节点每次的变化。Zookeeper 只能保证最终的一致性，而无法保证强一致性。
    
# Zookeeper ACL 权限控制机制

    权限 Permission
    （1）CREATE：数据节点创建权限，允许授权对象在该 Znode 下创建子节点
    （2）DELETE：子节点删除权限，允许授权对象删除该数据节点的子节点
    （3）READ：数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或子节点列表等
    （4）WRITE：数据节点更新权限，允许授权对象对该数据节点进行更新操作
    （5）ADMIN：数据节点管理权限，允许授权对象对该数据节点进行 ACL 相关设置操作
    
    身份的认证有4种方式：
        world：默认方式，相当于全世界都能访问,是一种特殊的 digest 模式，只有一个权限标识“world:anyone”
        auth：代表已经认证通过的用户(cli中可以通过addauth digest user:pwd 来添加当前上下文中的授权用户)
        digest：即用户名:密码这种方式认证，这也是业务系统中最常用的
        ip：使用Ip地址认证. 示例：setAcl /test2 ip:192.168.60.130:crwda // 将节点权限设置为Ip:192.168.60.130
        
# chroot 客户端命名空间

    3.2.0 版本后，添加了 Chroot 特性，该特性允许每个客户端为自己设置一个命名空间。
    如果一个客户端设置了 Chroot，那么该客户端对服务器的任何操作，都将会被限制在其自己的命名空间下。
    
    通过设置 Chroot，能够将一个客户端应用于 Zookeeper 服务端的一颗子树相对应，在那些多个应用公用一个 Zookeeper 进群的场景下，
    对实现不同应用间的相互隔离非常有帮助。

设置命名空间：

    在使用ZooKeeper构造方法时，用户传入的ZooKeeper服务器地址列表，即connectString参数，通常是这样一个使用英文状态逗号分隔的多个IP地址和端口的字符串：
    
    192.168.0.1:2181,192.168.0.1:2181,192.168.0.1:2181
    
    客户端可以通过在connecString中添加后缀的方式来设置Chroot，如下所示：
    
    192.168.0.1:2181,192.168.0.1:2181,192.168.0.1:2181/apps/X
    
    这个client的chrootPath就是/apps/X
    将这样一个connectString传入客户端的ConnectStringParser后就能够解析出Chroot并保存在chrootPath属性中。
    
# zookeeper服务器角色有哪些？

    1.Leader
    事务请求的唯一调度者和处理者。保证事务处理的顺序性
    事务请求：导致数据一致性的请求（数据发生改变）。如删除一个节点、创建一个节点、设置节点数据，设置节点权限就是一个事物请求，全局的事物id（zxid）只能由leader来分配
    集群内部各服务的调度者
    
    2.Follower
    （1）处理客户端的非事务请求，转发事务请求给 Leader 服务器
    （2）参与事务请求 Proposal 的投票
    （3）参与 Leader 选举投票
    
    3.Observer
    （1）3.0 版本以后引入的一个服务器角色，在不影响集群事务处理能力的基础上提升集群的非事务处理能力
    （2）处理客户端的非事务请求，转发事务请求给 Leader 服务器
    （3）不参与任何形式的投票
    在zk的配置server.1=master:2888:3888后面加:observer以后就是一个Observer
    server.1=master:2888:3888:observer

# zookeeper Proposal流程

    每个事务请求都需要集群中过半机器投票认可才能被真正应用到内存数据库中，这个投票与统计过程就是Proposal流程。
     · 发起投票。若当前请求是事务请求，Leader会发起一轮事务投票，在发起事务投票之前，会检查当前服务端的ZXID是否可用。
     · 生成提议Proposal。若ZXID可用，Zookeeper会将已创建的请求头和事务体以及ZXID和请求本身序列化到Proposal对象中，此Proposal对象就是一个提议。
     · 广播提议。Leader以ZXID作为标识，将该提议放入投票箱outstandingProposals中，同时将该提议广播给所有Follower。
     · 收集投票。Follower接收到Leader提议后，进入Sync流程进行日志记录，记录完成后，发送ACK消息至Leader服务器，Leader根据这些ACK消息来统计每个提议的投票情况，当一个提议获得半数以上投票时，就认为该提议通过，进入Commit阶段。
     · 将请求放入toBeApplied队列中。
     · 广播Commit消息。Leader向Follower和Observer发送COMMIT消息。向Observer发送INFORM消息，向Leader发送ZXID。
     

# Zookeeper 下 Server 工作状态？

    （1）LOOKING：寻 找 Leader 状态。当服务器处于该状态时，它会认为当前集群中没有 Leader，因此需要进入 Leader 选举状态。
    （2）FOLLOWING：跟随者状态。表明当前服务器角色是 Follower。
    （3）LEADING：领导者状态。表明当前服务器角色是 Leader。
    （4）OBSERVING：观察者状态。表明当前服务器角色是 Observer。
     
# Zookeeper 数据同步

    整个集群完成 Leader 选举之后，Learner（Follower 和 Observer 的统称）会向Leader 服务器进行注册。
    当 Learner 服务器向Leader 服务器完成注册后，进入数据同步环节。
    
数据同步流程：（均以消息传递的方式进行）

    1.Learner 向 Learder 注册
    2.数据同步
    3.同步确认
    
在进行数据同步前，Leader服务器会完成数据同步初始化：
    
    peerLastZxid：从learner服务器注册时发送的ACKEPOCH消息中提取lastZxid（该Learner服务器最后处理的ZXID）
    minCommittedLog：Leader服务器Proposal缓存队列committedLog中最小ZXID
    maxCommittedLog：Leader服务器Proposal缓存队列committedLog中最大ZXID
    
Zookeeper 的数据同步通常分为四类：

    1.直接差异化同步（DIFF 同步）:peerLastZxid介于minCommittedLog和maxCommittedLog之间
    
    2.先回滚再差异化同步（TRUNC+DIFF 同步）:
    当新的Leader服务器发现某个Learner服务器包含了一条自己没有的事务记录，那么就需要让该Learner服务器进行事务回滚–回滚到Leader服务器上存在的，同时也是最接近于peerLastZxid的ZXID
    
    3.仅回滚同步（TRUNC 同步）:
    peerLastZxid 大于 maxCommittedLog
    
    4.全量同步（SNAP 同步）:
    场景一：peerLastZxid 小于 minCommittedLog
    场景二：Leader服务器上没有Proposal缓存队列且peerLastZxid不等于lastProcessZxid

    

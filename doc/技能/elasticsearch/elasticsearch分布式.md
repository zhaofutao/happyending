# 一、ES集群构成
        一个Elasticsearch集群由许多节点（Node）构成，Node可以有不同的类型，通过以下配置，可以产生四种不同类型的Node（不考虑Ingest Node）：
```
conf/elasticsearch.yml:
    node.master: true/false
    node.data: true/false
```
        四种不同类型的Node是一个node.master和node.data的true/false两两组合。
        当node.master为true时，其表示这个node是一个master候选节点，可以参与选举，在ES的文档中常被称作master-eligible node，ES正常运行时只能有一个master(即leader)，多于1个时会发生脑裂。
        当node.data为true时，这个阶段作为一个数据节点，会存储分配在该node上的Shard的数据并负责这些Shard的写入、查询等。
        此外，任何一个集群内的Node都可以执行任何请求，其会负责将请求转发给对应的Node进行处理，所以当node.master和node.data都为false时，这个节点可以作为一个proxy及诶单，接收请求并进行转发、结果聚合等。
![blockchain](/resource/images/elasticsearch集群示意图.jpg "Elasticsearch集群示意图")  
        上图是一个ES集群的示意图，其中NodeA是当前集群的master，NodeB和NodeC是master的候选节点，其中NodeA和NodeB同时也是数据节点，此外NodeD是一个单纯的数据节点，NodeE是一个proxy节点，每个Node都会跟其他所有Node建立连接。

# 二、Elasticsearch节点发现
        ZenDiscovery是ES自己实现的一套用于节点发现和选主等功能的模块，没有Zookeeper等工具，简单来说，节点发现依赖以下配置：
```
conf/elasticsearch.yml
    discovery.zen.ping.unicast.hosts: [1.1.1.1, 1.1.1.2, 1.1.1.3]
```

# 三、Elasticsearch Master选举
        ES集群可能会有多个master-eligible node，此时就要进行master选举，保证只有一个当选master。如果有多个node当选mast，则集群会出现脑裂，破坏数据的一致性。
        为了避免脑裂，ES采用了常见的分布式系统思路，保证选举出的master被多数派的master-eligible node认可，以此来保证只有一个master
```
conf/elasticsearch.yml
    discovery.zen.minimum_master_nodes: 2
```
        master选举由master-eligible节点发起，当一个master-eligible节点发现满足以下条件时发起选举：
- 该master-eligible节点当前状态不是master;
- 该master-eligible节点通过ZenDiscovery模块的ping操作询问其一直的集群其他节点，没有任何节点连接到master；  
- 包括本节点在哪，当前已有超过minimum_master_nodes个节点没有连接到master。  
        总结一句话，即当一个节点发现包括自己在内的多数派的master-eligible节点认为集群没有master时，就可以发起master选举。

        当需要选举master时，选举谁，从下面源码所示，选举的是排序后的第一个MasterCandidate。
```
public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
    assert hasEnoughCandidates(candidates);
    List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
    sortedCandidates.sort(MasterCandidate::compare);
    return sortedCandidates.get(0);
}
```
        排序规则：
  - 当clusterStateVersion越大，优先级越高，这是为了保证master拥有最新的clusterState（即集群的meta），避免已经commit的meta变更丢失，因为master当选后，会以这个版本的clusterState为基础进行更新。
  - 当clusterStateVersion相同（或者所有集群全部重启，都没有meata）时、节点的id越小，优先级越高。即总是倾向于选择id小的Node。

        当一个master-eligible node(我们假设为NodeA)发起一次选举时，它会按照上述排序策略选出一个它认为的master。
        假设NodeA选NodeB当Master，NodeA会向NodeB发送join请求，此时：
        1. 如果NodeB已经成为Master，NodeB就会把NodeA加入到集群中，然后发布最新的cluster_state，最新的cluster_state就会包含NodeA信息，相当于一次正常情况的新节点加入。对于NodeA，等信的cluster_state发布到NodeA的时候，NodeA也就完成join了。
        2. 如果NodeB在竞选Master，那么NodeB会把这次join当做一张选票，对于这种情况，NodeA会等待一段时间，看NodeB是否能成为真正的NodeB，知道超时或者别的Master选举成功。
        3. 如果NodeB认为自己不是Master（现在不是，将来也选不上），那么NodeB会拒绝这次join，对于这种情况，NodeA会开启下一轮选举。

        假设NodeA选自己当Master，此时NodeA会等别的Node来join，即等待别的Node的选票，当收集选票超过半数的时候，认为自己已经成为Master，然后变更cluster_state的Master Node为自己，并向集群发布这一消息。

# 四、错误检测
   1. MasterFaultDetection和NodesFaultDetection  
        这里的错误检测可以理解为类似心跳的机制，有两类错误检测，一类是Master定期检测集群内其他的Node，另一类是集群内其他的Node定期检测当前集群的Master，检测的方法就是定期执行ping请求。  
        如果Master检测到某个Node连不上了，会执行removeNode的操作，将节点从cluster_state中移除，并发布新的cluster_state，当各个模块apply新的cluster_state时，就会执行一些恢复操作，比如选择新的primaryShard或者replica，执行数据复制等。  
        如果某个Node发现Master连不上了，会清空pending在内存还未commit的new cluster_state，然后发起rejoin，重新加入集群（如果达到选举条件则触发新的master选举）。  
    2. rejoin  
        处理上述两种情况，还有一种情况是Master发现自己已经不满足多数派条件了，需要主动退出master状态（退出master状态并执行rejoin）以避免脑裂的发生，那么master如何发现自己需要rejoin呢？
         - 当有节点连不上时，会执行removeNode，在执行removeNode时判断剩余的Node是否满足多数派条件，如果不满足，则执行rejoin。
         - 在publish新的cluster_state时，分为send阶段和commit阶段，send阶段要求多数派必须成功，然后再进行commit。如果在send阶段咩有实现多数派返回成功，那么可能是有了新的master或者是无法连接到多数派个节点，则master需要执行rejoin。
         - 在对其他阶段进行定期ping时，发现有其他节点也是master，此时会比较本节点与另一个master节点的cluster_state的version，谁的version大谁成为master，version小的执行rejoin。

# 五、Zookeeper
        Zookeeper分布式服务框架是Apache Hadoop的一个子项目，它主要是用来解决分布式应用汇总经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。
        简单来说，Zookeeper就是用来管理分布式系统中的节点、配置、状态，并完成各个节点间配置和状态的同步等，大量的分布式系统依赖Zookeeper或者类似的组件。
        Zookeeper通过目录树的形式来管理数据，每个节点成为一个znode，每个znode由3部分组成：
  - stat：状态信息，描述该znode的版本、权限等信息；
  - data：与该znode关联的数据；
  - children：该znode下的子节点；  
        stat有一项是ephemeralOwner，如果有值，代表一个临时节点，临时节点会在session结束后删除，可以用来辅助应用进行master选举和错误检测；
        Zookeeper提供watch功能，可以用于监听相应的时间，比如某个znode下的子节点的增减，某个znode本身的增减，某个znode的更新等。
    
        怎么使用Zookeeper实现ES的上述功能：
  - 节点发现：每个节点的配置文件中配置一下Zookeeper服务器的地址，节点启动后到Zookeeper中某个目录中注册一个临时的znode，当前集群的master监听这个目录的子节点增减的时间，放发现有新节点时，将新节点加入集群。
  - master选择：当一个master-eligible node启动时，都启动到固定位置注册一个名为master的临时znode，如果注册成功，即成为master,失败则监听这个znode的变化。当master初选故障时，由于是临时znode，会自动删除，这是集群中其他的master-eligible node就会尝试再次注册。使用Zookeeper后其实是把选master变成了抢master。
  - 错误检测：由于节点的znode和master的znode都是临时znode，如果节点故障，会与Zookeeper断开session，znode自动删除。集群的master只需要监听znode变更事件即可，如果master故障，其他的候选master则会坚挺到master znode被删除的事件，尝试成为新的master。
  - 集群扩缩容：容易；
  
  # 六、Raft
        与Zen的相同点：
  - 多数派选择：必须得到超过半数的选票才能成为master；
  - 选出的leader一定拥有最新提交数据。在Raft中，数据更新的节点不会给数据旧的节点投选票，二档选需要多数派的选择，则当选人一定有最新已提交数据。在ES中，version大的节点排序优先级高，同样用于保证这一点。
        与Zen的不同点：
  - 正确性论证：Raft是一个被论证过正确性的算法，而ES的算法是一个没有经过论证的算法，只能在实践中发现问题。
  - 是否有选举周期Term，Raft引入了选举周期的概念，每轮选举Term加1，保证了在同一个Term下每个参与人只能投1票，ES在选举时没有Term的概念，不能保证每轮每个节点只投1票。
  - 选举的倾向性：Raft中只要一个节点拥有最新的已提交的数据，则有机会选举成为Master，在ES中，version相同时会按照NodeId排序，总是NodeId小的人优先级高。
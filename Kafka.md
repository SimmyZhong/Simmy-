##### Kafka 架构
    - BrokerKafka: 集群包含一个或多个服务器，这种服务器被称为 broker
    - Topic: 每条发布到 Kafka 集群的消息都有一个类别，这个类别被称为 Topic
    （物理上不同 Topic 的消息分开存储，逻辑上一个 Topic 的消息虽然保存于一个或多个 broker 上, 
    但用户只需指定消息的 Topic 即可生产或消费数据而不必关心数据存于何处）
    - Partition: Parition 是物理上的概念，每个 Topic 包含一个或多个 Partition.
    - Producer: 负责发布消息到 Kafka broker
    - Consumer: 消息消费者，向 Kafka broker 读取消息的客户端
    - Consumer Group: 每个 Consumer 属于一个特定的 Consumer Group
    （可为每个 Consumer 指定 group name，若不指定 group name 则属于默认的 group）
##### Kafka 特性
    - Broker 可水平拓展
    - Topic 分成一个或多个 Partition, 消息可随机均匀分布在 Partition
    - 以时间复杂度为 O(1) 的方式提供消息持久化能力, (写入 Partion 文件, 顺序 IO)
    - Broker 是无状态的, 无需维持锁机制, offset 由 comsumer 维护
    - 同一 Topic 的一条消息只能被同一个 Consumer Group 内的一个 Consumer 消费，
    但多个 Consumer Group 可同时消费这一消息（广播机制）
##### Push vs Pull 
    Producer 向 broker push 消息并由 Consumer 从 broker pull 消息
    push 模式很难适应消费速率不同的消费者，因为消息发送速率是由 broker 决定的。
    push 模式的目标是尽可能以最快速度传递消息，但是这样很容易造成 Consumer 来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。
    而 pull 模式则可以根据 Consumer 的消费能力以适当的速率消费消息。   
    对于 Kafka 而言，pull 模式更合适。pull 模式可简化 broker 的设计，Consumer 可自主控制消费消息的速率，
    同时 Consumer 可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。
##### request.required.acks 
    - acks = 1 表示 producer push 消息, ISR 中 leader 发出 ack 确认即开始下一条消息, 如果 leader 挂了消息丢失
    - acks = 0 表示 producer push 消息, 不管有没收到 ack 确认即开始下一条, 效率最高， 可靠性最差
    - acks = -1 表示 producer push 消息，ISR 所有 follower ack确认后才开始下一条消息,
     如果 ISR 中 follower 都挂了，即回到了 acks = 1模式
##### ISR 和 AR
    - Partition 所有的副本即为 AR, ISR 是 AR 的一个子集，由 leader 负责维护，follower 从 leader 同步数据，
     延迟条数超过一个阈值 leader 会将 follower 从ISR 干掉。
##### leader 选举
    - Kafka 从 ISR 中选举 leader, 投票机制的一种缺点是可容忍挂掉follower的个数少
    - 如果所有副本 replica都挂了， 有两种机制恢复:
        1. 等待 ISR 中某一个副本 replica 恢复, 成为leader, 可用性太差
        2. 任意 一个副本 replica 恢复成为 leader, 可用性更好，但丢失更多（默认是这种）
##### 可靠性
    msg 默认是最少一次发送, at least once, 这肯定会存在消息重复, 不建议由客户端去重,
    这样不得不引入分布式缓存, 结构更复杂, 缓存大小难以界定。可以让消费方实现消息处理幂等性。
##### 其他
    replica 多了，acks = -1 或者其他配置 会增加系统可靠性，但是性能会下降, 需要权衡两者。
    要保证数据写入到 Kafka 是安全的，高可靠的，需要如下的配置：
        - topic 的配置：replication.factor>=3, 2<=min.insync.replicas<=replication.factor
        - broker 的配置：leader 的选举条件 unclean.leader.election.enable=false
        - producer 的配置：request.required.acks=-1(all)，producer.type=sync
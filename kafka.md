## kafka
- 卡夫卡topic消费不均匀导致性能抖动怎么办

- kafka消费如何保证一致性
    - 同一条消息来了两次
    - 通过回掉函数，等mysql确认入库成功了，才确认是给kafka回复成功

- kafka生产者流程
- buffer.memory满了怎么办
- batch.size的作用
- kafka的分区，备份等名词的概念，主分区和备份分区怎么保证消息一致，怎么保证消息不丢？
- kafka保证时序性有哪些方式？比如消费者消费要按照生产者生产的顺序进行？除了用kafka还有其他方式吗？
- kafka 如何避免重复消费
- kafka 版本2.4之后对rebalance进行新的机制，你了解吗？
  - incremental 协议允许消费者在重新平衡事件期间保留其分区，
  - 从而尽量减少消费者组成员之间的分区迁移。
  - 因此，通过 scaling out/down 操作触发的端到端重新平衡时间更短，这有利于重量级、有状态的消费者，比如 Kafka Streams 应用程序。
  - [refer]()


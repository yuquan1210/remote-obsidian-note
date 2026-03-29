#file:KafkaConsumerConfig.java:71-75
这三个参数**不是标配**，是这个项目自定义的 Bean，但 Spring 会自动注入它们。

### 工作原理

`@Bean` 方法的参数会被 Spring 通过**依赖注入**自动提供 —— Spring 从应用上下文中查找对应类型的 Bean，并在调用该工厂方法时自动传入。

### 三个参数各自的用途

在 `buildErrorHandler(...)` 中可以看到它们的具体作用：

| 参数                  | 类型                  | 用途                             |
| ------------------- | ------------------- | ------------------------------ |
| `messagePublisher`  | `MessagePublisher`  | 将处理失败的消息发送到 DLQ（死信队列 SQS）      |
| `cloudWatchService` | `CloudWatchService` | 当发送 DLQ 也失败时，向 CloudWatch 上报指标 |
| `jsonHelper`        | `JsonHelper`        | 序列化日志和消息内容为 JSON 字符串           |

### 总结

- 这三个参数**不是 Spring Kafka 的内置标配**，是项目自己定义的 Bean
- Spring 会自动识别并注入，不需要手动传值
- 三个 `@Bean` 工厂方法（`incomingLeadKafkaListenerContainerFactory`、`leadStatusKafkaListenerContainerFactory`、`leadFilteringEventKafkaListenerContainerFactory`）共享同一套参数，是因为它们都复用了同一个 `buildErrorHandler(...)` 私有方法


hydro topic 创建
https://hydro.reainternal.net/guides/hydro/topics/create-a-topic/#1-create-your-topic-manifest

copilot 
https://reagroup.atlassian.net/wiki/spaces/cp1/pages/977961102/Getting+started+wtih+CoPilot
- 公共资源 用这套部署系统
- 项目内部资源 自己部署？ecs, rds, ...

为什么3个分区 3个broker？
- 集群起了3个broker，3个分区可以一一对应（分区数是我们可以配的，broker数不是）
- 我们创建了topic 没有创建hydro
- hydro dev集群还被其他组使用？
- 对于lead store，只起一个实例/一个线程/一个消费者的情况下，3个分区 3个broker有必要吗
- 这时，三个分区 都给到这一个消费者，这个消费者会去3个broker拉数据，维持3个连接
- 一个分区 一个broker的情况，且流量小的情况，应该也可以承受，不过是这个broker压力大

线程指明一个消费组id的作用：确保同一组内的消费者不会重复消费信息
多个分区多个消费组的核心设计原因：运行多个消费组并行消费

kafka send 出错
- 抛出异常
- 被consumer的exception handler 接住





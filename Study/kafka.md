主题1：Kafka 基础概念

 kafka系统设计相关
 - kafka同样是 分布式系统，由至少一个服务器组成，这些服务器可位于不同位置 
	 - 高扩展：性能不够时，增加服务器
	 - 容错性：服务器挂了切另一个
- 整个kafka系统内部的服务器可以分为三类：
	- broker：可以理解为kafka服务器，负责数据的接收，发出，持久化，应用直接与它通信
	- connector：Kafka Connect 框架中的组件，独立于应用之外运行；常见于 kafka 与其他系统做数据自动导入 或 导出的工作（数据搬运场景，我们是业务场景需要对数据作处理，有各类下游逻辑）
	- controller：负责整个kafka集群的元信息，管理broker
- Source Connector：将外部系统数据拉入kafka
- Sink Connector：将kafka数据推送到外部系统
	- 数据仓库同步：Kafka → Snowflake/BigQuery，用 Sink Connector 批量写入

常见概念：
- **生产者**是向 Kafka 发布（写入）事件的客户端应用，而**消费者**则是读取和处理（订阅）这些事件的应用
- 一个事件被存储到一个主题里
- 一个主题中有多个分区
- 每个分区分散在不同的broker（或代理或服务器）中，假设一个broker是一个分区的leader，这个broker可能是其它分区的follower
- 同一个 broker 可以同时承载 同一个 topic 的多个分区（或这些分区的副本）
- 极少情况下 会出现 broker不是任何分区的leader 也不是任何分区的follower这种情况
	- 新加入broker会是，kafka不会自动迁移分区（避免隐式大规模数据移动）
	- 或者 分区数 / 副本因子太小
```
Brokers = 5 
Topic partitions = 1 -> 一个主题一个分区
replication.factor = 1 -> 一个分区一个副本
结果4个broker都不作用，合法不合理
```
- （kafka负载均衡）通过使一个topic 有多个分区，并且多个分区分散在不同broker使数据经可能均匀分散
- 一对键值放在哪个分区是由hash的方式计算的，所以同个键一定会放到同个分区
- 不同 key → 不同 partition 的消息，在读取时不会、也无法按照“整体写入顺序”被读取。Kafka 只保证“分区内有序”，不保证“分区之间有序”
- 客户端也可以指定不同key写入同一分区，但是不推荐依赖不同key的写入顺序，容易出错

Kafka 是客户端驱动的系统：Producer 主动写，Consumer 主动读，Broker 只响应请求
- kafka pull
	- Long Poll（长轮询）
	- 批量返回（Batch）

consumer group是什么
-  消费组 使得 多个消费者可以并行消费同一个主题中的信息，但是同一个消费组的并行必须在（同一主题的）不同分区
	- 分区内的信息是按写入顺序的，如果同一分区允许（同一消费组）多个消费者并行消费，执行顺序不可保证
	- offset（消费位点）是每个消费组 每个分区 维护一个（如果。。消费位点可能回退，导致消息在这个消费组中重复消费）
	- 可以配置改变第一条，不推荐
- 例子：
多个消费者并行处理 同一主题内的信息（消费组设计的本意）
```
Topic: order-events 
Consumer Group: order-service-group 
Consumers: order-service-1 ~ order-service-5
```
并行下游系统，每个group各自消费主题内的信息
```
Topic: order-events 
order-core-group 
risk-control-group
data-sync-group
```

offset还有什么要点
- 消息 offset=100 -> Consumer 处理成功 -> 提交 offset=101
- 异常情况
	- 消息 offset=100 -> Consumer 处理成功 ✅ -> （还没来得及提交 offset） -> Consumer 崩溃 / rebalance / 网络抖动 
	- offset=100 -> 再消费一次 （所以kafka只能保证at least once）
- offset是单调递增的，可以手动reset但是会导致消息重复消费
- 实现中，确保消费完成再提交offset，而不是确认收到消息就提交offset
- Kafka 允许**主动把消费组的 offset 调整到更小的值**，从而**重新消费历史消息**
- offset由broker维护（三元组 消费组+主题+分区 指向一个offset），跟踪下一条消费哪条信息
	- 一个特殊的broker -> group coordinator
	- offset存在一个主题里，但不向正常消费者一样消费信息（不深究）

什么是cloud event:
- 定义事件的统一格式，各个系统发送或接收事件 可以统一处理的规范
- 必须字段：specversion，id，source，type
- 常见可选：time，subject，datacontenttype，data

关于kafka与raft：
- raft 集群 选主大致解释：节点之间会健康检查，一个节点发现主节点不健康后会启动选举，健康节点之间会通过通票机制选出 主节点，一般日志最新/数据最全的节点一般情况会成为主节点 并让其它节点跟随
	- raft协议 强调 大多数从节点的数据 与 主节点的数据保持一致，这样当主节点宕机，几乎可以保证数据不丢失
- Kafka Controller 使用的 KRaft 在“共识语义”上与标准 Raft 没有本质不同，可以理解为针对kafka 场景的优化版raft（例如raft存任意数据，kraft存的是固定结构的元数据）
- controller kraft 一样强调数据的强一致性，被从节点多数提交才能确认，日志落后的节点不能当 Leader，。。。
- Kafka Partition 中，数据同步程度满足要求的副本会进入 ISR，只有 ISR 中的副本才默认有资格被 Controller 选为 Partition Leader。
	- Kafka 可以配置允许 ISR 之外的副本成为 Leader（可能丢数据），这是 Kafka 数据模型的取舍；而 Raft 协议中不允许日志落后的节点当 Leader
- Kafka 中 Partition Leader 的变更不是通过副本投票，而是由 Controller 统一决定
- Raft 中，Leader 主动向 Follower 推送日志并维护复制进度，但是 Kafka Partition 的数据复制中，Follower 主动从 Leader 拉取数据
	- follower 拉取数据时，会告知leader当前同步到哪里，leader通过这些信息 （被动）得知各个follower的同步状态，从而决定谁进isr
	- 允许少量日志不同步，不像raft一样严格
- partition lead (broker) 宕机 由controller直接从候选isr中选出下一个（根据配置的既定顺序，Partition 的 replica 列表）
	- replica 列表是 topic 创建时由 controller 自动生成的 partition assignment
- controller 的日志存集群状态：哪些 topic / partition 存在、每个 partition 的副本分布、谁是 leader、ISR 是谁、broker 是否存活、配置变更等——也就是**所有 Broker 必须一致认同的集群元数据**。


lead-store 中的 kafka 使用
这部分笔记基于lead-store中的关于kafka的代码
deploy 脚本中的kafka配置:
参考：https://git.realestate.com.au/financial-experiences/lead-store/blob/main/deploy/rea-shipper-ecs-dev.yml
- kafka_bootstrap_address -> Kafka 集群的入口地址列表
	- 应用会任选一个连接，拿到集群的信息（哪个broker负责哪个topic的哪个partition）
	- 客户端再基于这些信息 按实际情况 去连接 需要的broker
	- 可只选一部分，通常2-3个，有一个可连就不会影响功能（多写少写实际效果一样）
	- 地址从aws console或者client 取得（aws kafka get-bootstrap-brokers --cluster-arn <MSK_CLUSTER_ARN>）
	- 集群多少个broker在cloudformation配置里
- incoming_lead_topic_name -> topic 名
- incoming_lead_topic_role_arn -> lead-store 服务会借role得到读权限，这个role属于888892762448账号（hydro-dev），并不属于lead-store所以需要借
	- 在kafka topic配置中，声明了lead-store（属于567418462583账号）可以借用这个role


KafkaConsumerConfig:
参考：https://git.realestate.com.au/financial-experiences/lead-store/blob/a8bec0e94c55d7d92983129778e0c241a727343b/src/main/java/com/reagroup/finx/leadstore/config/KafkaConsumerConfig.java
- @EnableKafka 注解在一个@Configuration 类上 告诉spring去扫描所有标有 @KafkaListener 的方法（这些也是bean），并为它们创建对应的监听容器
- 构建ConcurrentKafkaListenerContainerFactory 监听容器
- 容器内部持有 Kafka Consumer 线程，会主动不断的从kafka 拉取信息，给到@KafkaListener 处理方法
- Concurrent -> 这个容器内可以有多个并发线程同时消费，默认是1；同一时间，一个分区只能被一个线程拉取，不允许同一个分区同时被多个线程拉取
	- buildConsumerFactory -> 输入配置，集群地址，输入iam权限配置
	- map结构，key, val, key, val 的格式
	- buildErrorHandler -> 构建失败处理器，消费一个消息失败怎么处理
	- FixedBackOff(1000, 1L)  // 重试间隔1秒，最多重试1次
	- ```
	  消息消费失败
    │
    ├─ FixedBackOff: 等1秒，最多重试1次
    │
    └─ 还是失败 → 执行 recover 回调：
            │
            ├─ 是反序列化异常？ → 把原始字节转成字符串
            │
            └─ 是业务处理异常？ → 把 CloudEvent 序列化成 JSON
            │
            └─ 发送到 SQS 死信队列（DLQ）
                    │
                    └─ SQS 也失败了？ → 上报 CloudWatch 指标
	  ```
- 这些 Bean 方法 在 Spring 初始化阶段执行，创建 ConsumerFactory、ErrorHandler、Factory；kafka连接是在应用启动阶段建立，开始监听拉取
- containerFactory 配在 @KafkaListener 上 告诉 Spring 用哪个 Factory 来为这个监听方法创建容器
- 如果不指定 containerFactory，Spring 会找名为 kafkaListenerContainerFactory 的默认 Bean，不存在就报错

IncomingLeadConsumer
- ConsumerRecord 包含消息体还有所有元数据（哪个topic，哪个分区，offset多少，消息在kafka中的位置）
- groupId 指明属于哪个消费组，一个消费组 在各个分区追踪维护各自的offset，同一个消费组只能消费同一个消息一次
- 这里虽然单个容器内只有 1 个线程，但这个应用在生产环境通常会部署多个 ECS Task 实例
	- groupid 是可以跨机器的
	- 多个ecs 中的消费者共用一个消费组，实现并行消费同一个topic


主题3： 配置

hydro/kafka 配置 
- incoming_lead_topic 所在 cluster 是 hydro-dev
- 配置在 hydro repo: hydro/hydro_env/hydro-dev/hydro-dev-cluster.yaml
- broker_count = 3
- Hydro 用 CDK 管理 AWS 资源 → CDK synth 生成 CFN 模板 → CDK deploy 提交到 CloudFormation 执行。CloudFormation 是幕后机制，CDK 是实际使用的工具（CloudFormation 模板生成器 + 部署工具）
- 没有用 rea-metacontroller：metacontroller 通常用于 Kubernetes CRD 控制器，这里是 MSK/AWS 资源，不涉及 K8s 控制平面
- Kubernetes CRD 控制器是用于管理自定义资源（custom resource definition）的逻辑代码。CRD 扩展了 Kubernetes API，允许用户定义非内置资源（如数据库、监控实例）
	- kafka cluster 是 aws msk 资源，kafka topic是自定义资源
	- MSK cluster 是 AWS 原生资源，CFN 能管；Kafka topic 不是 AWS 原生资源，需要通过 Kafka API 操作，CoPilot 提供了"谁的 topic 谁管理"的去中心化入口，Lambda 则是实际执行 Kafka API 调用的执行层。

kafka topic 配置
IAM权限相关:
- lead_filtering_event_topic_role_arn: arn:aws:iam::888892762448:role/hydro-dev-60026af-lead-filtering-event-readonly
- lead-filtering topic 由888892762448 aws账号管理，这里设置lead-store 运行的aws 账号有这个topic的只读权限
- lead-store dev环境用的账号是567418462583 （取决于用那个aws 账号执行cloudformation部署，部署时应该会要求登陆）
- hydro_env: hydro-dev -> 指定集群环境（服务通过它去ssm 查 /kafka/hydro/hydro-dev/{cluster}/ 下的参数（cluster ARN、deployment role ARN 等））
- hydro_metadata：-> 一些类似标签作用的元数据
- partitions: 3 -> 分区数，决定最大并行消费者数，创建后不能减少 （影响服务补全 retention.bytes 的计算：总字节上限 ÷ 分区数 = 每分区上限）
- kafka_config.cleanup.policy: delete -> 消息过期后的处理策略 
	- 一旦 topic 创建后此值不能修改，validate_cleanup_policy_cant_change 校验拦截变更
	- < 21天（MAX_RETENTION_MS_LOCAL）→ 不启用 Tiered Storage，remote.storage.enable 补全为 false
	- 决定 retention.bytes 默认值的档位（< 21天用 30GB 总量，> 21天用 300GB 总量）
- retention.bytes -> 每个分区最多多少数据 （服务补全为 10737418240（10GB/分区），即 3 个分区共 30GB 上限）
- Tiered Storage -> 分层存储
	- 标准 Kafka 数据全存在 broker 本地磁盘，成本高、容量有限。Tiered Storage 把历史数据卸载到 S3，大幅降低成本，同时允许更长的保留时间（最长 2 年）。
	- retention.ms > 21天（MAX_RETENTION_MS_LOCAL） → 自动启用，remote.storage.enable = true
	- 启用后，本地只保留最近 21 天热数据，其余转存 S3
	- 同时 retention.bytes 上限提升到 300GB/分区（MAX_RETENTION_BYTES_TIERED）
```
热数据（本地 broker EBS 磁盘）  ←  local.retention.ms / local.retention.bytes 控制
            ↓ 超出后自动迁移
冷数据（AWS S3 远程存储）        ←  retention.ms / retention.bytes 控制总量
```
- permissions -> 定义权限组
	- principals：权限组内被授权的应用列表，每个 principal 代表需要访问该 Kafka topic 的应用
	- incoming-lead-readonly（权限组名）-> 服务将其与 cluster_id 和 topic hash 拼接，生成实际 IAM 资源名：hydro-dev-27efb0e-incoming-lead-readonly
	- access_level -> 可选 topic-read、topic-write、topics-describe，决定该权限组下的 IAM Role 被附加哪些 MSK 操作权限
	- principal_type: consumer-app -> 标注该应用是消费者，配合 access_level 共同确定 IAM Policy 的内容
	- arn: arn:aws:iam::567418462583:role/application/lead-store-role-ap-southeast-2 -> lead-store 应用运行时使用的 Role
		- 服务生成的 IAM Role（hydro-dev-27efb0e-incoming-lead-readonly）的 Trust Policy 会允许这个 ARN 来 sts:AssumeRole
	- replication: 3 -> Kafka 的副本因子，每条消息被存储在 3 个不同的 broker 节点
	- retired -> 是否退役/删除这个topic 
		- 删除时，现设置retired=true，重新apply manifest 启动删除流程，会清空topic数据，并删除子资源
		- 删除manifest 只会 operator 停止管理
- Lambda 执行角色（Custom Resource）-> CloudFormation 在创建/更新/删除MSK Topic时，把事件发给一个 Lambda 函数（即 Provider Lambda），lambda函数实际调用msk API去操作kafka内部
	- 这个lambda函数需要相应的角色&权限（AWS::IAM::ManagedPolicy）
	- CloudFormation 管理 AWS 资源 （创建 Broker 集群，broker配置，管理 Broker 的 EC2、网络、监控等），不能管理kafka内的topic
	- 创建 Topic 不是调用 AWS API（如 msk:CreateTopic），而是要通过 Kafka AdminClient 协议连接到 Broker，发送 CreateTopics 请求：
- CloudFormation Custom Resource -> cloudformation 扩展机制，让cloudformation stack 可以配置 cf 原生不支持的（更细致/资源内部）配置工作，底层是通过lambda接入资源内部去做配置，需要自己定义lambda逻辑
- Type: AWS::IAM::Policy -> 内联策略，只与一个role强绑定，role删除，该策略一起删除
- AWS::IAM::ManagedPolicy -> 托管策略，是一个独立的资源（有自己的arn），多个role可共享，删除role不影响
	- 策略是把权限打包，方便管理和复用
```
AWS::IAM::Role
└── hydro-dev-27efb0e-incoming-lead-readwrite
      │
      │  附加了
      ▼
AWS::IAM::ManagedPolicy
└── （Kafka topic 读写托管策略）
      │
      │  包含以下权限（Permissions）
      ▼
      ├── kafka-cluster:Connect
      ├── kafka-cluster:DescribeTopic
      ├── kafka-cluster:ReadData        ← topic-read
      ├── kafka-cluster:WriteData       ← topic-write
      └── kafka-cluster:AlterGroup
```
- OnEvent Lambda 自动生成的CDK 框架代码，用于协助lambda 跟cloudformation衔接通信，超时处理等
- 


部署操作:
-Add or modify the YAML manifest under the appropriate copilot/<env>/ap-southeast-2/hydro-topics/ directory
- Merge to main — CoPilot automatically deploys it
- After the topic is created, go to Slack #hydro-streaming-platform to request trust relationship updates if IAM roles need to be re-trusted


部署相关：
CoPilot 是 REA 内部的基础设施自助服务平台，所有团队通过提交 YAML manifest 来申请资源，底层统一走:
Kubernetes YAML → Metacontroller → CoPilot Service → rea-copilot-cdk → CloudFormation → AWS 资源
- 这套模式在 REA 内部被用于部署各类数据平台资源（ECS 服务、数据库、消息队列等），hydro-topic-service 只是其中专门负责 Kafka topic 的一个 service。

copilot-hydro-topic-service 直接涉及的资源
  ├─ MSK Kafka Topic          ← CloudFormation（rea-copilot-cdk）
  ├─ IAM Role + Policy        ← CloudFormation（topic 的 permissions）
  ├─ CloudWatch Alarm         ← CloudFormation（topic 的 observability）
  └─ S3 Sink Connector        ← HydroConnect service（可选，S3备份）

copilot-hydro-topic-service 依赖/涉及到 copilot-hydro-connect 
copilot-hydro-connect 涉及资源
HydroConnect
  └─ MSK Connect Connector    ← 部署 Kafka Connect，实现数据流转
       例如：topic → S3 sink backup

事件驱动的循环控制 / 部署具体流程
- 在aws-infra中，调用 auto/kubectl apply 将配置文件发到 kubernetes
- metacontroller 监听kubernetes资源变动，调用 hydro-topic-service 的/sync 接口 
- hydro-topic-service 验证配置 检查现有资源状态 然后计算还需要什么资源（以及 配置）返回给metacontroller
- metacontroller 收到资源信息，（有必要的话）调用rea-copilot-cdk框架去实际部署
  （如果HydroConnectResource在desired children中，metracontroller调用copilot-hydro-connect-service的sync/ 以同样方式收到资源具体部署信息）
- 部署完成后 资源状态变更为ready 触发metacontroller
- 此时 hydro-topic-service.sync() 返回 Ready=True，部署完成
（任何资源更新/状态变动都会触发metacontroller的流程）

与之前讨论的部署postgres db的方式 作为例子 比较两种部署方式
db-in-a-box vs. CoPilot RDS Service (via Metacontroller)
- db-in-a-box 是应用侧自驱动的 CI 工具；CoPilot RDS Service 是平台侧控制的 Operator 模式。前者更灵活独立，后者强调平台统一治理和声明式管理。

角色/概念
- HydroTopic Manifest： 用户写的 Kubernetes YAML 配置文件（kind, metadata, spec）
- Children: 子资源
- rea-copilot-cdk 框架： 封装了"用 CDK 生成 CloudFormation 并部署"这个过程
- Metacontroller：监听资源变化，协调父子资源的创建/更新/删除，驱动控制循环
- hydro-topic-service：理解业务意图，验证配置，计算完整参数，决定需要哪些子资源
- rea-copilot-cdk：把技术方案变成 CloudFormation，真正部署 AWS 资源


TODO:
- lead-engine-hydro-topic-replay
- kafka 与 sqs 的区别，什么场景用kafka 什么场景用sqs？

问题：
- aws-infra kubectl 没有权限执行？文中示例用的是fin-exp-dev-Privileged权限
	- when pull kubectl image, 403 forbidden
- db-in-a-box部署 rds也是通过/用到metacontroller吗？
	- 好像不是，db-in-a-box直接可生成cloudformation
	- metacontroller 控制循环？最终才生成cloudformation

MSK Connect connector：把 Kafka topic 的消息持续同步写入 S3


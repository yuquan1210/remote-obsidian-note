业务角度
- lead store 的主要职责
- 上下游都是谁，他们用lead-store的数据做什么，或者调用lead-store帮助他们实现什么目的
- 

技术角度
- 关于一些包 例如 jackson, mdc（不作为重点）
- 和其他资源怎么连接的，资源调用涉及什么问题（例如dlq，这些之前有学到，不作为重点）
- 性能相关 aws ecs 配置（不作为重点）

- 整体系统设计（spring boot 经典架构吗）
- 数据流图 （各种情况下的数据流程图）
- 架构图 
- 测试代码（测试都测了哪些）

LeadRepository：
1. LeadRepository 是一个 Spring Data 的接口，用于管理 Lead 实体的数据访问，提供了自定义和标准的数据库操作方法。
2. 由于它只继承了 org.springframework.data.repository.Repository（一个标记接口），所以必须手动声明 findById、existsById 等方法，否则这些方法不会自动提供。
3. Repository 只是接口，没有实现。 Spring Data JDBC 会在运行时根据**接口定义**自动生成这些方法的实现，无需开发者手动实现。（与注解没有关系）
@Repository注解
- **标记为数据访问层组件**：告诉 Spring 这是一个数据访问对象（DAO，Data Access Object）
- **自动扫描注册**：Spring 容器会自动发现并注册为 Bean
- **异常转换**：将底层数据访问异常转换为 Spring 的 DataAccessException 层次结构

典型的 Spring Boot 分层结构是：
Controller（控制层）
    ↓
Service（业务逻辑层）
    ↓
DAO / Repository（数据访问层）
    ↓
Database（数据库）

Spring Boot 中 DAO 的常见形式
1️⃣ 使用 Spring Data JPA（最常见）
DAO 通常叫 Repository，本质就是 DAO。
```
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email);
}
```
UserRepository 就是 DAO
Spring 会自动帮你实现数据库操作

实现细节：
constraintValidator.validate(partialRawLeadPayload) seen everytime after construct an object
- validates the data structure of partialRawLeadPayload using Jakarta Bean Validation (formerly Java Bean Validation) constraints
- If any constraints are violated, it throws a ConstraintViolationException (If pass, continue silently)
- must have this validation function call, validation annotations are just metadata attached

Lombok.Builder seen when constructing an object with field values
- can add field values with conditions, no need to have constructor with different params
- immutable, once built, if want to update, must build a new one
- avoid with small class or all fields mandatory

关于 JsonHelper (为什么不直接用jackson objectmapper做string和json之间的转换)
- 为什么不直接用jackson objectmapper做string和json之间的转换
	- 主要是为了将objectMapper再包一层，为了抓到jacksonexception并用我们自己定义的InvalidJsonException再抛出去
	- 我们自己定义的exception只会打我们定义的信息，避免jacksonexception打出了敏感信息
- readValueWithoutJacksonException返回值用的是泛型将string转换成任意类型:
	- readValue 的作用是把 JSON 字符串变成一个 Java 对象，同一段 JSON 可以变成很多不同的 Java 类型
	```
	String json = "{\"name\":\"Alice\", \"age\":30}";

	// 同一段 JSON，可以变成不同类型：
	objectMapper.readValue(json, User.class);       // 变成你自己定义的 User 对象
	objectMapper.readValue(json, Map.class);        // 变成 Map<String, Object>
	objectMapper.readValue(json, ObjectNode.class); // 变成 Jackson 的 ObjectNode
	objectMapper.readValue(json, JsonNode.class);   // 变成 Jackson 的 JsonNode
	```
- JsonNode vs ObjectNode 的区别
	- jsonnode是抽象基类（不是interface），ObjectNode，ArrayNode，BooleanNode等都是它的子类
	- jsonnode（通用类）只有读方法，没有写方法，写方法定义在子类
	- Java 对象 ➜ JSON 字符串 是序列化
	- 区分三种read方式
		- readValueWithoutJacksonException：string转换成某个java对象，但是不会recursive转换内部的java对象（某个字段是另一个java对象，不会自动转换，只会转换一层）（例如转换为objectNode, jsonNode, LeadIncomingPayload）
		- content 必须是 合法 JSON，JSON 结构 能映射到 valueType，才能反序列化成功
		- treeToValueWithoutJacksonException：string会递进的转换内部的java对象（例如PartialRawLeadPayload）
		- writeValueAsStringWithoutJacksonException：将一个java对象转成string（json结构）（需要是jackson认识的类型，基本类型，包括数组等）
		- bytesToObjectWithoutJacksonException：将bytes转为某个类型，用于将cloudevent data 转为某个json类型
		- convertCloudEventToJsonString：将整个cloudevent序列化，不能直接用jackson去序列化，因为data不一定能被jackson序列化（data可能是任意类型）
		- convertToObjectNode：将一个类型转换为jackson的json对象，objectnode可以增删改字段，方便操作
		- buildObjectNode(String key, String value) 的作用：创建一个只有一个字段的 ObjectNode，并且智能处理 value


DeadLetterMessageFormatter：
- 为什么需要先将deadLetterMessage中的key:message构建成一个objectnode，再创建一个deadLetterMessage objectnode然后把message放回去
- 不能直接将deadLetterMessage序列化吗
- 原因是要将deadLetterMessage中的message保留它的json结构来序列化（可以用jackson readvalue 将message转换为objectnode但要考虑到message不是json结构的情况，所以通过buildobjectnode来处理）
- 如果直接序列化 那么message中的json结构可能就变成普通文本，如果message是json结构，输出会是json结构
- 以下是区别
```
{
  "message": { "a": 1 }
}
```
```
{
  "message": "{\"a\":1}"
}
```
- put将value作为字符串，set将value作为json结构，所以set value接受的是jsonnode

generateLeadId(ctx.getEnquiryRefId(), ctx.getOrg())
- 如果leadID没有提供，那么用EnquiryRefId和Org生成leadid
- EnquiryRefId是外部系统对于这条lead的id 配合所属组织 唯一标识一条lead
- uuidGenerator.nameUUIDFromString 对于相同的字符串生成相同的输出id

ObjectNode existingRawLeadPayload =
        leadRepository
            .findById(leadId)
            .map(Lead::getRawPayload)
            .orElseThrow(() -> new UnsupportedLeadException("This lead can not be found in DB"));
-  leadRepository.findById(leadId) 返回类型通常是 Optional<\Lead>。
- 有 Lead 时，执行 lead.getRawPayload()，得到 ObjectNode；没有 Lead 时，保持为空 Optional，不执行方法

buildNewRawPayload：
- (ObjectNode) newRawPayload.at("/enquiry") 取enquiry字段的值，可以用get 或者 path
- updateEnquiryDetails->putStringFields：遍历要更新的字段，构建字段名与对应值的pair，用filter将incominglead中对应字段为空的过滤掉，剩下的有值的pair覆盖原来的值

updateLeadDetails->updateMortgageChoiceDetails：
- 更新房贷（mortgage choice）信息，将incoming lead中的existingCustomer 和 customerId 字段更新进原有的房贷信息
- isMissingNode()：这个json节点是否存在
- isNull：这个字段是否为null
```java
// 存在但值为null的字段
JsonNode emailNode = root.at("/user/email");
emailNode.isMissingNode(); // 返回 false (节点存在，只是值为null)
emailNode.isNull();        // 返回 true (值为null)

// 完全不存在的字段  
JsonNode phoneNode = root.at("/user/phone");
phoneNode.isMissingNode(); // 返回 true (节点不存在)
phoneNode.isNull();        // 返回 false (因为节点本身就不存在)
```

（不重要）关于为什么isMissingNode()的定义是直接返回false
当你使用 at() 方法访问JSON路径时：
```
JsonNode root = objectMapper.readTree("{\"user\": {\"name\": \"John\"}}");

// 访问存在的路径
JsonNode nameNode = root.at("/user/name");     
// 返回 TextNode 实例，isMissingNode() 返回 false

// 访问不存在的路径  
JsonNode phoneNode = root.at("/user/phone");   
// 返回 MissingNode 实例，isMissingNode() 返回 true
```

updateExternalReferralData：
- 同理，如果incoming lead 有customerId，referrerId，source，就更新到现有的lead

updateExternalReferenceData：
- 同理，更新zendesk相关信息

为什么要创建rawPayloadUpdater class，而不是utils? Utils vs Helper/Component：
- rawPayloadUpdater 涉及业务逻辑（可能需要注入依赖），涉及多个函数，一般放在utils的为单一的静态的简单函数（不需要注入）
- rawPayloadUpdater @component 方便写单测，可以mock rawPayloadUpdater 指定它的返回值，而utils中的静态方法难以mock
- utils的方法一定是静态？为什么静态方法不好mock
	- 简单理解，静态方法在编译时被固定，难以被mock指定行为
	- 理论上好的utils方法不依赖任何状态（不依赖任何class中的值？只依赖输入值）
	- 现代观念是不静态，用@component
	- rawPayloadUpdater 中的方法都没有状态（输入确定输出）

getPublishAction：
- zendesk ticketstatus 空白的ticket status会被当作"solved"状态处理，允许lead继续流转
- isSolvedAsUnqualified中finxConciergeEnquiryType 的判断写法可以优化为 类似 ZENDESK_TICKET_STATUS_SOLVED.equalsIgnoreCase(ticketStatus)

doProcessWithTransaction：
- 用transaction将数据库更新和信息发布绑定执行
- TransactionTemplate 是 Spring Framework 提供的一个编程式事务管理工具（都成功/都失败）
- transactionTemplate.execute 参数接收 TransactionCallback<\T> action 是一个接口，这个TransactionCallback接口定义了一个（必须实现）方法
	- execute 中调用action.doInTransaction()
- 匿名内部类 = 没有名字的类 + 马上创建一个对象
```java
// 方式1：正常的类（有名字）
class MyWorker implements TransactionCallbackWithoutResult {
    @Override
    protected void doInTransactionWithoutResult(TransactionStatus status) {
        System.out.println("干活");
    }
}
// 使用：
MyWorker worker = new MyWorker();
transactionTemplate.execute(worker);

// 方式2：匿名内部类（没名字，直接用）
transactionTemplate.execute(
    new TransactionCallbackWithoutResult() {  // ← 这就是匿名内部类
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status) {
            System.out.println("干活");
        }
    }
);
```
- 代码中execute里面是一个类的实例（对象）
- TransactionCallback<\T> 带泛型的核心作用就是：允许不同的返回类型
```java
// 情况1：返回字符串
TransactionCallback<String> stringOperation = status -> {
    // 执行数据库操作
    return "操作成功";
};
String message = transactionTemplate.execute(stringOperation);
```
- 优化
- 现代 Java（8+）推荐使用 **Lambda 表达式** 替代匿名内部类
```java
// 现代写法（推荐）
transactionTemplate.execute(status -> {
    updateDB(ctx, dbAction);
    publish(ctx, publishAction);
    return null;
});
```
- 最推荐的方式是使用 **声明式事务**（`@Transactional` 注解）：
```java
@Transactional
public void processWithTransaction(ProcessContext ctx, DBAction dbAction, PublishAction publishAction) {
    updateDB(ctx, dbAction);
    publish(ctx, publishAction);
}
```
- 使用注解必须非private，因为sping会开启transaction调用这个方法，private的话只能在类里面调用，spring 没法调用

关于Spring 注解与 `private` 的关系：
- 根本原因：Java 的访问控制和代理机制的限制
Spring 注解的两大类别
类别1：需要 AOP 代理的注解（**不能** private）
这些注解需要 Spring 在运行时拦截方法调用：
```java
// ❌ 都不能是 private
@Transactional          // 事务管理
@Cacheable             // 缓存
@Async                 // 异步执行  
@Retryable             // 重试机制
@Scheduled             // 定时任务
@PreAuthorize          // 权限检查
@PostAuthorize         // 权限检查
@Timed                 // 监控计时
```
**原因**：这些都需要在方法执行前后添加额外逻辑。

类别2：通过反射调用的注解（**可以** private）
这些注解 Spring 通过反射直接访问，不需要代理：
```java
// ✅ 可以是 private
@PostConstruct         // 初始化方法
@PreDestroy           // 销毁方法  
@EventListener        // 事件监听
@Autowired            // 依赖注入（字段/方法）
@Value                // 值注入
```

为什么有这个区别？
AOP 代理的限制：
```java
// Spring 创建的代理类
public class MyServiceProxy extends MyService {
    
    private MyService target;
    
    // 只能重写 public/protected 方法
    @Override  
    public void saveData() {  // 不能重写 private 方法！
        // 开启事务
        transactionManager.begin();
        try {
            target.saveData();  // 调用真实对象
            transactionManager.commit();
        } catch (Exception e) {
            transactionManager.rollback();
        }
    }
}
```

反射的能力：
```java
// Spring 可以通过反射访问 private 方法
Method method = MyService.class.getDeclaredMethod("init");
method.setAccessible(true);  // 打破 private 限制
method.invoke(serviceInstance);  // 直接调用
```

记忆技巧
**判断规则**：
- 如果注解需要在**方法执行前后**添加逻辑 → 需要代理 → **不能** private
- 如果注解只是**标记方法**让 Spring 知道 → 通过反射 → **可以** private

enquiryNode instanceof ObjectNode maskedEnquiry：
- 如果 enquiryNode 是 ObjectNode 就 自动将它强转并赋值给变量 maskedEnquiry

updateExistingLeadWithStatus：
- 如果已存在的lead是allocated，那么直接调用leadEventRepository.insert(leadEvent) 在lead event 表中插入一条新数据 其中rawleadpayload作为change descripiton，不更新lead表（lead表对于allocated的更新应该在lead status consumer已完成）
- 如果不是allocated，正常 更新lead 和 插入lead event 表
- lead event 表中的数据 一律都mask pii，但是lead表中的数据只有当junk 或 unqualified 才 mask pii
	- lead 表是业务运营数据，下游还需要用到pii 信息，除非lead进来时已经是最终状态（junk / unqualified），不会再下游，那就需要mask pii（allocated状态更新应该在leadstatusconsumer 有处理，这里不用考虑）

leadAggregateRepository：涉及多个db表的操作所以叫aggregate

doProcessWithTransaction 用transactionTemplate开启事务，内部createLeadWithStatus用Transactional注解开启事务：
- 内层沿用外层的事务（事务传播，默认行为），所以在这个情况下内部开启事务没有实际作用
- createLeadWithStatus保留transaction注解，以防直接调用
- 推荐用同样的事务管理方式，都用注解



todo：
- 数据类型像 ProcessContext.java 注释放在一起看再check
- rawPayloadUpdater 为什么需要这个




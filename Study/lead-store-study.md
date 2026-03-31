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

IncomingLeadConsumer:
jsonHelper convert bytes to LeadIncomingPayload
processLead
constraintValidator.validate(leadIncomingPayload)
jsonHelper convert RawLeadPayload to PartialRawLeadPayload
constraintValidator.validate(partialRawLeadPayload)
incomingLeadProcessor.process(

constraintValidator.validate(partialRawLeadPayload):
- validates the data structure of partialRawLeadPayload using Jakarta Bean Validation (formerly Java Bean Validation) constraints
- If any constraints are violated, it throws a ConstraintViolationException (If pass, continue silently)
- must have this validation function call, validation annotations are just metadata attached

Lombok.Builder:
- can add field values with conditions, no need to have constructor with different params
- immutable, once built, if want to update, must build a new one
- avoid with small class or all fields mandatory

LeadRepository：
1. LeadRepository 是一个 Spring Data 的接口，用于管理 Lead 实体的数据访问，提供了自定义和标准的数据库操作方法。
2. 由于它只继承了 org.springframework.data.repository.Repository（一个标记接口），所以必须手动声明 findById、existsById 等方法，否则这些方法不会自动提供。
3. Repository 只是接口，没有实现。 Spring Data JDBC 会在运行时根据**接口定义**自动生成这些方法的实现，无需开发者手动实现。（与注解没有关系）
@Repository注解
- **标记为数据访问层组件**：告诉 Spring 这是一个数据访问对象（DAO）
- **自动扫描注册**：Spring 容器会自动发现并注册为 Bean
- **异常转换**：将底层数据访问异常转换为 Spring 的 DataAccessException 层次结构
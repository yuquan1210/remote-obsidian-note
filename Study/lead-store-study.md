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

IncomingLeadConsumer
jsonHelper convert bytes to LeadIncomingPayload
processLead
constraintValidator.validate(leadIncomingPayload)
jsonHelper convert RawLeadPayload to PartialRawLeadPayload
constraintValidator.validate(partialRawLeadPayload)


constraintValidator.validate(partialRawLeadPayload):
- validates the data structure of partialRawLeadPayload using Jakarta Bean Validation (formerly Java Bean Validation) constraints
- If any constraints are violated, it throws a ConstraintViolationException (If pass, continue silently)
- must have this validation function call, validation annotations are just metadata attached
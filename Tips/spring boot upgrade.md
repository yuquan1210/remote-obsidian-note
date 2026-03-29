原因是 Spring Framework 7.0 的新替代类（JacksonJsonEncoder/JacksonJsonDecoder）使用的是 Jackson 3.x (tools.jackson.databind.json.JsonMapper)，而本项目全面使用的是 Jackson 2.x (com.fasterxml.jackson.databind.ObjectMapper)。

具体来说：

Jackson2JsonEncoder / Jackson2JsonDecoder → 使用 Jackson 2.x ObjectMapper（当前项目用的）
JacksonJsonEncoder / JacksonJsonDecoder → 要求 Jackson 3.x tools.jackson.databind.json.JsonMapper
要消除这些 warning 需要将项目的 整个 Jackson 依赖从 2.x 迁移到 3.x，包括：

ObjectMapper → tools.jackson.databind.json.JsonMapper
com.fasterxml.jackson.* → tools.jackson.*
所有 Jackson 相关模块（JavaTimeModule, ParameterNamesModule 等）换成 Jackson 3.x 版本
JsonHelper 及所有序列化/反序列化代码也需要更新
建议：这些 warning 是 deprecated and marked for removal，但在当前 Spring Framework 7.0.x 中仍然完全可用。可以暂时保留，等 Jackson 3.x 生态更成熟后再做统一迁移。
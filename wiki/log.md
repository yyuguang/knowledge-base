## [2026-07-13] create | mall-swarm mall-search 源码深度学习

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-search设计.md`：基于 mall-search、商品写入/portal 检索源码、Nacos 配置、SQL 与部署文件，记录 `pms` 索引/Document/mapping、MySQL→ES→API Mermaid 数据流、字段映射、查询聚合、手工同步、一致性风险、Redis 边界、防滥用缺口及关键词/过滤/向量/RAG 对比。

### updated

- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`
- `index.md`
- `log.md`（本次记录）

### note

- 本轮未修改 `mall-swarm` 源码；只写入 wiki。
- 未检出 mall-search 自动消息同步、定时补偿、失败重试、向量字段或 RAG/LLM 实现；均按风险或“二开建议”记录，未表述为现有能力。

---

## [2026-07-13] deep-learning | mall-swarm mall-auth

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-auth设计.md`
  - 以 auth/admin/portal/gateway 源码、POM、Nacos 导入配置和部署清单为证据，确认 `mall-auth` 是按 `clientId` 的 Feign 登录分流层，不是 OAuth2 或自主签发 Token 的统一认证中心。
  - 覆盖管理员/会员身份边界、Sa-Token Simple JWT、Redis 路径权限表、Token 生命周期、服务间上下文传播缺口、Gateway/下游旁路风险。
  - 新增登录到受保护 API 时序图、Token/用户上下文传播图、权限边界表及明确标记为“二开建议”的 AI 工具最小权限设计。

### updated

- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`（补充认证与授权结论及部署旁路风险）
- `index.md`（新增 mall-auth 概念页索引）
- `log.md`（本次记录）

### note

- 本轮未修改 `mall-swarm` 源码；只写入 wiki。
- Nacos 运行时 `mall-auth-*.yaml`、Sa-Token JWT/Redis 撤销行为及真实部署网络隔离未在受控源码中确认，均保留为待验证。

---

## [2026-07-13] create | mall-swarm 第三轮：mall-gateway 深度学习

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-gateway设计.md`：仅以网关源码、Nacos 配置导入、允许范围内的 auth/common 源码和目标服务基础配置为证据，梳理静态 `lb://` 路由与 discovery locator、路由图/路由表、Sa-Token 双账号鉴权、权限 Redis 状态、CORS、异常 envelope、Knife4j 聚合、WebFlux/MVC 衔接、过滤器顺序证据边界与 AI API 二开建议。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`：补入 Gateway 静态/动态路由、鉴权与 AI 边缘层结论入口。
- `index.md`：新增 mall-swarm mall-gateway 概念页入口。
- `log.md`（本次记录）。

### note
- 本轮未修改 `mall-swarm` 源码；只写入 wiki。
- Gateway 未配置自定义 `GlobalFilter` / `GatewayFilter` 排序、限流或流式专用 route filter；文档明确将框架内部 filter order、动态 locator 路由冲突、HTTP status、流式代理与 Redis 阻塞影响列为待运行时验证，未将其伪装为已证实能力。

---

## [2026-07-13] create | mall-swarm 第三轮：mall-admin 后台管理深度学习

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-admin设计.md`：基于 admin、common、mbg、admin 配置、相关 SQL 与 auth→admin Feign 接口，建立后台功能模块图、三张调用链时序图、RBAC 数据关系、SPU/SKU/库存和订单发货证据、OSS/MinIO 边界、高中低 API 风险清单与 AI 管理助手工具白名单草案。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`：补充 mall-admin 已证实边界、搜索同步与路径鉴权待验证项。
- `index.md`：新增 mall-admin 概念页入口。
- `log.md`（本次记录）

### note
- 本轮未修改 `mall-swarm` 中任何文件；只写入 wiki。
- `mall-admin` 内未找到本地 `@FeignClient`；商品上下架和订单后台操作均仅见本地 MySQL 写入，搜索/库存/物流/支付/消息的跨服务一致性没有被伪装为现有能力。
- AI 相关内容均明确为二开建议；所有写操作列为人工确认，不自动执行。

---

## [2026-07-13] create | mall-swarm 第三轮：mall-monitor 深度学习

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-monitor设计.md`：以 mall-monitor 源码、六个业务服务的 POM/Actuator 配置、共享 Logback、Docker/K8s 部署文件为证据，记录 Admin Server 启动/认证/Nacos 发现、客户端与 Actuator 暴露、ELK/Logstash 链路、请求关联边界、风险清单及 AI 可观测性二开指标。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`：补充监控与可观测性学习入口和关键安全边界。
- `index.md`：新增 mall-monitor 概念页入口。
- `log.md`（本次记录）

### note
- 本轮未修改 `mall-swarm` 源码；只写入 wiki。
- Nacos 中心配置、Spring Boot Admin 实际登记/探测、Actuator 运行时 endpoint 清单、Logstash pipeline/索引策略均未由仓库静态文件证实，已明确标记为待验证。

---

## [2026-07-13] create | mall-swarm 第二轮：mall-mbg 数据访问层深度学习

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-mbg设计.md`：以 MBG 配置、生成物、`mall.sql` 与 admin/portal 直接调用为证据，记录 76 表生成边界、领域分组、生成/手写 DAO/数据库关系图、Example/分页/批量更新风险、状态与金额约定、AI 数据访问安全表、再生成流程和待验证事项。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`：补充 mall-mbg 学习入口与数据访问/AI 数据最小化结论。
- `index.md`：新增 mall-swarm mall-mbg 概念页入口。
- `log.md`（本次记录）

### note
- 本轮未修改 `mall-swarm` 中任何文件；只写入 wiki。
- 表关系以字段命名和关系表只能作逻辑推断；SQL 未见物理 `FOREIGN KEY`，均保留为待验证而非强约束结论。

---

## [2026-07-13] update | mall-swarm mall-common 源码深度学习

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-common设计.md`：基于 `mall-common/**`、根 POM 与直接依赖服务的 POM/import/config，记录共享契约、Redis、异常、日志、OpenAPI/Feign/LoadBalancer 的真实边界；包含依赖图与 AI 二开边界表。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`（补充 mall-common 边界链接）
- `index.md`（新增 mall-common 概念页索引）
- `log.md`（本次记录）

### note
- 标识 MVC/Servlet 与 Gateway/WebFlux 的不兼容风险、分页页号差异、Redis 多态序列化、日志脱敏与请求追踪缺口；二开建议均不代表已有 AI 能力。

---

## [2026-07-13] create | mall-swarm 项目全景摸底（受限工程/部署证据）

### created
- `sources/ecommerce/mall-swarm/来源_mall-swarm_项目源码.md`：记录根构建、模块 POM、启动入口、配置、Docker/K8s/脚本、数据库脚本目录与 AI 依赖检索证据边界。
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`：输出模块地图、Mermaid 运行时组件图、高层请求/消息/数据边界、技术证据、AI 不确定项和核心业务学习路线。
- `entities/ecommerce/项目_mall-swarm.md`：建立项目实体与来源/综述入口。

### updated
- `index.md`：新增 Ecommerce / mall-swarm 的 source、overview、entity 索引。
- `log.md`（本次记录）

### note
- 本轮严格未读取 controller/service/mapper/SQL 内容；README 功能与在线演示均未升级为“已实现”结论。
- 可见工程和部署层未检出 LLM、向量库、RAG、Agent 或工作流依赖；这不排除后续源码范围存在相关实现。

---

## [2026-07-13] create | mall-swarm 第一轮：项目入口与模块地图

### created
- `sources/java/mall-swarm/来源_mall-swarm_项目入口与模块地图.md`：以根 POM、模块 POM、启动类、环境配置与网关配置为证据，记录模块依赖、七服务入口、Nacos 配置加载、路由与风险信号。
- `overview/java/mall-swarm/主题_mall-swarm_项目入口与模块地图.md`：形成 mall-swarm 的首张架构地图，明确已知边界、AI 二开切入原则与下一轮问题。

### updated
- `index.md`：新增 mall-swarm 的 source 与 overview 入口。
- `log.md`（本次记录）

### note
- 只读取了项目入口、模块与配置层，不对 controller/service/mapper 内部业务下结论。
- `mall-auth` 的本地 profile 会导入 Nacos 配置，但仓库 `config/` 中未发现对应 YAML；已作为待验证的可复现性风险记录。

---

## [2026-05-14] create | wiki/comparisons/java/ai-frameworks/Spring_AI_vs_LangChain4j.md

### created
- `comparisons/java/ai-frameworks/Spring_AI_vs_LangChain4j.md`：Spring AI vs LangChain4j 系统性对比（出身、设计哲学、API 风格、编排机制、Tool Calling、RAG、厂商可移植、Spring 耦合、成熟度、选择建议）。

### updated
- `index.md`：新增 Comparisons / Java AI Frameworks 索引。
- `log.md`（本次记录）

### note
- 当前 wiki 对 LangChain4j 无覆盖，对比中 LangChain4j 部分基于内部知识。建议后续对 LangChain4j 做一次 ingest。

---

## [2026-05-13] update | MyBatis 第三轮深挖：DefaultResultSetHandler 结果映射机制

### created
- `concepts/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制.md`：深挖 `DefaultResultSetHandler`，整理自动映射、显式属性映射、nested query、nested resultMap、延迟加载、对象创建策略、collection 合并和 `PendingConstructorCreation`。
- `Attachments/mybatis-resultsethandler-mapping-flow.excalidraw`：ResultSetHandler 结果映射流程图。

### updated
- `sources/java/mybatis/来源_MyBatis_3_源码.md`：新增结果映射链路与源码入口。
- `overview/java/mybatis/主题_MyBatis源码架构_综述.md`：新增第三轮深挖结论，移除 ResultSetHandler TODO，并更新后续路线。
- `concepts/java/mybatis/概念_MyBatis执行器缓存与插件链.md`：补充到 ResultSetHandler 深挖页的链接。
- `concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建.md`：补充 `ResultMap/ResultMapping` 与结果映射阶段的关系。
- `index.md`：新增第三轮 concept 和图表入口。
- `log.md`（本次记录）

---

## [2026-05-13] update | MyBatis 第二轮深挖：XML SQL 编译为 MappedStatement

### created
- `concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建.md`：深挖 `XMLMapperBuilder -> XMLStatementBuilder -> MapperBuilderAssistant -> MappedStatement.Builder`，解释 namespace、sql/include、databaseId、selectKey、SqlSource、resultMap/parameterMap、cache 和最终 `MappedStatement` 构建。
- `Attachments/mybatis-mappedstatement-build-flow.excalidraw`：MyBatis XML SQL 到 `MappedStatement` 的详细构建流程图。

### updated
- `sources/java/mybatis/来源_MyBatis_3_源码.md`：新增 XML SQL 编译链路与源码入口。
- `overview/java/mybatis/主题_MyBatis源码架构_综述.md`：新增第二轮深挖结论，移除 XML mapper 构建 TODO，并更新后续路线。
- `concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement.md`：补充到 XML 编译页的关系链接。
- `index.md`：新增第二轮 concept 和图表入口。
- `log.md`（本次记录）

---

## [2026-05-13] update | Spring Framework 第四轮深度学习：复杂源码链路专题

### created
- `concepts/java/spring-framework/概念_MergedAnnotations合并注解模型.md`：源码级整理 `MergedAnnotations`、`AnnotationTypeMapping`、`@AliasFor`、mirror set 与组合注解。
- `concepts/java/spring-framework/概念_ConfigurationClassParser配置类解析细节.md`：拆 `@ComponentScan`、普通 `@Import`、`DeferredImportSelector` 的解析顺序。
- `concepts/java/spring-framework/概念_Spring_WebFlux响应式处理链.md`：整理 `DispatcherHandler`、`HandlerResultHandler`、异常恢复和 Reactor Context。
- `concepts/java/spring-framework/概念_Spring_TestContext缓存与事务测试.md`：拆 `MergedContextConfiguration` cache key、listener 顺序、事务测试、MockMvc/WebTestClient。
- `concepts/java/spring-framework/概念_Spring_AOP责任链源码时序.md`：整理 advisor chain、`ReflectiveMethodInvocation.proceed()`、`TransactionInterceptor` 事务时序。
- `Attachments/spring-configuration-class-parser-flow.excalidraw`：配置类解析流程图。
- `Attachments/spring-aop-advisor-chain-sequence.excalidraw`：AOP advisor chain 与事务拦截时序图。

### updated
- `sources/java/spring-framework/来源_Spring_Framework_源码.md`：补充第四轮源码入口和对应概念页。
- `overview/java/spring-framework/主题_Spring_Framework源码学习_综述.md`：新增第四轮结论，更新未解决问题和下一步。
- `index.md`：新增 5 个 Spring Framework 源码专题入口和图表引用。
- `log.md`（本次记录）

---

## [2026-05-12] ingest | raw/code/mybatis-3 → MyBatis 3 源码知识层

### created
- `sources/java/mybatis/来源_MyBatis_3_源码.md`：整理 MyBatis 3 源码入口、版本边界、核心设计理念、生命周期、Mapper 代理、执行器、缓存和插件主线。
- `concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration.md`：编译 Resources、SqlSessionFactoryBuilder、XMLConfigBuilder、Configuration、SqlSessionFactory、SqlSession 生命周期。
- `concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement.md`：整理 MapperRegistry、MapperProxyFactory、MapperProxy、MapperMethod、MappedStatement 的绑定关系。
- `concepts/java/mybatis/概念_MyBatis执行器缓存与插件链.md`：整理 Executor 类型、一级缓存、二级缓存、四大 handler、InterceptorChain 插件机制。
- `overview/java/mybatis/主题_MyBatis源码架构_综述.md`：形成 MyBatis 源码架构综述与后续深挖路线。
- `Attachments/mybatis-source-architecture.excalidraw`：MyBatis 3 源码架构图。
- `Attachments/mybatis-execution-flow.excalidraw`：Mapper 调用与 SQL 执行流程图。

### updated
- `index.md`：新增 MyBatis source、overview、concept 索引入口。
- `log.md`（本次记录）

### note
- 本地 `raw/code/mybatis-3` 的 `pom.xml` 为 `3.6.0-SNAPSHOT`；官方稳定发布页面显示 `3.5.19` 为当前稳定版本且 Java Version 为 8。本次笔记已标注 snapshot 与稳定版边界。

---

## [2026-05-12] create | Spring 0-6 知识体系大纲：根基、核心容器、AOP、数据访问、Web、Boot、扩展

### created
- `overview/java/spring/主题_Spring知识体系_综述.md`
- `concepts/java/spring/概念_Spring根基_IoC_DI_Bean生命周期_资源属性.md`
- `concepts/java/spring/概念_Spring核心容器_BeanFactory_依赖注入_自动装配_循环依赖.md`
- `concepts/java/spring/概念_Spring_AOP模块_动态代理_通知_切点_事务代理.md`
- `concepts/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源.md`
- `concepts/java/spring/概念_Spring_Web层_DispatcherServlet_HandlerMapping_参数返回值_过滤器拦截器.md`
- `concepts/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter.md`
- `concepts/java/spring/概念_Spring扩展进阶_事件_FactoryBean_后置处理器_SPI.md`
- `Attachments/spring-knowledge-outline.excalidraw`
- `Attachments/spring-runtime-core-flows.excalidraw`

### updated
- `index.md`：新增 Spring 0-6 知识体系索引和流程图入口。
- `log.md`（本次记录）

### note
- 当前 raw 中没有独立 `spring-boot` 与 `mybatis-spring` 源码；Boot/MyBatis 相关页已标注源码边界和待补 raw。

---

## [2026-05-12] update | Spring AI 第三轮学习：设计模式整理

### created
- `concepts/java/spring-ai/概念_Spring_AI_设计模式.md`：系统整理 Spring AI 使用到的 Builder/Fluent Interface、Facade、责任链、模板方法、策略、适配器、AutoConfiguration 工厂、Repository、Observation、Prototype、Command 等模式，并解释为什么这样设计及工程收益。
- `Attachments/spring-ai-design-patterns.svg`：Spring AI 设计模式地图，可直接在 Obsidian 页面中展示。

### updated
- `index.md`：新增 Spring AI 设计模式概念索引。
- `sources/java/spring-ai/Spring_AI_源码.md`、`overview/java/spring-ai/主题_Spring_AI源码架构_综述.md`：补充到设计模式页的反向链接。
- `log.md`：记录本轮更新。

---

## [2026-05-12] fix | Spring AI 图表从 Excalidraw 源文件改为 SVG 内嵌展示

### created
- `Attachments/spring-ai-source-architecture.svg`
- `Attachments/spring-ai-chatclient-call-flow.svg`
- `Attachments/spring-ai-tool-calling-loop.svg`
- `Attachments/spring-ai-rag-pipeline.svg`
- `Attachments/spring-ai-autoconfiguration-chain.svg`

### updated
- Spring AI 相关来源页、综述页、概念页和 `index.md`：将图表嵌入从 `.excalidraw` 源文件切换为 `.svg`，让笔记页面直接展示图片；保留 `.excalidraw` 作为可编辑源文件。

---

## [2026-05-12] update | Spring Framework 第三轮深度学习：设计模式专题

### created
- `overview/java/spring-framework/主题_Spring_Framework设计模式_综述.md`
- `concepts/java/spring-framework/概念_Spring_Framework_创建型设计模式.md`
- `concepts/java/spring-framework/概念_Spring_Framework_结构型设计模式.md`
- `concepts/java/spring-framework/概念_Spring_Framework_行为型设计模式.md`
- `concepts/java/spring-framework/概念_Spring_Framework_面试高频设计模式问答.md`
- `Attachments/spring-framework-design-patterns-map.excalidraw`

### updated
- `sources/java/spring-framework/来源_Spring_Framework_源码.md`：补充设计模式专题入口和源码锚点。
- `overview/java/spring-framework/主题_Spring_Framework源码学习_综述.md`：新增第三轮设计模式结论。
- `index.md`：新增 Spring Framework 设计模式专题索引。
- `log.md`（本次记录）

---

## [2026-05-12] update | Spring Framework 第二轮深度学习：Core + 配置/依赖解析 + 数据访问 + Web/Test/Messaging

### created
- `concepts/java/spring-framework/概念_Spring_Core资源类型注解与转换底座.md`
- `concepts/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析.md`
- `concepts/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块.md`
- `concepts/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展.md`
- `Attachments/spring-framework-second-round-map.excalidraw`

### updated
- `sources/java/spring-framework/来源_Spring_Framework_源码.md`：补充第二轮关键源码入口与核心观点。
- `overview/java/spring-framework/主题_Spring_Framework源码学习_综述.md`：更新为两轮学习地图与后续深挖路线。
- `index.md`：新增 Spring Framework 第二轮概念页与图表引用。
- `log.md`（本次记录）

---

## [2026-05-12] ingest | raw/code/spring-framework → wiki/sources/java/spring-framework/来源_Spring_Framework_源码.md

## [2026-05-12] create | wiki/concepts/java/spring-framework/概念_Spring_Framework_模块体系与扩展点.md

## [2026-05-12] create | wiki/concepts/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期.md

## [2026-05-12] create | wiki/concepts/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发.md

## [2026-05-12] create | wiki/overview/java/spring-framework/主题_Spring_Framework源码学习_综述.md + Attachments/spring-framework-source-flow.excalidraw

---

## [2026-05-12] update | Spring AI 源码级深化：调用链 + 5 张 Excalidraw 架构/流程图

### created
- `overview/java/spring-ai/主题_Spring_AI源码架构_综述.md`：新增源码级主线综述，串联 AutoConfiguration、ChatClient、Advisor、Tool Calling、RAG、VectorStore、MCP。
- `Attachments/spring-ai-source-architecture.excalidraw`：Spring AI 源码分层架构图。
- `Attachments/spring-ai-chatclient-call-flow.excalidraw`：ChatClient call/stream 执行流图。
- `Attachments/spring-ai-tool-calling-loop.excalidraw`：ToolCallAdvisor ReAct 循环图。
- `Attachments/spring-ai-rag-pipeline.excalidraw`：RetrievalAugmentationAdvisor Modular RAG 流程图。
- `Attachments/spring-ai-autoconfiguration-chain.excalidraw`：Starter 到 AutoConfiguration 到 Bean 的装配链图。

### updated
- `concepts/java/spring-ai/概念_Spring_AI_架构设计.md`：补充源码级阅读路径、DefaultChatClient/RequestSpec/终端 Advisor 机制和架构图。
- `concepts/java/spring-ai/概念_Spring_AI_ChatClient_API.md`：补充 call/stream 源码执行链、Observation、entity() 上下文写入和扩展点。
- `concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链.md`：补充 DefaultAroundAdvisorChain 的 Deque.pop、OrderComparator、copy(after)、流式聚合等源码细节。
- `concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用.md`：补充 ToolCallAdvisor 接管路径、chain.copy(this)、DefaultToolCallingManager 和 MethodToolCallback 反射细节。
- `concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成.md`：补充 RetrievalAugmentationAdvisor before/after 源码步骤、默认并发线程池和调优点。
- `concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置.md`：补充 Bean 依赖图式阅读方式和覆盖点。
- `sources/java/spring-ai/Spring_AI_源码.md`：补充源码阅读路线和图表入口。
- `index.md`：新增 Spring AI 源码架构综述索引。

---

## [2026-05-12] update | Spring AI 知识库补缺：VectorStore + AutoConfiguration + MCP

### checked
- 已读取现有 Spring AI 来源页、实体页和同 area 概念页。
- 同 area 当前只有 1 source + 1 entity + 7 concept，未发现存活重复页；历史日志中的旧 `整体架构` / `DashScope集成` 等页面已不在 wiki 中，本次未恢复重复内容。

### created
- `concepts/java/spring-ai/概念_Spring_AI_VectorStore向量存储.md`：补充 VectorStore 接口、SearchRequest、Filter.Expression、Observation 模板方法。
- `concepts/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置.md`：补充 Starter 依赖组合、模型/ChatClient/VectorStore 自动配置机制。
- `concepts/java/spring-ai/概念_Spring_AI_MCP集成.md`：补充 MCP Client/Server、ToolCallbackProvider、McpToolUtils 双向桥接。

### updated
- `sources/java/spring-ai/Spring_AI_源码.md`：补充 3 个缺失核心观点和相关链接。
- `entities/java/spring-ai/项目_Spring_AI.md`：补充 VectorStore / AutoConfiguration / MCP 关联。
- `index.md`：Spring AI 域新增 3 个概念索引。

---

## [2026-05-10] ingest | Spring AI 完整源码重入库：1 source + 7 concepts + 1 entity

### created
- `sources/java/spring-ai/Spring_AI_源码.md`：10 核心观点 + 14 概念表 + 5 未解决问题
- `concepts/java/spring-ai/概念_Spring_AI_架构设计.md`：Generic Model + ChatModel + Advisor 链 + 多厂商原理
- `concepts/java/spring-ai/概念_Spring_AI_ChatClient_API.md`：Builder/RequestSpec/CallResponseSpec/StreamResponseSpec 接口级分析
- `concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链.md`：BaseAdvisor 模板方法 + 12 内置 Advisor + 链式流程
- `concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用.md`：@Tool → ToolCallAdvisor ReAct 循环全链路
- `concepts/java/spring-ai/概念_Spring_AI_RAG检索增强生成.md`：7 步 Modular RAG 流水线源码级分析
- `concepts/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆.md`：滑动窗口 + SystemMessage 保护 + 6 后端
- `concepts/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出.md`：JSON Schema 自动生成 + BeanOutputConverter
- `entities/java/spring-ai/项目_Spring_AI.md`：项目实体页

### updated
- `index.md`（新增 Java / Spring AI 域，7 个概念 + 1 个实体 + 1 个来源）
- `log.md`（本次记录）

---

## [2026-05-10] delete | 清理 Spring AI 全量内容（sources/concepts/entities + index）

用户确认 Spring AI 非必要知识域，删除：
- `sources/java/spring-ai/`（1 页）
- `concepts/java/spring-ai/`（7 页）
- `entities/java/spring-ai/`（2 页）
- index.md 中 Spring AI 相关条目

---

## [2026-05-10] update | Spring AI 续更：ChatMemory + StructuredOutput + ToolCalling 3 新页

### created
- `concepts/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆.md`：ChatMemory/ChatMemoryRepository 接口、MessageWindowChatMemory 滑动窗口、JDBC 持久化、MessageChatMemoryAdvisor 集成
- `concepts/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出.md`：BeanOutputConverter/listOutputConverter/MapOutputConverter 源码、JsonSchema 自动生成、entity() API 完整流程
- `concepts/java/spring-ai/概念_Spring_AI_ToolCalling工具调用.md`：@Tool/@ToolParam 注解、MethodToolCallback 反射执行、ToolCallAdvisor 编排循环、ToolCallingManager

### updated
- `index.md`（Spring AI 域扩展为 7 个概念页）
- `log.md`（本次记录）

---

## [2026-05-10] update | Spring AI 深度重写：反编译源码级架构分析 + 4 页全面重写/新增

### updated（4 页基于反编译 jar 深度重写）
- `concepts/java/spring-ai/概念_Spring_AI_ChatClient_API.md`：新增 ChatClient/Builder/RequestSpec/CallResponseSpec/StreamResponseSpec 接口级源码、DefaultChatClient 实现分析、完整数据流图
- `concepts/java/spring-ai/概念_Spring_AI_Alibaba_DashScope集成.md`：新增 DashScopeChatAutoConfiguration 源码级 Bean 创建链、DashScopeChatModel 构造器分析、DashScopeApi REST 层、DashScopeChatOptions 40+ 参数全表

### created（2 个新概念页）
- `concepts/java/spring-ai/概念_Spring_AI_整体架构.md`：四层架构图、ChatModel 接口、Auto-Configuration 启动链、多厂商可移植性设计
- `concepts/java/spring-ai/概念_Spring_AI_Advisor拦截链.md`：CallAdvisor/StreamAdvisor 接口、DefaultAroundAdvisorChain 链式调用、8 内置 Advisor 清单、自定义 BillingAdvisor 示例

### updated
- `index.md`（Spring AI 域扩展为 4 个概念页）
- `log.md`（本次记录）

---

## [2026-04-29] ingest | raw/AI/SpringAI → wiki/sources/java/spring-ai/SpringAI示例项目.md (+ 2 concepts + 2 entities + index + log)

### created
- `sources/java/spring-ai/SpringAI示例项目.md`
- `concepts/java/spring-ai/概念_Spring_AI_ChatClient_API.md`
- `concepts/java/spring-ai/概念_Spring_AI_Alibaba_DashScope集成.md`
- `entities/java/spring-ai/项目_Spring_AI.md`
- `entities/java/spring-ai/项目_Spring_AI_Alibaba.md`

### updated
- `index.md`（新增 Java / Spring AI 域）
- `log.md`（本次记录）

---

## [2026-04-27] ingest | raw/books/深入理解Java虚拟机 → wiki/sources/深入理解Java虚拟机.md (+ 10 concepts + 2 entities + index + log)

### created
- sources/深入理解Java虚拟机.md
- concepts/概念_JVM内存区域.md
- concepts/概念_垃圾收集算法.md
- concepts/概念_垃圾收集器.md
- concepts/概念_类加载机制.md
- concepts/概念_Class文件结构.md
- concepts/概念_字节码执行引擎.md
- concepts/概念_Java内存模型JMM.md
- concepts/概念_JIT编译优化技术.md
- concepts/概念_Java语法糖.md
- concepts/概念_Java线程安全与锁优化.md
- entities/人物_周志明.md
- entities/项目_HotSpot_VM.md

### updated
- index.md（新增索引条目）
- log.md（本次记录）

---

## [2026-04-27] fix | 规范修正：重命名+补frontmatter+补结构

全部页面按 TheSchema 规范修正：
- 命名加前缀（概念_/人物_/项目_）
- frontmatter 补 aliases / status / confidence / updated(time)
- 概念页补"在本知识库中的应用示例"
- 实体页补"事件/计划/实验链接"
- 内部链接全部更新为新路径

---

## [2026-04-27] fix | 关联加上下文说明

所有页面的关联链接增加 why 说明，解释每条链接的关联原因

---

## [2026-04-27] restructure | 迁移 JVM 内容至 java/ 子目录 + 更新 TheSchema.md

将 13 个 wiki 页面从扁平目录迁移至领域子目录：
- `sources/java/`、`concepts/java/`、`entities/java/`
- 全部 wikilink 更新为新路径
- TheSchema.md 新增「领域子目录」说明
- README.md 目录结构同步更新

---

## [2026-04-28] ingest | raw/AI/hello-agents → wiki/sources/agent/Hello-Agents教程.md

### created
- `sources/agent/Hello-Agents教程.md`
- `concepts/agent/概念_Agent_Loop.md`
- `concepts/agent/概念_ReAct范式.md`
- `concepts/agent/概念_Plan-and-Solve范式.md`
- `concepts/agent/概念_Reflection范式.md`
- `concepts/agent/概念_PEAS模型.md`
- `concepts/agent/概念_多智能体协作.md`
- `entities/agent/项目_HelloAgents.md`
- `entities/agent/人物_陈思州.md`

### updated
- `index.md`（新增 AI / Agent 域）
- `log.md`（本次记录）
- `index.md`（新增 AI / Agent 域，含 6 个概念条目 + 2 个实体条目）
- `log.md`（本次记录）

待确认后展开：
- 6 个概念页（Agent Loop / ReAct / Plan-and-Solve / Reflection / PEAS / 多智能体协作）
- 1 个实体页（人物_陈思州）

## [2026-04-28] restructure | 细分至 java/jvm/ 子目录 + 细化标签

进一步细分目录层级 (java/ → java/jvm/)：
- 路径：`java/jvm/` 子目录，为后续 java/concurrency/、java/framework/ 等腾出空间
- 标签：全部 13 个页面 tags 细化（如 gc → gc/gc-algorithm/garbage-collector）
- TheSchema.md / README.md / CLAUDE.md 同步更新

---

## [2026-04-28] update | Agent 概念页深度重写 + 新增 6 个概念页

### updated
6 个现有概念页补充完整 Python 代码实现：
- `concepts/agent/概念_Agent_Loop.md`：补充 OpenAICompatibleClient + 主循环 + 工具定义完整代码
- `concepts/agent/概念_ReAct范式.md`：补充 HelloAgentsLLM + ToolExecutor + ReActAgent 完整实现
- `concepts/agent/概念_Plan-and-Solve范式.md`：补充 Planner + Executor + PlanAndSolveAgent 完整实现
- `concepts/agent/概念_Reflection范式.md`：补充 Memory + ReflectionAgent + 三个提示词模板完整实现
- `concepts/agent/概念_PEAS模型.md`：补充旅行助手 PEAS 实例 + 工具代码映射
- `concepts/agent/概念_多智能体协作.md`：补充赛博小镇 SimpleAgent/NPCBatchGenerator/混合模式完整代码

### created
6 个新概念页：
- `concepts/agent/概念_ELIZA.md`：1966 首个对话系统，正则规则库 + 代词转换
- `concepts/agent/概念_Transformer架构.md`：MultiHeadAttention / PositionalEncoding / Encoder/Decoder 完整实现
- `concepts/agent/概念_BPE分词算法.md`：字节对编码算法完整实现与迭代过程
- `concepts/agent/概念_提示工程.md`：Zero/Few-shot、CoT、角色扮演、采样参数
- `concepts/agent/概念_记忆系统.md`：四层记忆 + RAGTool/MQE/HyDE + 评分公式
- `concepts/agent/概念_上下文工程.md`：GSSC 流程（Gather→Select→Structure→Compress）

### updated
- `sources/agent/Hello-Agents教程.md`：更新为完整章节摘要 + 12 个概念链接
- `index.md`：Agent 域重新组织为三组（核心范式 / LLM 基础 / 系统工程）
- `log.md`（本次记录）

---

## [2026-04-28] update | JVM 概念页深度重写 + 新增代码示例

### updated
3 个概念页补充生产级 Java 代码示例：
- `concepts/java/jvm/概念_JVM内存区域.md`：新增 StackOverflowError / Heap OOM / Metaspace OOM / Direct Memory OOM 触发代码，JOL 对象布局解析，TLAB 分配演示，jmap/jstat/jcmd 工具速查
- `concepts/java/jvm/概念_垃圾收集算法.md`：新增四种引用完整演示、GC Roots 枚举代码、finalize() 自我救赎、ThreadLocal 内存泄漏复现与修复
- `concepts/java/jvm/概念_垃圾收集器.md`：新增完整收集器组合参数表、CMS GC 日志逐行注释解析、G1 Young/Mixed/Full GC 日志解析、G1 Humongous 大对象分配、ZGC 调优实践与日志解读

---

## [2026-04-28] update | JVM 3 个概念页深度重写：类加载/Class文件/执行引擎，补充生产级 Java 代码

### updated
3 个 JVM 概念页在保留原有结构和 frontmatter 基础上，新增大段实际 Java 代码和命令示例：

- `concepts/java/jvm/概念_类加载机制.md`：
  - 静态内部类单例（`<clinit>` 线程安全机制演示）
  - ClassLoader 层级打印代码（AppClassLoader → Bootstrap）
  - 自定义 PathClassLoader（打破双亲委派，从文件系统加载 .class）
  - SPI/线程上下文类加载器 JDBC 演示（ServiceLoader + TCCL）
  - `-verbose:class` 输出与类加载顺序观察命令

- `concepts/java/jvm/概念_Class文件结构.md`：
  - `xxd` 十六进制转储（cafe babe、major_version 对照表）
  - 方法描述符格式完整对照表（(II)I、(Ljava/lang/String;)V 等）
  - `javap -verbose` 常量池 + Code 属性逐条字节码指令解读
  - ASM 运行时动态生成类完整代码（ClassWriter + MethodVisitor）

- `concepts/java/jvm/概念_字节码执行引擎.md`：
  - 静态分派（重载）完整演示（Human/Man/Woman 类型体系）
  - 动态分派（重写）vtable 演示（Son/Father/Daughter）
  - invokedynamic 双入口（Lambda + MethodHandle 四种句柄：静态/虚/构造/字段）
  - jstack 栈帧可视化 Java 程序 + 输出解读
  - MethodHandle vs Reflection 性能基准测试代码（1 亿次迭代对比）

---

## [2026-04-28] update | JVM 全 10 页深度重写完成 + 4 张 Excalidraw 图表

### updated（全 10 页已补充 Java 代码 + JVM 诊断命令 + Excalidraw 图表）
- 内存区域：OOM 触发 / JOL 分析 ｜ 垃圾收集算法：引用类型 / finalize 自救
- 垃圾收集器：GC 日志注解 ｜ 类加载：自定义 ClassLoader / TCCL
- Class文件：javap 全注解 ｜ 执行引擎：静态/动态分派 / invokedynamic
- JMM：volatile 可见性 / DCL ｜ JIT：逃逸分析 / 方法内联
- 语法糖：泛型擦除反射攻击 / Lambda 反编译 ｜ 线程安全：JOL 锁膨胀 / 死锁检测

### created（4 张 Excalidraw）
- `jvm-memory-layout` · `jvm-classloading-lifecycle` · `jvm-lock-inflation` · `jvm-object-layout`

### updated
- `index.md`（JVM 条目含代码描述 + 图表引用）
- `sources/java/jvm/深入理解Java虚拟机.md`（新增图表引用）
- `log.md`（本次记录）
## [2026-07-13] deep-learning | mall-swarm mall-demo

### created
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-demo设计.md`：定位、模块/运行时依赖、Feign 调用关系图、六项服务契约表、购物车调用时序、认证/TraceId/异常/可靠性边界，以及 AI 编排调用规范建议。

### updated
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`（补充 mall-demo 的教学定位、调用治理缺口与 AI 二开边界）
- `index.md`（新增 mall-demo 概念页索引）
- `log.md`（本次记录）

---

## [2026-07-13] deep-learning | mall-swarm mall-portal

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-portal设计.md`：前台功能地图；会员登录/身份、购物车促销、提交订单与库存/优惠/积分、支付宝回调与 RabbitMQ 延迟取消四条 Mermaid 时序；用户端领域模型、订单状态机、MySQL/Redis/MongoDB/RabbitMQ/Alipay 职责表、AI 客服工具权限矩阵及源码风险/待验证项。

### updated

- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`（补充 portal 本地订单事实链、存储职责与高优先级一致性/授权风险）
- `index.md`（新增 mall-portal 概念页索引）
- `log.md`（本次记录）

---
## [2026-07-13] deep-scan | mall-swarm 第二轮 P0：订单一致性与支付安全

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_订单支付一致性.md`：逐项复核订单事务代理、下单/库存并发、RabbitMQ 延迟取消、支付宝发起/回调、前后台状态约束、补偿与测试；严格区分代码事实、运行时事实和 AI 二开建议，并标注 A/B/C 证据等级。

### updated

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-portal设计.md`（修正“订单方法没有事务”的历史表述，并链接 P0 复核页）
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`（RabbitMQ/Alipay 调用事实与 P0 风险优先级）
- `index.md`、`log.md`（本次记录）

### verification note

- 未修改 `mall-swarm` 源码；只增量更新 wiki。
- 已执行 `mvn -pl mall-portal -am test -DskipTests=false`：依赖模块编译通过；portal 两个 `@SpringBootTest` 因本机 MySQL `Connection refused` 无法加载上下文，Nacos 也不可达。现有测试不覆盖订单、支付、MQ、授权或并发，故本轮没有 A 级业务结论。

---
## [2026-07-13] deep-learning | mall-swarm 第二轮 P0：认证、网关旁路与部署暴露面

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_认证网关与部署安全.md`：汇总 Gateway 过滤顺序和白名单、Docker/Kubernetes/Nacos 可达性、双 Sa-Token/JWT/Redis session、下游身份验证缺口、Actuator/OpenAPI/ES/回调/Monitor 面、Feign Header 风险，以及 AI 用户委托、服务身份、tool scope、二次确认、审计和拒绝策略。

### updated

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-auth设计.md`
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-gateway设计.md`
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-monitor设计.md`
- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`
- `index.md`、`log.md`

### note

- 仅将知识写入正式知识库；未修改 mall-swarm 源码。
- Docker Compose 可解析；本机无 Kubernetes context 且未运行 mall 服务。真实 HTTP、生产 Nacos/Ingress/NetworkPolicy/Secret 和 Redis 多实例结果均保留为 C 级待验证。

---
## [2026-07-13] create | mall-swarm 第二轮 P1：Nacos 配置与运行态验证

### created

- `concepts/ecommerce/mall-swarm/概念_mall-swarm_Nacos配置与运行态验证.md`：记录 Nacos 配置加载模型、服务—Data ID—中间件矩阵、Compose/Kubernetes 对照、静态配置漂移、A/B/C 证据、只读验证命令和 AI 二开部署前置条件。

### updated

- `overview/ecommerce/mall-swarm/主题_mall-swarm_架构全景_综述.md`
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_认证网关与部署安全.md`
- `concepts/ecommerce/mall-swarm/概念_mall-swarm_mall-monitor设计.md`
- `index.md`
- `log.md`（本次记录）

### note

- 未修改 `mall-swarm` 源码、Docker/Kubernetes/Nacos 配置或外部环境；只写入正式 wiki。
- 本机仅完成 Docker、Compose、kubectl context 与 Nacos readiness 的只读探测。未发现 mall 容器、无 Kubernetes context、Nacos 配置地址不可达；不将这些局部观察写成生产结论。
- 未写入任何密码、Token、私钥或连接串值；后续 Nacos/Secret/Actuator 核验只记录键、来源、版本/状态和脱敏结果。

---

## [2026-07-13] update | Windows 路径兼容性修复

- 将 Spring AI 源码目录从 `raw/code/spring-ai/` 缩短为 `sai/`，消除 15 个超过 Windows 传统 260 字符限制的路径。
- 同步更新 16 个 Spring AI wiki 页面中的源码引用，保持来源可追溯。

---

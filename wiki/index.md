---
updated: "2026-07-19"
---

# Wiki 索引

> [!important] 当前架构（2026-07-19 起）
> 旧的类型优先目录已完成迁移并删除。请从 [[90-system/architecture/知识库架构骨架|知识库架构骨架]] 开始，按领域、项目、来源或导航地图归档。

## 当前入口

- [[40-maps/知识库导航]] — 从问题、领域、项目和资料四种入口进入知识库
- [[10-domains/java/_index]] — Java 长期知识地图；连接 JVM、Spring、MyBatis、算法等历史内容
- [[10-domains/ai/_index]] — AI 长期知识地图；当前含 Agent 主题
- [[10-domains/ecommerce/_index]] — 电商领域的可复用知识入口
- [[20-projects/mall-swarm/_index]] — mall-swarm 项目工作区，连接既有项目资料
- [[30-sources/_index]] — 资料摘要的新归档规则与历史来源入口
- [[_meta/迁移说明]] — 此次目录切换的范围、兼容策略和后续维护规则

## 已迁移页面索引

下方条目保留原有主题结构，并已全部改写到当前目录；新页面继续遵循当前架构。

## Sources（来源摘要）

### 数据结构 / 《大话数据结构》

- [[30-sources/books/大话数据结构/index]] — 程杰《大话数据结构》的书内专属索引：九章复习页、章节依赖、学习边界与待迁移算法主题

### 电商 / mall-swarm

- [[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]] — 受限于工程/部署层证据的技术栈、模块、部署、外部依赖与 AI 可见性索引
- [[30-sources/repositories/mall-swarm/来源_mall-swarm_项目入口与模块地图]] — mall-swarm 当前源码的 Maven 模块、服务入口、Nacos 配置、网关路由与风险证据
- [[30-sources/books/深入理解Java虚拟机]] — Java 虚拟机核心技术权威指南
- [[30-sources/courses/Hello-Agents教程]] — Datawhale 系统性智能体学习教程
- [[30-sources/repositories/Spring_AI_源码.md]] — Spring AI 2.0-SNAPSHOT 完整源码分析（40+ 模块，14+ 厂商，22+ 向量库）
- [[30-sources/repositories/来源_Spring_Framework_源码]] — Spring Framework 源码入口：模块地图、IoC refresh、Bean 创建、AOP/事务、MVC/WebFlux 分发
- [[30-sources/repositories/来源_MyBatis_3_源码]] — MyBatis 3 源码入口：Configuration、MapperProxy、MappedStatement、Executor、缓存和插件链

## Overviews（综述）

### Java / 数据结构与算法

- [[10-domains/java/data-structures-algorithms/主题_数据结构与算法_综述]] — 面向 Java 开发者的数据结构、算法范式与题型复盘章节入口

### 电商 / mall-swarm

- [[20-projects/mall-swarm/architecture/主题_mall-swarm_架构全景_综述]] — mall-swarm 模块、运行时组件、部署形态、数据/消息边界和 AI 二开学习路线
- [[20-projects/mall-swarm/architecture/主题_mall-swarm_项目入口与模块地图]] — mall-swarm 九模块、七服务、Nacos/Gateway/Monitor 的第一张源码证据地图
- [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]] — 从 AutoConfiguration 到 ChatClient、Advisor、Tool Calling、RAG、VectorStore、MCP 的源码级架构主线 ![[Attachments/spring-ai-source-architecture.svg]]
- [[10-domains/java/spring/主题_Spring知识体系_综述]] — 按 0-6 编号组织的 Spring 面试/源码知识体系 ![[Attachments/spring-knowledge-outline.excalidraw]] ![[Attachments/spring-runtime-core-flows.excalidraw]]
- [[10-domains/java/mybatis/主题_MyBatis源码架构_综述]] — MyBatis 从配置编译、XML mapper 解析、Mapper 代理到 Executor 执行、ResultSetHandler 映射、缓存和插件扩展的源码架构 ![[Attachments/mybatis-source-architecture.excalidraw]] ![[Attachments/mybatis-execution-flow.excalidraw]] ![[Attachments/mybatis-mappedstatement-build-flow.excalidraw]] ![[Attachments/mybatis-resultsethandler-mapping-flow.excalidraw]]

## Concepts（概念）

### Java / 数据结构与算法

- [[10-domains/java/data-structures-algorithms/foundations/概念_算法复杂度分析]] — 用渐近符号分析时间与空间成本，是比较方案和估算规模边界的共同语言
- [[10-domains/java/data-structures-algorithms/foundations/概念_数据结构抽象与Java实现]] — 区分 ADT 与具体实现，并将常见结构映射到 Java 集合和自定义实现

### 电商 / mall-swarm

- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-mbg设计]] — MBG 全表生成、76 张领域表、Example/分页/批量操作、手写 DAO 边界与 AI 数据安全边界
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-portal设计]] — 前台会员/购物车/订单/库存/支付功能地图、四条时序、状态机、存储职责及 AI 客服权限边界
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_订单支付一致性]] — P0 订单事务代理、库存并发、支付宝、延迟取消、前后台状态机、测试缺口与 AI 工具闸门复核
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-monitor设计]] — Spring Boot Admin Server/Nacos 发现、六个 Actuator Client、ELK 日志链路、管理面安全风险与 AI 可观测性二开边界
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-admin设计]] — 后台功能地图、Sa-Token/RBAC、SPU/SKU/库存、订单发货、对象存储及 AI 管理助手安全边界
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-gateway设计]] — Gateway 静态/服务发现路由、Sa-Token 双账号鉴权、Knife4j 聚合、WebFlux/MVC 边界与 AI API 接入建议
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-common设计]] — mall-common 的跨服务契约、Redis/MVC 日志封装、WebFlux 边界与 AI 二开公共层建议
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-search设计]] — ES `pms` 商品索引、MySQL 手工同步、关键词/聚合检索、最终一致性风险与 RAG 二开边界
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-auth设计]] — mall-auth 登录分流、admin/member 双身份体系、Sa-Token Simple JWT、Gateway 权限边界、旁路风险与 AI 二开授权建议
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_认证网关与部署安全]] — 第二轮 P0：Gateway 旁路、部署暴露、Sa-Token 生命周期、敏感端点与 AI 最小权限方案
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_Nacos配置与运行态验证]] — 第二轮 P1：Nacos Data ID/加载模型、Docker/Kubernetes 对照、运行态探测、配置漂移与 A/B/C 验证入口
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-demo设计]] — 可运行 MVC/MyBatis 示例、三组 OpenFeign 契约、全量 Header 透传风险与 AI 编排调用规范建议

### Java / JVM

- [[10-domains/java/jvm/概念_JVM内存区域]] — 运行时数据区划分、OOM 触发代码、JOL 对象布局分析 ![[Attachments/jvm-memory-layout.excalidraw]]
- [[10-domains/java/jvm/概念_垃圾收集算法]] — GC 算法、可达性分析、引用类型 demo、finalize 自救
- [[10-domains/java/jvm/概念_垃圾收集器]] — 各代收集器特性、GC 日志分析、JVM 参数组合
- [[10-domains/java/jvm/概念_类加载机制]] — 类加载 7 阶段、自定义 ClassLoader、线程上下文加载器 ![[Attachments/jvm-classloading-lifecycle.excalidraw]]
- [[10-domains/java/jvm/概念_Class文件结构]] — Class 文件格式、javap 字节码详解、ASM 字节码操作
- [[10-domains/java/jvm/概念_字节码执行引擎]] — 栈帧结构、静态/动态分派、invokedynamic 实现
- [[10-domains/java/jvm/概念_Java内存模型JMM]] — volatile 可见性 demo、DCL 单例、Happens-Before 验证
- [[10-domains/java/jvm/概念_JIT编译优化技术]] — 逃逸分析、方法内联、PrintCompilation 日志解读
- [[10-domains/java/jvm/概念_Java语法糖]] — 泛型擦除反射验证、自动装箱陷阱、Lambda 反编译
- [[10-domains/java/jvm/概念_Java线程安全与锁优化]] — 锁膨胀 JOL 实测、CAS ABA 解决、死锁 jstack 检测 ![[Attachments/jvm-lock-inflation.excalidraw]]

### Java / MyBatis

- [[10-domains/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — Resources、SqlSessionFactoryBuilder、Configuration、SqlSessionFactory、SqlSession 的生命周期与线程安全边界 ![[Attachments/mybatis-source-architecture.excalidraw]]
- [[10-domains/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — XMLMapperBuilder、XMLStatementBuilder、MapperBuilderAssistant 如何把 mapper XML 编译为 MappedStatement ![[Attachments/mybatis-mappedstatement-build-flow.excalidraw]]
- [[10-domains/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — MapperRegistry、MapperProxy、MapperMethod 如何把接口方法绑定到 MappedStatement
- [[10-domains/java/mybatis/概念_MyBatis执行器缓存与插件链]] — Executor、StatementHandler、ParameterHandler、ResultSetHandler、一级/二级缓存和 Interceptor 插件链 ![[Attachments/mybatis-execution-flow.excalidraw]]
- [[10-domains/java/mybatis/概念_MyBatis_ResultSetHandler结果映射机制]] — DefaultResultSetHandler 的自动映射、嵌套映射、延迟加载、对象创建与 collection 合并机制 ![[Attachments/mybatis-resultsethandler-mapping-flow.excalidraw]]

### Java / Spring AI

- [[10-domains/java/spring-ai/概念_Spring_AI_架构设计]] — Generic Model 抽象 + ChatClient + Advisor 链 + 多厂商可移植性
- [[10-domains/java/spring-ai/概念_Spring_AI_ChatClient_API]] — Builder 模式 + RequestSpec/CallResponseSpec/StreamResponseSpec + entity() 结构化输出
- [[10-domains/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — BaseAdvisor 模板方法 + 12 个内置 Advisor + 链式执行流程
- [[10-domains/java/spring-ai/概念_Spring_AI_设计模式]] — Builder、责任链、模板方法、策略、适配器、门面、自动配置工厂等模式如何服务于 Spring AI 的可扩展设计 ![[Attachments/spring-ai-design-patterns.svg]]
- [[10-domains/java/spring-ai/概念_Spring_AI_ToolCalling工具调用]] — @Tool 注解 → MethodToolCallback → ToolCallAdvisor ReAct Agent 循环
- [[10-domains/java/spring-ai/概念_Spring_AI_RAG检索增强生成]] — 7 步 Modular RAG 流水线（变换→扩展→检索→合并→后处理→增强）
- [[10-domains/java/spring-ai/概念_Spring_AI_ChatMemory对话记忆]] — MessageWindowChatMemory 滑动窗口 + 6 种持久化后端
- [[10-domains/java/spring-ai/概念_Spring_AI_StructuredOutput结构化输出]] — BeanOutputConverter JSON Schema 自动生成 + Jackson 反序列化
- [[10-domains/java/spring-ai/概念_Spring_AI_VectorStore向量存储]] — add/delete/similaritySearch + SearchRequest + Filter.Expression 跨存储过滤
- [[10-domains/java/spring-ai/概念_Spring_AI_AutoConfiguration自动配置]] — Starter 依赖组合 + 条件装配 + ChatClient.Builder prototype
- [[10-domains/java/spring-ai/概念_Spring_AI_MCP集成]] — MCP Client/Server 与 Spring AI ToolCallback 的双向桥接

### Java / Spring Framework

#### Spring 知识体系大纲

- [[10-domains/java/spring/概念_Spring根基_IoC_DI_Bean生命周期_资源属性]] — 0_Spring根基：IoC/DI、Bean 生命周期、Properties/YAML/占位符与资源属性解析
- [[10-domains/java/spring/概念_Spring核心容器_BeanFactory_依赖注入_自动装配_循环依赖]] — 1_核心容器：BeanFactory vs ApplicationContext、DI 方式、@Autowired、三级缓存
- [[10-domains/java/spring/概念_Spring_AOP模块_动态代理_通知_切点_事务代理]] — 2_AOP模块：JDK/CGLIB、通知顺序、切点表达式、@Transactional 代理原理
- [[10-domains/java/spring/概念_Spring数据访问_JdbcTemplate_事务传播_MyBatis_多数据源]] — 3_数据访问：JdbcTemplate、事务传播、MyBatis 整合机制、多数据源路由
- [[10-domains/java/spring/概念_Spring_Web层_DispatcherServlet_HandlerMapping_参数返回值_过滤器拦截器]] — 4_Web层：DispatcherServlet、HandlerMapping/Adapter、参数解析、返回值处理、Filter/Interceptor
- [[10-domains/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]] — 5_Spring Boot内核：自动配置、条件注解、run 启动过程、自定义 Starter（待补 Boot 源码）
- [[10-domains/java/spring/概念_Spring扩展进阶_事件_FactoryBean_后置处理器_SPI]] — 6_扩展与进阶：ApplicationEvent、FactoryBean、后置处理器、Spring Factories SPI

#### Spring Framework 源码专题

- [[10-domains/java/spring-framework/概念_Spring_Framework_模块体系与扩展点]] — core/beans/context 内核与 aop/tx/web/test 等模块如何通过生命周期和策略扩展点接入
- [[10-domains/java/spring-framework/概念_Spring_Core资源类型注解与转换底座]] — Resource、ResolvableType、MergedAnnotations、Environment、ConversionService 构成 Spring 元数据底座
- [[10-domains/java/spring-framework/概念_Spring_Framework_配置类解析与依赖解析]] — ConfigurationClassPostProcessor 编译注解配置，resolveDependency 解析注入点
- [[10-domains/java/spring-framework/概念_MergedAnnotations合并注解模型]] — MergedAnnotations、@AliasFor、mirror set 与组合注解如何形成统一注解语义
- [[10-domains/java/spring-framework/概念_ConfigurationClassParser配置类解析细节]] — @ComponentScan、@Import、DeferredImportSelector 的源码级解析顺序 ![[Attachments/spring-configuration-class-parser-flow.excalidraw]]
- [[10-domains/java/spring-framework/概念_Spring_Framework_IoC容器启动与Bean生命周期]] — `AbstractApplicationContext.refresh()` 与 `doCreateBean()` 的源码级主流程
- [[10-domains/java/spring-framework/概念_Spring_Framework_AOP事务与Web分发]] — AOP 代理、声明式事务、MVC/WebFlux 分发的共同执行骨架
- [[10-domains/java/spring-framework/概念_Spring_AOP责任链源码时序]] — advisor chain、ReflectiveMethodInvocation、TransactionInterceptor 的责任链时序 ![[Attachments/spring-aop-advisor-chain-sequence.excalidraw]]
- [[10-domains/java/spring-framework/概念_Spring_Framework_数据访问事务同步与集成模块]] — JdbcTemplate、DataAccessException、TransactionSynchronizationManager 与 JDBC/R2DBC/ORM/JMS 资源同步
- [[10-domains/java/spring-framework/概念_Spring_Framework_Web测试消息与应用层扩展]] — RequestMappingHandlerAdapter、TestContextManager、Messaging/JMS 的应用层策略模型
- [[10-domains/java/spring-framework/概念_Spring_WebFlux响应式处理链]] — DispatcherHandler、HandlerResultHandler、异常恢复与 Reactor Context
- [[10-domains/java/spring-framework/概念_Spring_TestContext缓存与事务测试]] — ContextCache key、TestExecutionListener 顺序、事务测试、MockMvc/WebTestClient
- [[10-domains/java/spring-framework/概念_Spring_Framework_创建型设计模式]] — BeanFactory、FactoryBean、AopProxyFactory、BeanDefinitionBuilder、DefaultSingletonBeanRegistry 的创建型模式
- [[10-domains/java/spring-framework/概念_Spring_Framework_结构型设计模式]] — AOP Proxy、HandlerAdapter、Composite、Decorator/Wrapper 的结构型模式
- [[10-domains/java/spring-framework/概念_Spring_Framework_行为型设计模式]] — Template Method、Strategy、Observer、Chain of Responsibility、Callback 的行为型模式
- [[10-domains/java/spring-framework/概念_Spring_Framework_面试高频设计模式问答]] — Spring 设计模式面试高频问答、回答公式和易错点

### AI / Agent

#### 核心范式
- [[10-domains/ai/agent/概念_Agent_Loop]] — 感知→思考→行动→观察循环（含完整 ~70 行实现）
- [[10-domains/ai/agent/概念_ReAct范式]] — ReActAgent + ToolExecutor 完整实现
- [[10-domains/ai/agent/概念_Plan-and-Solve范式]] — Planner + Executor + PlanAndSolveAgent
- [[10-domains/ai/agent/概念_Reflection范式]] — Memory + ReflectionAgent 三阶段迭代
- [[10-domains/ai/agent/概念_PEAS模型]] — 任务环境四要素度量框架
- [[10-domains/ai/agent/概念_多智能体协作]] — 赛博小镇 SimpleAgent per NPC + 批量生成

#### LLM 基础
- [[10-domains/ai/agent/概念_ELIZA]] — 1966 首个对话系统，正则模式匹配 + 代词转换
- [[10-domains/ai/agent/概念_Transformer架构]] — MultiHeadAttention / PositionalEncoding 完整实现
- [[10-domains/ai/agent/概念_BPE分词算法]] — 字节对编码，子词分词基础
- [[10-domains/ai/agent/概念_提示工程]] — Zero/Few-shot、CoT、角色扮演、采样参数

#### 系统工程
- [[10-domains/ai/agent/概念_记忆系统]] — 四层记忆 + RAG/MQE/HyDE 增强检索
- [[10-domains/ai/agent/概念_上下文工程]] — GSSC 流程：Gather→Select→Structure→Compress

## Overviews（综述）

### Java / Spring Framework

- [[10-domains/java/mybatis/主题_MyBatis源码架构_综述]] — MyBatis 源码架构、学习路径与后续深挖路线 ![[Attachments/mybatis-source-architecture.excalidraw]] ![[Attachments/mybatis-execution-flow.excalidraw]] ![[Attachments/mybatis-mappedstatement-build-flow.excalidraw]] ![[Attachments/mybatis-resultsethandler-mapping-flow.excalidraw]]
- [[10-domains/java/spring/主题_Spring知识体系_综述]] — Spring 0-6 知识大纲、源码锚点与流程图 ![[Attachments/spring-knowledge-outline.excalidraw]] ![[Attachments/spring-runtime-core-flows.excalidraw]]
- [[10-domains/java/spring-framework/主题_Spring_Framework源码学习_综述]] — Spring Framework 源码逐模块深度学习路线 ![[Attachments/spring-framework-source-flow.excalidraw]] ![[Attachments/spring-framework-second-round-map.excalidraw]]
- [[10-domains/java/spring-framework/主题_Spring_Framework设计模式_综述]] — Spring Framework 设计模式总览与面试表达 ![[Attachments/spring-framework-design-patterns-map.excalidraw]]

## Comparisons（对比）

### Java / AI Frameworks

- [[40-maps/比较/java/ai-frameworks/Spring_AI_vs_LangChain4j]] — Spring AI 与 LangChain4j 系统性对比：出身、设计哲学、API 风格、编排机制、厂商可移植性、Spring 耦合度与选择建议

## Entities（实体）

- [[20-projects/mall-swarm/项目_mall-swarm]] — mall-swarm 项目实体与证据入口
- [[10-domains/java/jvm/人物_周志明]] — 《深入理解Java虚拟机》作者
- [[10-domains/java/jvm/项目_HotSpot_VM]] — OracleJDK/OpenJDK 默认 JVM
- [[10-domains/java/spring-ai/项目_Spring_AI]] — Spring 官方 AI 框架，ChatClient + Advisor + RAG + Tool Calling
- [[10-domains/ai/agent/项目_HelloAgents]] — Datawhale 智能体教程开源项目
- [[10-domains/ai/agent/人物_陈思州]] — Hello-Agents 项目负责人

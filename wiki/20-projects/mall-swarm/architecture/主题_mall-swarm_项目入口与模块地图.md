---
type: "overview"
tags: ["java", "spring-cloud", "mall-swarm", "microservices", "architecture"]
summary: "mall-swarm 的第一张源码证据地图：九模块 Maven 聚合，Nacos 驱动注册/配置，Gateway 承担北向路由和统一鉴权，Admin/Portal/Search/Auth 是待逐模块展开的业务服务。"
sources:
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目入口与模块地图]]"
  - "wiki/30-sources/repositories/mall-swarm/来源_mall-swarm_项目入口与模块地图.md"
aliases: ["mall-swarm 架构综述", "mall-swarm 模块地图"]
status: "evolving"
confidence: 0.88
created: "2026-07-13"
updated: "2026-07-13"
---

# 主题：mall-swarm 项目入口与模块地图

## 本轮结论

当前代码库不是单体应用拆包，而是一个 Maven 父聚合工程：`mall-common`、`mall-mbg` 两个 jar 层被多个服务复用；`mall-gateway` 是明确的北向 HTTP 入口；Nacos 承担服务发现，并为六个服务提供环境配置导入；`mall-monitor` 是独立的监控服务。

业务域的精确边界尚不能仅由模块名得出。当前只确认：admin、portal、search、auth 分别是四个可执行业务相关服务，后续需按 controller → service → mapper/Feign 的实际调用链分别展开。

## 结构图

```text
客户端
  |
  v
mall-gateway :8201  -- 静态路由 + Sa-Token 鉴权 + Redis 权限规则
  |-- /mall-auth/**   -> lb://mall-auth   :8401
  |-- /mall-admin/**  -> lb://mall-admin  :8080
  |-- /mall-portal/** -> lb://mall-portal :8085
  |-- /mall-search/** -> lb://mall-search :8081
  `-- /mall-demo/**   -> lb://mall-demo   :8082

Nacos :8848 <--- 服务发现（全部七个服务）
             `--- 配置导入（除 mall-monitor 外的六个服务）

mall-monitor :8101 -- Spring Boot Admin Server

共享 jar：mall-common <- mall-mbg <- admin / portal / search / demo
```

图中路由、端口、依赖箭头均来自 [[30-sources/repositories/mall-swarm/来源_mall-swarm_项目入口与模块地图]] 所列 POM 与配置。图中不表达具体业务调用或数据库表归属。

## 运行时主线

```text
启动服务
-> SpringApplication.run
-> 读取 application.yml（默认 dev）
-> 连接 Nacos discovery；多数服务 import Nacos 环境配置
-> 服务以 spring.application.name 注册
-> Gateway 以 lb://service-id 选择下游实例
-> Gateway Sa-Token 先做登录/权限处理，再转发请求
```

这条主线把三个扩展面分开：

- **服务入口扩展**：新增独立服务，需要 Maven 模块、启动类、Nacos discovery/config、网关路由四处同时一致；是否还要接入监控要按 POM 的 Admin Client 决定。
- **北向 API 扩展**：现有服务路径需要与网关 `Path=/mall-<service>/**` 和 `StripPrefix=1` 协作；不能假设下游 controller 会收到服务前缀。
- **权限扩展**：后台路径的权限要求不只在下游控制器；网关会读取 Redis 的 `PATH_RESOURCE_MAP`。写入/刷新该映射的实现尚未阅读，不能在此页规定更新方式。

## AI 二次开发的起点

建议把 AI 能力先作为某一个可验证服务域的内部能力，而不是直接把模型调用塞进网关：网关当前承担 Reactor/Sa-Token/Redis 路由与认证边界；业务服务的同步/异步风格、事务边界、数据权限和可观测性仍未验证。一个可靠的下一步是先选择 `mall-admin` 的一个业务切片，追踪其 Controller、Service、Mapper 和权限资源初始化，再决定 AI 工具调用应放在 service、异步消费者还是独立 AI 服务。

这不是当前代码已实现的 AI 架构，而是基于现有边界做出的演进建议；落地前可对照 [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]] 的 ChatClient、Advisor、Tool Calling 与观测扩展点。

## 与既有知识的连接

- [[10-domains/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]]：解释服务启动时 starter 与自动配置如何接入。
- [[10-domains/java/spring/概念_Spring_Web层_DispatcherServlet_HandlerMapping_参数返回值_过滤器拦截器]]：下游 MVC 服务的 controller 调用链可用此页做阅读参照；网关本身为 WebFlux，不能直接等同。
- [[10-domains/java/mybatis/概念_MyBatis核心生命周期与Configuration]]：后续从 `mall-mbg` 深入数据库访问时的运行时基础。
- [[10-domains/java/spring-ai/主题_Spring_AI源码架构_综述]]：后续评估 AI 适配层的候选模式。

## 待探索问题

1. `mall-admin`、`mall-portal`、`mall-search` 各自有哪些 controller API、业务服务和数据表？
2. `AuthConstant.PATH_RESOURCE_MAP` 由谁构建、何时刷新，网关与下游是否存在双重权限校验？
3. auth 在 Nacos 中的实际远端配置是什么，为什么仓库未含镜像？
4. Feign 远程调用目前只用于 auth/demo，还是 admin/portal 的业务实现中也存在？
5. `mall-portal` 的 RabbitMQ/MongoDB 在哪些业务流程中使用，是否适合作为 AI 异步任务或知识检索的接入点？

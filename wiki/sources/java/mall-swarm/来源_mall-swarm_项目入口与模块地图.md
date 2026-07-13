---
type: "source"
tags: ["java", "spring-cloud", "mall-swarm", "source-code", "architecture"]
summary: "mall-swarm 当前工作区的项目入口证据：Maven 聚合结构、九个模块、七个可执行服务、Nacos 配置导入、网关路由与基础设施依赖。"
sources:
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm"
aliases: ["mall-swarm 项目入口", "mall-swarm 模块地图"]
status: "evolving"
confidence: 0.91
created: "2026-07-13"
updated: "2026-07-13"
---

# 来源：mall-swarm 项目入口与模块地图

## 本轮范围与证据边界

- 允许阅读范围：整个仓库。
- 实际阅读范围：根 `pom.xml`、九个模块 POM、各可执行服务启动类与 `application*.yml`、`config/` 中的环境配置、网关最小配置类、部署编排与说明文档。
- 未展开读取 controller、service、mapper、领域模型或 SQL；因此本页只对**工程入口、模块职责、部署依赖和请求入口**下结论，不把模块名称当作业务实现的证据。

## 版本与构建入口

根工程 `pom.xml` 的 `packaging` 为 `pom`，`<modules>` 聚合 `mall-common`、`mall-mbg`、`mall-demo`、`mall-admin`、`mall-search`、`mall-portal`、`mall-monitor`、`mall-gateway`、`mall-auth` 共九个模块。其属性明确为 Java 17、Spring Boot 3.5.14、Spring Cloud 2025.0.2、Spring Cloud Alibaba 2025.0.0.0。

**证据**：`pom.xml` 的 `<packaging>`、`<modules>` 与 `<properties>`。

## 模块地图（以 POM 依赖和启动入口为准）

| 模块 | 交付形态 / 启动入口 | 已验证职责线索 | 关键直接依赖（POM 证据） |
| --- | --- | --- | --- |
| `mall-common` | jar，无 `main` | 跨模块公共库 | Web、Redis、校验、LoadBalancer、OpenAPI 相关依赖 |
| `mall-mbg` | jar，无 `main` | 数据访问生成层；模块名之外还由 MyBatis Generator、MyBatis、MySQL 驱动依赖佐证 | 依赖 `mall-common`、PageHelper、MyBatis、Druid、Generator |
| `mall-admin` | `MallAdminApplication#main`，8080 | 后台服务（文档称后台管理；业务细节未在本轮验证） | `mall-mbg`、Nacos、Redis、OSS/MinIO、OpenFeign、Sa-Token、Admin Client |
| `mall-portal` | `MallPortalApplication#main`，8085 | 面向前台的服务（文档称移动端商城；业务细节未在本轮验证） | `mall-mbg`、MongoDB、Redis、AMQP、Nacos、OpenFeign、Alipay、Sa-Token、Admin Client |
| `mall-search` | `MallSearchApplication#main`，8081 | 商品搜索服务的技术承载；业务范围待查 controller/service | `mall-mbg`、Elasticsearch、Nacos、Admin Client |
| `mall-auth` | `MallAuthApplication#main`，8401 | 独立认证服务；通过 Feign 声明依赖 `mall-admin`、`mall-portal` | `mall-common`、Web、Nacos、OpenFeign、Admin Client |
| `mall-gateway` | `MallGatewayApplication#main`，8201 | WebFlux API 网关、路由和统一认证/鉴权入口 | `mall-common`、Nacos、Gateway WebFlux、Redis、Sa-Token Reactor、Knife4j、Admin Client |
| `mall-monitor` | `MallMonitorApplication#main`，8101 | Spring Boot Admin 监控服务 | Web、Nacos、Boot Admin Server、Spring Security |
| `mall-demo` | `MallDemoApplication#main`，8082 | Feign 远程调用示例服务 | `mall-mbg`、Web、Nacos、OpenFeign、Admin Client |

**证据**：各模块 `pom.xml`；各 `Mall*Application.java`；端口来自对应 `src/main/resources/application.yml`。`mall-auth/src/main/java/com/macro/mall/auth/service/UmsAdminService.java` 的 `@FeignClient("mall-admin")`、`UmsMemberService.java` 的 `@FeignClient("mall-portal")`；`mall-demo` 的三个 `Feign*Service` 分别指向 admin、portal、search。

## 启动、注册与配置加载链

七个可执行服务均以 `SpringApplication.run(...)` 进入 Spring Boot。七者均标注 `@EnableDiscoveryClient`；admin、portal、auth、demo 额外标注 `@EnableFeignClients`。monitor 额外标注 `@EnableAdminServer`。

除 `mall-monitor` 外，六个业务/边缘服务的 `application.yml` 均默认激活 `dev`，同名 `application-dev.yml` 使用：

```text
spring.config.import
  -> nacos:<服务名>-dev.yaml?refreshEnabled=true
```

并同时配置 Nacos discovery 与 config 地址为 `localhost:8848`；相应 prod profile 改为 `nacos-registry:8848`。`mall-monitor` 只配置 Nacos discovery，未见 config import。

**证据**：

- 启动类：各模块 `src/main/java/**/Mall*Application.java`。
- 配置模板：`mall-{gateway,auth,admin,search,portal,demo}/src/main/resources/application.yml` 与其 `application-{dev,prod}.yml`；`mall-monitor/src/main/resources/application-{dev,prod}.yml`。
- 外置配置镜像：`config/{gateway,admin,search,portal,demo}/mall-*-{dev,prod}.yaml`。

## 请求入口与路由边界

网关的本地 `application.yml` 静态声明五条路由：`/mall-auth/**`、`/mall-admin/**`、`/mall-portal/**`、`/mall-search/**`、`/mall-demo/**`，均转发到 `lb://<service-id>`，并通过 `StripPrefix=1` 去掉服务前缀；同时启用 discovery locator（小写 service ID）。未见到 `mall-monitor` 的业务路由。

网关 `SaTokenConfig#getSaReactorFilter` 将 `/mall-portal/**` 交给 `StpMemberUtil.checkLogin()`，将 `/mall-admin/**` 交给 `StpUtil.checkLogin()`，再从 Redis Hash `AuthConstant.PATH_RESOURCE_MAP` 读取路径-权限映射进行后台权限校验；白名单从 `IgnoreUrlsConfig` 绑定的 `secure.ignore.urls` 注入。

**证据调用链**：

```text
HTTP /mall-admin/**
-> mall-gateway/application.yml（Path + StripPrefix + lb://mall-admin）
-> SaTokenConfig#getSaReactorFilter
-> RedisTemplate.opsForHash().entries(AuthConstant.PATH_RESOURCE_MAP)
-> 下游 mall-admin
```

**证据**：`mall-gateway/src/main/resources/application.yml` 的 `spring.cloud.gateway.server.webflux.routes`、`secure.ignore.urls`；`mall-gateway/src/main/java/com/macro/mall/config/IgnoreUrlsConfig.java`；`mall-gateway/src/main/java/com/macro/mall/config/SaTokenConfig.java#getSaReactorFilter`。

## 可复现性与风险信号（均为源码/配置事实）

1. **认证服务远端配置镜像缺失。** `mall-auth/application-{dev,prod}.yml` 强制导入 `nacos:mall-auth-<env>.yaml`，但仓库 `config/` 仅含 gateway、admin、search、portal、demo 的同类文件，未发现 auth 对应 YAML。这不证明 Nacos 中不存在该配置，但从仓库无法复现 auth 的远端配置。待验证：检查 Nacos 的 `mall-auth-dev.yaml` / `mall-auth-prod.yaml` 实际 Data ID 与内容。
2. **网关是认证/权限策略的集中点。** `SaTokenConfig` 从 Redis 读取路径权限映射；因此 Redis 数据不可用或规则未初始化时的行为，须在运行环境验证，不能仅由代码断言为“放行”或“拒绝”。待验证：追踪 `AuthConstant.PATH_RESOURCE_MAP` 的写入方并进行 Redis 不可用、空 Map、匹配多资源三种集成场景。
3. **本地默认配置含开发凭据与高可见性管理端点。** 多服务 `application.yml` 明文写有本地数据库/中间件账号，且 actuator 暴露 `*`，部分服务设置 `env/configprops.show-values: always`。这是当前源码配置的暴露面，而非线上环境结论；prod Nacos 配置及部署 Secret 管理待验证。
4. **README 与当前代码的架构描述不完全同步。** README 的组织结构仍称 auth 为 Spring Security OAuth2，而当前 POM 与网关代码使用 Sa-Token；部署文档还引用已不在根聚合模块内的 `mall-registory`、`mall-config`。学习与二开时应以当前 POM、启动类、配置为事实来源，历史文档仅作线索。

## 已有知识连接

- [[overview/java/spring/主题_Spring知识体系_综述]]：理解 Spring Boot 的启动、配置和 Web 请求骨架。
- [[concepts/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]]：本项目每个服务的 `SpringApplication.run(...)` 都落在该启动模型之内。
- [[concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration]]：`mall-mbg` 与 admin/search/portal/demo 的 MyBatis 依赖关系，是后续数据访问链路阅读的入口。

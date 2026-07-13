---
type: "source"
tags: ["ecommerce", "mall-swarm", "spring-cloud", "source-code", "architecture"]
summary: "mall-swarm 项目全景的受限证据索引：仅依据根构建、模块 POM、启动入口、application 配置、部署清单、脚本和数据库脚本目录结构。"
sources:
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/README.md"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/pom.xml"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/mall-*/pom.xml"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/document/docker"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/document/k8s"
status: "evolving"
confidence: 0.9
created: "2026-07-13"
updated: "2026-07-13"
---

# 来源：mall-swarm 项目源码

## 本轮证据边界

本页是“项目全景摸底”的来源索引。只读取了：根 `README.md`、`LICENSE`、`pom.xml`；顶层目录；全部 `application*.yml`；`document/docker/`、`document/k8s/`、`document/sh/`；`config/`；数据库脚本目录结构；模块 POM 与 `*Application.java` 启动入口。

本轮**没有读取** controller、service、mapper、实体、SQL 内容或消息消费者实现。因此，业务流程、消息生产/消费、支付回调、库存扣减和订单状态机一律不从模块名、README 或依赖反推。

## 构建与许可证据

- 根工程：`pom.xml` 是 `packaging=pom` 的 Maven 聚合，模块为 common、mbg、demo、admin、search、portal、monitor、gateway、auth 共九个。
- 版本：根 POM 指定 Java 17、Spring Boot 3.5.14、Spring Cloud 2025.0.2、Spring Cloud Alibaba 2025.0.0.0；MyBatis 3.5.19、Sa-Token 1.42.0。
- 容器镜像：根 POM 的 `docker-maven-plugin` 以 `eclipse-temurin:17-jre` 为基础镜像，镜像名规则为 `mall/${project.name}:${project.version}`。
- 许可证：`LICENSE` 为 Apache License 2.0。

**证据**：根 `pom.xml` 的 `<modules>`、`<properties>`、`docker-maven-plugin` 配置；`LICENSE`。

## 模块与启动入口证据

| 模块 | 构建/启动状态 | 可证实的技术职责 | 证据 |
| --- | --- | --- | --- |
| `mall-common` | jar，无启动入口 | 公共依赖层 | `mall-common/pom.xml`：Web、Redis、校验、OpenAPI、LoadBalancer |
| `mall-mbg` | jar，无启动入口 | MyBatis/生成器数据访问层 | `mall-mbg/pom.xml`：MyBatis、Generator、Druid、MySQL |
| `mall-admin` | 可启动，8080 | 后台服务技术承载 | POM：mbg、Redis、OSS/MinIO、Feign、Sa-Token；`MallAdminApplication` |
| `mall-portal` | 可启动，8085 | 前台服务技术承载 | POM：mbg、MongoDB、Redis、AMQP、Alipay、Feign、Sa-Token；`MallPortalApplication` |
| `mall-search` | 可启动，8081 | 搜索服务技术承载 | POM：mbg、Elasticsearch；`MallSearchApplication` |
| `mall-auth` | 可启动，8401 | 认证服务技术承载 | POM：Web、Nacos、Feign；`MallAuthApplication` |
| `mall-gateway` | 可启动，8201 | WebFlux 网关、服务发现与文档聚合 | POM：Gateway WebFlux、Nacos、Redis、Sa-Token Reactor、Knife4j；`MallGatewayApplication` |
| `mall-monitor` | 可启动，8101 | 监控服务 | POM：Boot Admin Server、Security；`MallMonitorApplication` 标注 `@EnableAdminServer` |
| `mall-demo` | 可启动，8082 | 演示/远程调用试验服务 | 模块名、POM 中 OpenFeign/Nacos、`MallDemoApplication` |

除 `mall-common`、`mall-mbg` 外的七个模块都有 `SpringApplication.run(...)` 入口，且都带 `@EnableDiscoveryClient`。admin、portal、auth、demo 另有 `@EnableFeignClients`；monitor 另有 `@EnableAdminServer`。

**证据**：各 `mall-*/pom.xml`，各 `mall-*/src/main/java/**/Mall*Application.java`，各模块 `application.yml`。

## 配置、注册与外部依赖证据

- 所有服务的基础 `application.yml` 默认激活 `dev` profile；端口为 admin 8080、search 8081、demo 8082、portal 8085、monitor 8101、gateway 8201、auth 8401。
- admin、search、portal、gateway、auth、demo 的 `application-{dev,prod}.yml` 都以 `spring.config.import` 导入 Nacos 同名 Data ID，并同时配置 Nacos discovery/config；monitor 只配置 discovery。
- 配置中或 POM 中可见的外部组件：MySQL、Redis、Nacos、Elasticsearch、MongoDB、RabbitMQ、Logstash/Kibana、OSS、MinIO、Spring Boot Admin；Alipay SDK 仅由 portal POM 声明。
- `document/sql/` 只有 `mall.sql` 一个初始化脚本文件；未发现 Flyway/Liquibase 或其他迁移目录。其表结构和初始化内容未在本轮读取。
- `config/` 存有 gateway/admin/search/portal/demo 的 dev/prod Nacos 配置镜像；未发现 auth 对应镜像，尽管 auth profile 会导入 `mall-auth-{dev,prod}.yaml`。

**证据**：所有模块 `src/main/resources/application*.yml`；`config/**/mall-*.yaml`；`document/sql/` 目录结构；对应模块 POM。

## 部署形态证据

- **本地进程**：每个可执行模块都有 main 入口；默认 profile 是 dev。
- **Docker 镜像**：admin/search/portal/auth/gateway/monitor 的 POM 启用 Spring Boot 与 Fabric8 Docker Maven 插件；根 POM 定义统一 JRE 17 镜像模板。
- **Docker Compose**：`document/docker/docker-compose-env.yml` 组合 MySQL 5.7、Redis 7、Nginx 1.22、RabbitMQ 3.9.11-management、Elasticsearch/Logstash/Kibana 7.17.3、Mongo 4、Nacos 2.1.0；`docker-compose-app.yml` 编排 admin/search/portal/auth/gateway/monitor 六个应用镜像，未包含 demo。
- **Docker 脚本**：`document/sh/` 有上述六个应用的启动脚本；其 `--link` 参数显示 gateway 连接 Redis/Nacos，portal 连接 MySQL/Redis/Mongo/RabbitMQ/Nacos，search 连接 MySQL/Elasticsearch/Nacos，admin 连接 MySQL/Nacos。
- **Kubernetes**：`document/k8s/` 提供上述六个应用各一份 Deployment 和 Service 清单，副本数均为 1、profile 为 prod；gateway/monitor Service 为 NodePort，其余为 ClusterIP。无 demo 清单。

**证据**：根 `pom.xml`；`document/docker/docker-compose-*.yml`；`document/sh/*.sh`；`document/k8s/*.yaml`。

## README 的证据等级

README 的项目简介、业务图、在线演示和功能清单是**项目声明/演示线索**，不能单独证明当前分支已实现。尤其是 README 仍将 auth 描述为 Spring Security OAuth2，而当前 POM/配置中可直接确认的是 Sa-Token 相关依赖（admin、portal、gateway）与 auth 的 Nacos/Feign/Web 依赖。

后续业务页应优先以实际 Controller → Service → Mapper/Repository → 配置/消息端点为证据，再用 README 补充业务语义。

## AI 相关检索结论

在本轮允许读取的 README、根/模块 POM、`application*`、`config/`、Docker、K8s 与脚本中，未检出 Spring AI、OpenAI、LangChain、Ollama、LLM、Embedding、向量数据库、RAG、Agent 或工作流编排依赖/配置。

这只能说明**工程入口与部署层没有可见 AI 集成证据**；不证明整个源码不存在 AI 调用。验证需要在后续获准范围内搜索 Java 源码、资源文件和 CI/部署变量。

## 关联知识

- [[sources/java/mall-swarm/来源_mall-swarm_项目入口与模块地图]]：同一工作区的入口、网关路由与鉴权证据补充。
- [[concepts/java/spring/概念_Spring_Boot内核_自动配置_条件注解_启动过程_Starter]]：启动入口与 profile/config import 的 Spring Boot 基础。
- [[concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration]]：后续展开 `mall-mbg` 数据访问时的源码参照。

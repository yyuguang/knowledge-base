---
type: "concept"
tags: ["ecommerce", "mall-swarm", "nacos", "runtime", "docker", "kubernetes", "configuration", "security"]
summary: "第二轮 P1：以源码、配置、Compose 与 Kubernetes 清单交叉核验 Nacos 配置加载、服务依赖、部署漂移和运行态缺口；不把静态清单误作实际生效。"
sources:
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]]"
source_project: "/Users/yangyuguang/Documents/project_code/mall/mall-swarm"
status: "evidence-based"
confidence: 0.92
created: "2026-07-13"
updated: "2026-07-13"
---

# 概念：mall-swarm Nacos 配置与运行态验证

## 定义与范围

本页是第二轮 P1 的配置治理和运行态复核。范围覆盖所有服务的 `src/main/resources/application*.yml`、`config/**`、根与模块 POM、配置绑定类、测试/启动脚本、`document/docker/docker-compose-*.yml` 及 `document/k8s/**`。不修改项目、部署清单、Nacos 或外部环境；不记录任何密码、令牌、私钥或连接串值。

证据等级：**A** 为本轮真实只读命令/API/容器/Kubernetes 观察；**B** 为源码、环境配置和部署清单相互佐证；**C** 为必须取得真实 Nacos、Docker、Kubernetes、Secret、Ingress 或中间件权限后才能确认的事项。静态配置存在只说明“声明/预期”，不说明服务已运行、配置已加载或值已生效。

## 配置加载模型

| 服务 | 本地应用名/端口 | Profile 与 Nacos import | 远端镜像在 `config/` | 静态结论 |
| --- | --- | --- | --- | --- |
| admin | `mall-admin` / 8080 | dev、prod 分别 import `mall-admin-{profile}.yaml`，`refreshEnabled=true` | dev、prod 均有 | B |
| portal | `mall-portal` / 8085 | 同模式，Data ID 为 `mall-portal-{profile}.yaml` | dev、prod 均有 | B |
| auth | `mall-auth` / 8401 | 同模式，Data ID 为 `mall-auth-{profile}.yaml` | **未找到** | B：启动期依赖远端 Data ID 的证据存在；内容/是否存在为 C |
| gateway | `mall-gateway` / 8201 | 同模式，Data ID 为 `mall-gateway-{profile}.yaml` | dev、prod 均有 | B |
| search | `mall-search` / 8081 | 同模式，Data ID 为 `mall-search-{profile}.yaml` | dev、prod 均有 | B |
| demo | `mall-demo` / 8082 | 同模式，Data ID 为 `mall-demo-{profile}.yaml` | dev、prod 均有 | B |
| monitor | `mall-monitor` / 8101 | 仅 Nacos discovery；未见 Nacos config import | 不适用 | B |

所有 import 服务声明 `file-extension: yaml`，但静态文件未显式设置 Nacos `namespace` 或 `group`。因此隔离边界、实际使用的默认 namespace/group、Nacos ACL 用户、加密、历史版本和回滚能力均为 **C**，不能由“有 server-addr”推导出来。仓库也未见 Nacos ACL、配置加密或配置历史导出物。

本地 `application.yml` 与 `config/` 镜像对 admin（MySQL/Redis/OSS/MinIO）、portal（MySQL/Redis/MongoDB/RabbitMQ/支付）、gateway（Redis）、search（MySQL/ES）、demo（MySQL）声明了同类键；导入的远端 Data ID 也可能含这些键。故“候选来源重叠”是 **B**，而任一进程最终的 property source、刷新版本和实际值是 **C**，应以 `/actuator/env` 的脱敏只读核验或 Nacos API 导出为准。Gateway 路由与 Sa-Token、portal 支付、admin 对象存储、各服务中间件地址均不得因本地文件存在而认定未被远端覆盖。

## 服务—配置—中间件依赖矩阵

| 服务 | 代码声明依赖（B） | Compose 提供/链接（B） | K8s 清单提供（B） | 真正运行（A） |
| --- | --- | --- | --- | --- |
| admin | Nacos、MySQL、Redis、OSS/MinIO、Actuator/Admin Client | MySQL、Nacos；日志卷 | Deployment/ClusterIP；仅 profile、时区、Nacos 地址 env | 未验证 |
| portal | Nacos、MySQL、Redis、MongoDB、RabbitMQ、支付、Actuator/Admin Client | MySQL、Redis、Mongo、RabbitMQ、Nacos | Deployment/ClusterIP；仅 profile、时区、Nacos 地址 env | 未验证 |
| auth | Nacos、Feign、Redis 依赖、Actuator/Admin Client；远端 config 内容缺失 | Nacos | Deployment/ClusterIP；仅 profile、时区、Nacos 地址 env | 未验证 |
| gateway | Nacos、Redis、WebFlux、Actuator/Admin Client | Redis、Nacos | Deployment/NodePort；仅 profile、时区、Nacos 地址 env | 未验证 |
| search | Nacos、MySQL、Elasticsearch、Actuator/Admin Client | MySQL、ES、Nacos | Deployment/ClusterIP；仅 profile、时区、Nacos 地址 env | 未验证 |
| monitor | Nacos discovery、Spring Boot Admin Server、Actuator | Nacos | Deployment/NodePort；仅 profile、时区、Nacos 地址 env | 未验证 |
| demo | Nacos、MySQL、Feign、Actuator/Admin Client | 未纳入应用 Compose | 无 Deployment/Service | 未验证 |

“代码声明”来自 POM、应用配置和配置绑定：`secure.ignore` 使用 `@ConfigurationProperties`；portal 支付使用 `@ConfigurationProperties`；admin OSS/MinIO、缓存等大量以 `@Value` 注入。由此可定位会受 property source 漂移影响的功能，但不能证明 Bean 已成功绑定或依赖可连接。

## Docker、Kubernetes、Nacos 配置对照

| 面 | 可确认的静态事实 | 未确认的运行态事实/风险 |
| --- | --- | --- |
| Docker Compose | 两份 Compose 可被 `docker compose config` 静态展开；环境文件声明 MySQL、Redis、Nginx、RabbitMQ、ES、Logstash、Kibana、Mongo、单机 Nacos；应用 Compose 以端口发布六个服务并用 `external_links` 取依赖别名。 | 应用 Compose 无 `depends_on` 或 healthcheck；`external_links` 不是就绪保证。是否同一网络、镜像存在、容器健康、卷是否可写、端口是否实际监听为 C。 |
| Docker 中间件 | MySQL、Redis、RabbitMQ、ES、Mongo、Nacos 与部分管理/日志端口被 Compose 发布；数据目录以宿主机卷形式声明，Redis 启用 AOF，ES 为单节点。 | 认证、TLS、访问网段、备份恢复、实际卷内容及端口暴露均为 C；不能从端口映射推出公网可达。 |
| Kubernetes | 六个服务各 1 副本；Gateway、Monitor 是 NodePort，下游为 ClusterIP；容器仅注入 profile、时区、Nacos discovery/config 地址，日志为 hostPath。 | 仓库未见 ConfigMap、Secret、Ingress、NetworkPolicy、探针或 resources/limits 清单。实际集群是否另有这些对象、RBAC/准入、DNS、节点卷、服务端点和入口策略均为 C。 |
| Nacos | 六个服务各声明 discovery；除 monitor 外另声明 config import；镜像缺少 auth 的两个 Data ID。 | namespace/group/ACL/账户、Data ID 实存内容、历史版本、回滚、加密及服务实例为 C。 |

## 已验证的运行态事实

| 只读检查 | 观察结果 | 等级与边界 |
| --- | --- | --- |
| `docker ps` | Docker CLI/守护进程可访问；未见 mall 容器（仅有与本项目无关的容器）。 | A：此工作站当时的容器列表；不代表其他主机或生产。 |
| `docker compose -f ... config` | `docker-compose-env.yml` 与 `docker-compose-app.yml` 均静态展开成功。 | A：语法/变量展开；不代表镜像、网络、容器或依赖健康。 |
| `kubectl config current-context` | 没有可用当前 context。 | A：本工作站无可用集群上下文；生产集群状态仍为 C。 |
| 配置地址的 Nacos readiness GET | 连接失败（HTTP 无响应）。 | A：本工作站到该配置地址当时不可达；不能推导远端 Nacos 不存在或生产不可用。 |

因此本轮**没有任何 mall 服务、Nacos Data ID、容器健康、Kubernetes workload 或 HTTP health 的正向 A 级运行结论**。

## 配置漂移与安全风险

1. **P1：远端配置不可复现。** auth 引用了两个 Nacos Data ID，但仓库没有对应镜像；其 Redis/Feign/Admin Client 等最终配置无法离线审计。其他五个镜像也不等于 Nacos 中实际版本。影响：启动失败、认证/网关策略或中间件目标漂移。等级 B/C。
2. **P1：部署形态没有可验证的配置治理闭环。** K8s 不引用 ConfigMap/Secret，只有 Nacos 地址；Docker 使用外部链接且没有健康依赖。若 Nacos 未准备好或 Data ID 缺失，启动/刷新行为和失败降级不明。等级 B/C。
3. **P1：管理面与基础设施暴露缺乏实证。** Compose 发布多个中间件和应用端口；业务服务配置暴露全部 Actuator 且显示 env/configprops 值；K8s 的 Gateway/Monitor 使用 NodePort。认证、网络隔离、Secret 引用与外部 Ingress 未验证。等级 B/C。
4. **P1：安全关键键的有效来源不明。** Gateway 白名单/路由、Sa-Token 键、支付回调、OSS/MinIO、Redis/MySQL/RabbitMQ/ES 目标均可能受 Nacos 导入和环境变量影响；只可确认本地声明，不能确认生产生效值。等级 B/C。
5. **可用性不足。** K8s 清单均为单副本且无探针/资源限制；Compose Nacos 为单机、ES 为单节点。不存在 HA、滚动升级、持久化恢复或健康联动的 A 级证据。等级 B/C。

## A/B/C 证据清单

- **A：** 本工作站 Docker 守护进程可读、无 mall 容器；两份 Compose 静态展开成功；无 Kubernetes current context；Nacos readiness 连接失败。均是环境探测事实，不是部署成功或失败的全局结论。
- **B：** 七个应用名/端口；六个 Nacos config imports、monitor 仅 discovery；十份本地 config 镜像和 auth 缺失；中间件 POM/配置/绑定类；Compose 服务/端口/卷；K8s 六个 Deployment/Service、单副本、NodePort/ClusterIP、无仓库内 ConfigMap/Secret/Ingress/NetworkPolicy/探针/资源限制；现有测试多数为上下文加载且依赖真实基础设施。
- **C：** Nacos ACL/namespace/group/用户、Data ID 实存内容与历史/回滚、最终 property source、refresh 成功；Docker 网络/健康/监听/卷；K8s 实际对象与 RBAC/Secret/Ingress/NetworkPolicy；MySQL/Redis/RabbitMQ/Mongo/ES 的认证、备份、HA、真实暴露；Gateway/直连/Monitor/Actuator HTTP 可达性；支付回调与对象存储真实连通性。

## 解除 C 级不确定性的命令与权限

以下命令均为只读。执行者须使用脱敏输出；禁止把 `Secret` 数据字段、Nacos 配置值或 HTTP 返回中的凭据写入 wiki。

```bash
# 权限：Nacos namespace、服务实例、配置读取/历史与 ACL 审计权限。
# 用实际、受批准的 NACOS_URL / namespace / group / dataId 替换占位符；只记录状态、版本和键名。
curl -fsS "$NACOS_URL/nacos/v1/ns/namespace/list"
curl -fsS "$NACOS_URL/nacos/v1/ns/instance/list?serviceName=mall-gateway&namespaceId=$NAMESPACE"
curl -fsS "$NACOS_URL/nacos/v1/cs/configs?tenant=$NAMESPACE&group=$GROUP&dataId=mall-auth-prod.yaml"
curl -fsS "$NACOS_URL/nacos/v1/cs/history?tenant=$NAMESPACE&group=$GROUP&dataId=mall-auth-prod.yaml"
# ACL 接口是否可用、是否需要认证，只记录 HTTP 状态和策略元数据。
curl -sS -o /dev/null -w '%{http_code}\n' "$NACOS_URL/nacos/v1/auth/users"

# 权限：目标主机 Docker 只读 socket 权限。
docker compose -f document/docker/docker-compose-env.yml -f document/docker/docker-compose-app.yml ps
docker inspect --format '{{.Name}} {{.State.Status}} {{.State.Health.Status}}' $(docker ps -q)
docker network ls && docker port mall-gateway

# 权限：目标集群 list/get；仅查询 Secret 名称和引用，绝不导出 data。
kubectl get ns
kubectl -n "$NS" get deploy,po,svc,endpoints,ingress,networkpolicy
kubectl -n "$NS" get configmap
kubectl -n "$NS" get secret -o custom-columns=NAME:.metadata.name,TYPE:.type
kubectl -n "$NS" get deploy -o yaml | rg 'secretKeyRef|configMapKeyRef|image:|readinessProbe|livenessProbe|resources:'

# 权限：经批准的只读网络路径；只访问 health/info，不携带用户 Token。
curl -fsS "$GATEWAY_URL/actuator/health"
curl -fsS "$MONITOR_URL/actuator/health"
for u in "$ADMIN_URL" "$PORTAL_URL" "$AUTH_URL" "$SEARCH_URL"; do curl -fsS "$u/actuator/health"; done
```

若 Nacos 为受认证管理面，需由平台管理员以只读审计角色执行；若 `kubectl` 无 context，应先由集群管理员提供只读 kubeconfig 或在受控跳板机执行。对 `/actuator/env` 的核验应只比较 property source 名称、键存在性和脱敏状态，不保存值。

## 对 AI 二开部署架构的影响

当前证据不足以把 AI 服务接入现有 Nacos/网关/监控即视为可安全上线。上线前至少应：将模型密钥、向量库凭据和用户委托放入 Secret/外部密钥系统而非普通 Nacos 文本；对 AI workload 建立独立 namespace、最小 Nacos group/Data ID 和 RBAC；以 NetworkPolicy/mTLS 限制其只到工具网关、模型提供方和批准的向量/审计存储；为限流、审计、模型/向量依赖健康、故障降级和成本指标配置可验证的探针与告警。

AI 工具不得直接继承 Gateway 白名单、下游 ClusterIP 可达性或 Nacos 读取权限。用户委托、模型密钥、向量库、审计存储、限流、观测和降级目前均无 A 级运行保障；在完成下述 A 级矩阵前，只能作为设计建议，不得开放订单、支付、库存、权限或索引删除写工具。

## 相关链接

- [[20-projects/mall-swarm/architecture/主题_mall-swarm_架构全景_综述]] — 服务全景和本轮 P1 摘要。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_认证网关与部署安全]] — Gateway 旁路、管理面和部署安全。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-monitor设计]] — Admin Server、Actuator 与日志链路。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_订单支付一致性]] — 支付/RabbitMQ 运行态缺口。

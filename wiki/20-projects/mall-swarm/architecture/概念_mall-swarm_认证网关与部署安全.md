---
type: "concept"
tags: ["ecommerce", "mall-swarm", "security", "authentication", "authorization", "gateway", "deployment", "ai-readiness"]
summary: "第二轮 P0 安全核查：Gateway 非唯一可信入口；Docker 直接端口、Kubernetes 集群内旁路、全前缀白名单、下游缺少 HTTP 鉴权与全 Header Feign 透传共同扩大攻击面。"
sources:
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]]"
  - "wiki/30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码.md"
  - "mall-gateway/src/main/java/com/macro/mall/config/SaTokenConfig.java"
  - "document/docker/docker-compose-app.yml"
  - "document/k8s/"
aliases: ["mall-swarm 认证网关与部署安全", "Gateway 唯一可信入口核查", "mall-swarm 第二轮 P0"]
status: "evolving"
confidence: 0.92
created: "2026-07-13"
updated: "2026-07-13"
---

# 概念：mall-swarm 认证网关与部署安全

## 定义

本页把“请求是否一定经过 Gateway、身份是否可被下游独立验证、部署是否限制旁路”作为同一条安全边界核查。静态代码与部署清单的结论为 **B**；需要真实网络、Nacos、Secret、Ingress 或 Redis 多实例观察的结论为 **C**。当前没有 A 级 HTTP/集成验证：本机无 Kubernetes context，Docker 未运行 mall 服务。

## 当前结论

**Gateway 不是唯一可信入口。**Docker Compose 将 admin:8080、portal:8085、auth:8401、search:8081 全部映射到宿主机；Kubernetes 仅将 Gateway 暴露为 NodePort 30201，但 admin/portal/auth/search 仍是可被集群内工作负载访问的 ClusterIP，仓库未见 NetworkPolicy、Ingress、mTLS 或服务网格。Nacos 仅解决发现与配置导入，不提供此处可见的请求授权。

| 路径/部署面 | 当前边界 | 证据等级 |
| --- | --- | --- |
| Docker admin/portal/auth/search | 宿主机端口直接可达，能绕过 Gateway | B |
| Kubernetes Gateway | `NodePort:30201` | B |
| Kubernetes 下游 | ClusterIP，不等于只允许 Gateway；无 NetworkPolicy 清单 | B；集群实际策略 C |
| Nacos | 服务注册和 `lb://` 发现，不是访问控制 | B；运行时 ACL/配置 C |

## Gateway 顺序与白名单

Gateway 对 `/mall-auth/**`、`/mall-admin/**`、`/mall-portal/**`、`/mall-search/**`、`/mall-demo/**` 路由并 `StripPrefix=1`。`SaReactorFilter` 先应用 `secure.ignore.urls`；排除请求完全不进入其 auth 回调。未排除请求中，顺序为：`OPTIONS` 放行；portal 以 `memberLogin` 校验并 stop；admin 以默认 `StpUtil` 校验；再读取 Redis `auth:pathResourceMap`，对命中的 admin 路径执行任一资源权限校验。异常映射为业务 JSON 401/403/failed。

高风险白名单包括整段 `/mall-auth/**`、`/mall-search/**`、`/actuator/**`、文档路径，以及 `/mall-portal/alipay/**`。搜索段因此包含 ES `importAll/create/delete` 写操作。成功响应未见统一 WebFlux CORS 配置；错误处理才写 `Access-Control-Allow-Origin: *`，不能视为完整 CORS 策略。

## Token、会话与权限失效

admin 与 portal 分别在本服务以 BCrypt 校验密码后签发 Sa-Token JWT Simple token；admin 使用默认 `StpUtil`，portal 使用 member login type。两者从 `Authorization: Bearer` 读取 token，关闭 cookie，允许并发、每次登录新 token，固定超时 7 天且无 active timeout。Gateway 配置为 30 天，存在漂移。admin/portal 还使用相同硬编码 `jwt-secret-key`，Gateway 未配置该 key；均接入 Redis DAO，但跨实例校验、session key/序列化和撤销行为仍需 C 级验证。

`mall-auth /auth/login` 只是按公开 `clientId` 的 Feign 登录分流，不签 token。admin session 保存含资源权限快照的 `UserDto`；资源 URL 增删改会刷新路径—资源 Redis Hash，但未发现角色分配、用户禁用、改密或权限变更后对全部既有会话的统一 kickout/刷新。因此权限撤销可能滞后。

## 下游与敏感面

下游未发现 Sa-Token Servlet Filter、Spring Security 身份过滤器或 MVC Interceptor。因而不能信任“Gateway 已校验”作为服务自身授权边界：直接端口、同集群 Pod 或伪造 `X-User-*` Header 都不应被接受。应由 Gateway 删除外部内部身份 Header，注入带 `iss/aud/exp/jti` 的签名上下文；下游还要验证该证明，或使用 mTLS/服务网格身份。

| 面 | 真实暴露/边界 | 首要处置 |
| --- | --- | --- |
| Actuator/OpenAPI | 多服务 `include: '*'`、显示 env/configprops 值；Gateway 白名单管理与文档路径 | 仅管理网可达，allowlist 端点，关闭敏感值显示。 |
| ES 管理 | `/mall-search/**` 白名单；ES 9200/9300 Docker 也发布 | 拆分读写，写操作改为内部服务身份 + 精确 scope。 |
| 对象存储/回调 | MinIO upload 白名单；OSS callback 直指 admin:8080；支付宝段整体白名单 | 回调签名、时间窗、重放与来源校验；回调外不应整体公开。 |
| Monitor | 独立 NodePort 30101，源码固定 `macro/123456`，health/info 匿名 | Secret/IdP、VPN/管理网络、关闭公网 NodePort。 |

## Feign 与 AI 最小权限

`mall-auth` 的 Feign 调用不传用户委托或服务身份；`mall-demo` 则转发除 `content-length` 外的全部入站 Header，可能扩散 Authorization、Cookie、伪造代理头和未来敏感头。生产调用应改为 allowlist trace header，并使用独立、短期的服务凭据。

AI 服务不得持有用户原始 Authorization，也不得因 Header 自称为用户。最低方案：

1. Gateway 在已认证用户请求内签发 5 分钟、绑定 `jti`/请求/工具的 delegation JWT，含 `sub`、`actor=ai-service`、`aud=tool-gateway`、最小 scope；
2. AI workload 同时持有 mTLS/SPIFFE 或 client-credentials 服务身份，只能访问工具网关；
3. 工具 scope 精确到读写与资源归属，例如 `catalog.read`、`admin.product.draft`，禁止 `admin.*` 和高危通配；
4. 下单、退款、价格/库存/商品发布、权限、索引删除等写操作默认只预览，须由人类对工具、参数摘要、资源版本和委托 jti 二次确认；
5. 审计 `request_id/trace_id/delegation_jti`、用户/服务/模型/工具、scope、资源、参数摘要哈希、确认和结果；缺委托、scope 不足、资源不归属、确认失效、重放或审计不可写时 fail closed。

## 验证清单

- 在隔离环境分别通过 Gateway、Docker 宿主机直连、集群内 Pod 直连和外部 Ingress 测匿名/admin/member/伪造 Header/过期/注销/变权 token。
- 导出生产 Nacos ACL、Ingress、NetworkPolicy、Secret、服务发现实例与 Redis 多副本会话状态。
- 验证支付/OSS 回调签名重放、Actuator/OpenAPI/ES 的内外网可达性，以及角色、禁用、改密触发的会话撤销。

## P1 配置治理补充（2026-07-13）

Gateway 的 Nacos config import、K8s 仅注入 Nacos 地址且无仓库内 Secret/ConfigMap/Ingress/NetworkPolicy，以及 Compose 的直接端口发布均已由静态文件交叉确认（**B**）。但本工作站未发现 mall 容器、没有 Kubernetes context，配置地址的 Nacos readiness 也不可达；这不是生产状态结论，故 Gateway 路由、`secure.ignore`、Sa-Token 和 Redis 的**实际生效来源**仍是 **C**。不能把本地 YAML 当作生产白名单或密钥来源的证明。

详见 [[20-projects/mall-swarm/architecture/概念_mall-swarm_Nacos配置与运行态验证]] 的 Data ID 覆盖、部署差异及只读验证命令；该页要求只核对键、来源与脱敏状态，禁止记录 Secret 值。

## 相关链接

- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-gateway设计]] — 过滤器、路由和异常边界。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-auth设计]] — 双身份签发与权限 session。
- [[20-projects/mall-swarm/architecture/主题_mall-swarm_架构全景_综述]] — 服务与部署全景。

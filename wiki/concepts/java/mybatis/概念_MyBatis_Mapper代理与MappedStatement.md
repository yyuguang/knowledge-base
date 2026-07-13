---
type: "concept"
tags: ["java", "mybatis", "mapper", "proxy", "mappedstatement"]
summary: "Mapper 接口调用通过 JDK 动态代理进入 MapperProxy，再由 MapperMethod 将接口方法解析为 statement id、参数对象和返回值策略，最终定位 Configuration 中的 MappedStatement。"
sources:
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/binding/MapperRegistry.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/binding/MapperProxyFactory.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/binding/MapperProxy.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/binding/MapperMethod.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/mapping/MappedStatement.java"
aliases: ["MapperProxy", "MapperMethod", "MappedStatement", "MyBatis Mapper 动态代理"]
status: "evolving"
confidence: 0.9
created: "2026-05-12 21:37:31"
updated: "2026-05-13 09:14:29"
---

# 概念：MyBatis Mapper代理与MappedStatement

## 定义

Mapper 代理机制是 MyBatis 将 Java 接口方法绑定到 SQL 语句的运行时桥梁。它由三部分组成：

- `MapperRegistry`：注册 mapper 接口和代理工厂。
- `MapperProxy/MapperProxyFactory`：用 JDK 动态代理拦截接口方法。
- `MapperMethod/MappedStatement`：把方法调用转换成 statement id、参数对象、返回值处理和 `SqlSession` 调用。

`MappedStatement` 则是一条 SQL 的标准化元数据对象，保存 SQL source、命令类型、参数映射、结果映射、缓存、超时、fetchSize、key generator 等执行所需信息。

## 为什么重要

MyBatis 的类型安全入口来自 Mapper 接口，但真正的执行单元不是接口方法，而是 `MappedStatement`。理解两者之间的转换，可以解释：

- 为什么 mapper XML 的 `namespace` 要等于接口全限定名。
- 为什么方法名必须匹配 statement id。
- 为什么启动时报 “Invalid bound statement”。
- 为什么 `List<T>`、`Map<K,V>`、`Cursor<T>`、`Optional<T>` 返回值行为不同。
- 为什么 default method 可以在 Mapper 接口中直接执行 Java 逻辑。

## 使用场景

- 排查 `BindingException: Invalid bound statement`。
- 分析 XML mapper 与注解 mapper 的共存规则。
- 设计通用 Mapper、代码生成器、Mapper 扫描逻辑。
- 阅读 MyBatis-Spring 的 `MapperFactoryBean`。
- 解释 Mapper 接口为什么不需要实现类。

## 核心机制

### 1. Mapper 注册

`MapperRegistry#addMapper()` 只处理接口：

1. 检查是否已注册。
2. 先放入 `knownMappers`，值是 `MapperProxyFactory`。
3. 再用 `MapperAnnotationBuilder` 解析注解和同名 XML。
4. 如果解析失败，回滚注册。

先注册再解析是为了避免 mapper 解析期间因相互引用触发重复绑定。

### 2. Mapper 实例创建

`DefaultSqlSession#getMapper(type)` 委托 `Configuration#getMapper(type, this)`。最终 `MapperProxyFactory#newInstance(sqlSession)` 创建：

```text
MapperProxy(sqlSession, mapperInterface, methodCache)
  -> Proxy.newProxyInstance(...)
```

因此 Mapper 实例本质是 JDK 动态代理，并且持有创建它的 `SqlSession`。

### 3. 方法调用分派

`MapperProxy#invoke()` 的分派规则：

- `Object` 方法：直接反射调用代理对象自身逻辑。
- default 方法：用 `MethodHandle` 调用接口默认实现。
- 普通 mapper 方法：从 `methodCache` 获取或构建 `MapperMethod`，再执行 SQL。

`methodCache` 避免每次调用都重复解析方法签名和 statement。

### 4. MapperMethod 的两层解析

`MapperMethod` 内部有两个核心对象：

| 对象 | 职责 |
| --- | --- |
| `SqlCommand` | 解析 statement id 和 SQL 命令类型 |
| `MethodSignature` | 解析返回值、参数、`RowBounds`、`ResultHandler`、`@MapKey` |

statement id 的规则是：

```text
mapperInterface.getName() + "." + methodName
```

如果当前接口找不到，还会沿声明方法所在的父接口递归查找。

### 5. MappedStatement 是 SQL 元数据

`MappedStatement.Builder` 至少需要：

- `Configuration`
- statement id
- `SqlSource`
- `SqlCommandType`

并逐步填充：

- `resultMaps`
- `parameterMap`
- `cache`
- `flushCacheRequired`
- `useCache`
- `keyGenerator`
- `statementType`
- `resultSetType`
- `timeout/fetchSize`
- `databaseId`
- `LanguageDriver`

调用期 `MappedStatement#getBoundSql(parameter)` 才会结合参数生成最终 `BoundSql`。

## 例子

Mapper XML：

```xml
<mapper namespace="com.example.UserMapper">
  <select id="selectById" resultType="User">
    select * from user where id = #{id}
  </select>
</mapper>
```

Mapper 接口：

```java
public interface UserMapper {
  User selectById(Long id);
}
```

调用链：

```text
mapper.selectById(1)
-> MapperProxy.invoke()
-> MapperMethod.execute()
-> SqlCommand.name = "com.example.UserMapper.selectById"
-> SqlSession.selectOne(statementId, param)
-> Configuration.getMappedStatement(statementId)
```

如果 namespace 写成 `com.example.UserDao`，接口方法就找不到 `com.example.UserMapper.selectById`，运行时会报 bound statement 不存在。

## 常见误区

- **误区：Mapper 代理直接执行 SQL。** 它只是把方法调用翻译为 `SqlSession` 调用；执行仍由 `Executor` 完成。
- **误区：Mapper 实例可以长期缓存。** Mapper 持有 `SqlSession`，生命周期不应超过 session。
- **误区：XML id 可以随意命名。** 使用接口绑定时，namespace 和方法名共同组成 statement id。
- **误区：`${}` 和 `#{}` 都只是占位符。** `#{}` 变成 JDBC 参数绑定，`${}` 是文本替换，安全边界完全不同。

## 和其它概念的关系

- [[concepts/java/mybatis/概念_MyBatis核心生命周期与Configuration]] — `MapperRegistry` 和 `MappedStatement` 都存放在 `Configuration` 中。
- [[concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — XML mapper 在启动期构建出 Mapper 代理运行期要查找的 `MappedStatement`。
- [[concepts/java/mybatis/概念_MyBatis执行器缓存与插件链]] — Mapper 方法转换为 `SqlSession` 调用后，进入执行器和缓存链。
- [[overview/java/mybatis/主题_MyBatis源码架构_综述]] — 将 Mapper 代理放回 MyBatis 整体架构中理解。

## 我的理解

Mapper 代理是 MyBatis “半自动 ORM”体验的关键：它给业务代码一个像 Repository 一样的类型安全接口，但没有隐藏 SQL。接口方法只是到 `MappedStatement` 的强类型索引；真正复杂度被压缩到配置解析、参数命名、返回值适配和执行器链中。

## 来源

- [[sources/java/mybatis/来源_MyBatis_3_源码]] — 源码入口与整体说明。
- `raw/code/mybatis-3/src/site/markdown/getting-started.md`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/binding/MapperMethod.java`

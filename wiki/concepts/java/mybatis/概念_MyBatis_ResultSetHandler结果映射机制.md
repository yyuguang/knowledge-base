---
type: "concept"
tags: ["java", "mybatis", "resultset", "resultmap", "lazy-loading"]
summary: "DefaultResultSetHandler 是 MyBatis 把 JDBC ResultSet 映射为对象图的核心：它根据 ResultMap 选择简单映射或嵌套映射，结合自动映射、显式属性映射、构造器创建、嵌套查询、延迟加载和 CacheKey 行去重完成结果组装。"
sources:
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/resultset/DefaultResultSetHandler.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/resultset/ResultSetWrapper.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/loader/ResultLoaderMap.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/loader/ResultLoader.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/loader/javassist/JavassistProxyFactory.java"
  - "raw/code/mybatis-3/src/main/java/org/apache/ibatis/reflection/factory/DefaultObjectFactory.java"
aliases: ["DefaultResultSetHandler", "MyBatis 结果映射", "ResultMap 嵌套映射", "MyBatis 延迟加载"]
status: "evolving"
confidence: 0.9
created: "2026-05-13 09:19:36"
updated: "2026-05-13 09:19:36"
---

# 概念：MyBatis ResultSetHandler结果映射机制

![[Attachments/mybatis-resultsethandler-mapping-flow.excalidraw]]

## 定义

`DefaultResultSetHandler` 是 MyBatis 查询结果映射的核心实现。它接收 JDBC `Statement/ResultSet`，读取 `MappedStatement.resultMaps`，再把每一行或多行结果组装成 Java 对象、集合、对象图或游标。

它解决的不是单纯的 “列名到属性名” 问题，而是四类问题的组合：

1. **对象如何创建**：简单类型、默认构造器、显式 constructor 映射、构造器自动映射。
2. **列如何写入属性**：显式 `ResultMapping` 与自动映射。
3. **对象图如何合并**：nested resultMap、association、collection、discriminator、行级去重。
4. **关联对象何时加载**：nested query 即时加载、延迟加载、一级缓存延迟填充。

## 为什么重要

MyBatis 最容易被低估的复杂度就在结果映射。理解 `DefaultResultSetHandler` 可以解释：

- 为什么 `resultType` 简单场景能自动映射。
- 为什么复杂 join 必须写 `<id>`，否则 collection 去重容易失效。
- 为什么嵌套 resultMap 不建议随意配 RowBounds/custom ResultHandler。
- 为什么 association/collection 有 nested select 和 nested result 两种完全不同路径。
- 为什么 lazy loading 需要代理对象。
- 为什么不可变对象、构造器注入、集合构造器映射会出现 pending creation。

## 使用场景

- 排查查询结果重复、collection 丢数据、对象字段为 null。
- 判断 `resultType`、`resultMap`、`association`、`collection` 应如何选择。
- 解释 `autoMappingBehavior`、`mapUnderscoreToCamelCase`、`callSettersOnNulls` 的影响。
- 优化复杂 join 映射和 N+1 nested query。
- 设计不可变对象、record-like DTO、构造器映射。

## 核心机制

### 1. 总入口：handleResultSets

`handleResultSets(Statement)` 的主流程：

```text
getFirstResultSet(stmt)
-> validate resultMaps
-> for each resultMap/resultSet
   -> handleResultSet()
      -> handleRowValues()
-> collapseSingleResultList()
```

`handleResultSet()` 会按是否存在外部 `ResultHandler` 决定使用用户 handler，还是创建默认 `DefaultResultHandler` 收集列表。处理完成后会关闭当前 `ResultSet`。

### 2. ResultSetWrapper：ResultSet 元数据缓存

`ResultSetWrapper` 在构造时读取：

- column label/name
- JDBC type
- JDBC driver 返回的 Java class name

它还缓存每个 `ResultMap + columnPrefix` 的：

- mapped columns
- unmapped columns
- column/property 对应的 `TypeHandler`

这让自动映射和显式映射不用反复读 JDBC metadata。

### 3. simple vs nested：两条行处理路径

`handleRowValues()` 先看 `resultMap.hasNestedResultMaps()`：

```text
无 nested resultMap -> handleRowValuesForSimpleResultMap()
有 nested resultMap -> handleRowValuesForNestedResultMap()
```

简单路径是一行生成一个结果对象。

嵌套路径通常是多行合并成一个对象图。例如一篇博客 join 多篇文章时，父对象重复出现，子集合逐行累加。它依赖 `CacheKey` 判断当前行是否属于已有对象。

### 4. 自动映射

自动映射发生在 `applyAutomaticMappings()`：

1. `ResultSetWrapper#getUnmappedColumnNames()` 找到未显式映射的列。
2. 去掉构造器自动映射已经消费过的列。
3. 如果有 `columnPrefix`，只处理带此前缀的列。
4. 用 `metaObject.findProperty(columnName, mapUnderscoreToCamelCase)` 找属性。
5. 属性存在且有 setter，且不在 `resultMap.mappedProperties` 中，才尝试找 `TypeHandler`。
6. 读取值并按 `callSettersOnNulls` 决定是否对 null 调 setter。

是否启用自动映射由 `shouldApplyAutomaticMappings()` 决定：

- `ResultMap.autoMapping` 显式配置优先。
- 非嵌套映射：`autoMappingBehavior != NONE`。
- 嵌套映射：只有 `autoMappingBehavior == FULL` 才默认启用。

### 5. 显式属性映射

显式映射发生在 `applyPropertyMappings()`，遍历 `resultMap.getPropertyResultMappings()`。

每个 `ResultMapping` 有几种分支：

| 分支 | 机制 |
| --- | --- |
| 普通列 | 用 `TypeHandler#getResult()` 读取列值，再 `metaObject.setValue()` |
| nested query | 走 `getNestedQueryMappingValue()`，可能即时加载或延迟加载 |
| nested cursor | 从当前列取 `ResultSet`，递归 `handleResultSet()` |
| multiple resultSets | 记录 pending relation，等后续 resultSet 回来再链接 |
| nested resultMap | 不在这里处理，而由 `applyNestedResultMappings()` 处理 |

这说明 `resultMap` 的 `<association select="...">` 和 `<association resultMap="...">` 是两套运行路径。

### 6. 对象创建策略

`createResultObject()` 的选择顺序：

1. 如果结果类型有直接 `TypeHandler`：用第一列或显式列创建简单值。
2. 如果 resultMap 有 constructor mappings：按 `<constructor>` 显式构造。
3. 如果类型是接口或有默认构造器：`objectFactory.create(resultType)`。
4. 如果允许自动映射：尝试构造器自动映射。
5. 否则抛出 “不知道如何创建实例”。

`DefaultObjectFactory` 会把接口类型转换成默认实现：

- `List/Collection/Iterable -> ArrayList`
- `Map -> HashMap`
- `Set -> HashSet`
- `SortedSet -> TreeSet`

构造器自动映射有两种：

- 按列顺序匹配构造器参数类型。
- 开启 `argNameBasedConstructorAutoMapping` 后按参数名匹配列名；多构造器时通常需要 `@AutomapConstructor`。

### 7. nested query 与延迟加载

nested query 由 `getNestedQueryMappingValue()` 处理：

1. 找到 nested query 的 `MappedStatement`。
2. 从当前行列值准备 nested query 参数。
3. 创建 nested query 的 `BoundSql` 和 `CacheKey`。
4. 如果 executor 已缓存该 key，调用 `executor.deferLoad()`，返回 `DEFERRED`。
5. 否则创建 `ResultLoader`。
6. 如果 `propertyMapping.isLazy()`，加入 `ResultLoaderMap`，返回 `DEFERRED`。
7. 否则立即 `resultLoader.loadResult()`。

如果当前结果对象存在懒加载属性，`createResultObject()` 会用 `configuration.getProxyFactory().createProxy()` 创建代理对象。默认 Javassist 代理会在：

- aggressive lazy loading 开启时触发全部加载；
- 调用 `equals/clone/hashCode/toString` 等触发方法时加载全部；
- 调用某个 getter 时只加载对应属性；
- 调用 setter 时移除该属性的 loader。

### 8. nested resultMap 与行级去重

join 嵌套映射由 `handleRowValuesForNestedResultMap()` 和 `applyNestedResultMappings()` 处理。

核心是 `CacheKey`：

```text
rowKey = resultMap.id + id列/映射列值
combinedKey = childRowKey + parentRowKey
```

`nestedResultObjects` 用这个 key 保存已经创建过的对象。这样多行 join 结果可以合并到同一个父对象或子对象，而不是每行创建新父对象。

如果 resultMap 配置了 `<id>`，`createRowKey()` 优先使用 id mappings；没有 `<id>` 时会退回 property mappings 或 unmapped properties，稳定性较差。

### 9. association/collection 的链接

`linkObjects()` 负责把子对象挂到父对象：

- 如果目标属性是 collection，先通过 `instantiateCollectionPropertyIfAppropriate()` 创建或取得集合，再 `add()` 子对象。
- 如果不是 collection，就直接 `metaObject.setValue(property, rowValue)`。

`instantiateCollectionPropertyIfAppropriate()` 会根据 resultMapping 的 `javaType` 或 setter 类型判断是否是 collection，并用 `ObjectFactory` 创建集合实例。

### 10. 构造器集合映射与 PendingConstructorCreation

对于不可变对象，如果 collection 作为构造器参数，就不能在第一行立刻创建最终对象，因为后续行还会继续补集合元素。

MyBatis 用 `PendingConstructorCreation` 暂存：

- result type
- constructor arg types
- constructor args
- 已链接的 collection 值
- 内层 pending creation

等同一父对象的所有行处理完，再递归调用 `pendingCreation.create(objectFactory)` 生成最终不可变对象。

这也是为什么构造器集合映射要求结果有序，源码会检查 `resultOrdered=true`。

## 例子

一对多 join 映射：

```xml
<resultMap id="blogMap" type="Blog">
  <id property="id" column="blog_id"/>
  <result property="title" column="blog_title"/>
  <collection property="posts" ofType="Post">
    <id property="id" column="post_id"/>
    <result property="subject" column="post_subject"/>
  </collection>
</resultMap>
```

执行时：

```text
第 1 行：创建 Blog#1，创建 Post#10，posts.add(Post#10)
第 2 行：rowKey 命中 Blog#1，创建 Post#11，posts.add(Post#11)
第 3 行：创建 Blog#2，创建 Post#20，posts.add(Post#20)
```

如果没有 `<id>`，MyBatis 只能用更多列推导 rowKey，重复合并更容易不稳定。

## 常见误区

- **误区：自动映射会覆盖显式映射。** 自动映射只处理 unmapped columns，并跳过已在 resultMap 中映射的属性。
- **误区：nested select 和 nested result 只是写法不同。** nested select 会额外发 SQL，可 lazy；nested result 从同一个 join resultSet 合并对象图。
- **误区：collection 重复是 MyBatis bug。** 很多时候是 resultMap 没有稳定 `<id>`，导致 rowKey 无法正确去重。
- **误区：延迟加载只是晚点 set 属性。** 它需要代理对象、`ResultLoaderMap`、nested query 的参数和 `CacheKey`。
- **误区：不可变对象只能简单映射。** MyBatis 支持 constructor 映射，但 collection 构造器注入需要 pending creation 和有序结果。

## 和其它概念的关系

- [[concepts/java/mybatis/概念_MyBatis执行器缓存与插件链]] — `StatementHandler.query()` 后进入 `ResultSetHandler.handleResultSets()`。
- [[concepts/java/mybatis/概念_MyBatis_XML语句解析与MappedStatement构建]] — XML 中的 `resultMap/resultType/association/collection` 会被编译成这里使用的 `ResultMap/ResultMapping`。
- [[concepts/java/mybatis/概念_MyBatis_Mapper代理与MappedStatement]] — Mapper 方法定位 `MappedStatement` 后，`MappedStatement.resultMaps` 决定本页的映射行为。
- [[overview/java/mybatis/主题_MyBatis源码架构_综述]] — 本页补齐第三轮 `DefaultResultSetHandler` 深挖主题。

## 我的理解

`DefaultResultSetHandler` 是 MyBatis 里最接近 ORM 的部分，但它仍然保持 SQL Mapper 的边界：不推导查询，不管理实体状态，只把一份明确的 `ResultMap` 应用到 JDBC 行集上。它的复杂度主要来自 “一行不一定等于一个对象”：join 会让父对象重复，collection 要跨行累加，nested query 可能延迟执行，构造器对象可能要等集合收集完整才能创建。

## 来源

- [[sources/java/mybatis/来源_MyBatis_3_源码]] — MyBatis 3 源码总入口。
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/resultset/DefaultResultSetHandler.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/resultset/ResultSetWrapper.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/loader/ResultLoaderMap.java`
- `raw/code/mybatis-3/src/main/java/org/apache/ibatis/executor/loader/ResultLoader.java`

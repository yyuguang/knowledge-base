---
type: concept
tags:
  - java
  - spring-ai
  - structured-output
  - json-schema
  - bean-output-converter
summary: "Spring AI StructuredOutput 结构化输出：BeanOutputConverter 自动从 Java 类生成 JSON Schema → 注入 System Prompt → LLM 返回 JSON → Jackson 反序列化为 DTO，全过程类型安全。支持 Record/Class/ParameterizedTypeReference"
sources:
  - "[[30-sources/repositories/Spring_AI_源码]]"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/converter/StructuredOutputConverter.java"
  - "raw/sai/spring-ai-model/src/main/java/org/springframework/ai/converter/BeanOutputConverter.java"
aliases:
  - Structured Output
  - 结构化输出
  - BeanOutputConverter
  - JSON Schema 注入
status: evolving
confidence: 0.95
updated: "2026-07-14 15:05:00"
---

# 概念_Spring_AI_StructuredOutput结构化输出

## 定义

StructuredOutput 是 Spring AI 中将 LLM 自由文本输出**自动转换为类型安全的 Java 对象**的机制。核心实现 `BeanOutputConverter<T>` 通过以下流程工作：Java 类（Record/Class）→ 自动生成 JSON Schema（victools/jsonschema-generator）→ 注入 System Prompt 约束 LLM 输出格式 → LLM 返回 JSON → 文本清洗（去 markdown/thinking tag）→ Jackson 反序列化为目标 DTO。

## 为什么重要

LLM 的本质是文本生成，但业务系统需要结构化数据：

1. **类型安全**：编译期检查，杜绝 LLM 输出的字段名/类型错误
2. **自动 Schema 生成**：无需手写 JSON Schema，Java 类即 Schema
3. **无缝集成**：DTO 可直接传入 Service/Repository，无需中间解析层
4. **泛型支持**：`List<Person>`、`Map<String, Order>` 等复杂类型直接转换

没有 StructuredOutput，你需要手写正则/JSON 解析逻辑，且每次修改 DTO 都要同步修改 Prompt。

## 使用场景

- **信息提取**：从对话中提取姓名、日期、金额等结构化字段
- **实体识别**：`Person`、`Company`、`Address` 等实体抽取
- **格式规范化**：将 LLM 的自然语言输出映射为 API 响应格式
- **批量处理**：一次 Prompt 返回 `List<Entity>` 批量抽取
- **多字段分类**：情感分析 + 主题分类 + 摘要 = 一个 DTO

## 核心机制

### 1. StructuredOutputConverter 接口

```java
// spring-ai-model: StructuredOutputConverter.java
public interface StructuredOutputConverter<T> extends Converter<String, T>, FormatProvider {

    String NO_JSON_SCHEMA = "";

    // 返回 JSON Schema，用于 LLM 的结构化输出约束（v2.0 新增）
    default String getJsonSchema() { return NO_JSON_SCHEMA; }

    // FormatProvider.getFormat(): 返回注入 Prompt 的格式指令
    // Converter.convert(String): LLM 文本 → T
}
```

### 2. BeanOutputConverter 工作流程

```
① 构造
BeanOutputConverter<Person> converter = new BeanOutputConverter<>(Person.class);
    │
② 生成 JSON Schema（自动，构造函数中完成）
    │  使用 victools/jsonschema-generator (DRAFT_2020_12)
    │  ├── 扫描 Java 类的所有字段
    │  ├── 映射 Java 类型 → JSON Schema 类型
    │  ├── 注解支持：@JsonProperty, @JsonClassDescription, @JsonInclude
    │  ├── FORBIDDEN_ADDITIONAL_PROPERTIES_BY_DEFAULT (防止 LLM 编造字段)
    │  └── 生成 Pretty-printed JSON Schema String
    │
③ getFormat() 生成 Prompt 注入文本
    │  "Your response should be in JSON format.
    │   Do not include any explanations, only provide a RFC8259 compliant JSON response.
    │   Do not include markdown code blocks in your response.
    │   Remove the ```json markdown from the output.
    │   Here is the JSON Schema instance your output must adhere to:
    │   ```{jsonSchema}```"
    │
④ 注入 System Prompt  → ChatClient.entity() 自动调用
    │
⑤ LLM 返回 JSON 文本
    │  可能包含 markdown 包裹、thinking tag、多余空白等
    │
⑥ 文本清洗（ResponseTextCleaner 链）
    │  默认清洁器链：
    │  ├── WhitespaceCleaner: 去除首尾空白
    │  ├── ThinkingTagCleaner: 移除 Amazon Nova/Qwen 的 <thinking> 标签
    │  ├── MarkdownCodeBlockCleaner: 移除 ```json ... ``` 包裹
    │  └── WhitespaceCleaner: 再次去除空白
    │
⑦ Jackson 反序列化
    jsonMapper.readValue(cleanedText, type)
    │  配置：FAIL_ON_UNKNOWN_PROPERTIES = false（宽容模式）
    │
⑧ 返回类型安全的 T 对象
    Person[name="张三", age=30, role="工程师"]
```

### 3. ChatClient.entity() 自动集成

```java
// ChatClient 内部：entity() 自动注入 Schema 到 System Prompt
<T> T entity(Class<T> type) {
    // ① 创建 BeanOutputConverter
    // ② 将 getFormat() 注入 SystemMessage
    // ③ call() 获取 LLM 响应
    // ④ converter.convert(response) 返回 T
}
```

调用 `entity()` 时，框架自动做了三件事：
1. 创建 `BeanOutputConverter<T>`
2. 将 JSON Schema 指令追加到 System Prompt
3. 调用 LLM → 清洗 → 反序列化 → 返回 T

### 4. Schema 生成细节

```java
// BeanOutputConverter.generateSchema()
JacksonSchemaModule jacksonModule = new JacksonSchemaModule(
    RESPECT_JSONPROPERTY_REQUIRED,
    RESPECT_JSONPROPERTY_ORDER
);
SchemaGeneratorConfig config = new SchemaGeneratorConfigBuilder(
    DRAFT_2020_12,
    OptionPreset.PLAIN_JSON
)
.with(jacksonModule)
.with(Option.FORBIDDEN_ADDITIONAL_PROPERTIES_BY_DEFAULT)  // ⭐ 关键：防止 LLM 编造字段
.build();

config.forFields().withRequiredCheck(f -> true);  // 所有字段标记为 required

SchemaGenerator generator = new SchemaGenerator(config);
JsonNode jsonNode = generator.generateSchema(this.type);

// Kotlin 支持
if (KotlinDetector.isKotlinReflectPresent())
    config.with(new KotlinModule());
```

## 例子

### Record 结构化提取

```java
record Person(String name, int age, String role) {}

Person person = chatClient.prompt()
    .user("张三，30岁，Java开发工程师。提取信息。")
    .call()
    .entity(Person.class);
// → Person[name="张三", age=30, role="Java开发工程师"]
```

### 泛型列表提取

```java
List<Person> people = chatClient.prompt()
    .user("张三30岁工程师，李四25岁设计师，王五35岁经理。全部提取。")
    .call()
    .entity(new ParameterizedTypeReference<List<Person>>() {});
// → [Person["张三", 30, "工程师"], Person["李四", 25, "设计师"], Person["王五", 35, "经理"]]
```

### 复杂嵌套 DTO

```java
record Address(String city, String street) {}
record Employee(String name, int age, Address address) {}

// JSON Schema 自动包含嵌套关系
Employee emp = chatClient.prompt()
    .user("王小明，28岁，住在北京市朝阳区望京街道。")
    .call()
    .entity(Employee.class);
// → Employee[name="王小明", age=28, address=Address[city="北京", street="朝阳区望京街道"]]
```

### 自定义 JSON Mapper

```java
JsonMapper customMapper = JsonMapper.builder()
    .enable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES) // 严格模式
    .build();

BeanOutputConverter<Person> converter = new BeanOutputConverter<>(Person.class, customMapper);
Person person = chatClient.prompt().user("...").call().entity(converter);
```

## 常见误区

1. **"entity() 和 content() 可以一起用"**：错。`entity()` 注入 JSON Schema 到 System Prompt，约束 LLM 必须返回 JSON。如果再调用 `content()` 获取纯文本，结果不可预期。
2. **"LLM 一定会返回合法 JSON"**：不一定。LLM 可能返回带 markdown 包裹的 JSON（```json ... ```）或额外解释文本。BeanOutputConverter 的 `ResponseTextCleaner` 链会处理这些情况，但极端情况下仍可能反序列化失败。
3. **"JSON Schema 约束了 LLM 就一定不编造字段"**：`FORBIDDEN_ADDITIONAL_PROPERTIES_BY_DEFAULT` 会告知 LLM 不要添加额外字段，但无法 100% 保证。验证仍需要 `StructuredOutputValidationAdvisor`。
4. **"BeanOutputConverter 只能用 Record"**：错。任何 Java 类都可以，支持 `Class<T>`、`ParameterizedTypeReference<T>`，以及 Kotlin data class（自动检测 KotlinReflect）。

## 和其它概念的关系

- [[10-domains/java/spring-ai/概念_Spring_AI_ChatClient_API]] — `entity()` 是 CallResponseSpec 的核心方法，自动触发 BeanOutputConverter
- [[10-domains/java/spring-ai/概念_Spring_AI_Advisor拦截链]] — StructuredOutputValidationAdvisor 可校验 entity() 的输出
- [[10-domains/java/spring-ai/概念_Spring_AI_架构设计]] — StructuredOutputConverter 是架构中 Model 层的转换器抽象

## 来源

- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/converter/StructuredOutputConverter.java`
- `raw/sai/spring-ai-model/src/main/java/org/springframework/ai/converter/BeanOutputConverter.java`

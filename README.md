# 目录结构
```
.
├── prompt/                   # 自定义提示词模版
├── raw/                    # 原始资料（只读）
│   ├── papers/            # 论文、文章
│   ├── books/             # 书籍摘录
│   └── media/             # 图片、视频等
│
├── wiki/                   # 知识层（LLM 维护）
│   ├── sources/           # 来源摘要页
│   │   └── java/         # 按领域划分子目录
│   ├── entities/          # 实体页（人物、项目等）
│   │   └── java/
│   ├── concepts/          # 概念页（方法、理论等）
│   │   └── java/
│   ├── comparisons/       # 比较分析页
│   │   └── java/         # 等新增领域（redis/、mq/…）
│   ├── overview/          # 总览综合页
│   ├── index.md           # 内容索引
│   └── log.md             # 操作日志
│
├── Attachments/           # 图片资源
├── TheSchema.md           # 系统配置文档
└── README.md              # 本文档
```

# 框架总览
![[Pasted image 20260427213646.png]]
系统通过三层架构实现知识的持续积累和复利增长。

#  Architecture · 三种文件类型
![[Pasted image 20260427213710.png]]
- **Raw Sources**：原始资料，不可变的事实来源
- **Wiki Pages**：LLM 生成和维护的结构化知识页面
- **Schema**：系统配置，定义工作流和规则

#  Operations · 三个日常操作
![[Pasted image 20260427213729.png]]
### 1. Ingest（导入）

添加新资料到 `raw/`，LLM 阅读并：

- 创建来源摘要页
- 更新相关实体/概念页
- 维护交叉引用
- 记录到操作日志

### 2. Query（查询）

向 Wiki 提问，LLM：

- 搜索相关页面
- 综合回答并附上引用
- 可选：将有价值的回答写回为新页面

### 3. Lint（检查）

定期审计 Wiki 健康度：

- 发现矛盾和过时内容
- 识别孤立页面
- 建议合并/拆分
- 补充缺失的交叉引用
[English](README.md) | [中文](README.zh.md)

# Skill-TAPD

> 本 README 面向 skill 维护者与人类读者，不是 skill 的运行时入口。
>
> - Skill 运行入口：[`skill.md`](./skill.md)
> - 模块目录：[`guides/`](./guides/) 和 [`reference/`](./reference/)
> - 运行时规则（工作流、MCP 工具、状态流转）全部在 `skill.md` 内，不在本文件复述。

这个 skill 用于 AI 驱动的 TAPD 项目管理：需求分析、迭代规划、Bug 分诊、状态流转、批量操作，集成 MCP 实现 TAPD API 访问。

## 适合什么时候用

当你在 TAPD 中需要做这些事情时使用：

- **需求分析** — 从 TAPD URL 提取内容、解析原型图片、应用六维度分析框架
- **迭代规划** — 拉取迭代数据、分析容量、批量分配需求
- **Bug 分诊** — 解析错误截图、分类严重性/优先级、批量更新
- **状态流转** — 只能向前流转的状态机，带开发人员验证
- **批量操作** — 批量更新需求/缺陷/任务，带二次确认
- **日报生成** — 从 TAPD 数据生成进度报告
- **评论管理** — 列出和创建需求/缺陷/任务的评论
- **成员查找** — 查找用户及其分配的工作项

## 当前结构

```text
skill-tapd/
├── skill.md                        # 运行入口：触发条件、硬规则、工作流索引
├── README.md                       # 维护者入口（英文版）
├── README.zh.md                    # 维护者入口（中文版，本文件）
├── guides/
│   ├── requirement-analysis.md    # 六维度分析方法论
│   ├── vision-parser-quick.md     # 快速图片解析指南
│   ├── vision-parser.md           # 完整 14 节图片解析标准
│   ├── workflows.md               # 迭代规划、Bug 分诊、日报生成等
│   └── templates/                 # 即用型模板（7 个文件）
└── reference/
    ├── mcp-tools.md               # 完整 TAPD MCP 工具文档
    └── status-mapping.md          # 工作区特定的状态流转规则
```

**模块加载**：`skill.md` 通过路径引用 guides/ 和 reference/，Claude 在处理特定工作流时按需加载。

## 核心特性

- **MCP 集成** — 使用 `@npm_xiyuan/mcp-tapd-radar` 进行 TAPD API 认证访问
- **图片解析器** — 从原型和截图中提取 UI 元素、文本、布局
- **状态流转校验** — 只能向前流转，带开发人员存在性检查（`has_developer()`）
- **批量操作** — 高效批量更新，带确认提示
- **Token 效率** — 模块化架构平均节省约 31% tokens

## 使用要求

- **MCP 服务器**：在 Claude Desktop 中配置 `@npm_xiyuan/mcp-tapd-radar`
- **TAPD 账号**：工作区 ID 和 API 凭证
- **工作区 ID**：默认 `48801209`（可在 skill.md 中配置）

## 维护规范

维护这个 skill 时：

- 保持 `skill.md` 作为入口（触发条件、硬规则、工作流索引）
- 把详细工作流和模板放在 `guides/`
- 把 API 参考和配置放在 `reference/`
- 如果 `skill.md` 超过 1000 行，考虑把章节移到模块中
- 添加新能力时更新工作流索引表（第 161-171 行）
- 保持 `guides/vision-parser-quick.md` 和 `guides/vision-parser.md` 的图片解析标准同步
- 不要删除 `guides/templates/` — 它们被 `requirement-analysis.md` 引用

## 模块说明

### skill.md（789 行）
- frontmatter：name、description、whenToUse
- 核心原则：信息完整、图片解析、二次确认
- 状态流转规则，带 `has_developer()` 验证
- 工作流索引表，含模块引用
- MCP 工具使用指南
- 异常处理和模糊匹配策略

### guides/requirement-analysis.md
- 六维度分析框架（目标、用户、场景、边界、价值、验收）
- Story 拆解和任务分解
- 验收标准生成（Given/When/Then）

### guides/vision-parser-quick.md
- 快速 3 步图片解析指南（结构、内容、技术提示）
- 优化用于快速原型分析

### guides/vision-parser.md
- 完整 14 节图片解析标准
- 用于详细 UI/UX 规格提取

### guides/workflows.md
- 13 个工作流章节（迭代规划、Bug 分诊、日报生成等）
- 分步执行指南，含 MCP 工具调用
- 确认提示和错误恢复

### reference/mcp-tools.md
- 完整 TAPD MCP API 工具文档
- 参数规格、返回格式、使用示例

### reference/status-mapping.md
- 工作区特定的状态序列（如 48801209: new→planned→developing→testing→done）
- 流转校验规则

## 常见陷阱

- **不要绕过 `has_developer()` 检查** — 状态流转需要验证开发人员分配
- **不要跳过确认提示** — 所有写入操作（创建/更新/删除）需用户批准
- **不要用 `tapd_get_*` 做批量操作** — 应使用 `tapd_count_*` + `tapd_list_*` + 批量工具
- **不要忽略图片解析触发器** — description 含 `<img>` 标签或图片 URL 必须调用解析器

## 当前 skill 文件路径

```text
~/.claude/skills/skill-tapd/skill.md
~/.claude/skills/skill-tapd/guides/
~/.claude/skills/skill-tapd/reference/
```

## 相关项目

- [MCP TAPD Radar](https://github.com/jiyi1990118/mcp-tapd-radar) — 底层 MCP 服务器
- [Skill Project Knowledge System](../skill-project-knowledge-system/) — 项目知识库构建器

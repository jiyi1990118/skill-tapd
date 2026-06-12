---
name: skill-tapd
description: TAPD 项目管理工作流 - 需求分析、迭代规划、缺陷管理、进度跟踪
whenToUse: |
  TAPD 项目管理任务：需求分析、迭代规划、bug 分诊、日报生成、需求跟踪、质量分析
  触发词：需求分析、迭代规划、sprint、bug 分诊、日报、需求跟踪、质量分析
---

# TAPD 项目管理 Skill

**目标**：标准化使用 MCP 与 TAPD 进行需求管理、任务拆解、缺陷跟踪的方式。

---

## 核心原则

### 1. 信息完整性优先
- ✅ 信息不足时，**必须澄清**，禁止臆造
- ✅ 需求必须包含：业务目标、用户角色、使用场景、验收标准
- ❌ 禁止创建不完整需求

### 2. 图片内容解析 ⭐ 关键能力
当获取 Story/Bug/Task 详情包含图片时，**必须执行 Vision Parser** 解析图片内容。

**触发条件**：
- `tapd_get_story/bug/task` 返回的 description 包含图片引用
- 用户描述中提到"原型图"、"设计稿"、"截图"、"效果图"
- 检测方法：description 中包含 `<img>`、图片URL、或 base64 图片数据

**立即执行（3步）**：
1. **读取快速规范**：`guides/vision-parser-quick.md`（核心原则+执行步骤）
2. **执行解析**：按照标准输出格式，提取UI元素、文本、布局
3. **补充到需求**：将解析结果添加到需求理解中

**需要完整14部分解析？** 读取 `guides/vision-parser.md`

**核心原则**（执行时牢记）：
- 观察 > 解释（只描述可见内容，不推测意图）
- 保留一切（所有文本、数字、UI元素、表格、图表）
- 事实与推理分离（推理必须标记 `[Inference]`）
- 无损编码（下游模型能通过输出重建原图）

**TAPD场景特化**：
- 原型图 → 重点提取：UI元素清单、交互流程、页面流转
- Bug截图 → 重点提取：错误信息、环境信息、页面状态
- 设计稿 → 重点提取：颜色、字体、尺寸标注、组件层级

### 3. 二次确认机制
- ✅ 所有写入操作（create/update/delete）需用户明确确认
- ✅ 批量操作前展示变更清单
- ✅ 增量更新，不覆盖原始内容

### 4. 优先统计后详情
- ✅ 使用 `tapd_count_*` 获取统计数据
- ✅ 按需使用 `tapd_list_*` 拉取列表
- ✅ 只在必要时使用 `tapd_get_*` 获取详情

---

## 工作流快速索引

### 需求分析流程 ⭐ 最常用
```
1. 用户提出需求
   ↓
2. 【检测图片】→ Yes → 立即执行 Vision Parser（见下方"图片解析执行清单"）
   ↓
3. 【需求理解】明确6维度（Why/Who/When/Where/Value/Boundary/Done）
   ↓
4. 【完整性检查】缺失信息？→ Yes → 输出缺失清单 → 等待用户补充 → 返回步骤3
   ↓ No
5. 【需求拆解】Story → Module → Feature → Task
   ↓
6. 【验收标准】设计 Given/When/Then 格式
   ↓
7. 【用户确认】展示完整需求文档
   ↓
8. 【创建记录】tapd_create_story + tapd_create_task
```

**详细规范**：`guides/requirement-analysis.md`  
**模板参考**：`guides/templates/requirement.md`

---

### 迭代规划流程
```
1. tapd_list_iterations(status="open") → 获取开放迭代
2. tapd_count_stories(iteration_id="<>") → 统计未分配需求
3. tapd_list_stories(iteration_id="<>", limit=50) → 拉取列表
4. 按 priority 排序 + 容量分析（团队人天 vs 需求工作量）
5. 生成规划建议（建议纳入/不纳入 + 风险提示）
6. 用户确认
7. 批量执行：tapd_update_story(iteration_id="迭代X")
```

**详细规范**：`guides/workflows.md` → 迭代规划部分

---

### Bug 分诊流程
```
1. tapd_count_bugs(status="new") → 统计概况
2. tapd_list_bugs(status="new", limit=30) → 拉取列表
3. 逐个分析：
   - 有截图？→ Yes → 执行 Vision Parser（提取错误信息）
   - 判断 severity（基于影响范围）→ fatal/serious/normal
   - 判断 priority（基于业务优先级）→ urgent/high/medium
   - 推荐 owner（基于 module）
4. 展示分诊报告
5. 用户确认
6. 批量执行：tapd_update_bug(severity, priority, current_owner)
```

**详细规范**：`guides/workflows.md` → 缺陷管理部分

---

### 日报生成流程
```
1. 计算时间：yesterday = today - 1天
2. 查询昨日完成：tapd_list_stories/tasks/bugs(modified="昨天", status="done/resolved")
3. 查询今日计划：tapd_list_stories/tasks(status="in_progress")
4. 识别阻塞项：tapd_list_stories(priority="urgent|high", modified="<昨天")
5. 格式化输出：📅 标题 + ✅ 昨日完成 + 🚀 今日计划 + ⚠️ 阻塞风险
```

**详细规范**：`guides/workflows.md` → 日报生成部分

---

## MCP 工具使用规范

### 查询工具
- `tapd_ping` - 检查连通性
- `tapd_list_workspaces` - 列出工作区
- `tapd_count_stories/bugs/tasks` - **优先使用**，快速统计
- `tapd_list_stories/bugs/tasks` - 列表查询，支持过滤和分页
- `tapd_get_story/bug/task` - 获取详情（含图片）
- `tapd_list_iterations` - 列出迭代
- `tapd_list_comments` - 列出评论
- `tapd_list_users` - 列出成员

### 写入工具（需二次确认）
- `tapd_create_story/bug/task` - 创建记录
- `tapd_update_story/bug/task` - 更新记录（增量更新）
- `tapd_delete_story/bug/task` - 删除记录（实为状态变更）
- `tapd_create_comment` - 添加评论

### 图片工具
- `tapd_download_image` - 下载认证图片（已内嵌在详情中）

**完整工具参考**：读取 `reference/mcp-tools.md`

---

## 需求分析六维度（核心）

执行需求分析时，**必须明确**以下六个维度：

### ✅ 业务目标（Why）
- 要解决什么问题？
- 预期带来什么价值？
- 如何衡量成功？

### ✅ 用户角色（Who）
- 谁会使用？
- 不同角色的需求差异？

### ✅ 使用场景（When & Where）
- 用户在什么情况下使用？
- 典型使用路径？

### ✅ 业务价值（Value）
- 为什么现在做？
- 优先级依据？

### ✅ 边界条件（Boundary）
- 范围内/范围外？
- 约束条件？

### ✅ 成功标准（Done）
- 如何判断完成？
- 验收标准？

**如有任何维度信息缺失，必须输出缺失清单并要求用户补充。**

---

## 图片解析执行清单 ⭐

**何时执行**：检测到图片立即执行（不等待用户要求）

**检测方法**：
```python
# description 包含以下任一标记
has_image = (
    "<img" in description or 
    "http" in description and (".jpg" in description or ".png" in description) or
    "base64" in description or
    "原型图" in user_input or 
    "设计稿" in user_input or
    "截图" in user_input
)
```

**执行流程（强制）**：
```
1. 读取 guides/vision-parser.md → 获取完整14部分解析标准
2. 按标准执行 → 输出结构化JSON（14部分）
3. 补充到需求 → 在需求理解中引用解析结果
```

**输出示例**：
```markdown
## 需求理解

### 原型图解析结果
- **UI元素**：登录按钮、用户名输入框、密码输入框、"记住我"复选框
- **文本内容**：标题"用户登录"、按钮"立即登录"、链接"忘记密码？"
- **布局关系**：标题居中、输入框垂直排列、按钮在底部
- **交互流程**：输入账号密码 → 点击登录 → 跳转首页

基于原型图，业务目标是：[推断业务目标]
```

**完整解析规范**：`guides/vision-parser.md`（14部分标准）

---

## 从 TAPD URL 提取内容 ⭐ 通用场景

### 支持的URL类型
- Story（需求）：`https://www.tapd.cn/.../story/detail/...`
- Bug（缺陷）：`https://www.tapd.cn/.../bug/bugs/view/...`
- Task（任务）：`https://www.tapd.cn/.../task/...`

### 触发指令模式
```
模式1：单个提取
"帮我提取【标题】{URL} 的内容"
"提取这个需求：{URL}"

模式2：批量提取
"提取以下需求的内容：{URL1} {URL2} {URL3}"

模式3：对比提取
"对比这两个需求的差异：{URL1} vs {URL2}"
```

### 通用执行规则 🔒

#### 1. URL解析
```python
# 从URL提取关键信息
import re

url_pattern = r"tapd\.cn/(\d+)/(story|bug|task)/.*?(\d{19})"
workspace_id, entity_type, entity_id = extract_from_url(url)

# 映射到工具
tool_map = {
    "story": tapd_get_story,
    "bug": tapd_get_bug,
    "task": tapd_get_task
}
```

#### 2. 内容提取原则
- ✅ **完整性**：一字不改地提取所有内容
- ✅ **原始性**：保留原文格式、结构、顺序
- ❌ **禁止加工**：不修改、不总结、不优化

#### 3. 图片处理（统一规则）

**检测位置**：
```python
# 找出所有图片及其位置
images = []
for match in re.finditer(r'(<img.*?>|!\[.*?\]\(.*?\))', description):
    images.append({
        'position': match.start(),
        'url': extract_image_url(match.group()),
        'type': detect_image_type(match.group())
    })
```

**原位插入标准格式**：
````markdown
[原文内容...]

![图片描述](图片URL)

---
**📸 图片解析（Vision Parser）**

**图片类型**：需求原型 | Bug截图 | 设计稿 | 流程图 | 架构图

**UI元素**（如适用）：
- 元素名称：[类型] - [状态] - [位置]

**文本内容（OCR）**：
- "[提取的文本1]"
- "[提取的文本2]"

**布局关系**：
- [元素A] 在 [元素B] 的 [方位]

**交互流程**（如适用）：
[动作1] → [动作2] → [结果]

**数据/表格**（如适用）：
| 列1 | 列2 |
|-----|-----|
| 值1 | 值2 |

**观察事实**：
1. [事实1 - 可直接观察到的]
2. [事实2 - 不含推理]

---

[原文继续...]
````

### 场景特化处理

#### 场景A：提取Story（需求）
```python
story = tapd_get_story(workspace_id, story_id)

# 完整输出（按固定结构）
output = f"""
# {story.name}

**需求ID**：{story.id}
**工作区**：{story.workspace_id}
**状态**：{story.status}
**优先级**：{story.priority}
**负责人**：{story.owner}
**创建人**：{story.creator}
**创建时间**：{story.created}
**修改时间**：{story.modified}
**迭代**：{story.iteration_id or '未分配'}
**开始日期**：{story.begin or '未设置'}
**截止日期**：{story.due or '未设置'}

## 需求描述

{process_with_images(story.description)}

## 自定义字段

{format_custom_fields(story)}
"""
```

#### 场景B：提取Bug（缺陷）
```python
bug = tapd_get_bug(workspace_id, bug_id)

output = f"""
# {bug.title}

**Bug ID**：{bug.id}
**状态**：{bug.status}
**严重程度**：{bug.severity}
**优先级**：{bug.priority}
**报告人**：{bug.reporter}
**当前处理人**：{bug.current_owner}
**创建时间**：{bug.created}
**修改时间**：{bug.modified}
**平台**：{bug.platform}
**模块**：{bug.module}

## Bug描述

{process_with_images(bug.description)}

## 解决方案

{bug.resolution or '待解决'}
"""
```

#### 场景C：提取Task（任务）
```python
task = tapd_get_task(workspace_id, task_id)

output = f"""
# {task.name}

**任务ID**：{task.id}
**状态**：{task.status}
**优先级**：{task.priority}
**负责人**：{task.owner}
**创建时间**：{task.created}
**开始日期**：{task.begin}
**截止日期**：{task.due}

## 任务描述

{process_with_images(task.description)}
"""
```

#### 场景D：批量提取
```python
urls = extract_urls_from_input(user_input)

for url in urls:
    workspace_id, entity_type, entity_id = parse_url(url)
    content = tool_map[entity_type](workspace_id, entity_id)
    
    print(f"\n{'='*60}")
    print(format_content(content, entity_type))
    print(f"{'='*60}\n")
```

#### 场景E：对比分析
```python
url1, url2 = extract_two_urls(user_input)

content1 = fetch_content(url1)
content2 = fetch_content(url2)

# 对比输出
print("## 需求对比分析\n")
print("### 左侧（原需求）")
print(format_content(content1))
print("\n### 右侧（新需求）")
print(format_content(content2))
print("\n### 差异分析")
print(compare_contents(content1, content2))
```

### 输出质量检查清单

提取完成后，验证：
- [ ] URL解析正确（workspace_id + entity_id）
- [ ] 原文完整（无遗漏字段）
- [ ] 格式未破坏（段落、列表、表格）
- [ ] 图片已检测（所有图片都找到）
- [ ] 图片解析已插入原位（紧跟图片）
- [ ] 标识清晰（使用📸 emoji + 分隔线）
- [ ] 无二次加工（原文一字不改）

---

## Agent 禁止行为

### ❌ 禁止臆造需求信息
- 不猜测用户意图
- 不补充未提供的业务规则
- 不假设验收标准

### ❌ 禁止跳过需求澄清
- 信息不完整时不直接生成方案
- 验收标准缺失时不创建需求

### ❌ 禁止创建不完整需求
- 无验收标准不创建
- 无业务目标不创建

### ❌ 禁止覆盖原始内容
- 不用 Agent 内容替换用户输入
- 不删除用户已写入信息
- 通过评论补充或增量更新

---

## 使用示例

### 示例 1：需求分析（含图片）
```
用户：帮我分析这个需求，这是原型图 [图片]

Agent：
1. 读取 guides/vision-parser.md
2. 执行图片解析（提取所有 UI 元素、文本、布局）
3. 执行需求理解（6 维度）
4. 检查信息完整性
5. 如有缺失，输出缺失清单
6. 用户补充后，执行需求拆解
7. 生成验收标准
8. 创建 TAPD 记录
```

### 示例 2：迭代规划
```
用户：帮我规划 Sprint 5

Agent：
1. tapd_list_iterations(status="open")
2. tapd_count_stories(iteration_id="")
3. tapd_list_stories(iteration_id="", limit=30)
4. 按 priority 排序
5. 计算容量（读取 guides/workflows.md）
6. 生成规划建议
7. 用户确认后批量更新
```

### 示例 3：Bug 分诊（含截图）
```
用户：帮我分诊这个 Bug [截图]

Agent：
1. 读取 guides/vision-parser.md
2. 执行图片解析（提取错误信息、UI 状态、环境信息）
3. 分析 severity（基于影响范围）
4. 分析 priority（基于业务优先级）
5. 推荐 owner（基于 module）
6. 展示分诊报告
7. 用户确认后创建/更新 Bug
```

---

## 详细参考文档

当需要详细规范时，按需读取以下文件：

### 指南（Guides）
- `guides/requirement-analysis.md` - 需求分析专业规范（需求理解、澄清、拆解、验收标准设计）
- `guides/workflows.md` - 最佳实践工作流（迭代规划、缺陷处理、需求变更、版本发布）
- `guides/templates.md` - Prompt 模板（需求分析、用户故事、任务拆解、缺陷分析、需求评审）
- `guides/vision-parser.md` - 图片解析完整规范（Lossless Visual Encoder）

### 参考（Reference）
- `reference/mcp-capabilities.md` - MCP 能力详解（工具列表、查询语法、字段说明、协作链路）
- `reference/tapd-scenarios.md` - TAPD 场景适配（7 大场景的业务目标、MCP 调用、输入输出规范）
- `reference/tool-reference.md` - 工具速查表

---

## 快速决策树

### 我应该做什么？
```
用户提到需求/想法？
  ↓ Yes
执行需求分析流程
  有图片？ → Yes → 执行 Vision Parser
  ↓ No
需求信息完整？
  ↓ No
输出缺失清单，要求补充
  ↓ Yes
需求拆解 → 验收标准 → 用户确认 → 创建记录
```

```
用户提到迭代规划/Sprint？
  ↓ Yes
读取 guides/workflows.md（迭代规划流程）
  ↓
tapd_list_iterations + tapd_count_stories
  ↓
生成规划建议 → 用户确认 → 批量分配
```

```
用户提到 Bug 分诊？
  ↓ Yes
tapd_list_bugs(status="new")
  有截图？ → Yes → 执行 Vision Parser
  ↓
分析 severity/priority/owner
  ↓
展示分诊报告 → 用户确认 → 批量更新
```

```
用户提到日报/进度？
  ↓ Yes
读取 guides/workflows.md（日报生成流程）
  ↓
查询昨日完成 + 今日计划 + 阻塞项
  ↓
格式化输出日报
```

---

**版本**：v2.0.0 (模块化)  
**最后更新**：2026-06-12  
**核心文件**：skill.md（本文件）+ guides/ + reference/

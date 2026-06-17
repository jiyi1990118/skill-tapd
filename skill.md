---
name: skill-tapd
description: TAPD 项目管理 - 需求/迭代/缺陷/任务管理，支持内容提取、状态流转、批量更新
whenToUse: |
  TAPD 项目管理：需求分析、迭代规划、bug分诊、日报、状态流转、内容提取
  触发词：需求分析、迭代规划、sprint、bug分诊、日报、流转、提取、获取、搜索、评论、成员
  触发模式：
    - "提取/获取/查看 [需求] {URL}" → 提取+分析
    - "{URL} 分析" → 提取+分析
    - "提取...融合(不要修改)" → 原始内容+图片解析（一字不改）
    - "07/22迭代的需求清单" → 匹配迭代→拉取→表格
    - "流转 [需求] 为 开发中" → 查找→**has_developer()**→判断→更新
    - "批量修改 [迭代] [状态] 为 [目标]" → 匹配→**开发人过滤**→筛选→批量更新
    - "[迭代] 技术评估→需求排期，规模=开发人数，处理人=开发人名，无开发人忽略" → 匹配→提取开发人→**默认忽略无开发人**→计算→批量更新(status+size+owner)
---

# TAPD 项目管理 Skill

**目标**：标准化使用 MCP 与 TAPD 进行需求管理、任务拆解、缺陷跟踪的方式。

---

## 核心原则

1. **信息完整**：不足时**必须澄清**，禁止臆造
2. **图片解析**：description 含 `<img>` / 图片URL / base64 → 立即 Vision Parser（读 `guides/vision-parser-quick.md`）
3. **写入确认**：create/update/delete 需用户确认，批量操作展示清单
4. **先统计后详情**：count → list → get

### 3. 二次确认机制
- ✅ 所有写入操作（create/update/delete）需用户明确确认
- ✅ 批量操作前展示变更清单
- ✅ 增量更新，不覆盖原始内容

### 4. 优先统计后详情
- ✅ 使用 `tapd_count_*` 获取统计数据
- ✅ 按需使用 `tapd_list_*` 拉取列表
- ✅ 只在必要时使用 `tapd_get_*` 获取详情

---

## 状态流转规则 ⭐

**原则 1**：只能向前流转，禁止向后回退。

**Workspace 48801209 状态顺序**：
```
new → planned → tech_review → developing → testing → resolved → done → closed
 1       2           3            4           5          6         7        8
```

**流转判断**：目标状态位置 ≤ 当前位置 → 跳过；目标位置 > 当前位置 → 执行。

**快捷流转**："完成需求"→`done`，"开始开发"→`developing`，"排期"→`planned`，"提测"→`testing`。

---

**原则 2（默认前置条件）**：状态变更前，需求**必须**有明确的开发人员，否则忽略。

**开发人员判定规则**（按优先级匹配 description 文本）：
1. 命中 `开发人员[：:]\s*<名字>` 标注 → 提取名字
2. 命中 `负责人[：:]\s*<名字>` 标注 → 提取名字
3. 命中 `developer[s]?[：:]\s*<name>` 标注 → 提取名字
4. 命中 `@<名字>` 提及（中文 2-4 字）→ 提取名字
5. **以上都未命中** → ⚠️ 视为无开发人员，**忽略不处理**（非报错）

```python
import re
DEV_PATTERNS = [
    r'开发人员[：:]\s*([一-龥\w\s、,，/|]+?)(?=\n|;|；|$|（|\(|\s{2,})'),
    r'负责人[：:]\s*([一-龥\w\s、,，/|]+?)(?=\n|;|；|$|（|\(|\s{2,})'),
    r'developer[s]?[：:]\s*([\w\s,，/|]+?)(?=\n|;|;|$|\(|\s{2,})',
    r'@([一-龥]{2,4})',
]

def has_developer(description: str) -> tuple[bool, list[str]]:
    """
    返回 (是否有开发人员, 提取到的开发人列表)
    """
    devs = set()
    for pat in DEV_PATTERNS:
        for m in re.finditer(pat, description or ''):
            names = re.split(r'[、,，/|;\s]+', m.group(1).strip())
            devs.update(n.strip() for n in names if n.strip() and len(n.strip()) >= 2)
    return (len(devs) > 0, list(devs))
```

**典型示例**：
| description 片段 | 是否视为有开发人员 |
|---|---|
| `开发人员：张三、李四` | ✅ 有（张三、李四） |
| `负责人：王五` | ✅ 有（王五） |
| `developer: Alice, Bob` | ✅ 有（Alice、Bob） |
| `需告知 @王登林(王登林) 券code` | ✅ 有（王登林） |
| `需 @阙竟毅 给一个表头字段` | ✅ 有（阙竟毅） |
| `需求说明：1、增敏这边的调整 @张增敏 ...` | ✅ 有（张增敏） |
| 全文无 `开发人员` / `负责人` / `@` 标注 | ❌ 无，忽略 |
| `协调 @王登林 关于券code`（@ 后跟对话/对接人） | ✅ 有，但需人工事后复核 |

**复杂批量**："[迭代] [状态]→[目标]，规模=开发人数，处理人=开发人名，无开发人忽略"
- 从 description 提取开发人（按 `DEV_PATTERNS` 正则）
- size=人数，owner=第一人，无开发人→跳过
- 输出：✅更新 / ⚠️忽略(无开发人) / ❌跳过(方向不对)

**确认清单**：确认状态顺序 → 获取当前状态 → 判断方向 → **判断有无开发人员** → 展示清单 → 确认 → 执行 → 记录评论。

---
if user_confirms():
    for item in can_update:
        # 使用 tapd_update_story（单条更新，因为字段不同）
        # 或使用 batch_update 如果所有字段相同
        tapd_update_story(
            workspace_id=workspace_id,
            story_id=item['story'].id,
            status=item['target_status'],
            size=str(item['size']),
            owner=item['owner']
        )
        
        # 记录评论
        tapd_create_comment(
            workspace_id=workspace_id,
            entry_type="stories",
            entry_id=item['story'].id,
            description=f"""批量字段更新：
- 状态：{item['current_status']} → {item['target_status']}
- 规模：{item['size']}（开发人员数量）
- 处理人：{item['owner']}（开发人员）
- 开发人员：{",".join(item['dev_names'])}"""
        )
    
    print(f"✅ 成功更新 {len(can_update)} 个需求")
    print(f"⚠️ 忽略 {len(skip_no_dev)} 个需求（无开发人员）")
    print(f"❌ 跳过 {len(skip_transition)} 个需求（流转方向不对）")
```

**关键约束**：
- **无开发人员的需求 → 一律忽略不处理**（适用于所有状态变更场景，不仅是复杂批量）
- 规模字段（size）= 提取到的开发人员数量
- 处理人（owner）= 取第一个开发人员名字（或按用户指定规则）
- 流转方向判断仍然生效（向前流转才执行）
- 批量操作中字段值可能不同（每个需求的开发人员不同），需逐个更新或使用 batch_update 分组
- `has_developer()` 必须从 description 文本中匹配 `DEV_PATTERNS` 正则，不能仅依据 `owner` 字段（owner 是历史处理人，可能已离职/转岗）

---

### 7. 流转执行前确认清单

- [ ] 已确认目标 workspace 的状态顺序
- [ ] 已获取需求当前状态
- [ ] 已判断流转方向（向前/向后/同级）
- [ ] 向后流转 → 输出跳过说明
- [ ] **已通过 `has_developer()` 判断需求是否有开发人员**
- [ ] **无开发人员 → ⚠️ 忽略，不展示在确认清单**
- [ ] 向前流转 + 有开发人员 → 展示变更清单 → 用户确认 → 执行

---

## 工作流快速索引

| 工作流 | 核心步骤 | 详细规范 |
|---|---|---|
| **需求分析** | 检测图片→Vision Parser→6维度理解→拆解→验收标准→创建 | `guides/requirement-analysis.md` |
| **迭代规划** | list_iterations→count_stories→list_stories→排序→容量分析→批量分配 | `guides/workflows.md` 第一章 |
| **Bug分诊** | count_bugs→list_bugs→Vision Parser(截图)→判断severity/priority→批量更新 | `guides/workflows.md` 第二章 |
| **日报生成** | 计算时间→查昨日完成→查今日计划→识别阻塞→格式化输出 | `guides/workflows.md` 第三章 |
| **状态流转** | 解析意图→查找需求→获取状态→判断方向→展示清单→确认→执行 | `guides/workflows.md` 第十章 |
| **迭代清单** | 提取标识→list_iterations→模糊匹配→list_stories→表格展示 | `guides/workflows.md` 第九章 |
| **搜索过滤** | 解析条件→count→list→表格展示 | `guides/workflows.md` 第十一章 |
| **评论管理** | list_comments / create_comment（entry_type用复数：stories/bug/tasks） | `guides/workflows.md` 第十二章 |
| **成员查找** | list_users→过滤→表格展示 | `guides/workflows.md` 第十三章 |

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

### 批量更新工具（高效优先）
- `tapd_batch_update_stories` - 批量更新需求（分配迭代、修改优先级等）
- `tapd_batch_update_bugs` - 批量更新缺陷（分诊后批量设置severity/priority）
- `tapd_batch_update_tasks` - 批量更新任务（批量完成/分配）

### 迭代管理
- `tapd_lock_iteration` - 锁定迭代（防止修改）
- `tapd_unlock_iteration` - 解锁迭代（允许修改）

### 写入工具（需二次确认）
- `tapd_create_story/bug/task` - 创建记录
- `tapd_update_story/bug/task` - 更新单个记录（增量更新）
- `tapd_delete_story/bug/task` - 删除记录（实为设置status='deleted'）
- `tapd_create_comment` - 添加评论（entry_type用复数：stories/bug/tasks）

⚠️ **状态值映射**：不同 workspace 的状态值不同，修改状态前务必查阅 `reference/status-mapping.md`，切勿臆造状态值（如 `status_2`）。
- 如需设置"需求排期"，先通过 `tapd_get_story` 确认参考需求的状态值

### 图片工具
- `tapd_download_image` - 下载认证图片（已内嵌在详情中）

**完整工具参考**：读取 `reference/mcp-tools.md`

---

## 异常处理与边界情况

### 常见错误及应对

| 错误类型 | 表现 | 处理方式 |
|---|---|---|
| **401/403 权限不足** | 无法访问项目或执行操作 | 提示用户前往 TAPD 开放平台配置应用权限 |
| **404 需求/迭代不存在** | 找不到指定的需求或迭代 | 检查 ID 是否正确，或列出候选让用户选择 |
| **400 状态值无效** | 设置的状态不存在于该 workspace | 查询参考需求确认正确状态值 |
| **网络超时** | API 无响应 | 重试一次，仍失败则提示稍后重试 |
| **需求查找失败** | 按标题搜索无结果 | 建议用更精确的关键词、ID 或 URL |
| **迭代匹配失败** | 找不到匹配的迭代 | 列出所有迭代让用户选择 |
| **批量操作部分失败** | 部分需求更新成功，部分失败 | 汇总成功/失败列表，展示失败原因 |

### 模糊匹配兜底策略

```
需求查找：
  按标题搜索 → 0 条 → 提示用 ID 或 URL
  按标题搜索 → 多条 → 展示候选列表让用户选择
  按标题搜索 → 1 条 → 直接命中

迭代匹配：
  模糊匹配 → 无结果 → 列出所有迭代
  模糊匹配 → 1 条 → 直接命中
```

---

## 需求分析六维度

Why（业务目标）、Who（用户角色）、When&Where（使用场景）、Value（业务价值）、Boundary（边界条件）、Done（成功标准）。**任一维度缺失，输出缺失清单要求补充。**

---

## 图片解析

**检测**：description 含 `<img>` / `.jpg|.png` / `base64` / 用户提到"原型图/设计稿/截图"。
**执行**：读 `guides/vision-parser.md` → 按14部分标准解析 → 补充到需求理解。

---

## 从 TAPD URL 提取内容

**支持**：story / bug / task URL。

**模式**：
- **A（分析）**："提取 {URL} 内容" → 提取 + 结构化展示 + 图片解析
- **B（融合）**："提取...融合（不要修改）" → 原始内容 + 图片解析（一字不改，供下游使用）
- **C（批量）**：多个 URL → 逐个提取
- **D（对比）**："{URL1} vs {URL2}" → 对比差异

**判断逻辑**：
```
含 tapd.cn URL →
  含"融合"/"不要修改" → 模式B（禁止修改）
  含"对比"/"区别" → 模式D
  多个URL → 模式C
  其他 → 模式A
```

模式C：批量提取
"提取以下需求的内容：{URL1} {URL2} {URL3}"
"帮我看看这几个需求：{URL1} {URL2}"

模式D：对比提取
"对比这两个需求的差异：{URL1} vs {URL2}"
"{URL1} 和 {URL2} 有什么区别"
```

### 指令意图快速识别

当用户输入包含以下元素时，**立即触发提取流程**：
1. **含 TAPD URL** → 无论前面说什么，先提取内容
2. **关键词**：`获取`/`提取`/`查看`/`分析` + `需求`/`tapd`/`链接`/`内容`
3. **URL 后直接跟指令**：`{URL} 分析一下`、`{URL} 帮我看看`
4. **融合指令关键词**：`融合`/`合并`/`原始内容`/`不要修改`/`不要二次` → 触发模式B（一字不改）

**判断逻辑**：
```
if 包含 tapd.cn URL:
    if 包含 "融合" 或 "不要修改" 或 "原始":
        → 模式B（提取+融合，禁止修改）
    else if 包含 "对比" 或 "区别":
        → 模式D（对比提取）
    else if 多个URL:
        → 模式C（批量提取）
    else:
        → 模式A（提取+分析）
```

### 通用执行规则 🔒

#### 1. URL解析（自动检测）
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

**自动检测规则**：用户消息中只要包含 `tapd.cn` 域名 + `story/bug/task` 路径，**无论前面说什么**，都触发提取流程。

#### 2. 内容提取原则
- ✅ **完整性**：一字不改地提取所有内容
- ✅ **原始性**：保留原文格式、结构、顺序
- ❌ **禁止加工**：不修改、不总结、不优化

#### 3. 模式B特殊规则：融合输出（一字不改）

当用户明确要求"融合"、"不要二次修改"、"原始内容"时：

```
执行规则：
1. 提取 story.description（原始文本，一字不改）
2. 检测所有图片 → Vision Parser 解析
3. 输出格式：原始内容 + 图片解析结果（原位插入）
4. ❌ 禁止：总结、改写、添加分析、添加个人理解
5. ✅ 允许：格式化排版、添加图片解析区块
```

**输出模板（模式B）**：
```markdown
# {需求名称}（原始内容）

**需求ID**：{id}
**状态**：{status}
**优先级**：{priority}

---

## 需求描述（原文）

{原始 description，一字不改}

---

## 图片内容解析

### 图片1：{位置描述}
**图片类型**：需求原型 | Bug截图 | 设计稿 | 流程图

**UI元素**：
- ...

**文本内容（OCR）**：
- "..."

**布局关系**：
- ...

---

## 自定义字段（如有）

...
```

**关键约束**：
- `description` 字段原文**必须完整保留**，不得删减
- 图片解析结果以**追加区块**形式放在后面，不替换原文
- 不添加"需求理解"、"分析"、"建议"等二次加工内容
- 输出目的是**供下游模型/人工进一步处理使用**

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

#### 场景A：提取Story（需求）- 标准分析模式
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

#### 场景A+：提取Story（需求）- 融合模式（一字不改）
当用户要求"融合"、"不要修改"、"供下游使用"时：

```python
story = tapd_get_story(workspace_id, story_id)

# 步骤1：提取原始内容（一字不改）
raw_description = story.description  # 完整保留，不处理

# 步骤2：检测图片并解析
images = extract_images(raw_description)
vision_results = []
for img in images:
    vision_results.append({
        'url': img.url,
        'position': img.position,
        'result': vision_parse(img.url)  # Vision Parser
    })

# 步骤3：输出（原始内容 + 图片解析区块）
output = f"""
# {story.name}（原始内容）

**需求ID**：{story.id}
**状态**：{story.status}
**优先级**：{story.priority}

---

## 需求描述（原文，未修改）

{raw_description}

---

## 图片内容解析（供参考）
"""

for vr in vision_results:
    output += f"""
### 📸 图片：{vr['url']}

**UI元素**：
{format_ui_elements(vr['result'])}

**文本内容（OCR）**：
{format_ocr_text(vr['result'])}

**布局关系**：
{format_layout(vr['result'])}

---
"""

output += f"""
## 元数据

**负责人**：{story.owner}
**创建人**：{story.creator}
**创建时间**：{story.created}
**迭代**：{story.iteration_id or '未分配'}

---
⚠️ 以上内容中"需求描述"为 TAPD 原文，未做任何修改。
图片解析结果仅供参考，供下游处理使用。
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

---

## 迭代需求清单智能提取 ⭐ 新增强

**一句话指令，多步骤自动执行。**

### 触发指令（极简表达 → 智能推断）

| 用户说法 | AI 理解 | 执行步骤 |
|---|---|---|
| "帮我获取 07/22 迭代的需求清单" | 找到名字/日期含"07/22"的迭代 → 拉取其需求列表 | 1. 列出迭代 2. 模糊匹配 3. 拉取需求 4. 表格展示 |
| "看看本周迭代有什么需求" | 找到当前开放迭代 → 列出需求 | 1. 列出开放迭代 2. 拉取需求 3. 表格展示 |
| "Sprint 5 有哪些需求" | 找到名字含"Sprint 5"的迭代 → 列出需求 | 1. 列出迭代 2. 模糊匹配 3. 拉取需求 4. 表格展示 |
| "7月份迭代的需求列表" | 找到 7 月份相关的迭代 → 列出需求 | 1. 列出迭代 2. 日期匹配 3. 拉取需求 4. 表格展示 |
| "当前迭代有哪些需求没完成" | 找到当前开放迭代 → 列出未完成需求 | 1. 列出开放迭代 2. 过滤状态 3. 表格展示 |
| "这个迭代的需求" | 找到最近/唯一的开放迭代 → 列出需求 | 1. 列出迭代 2. 取最近一个 3. 拉取需求 4. 表格展示 |

### 执行流程（自动推断）

```python
def extract_iteration_needs(user_input, workspace_id):
    """
    一句话提取迭代需求清单
    """
    # Step 1: 提取用户提到的迭代标识
    iteration_hint = extract_iteration_hint(user_input)
    # 示例: "07/22" → hint="07/22"
    # 示例: "本周" → hint="current_week"
    # 示例: "Sprint 5" → hint="Sprint 5"
    # 示例: "7月份" → hint="2026-07"
    # 示例: "当前" → hint="current"

    # Step 2: 获取迭代列表
    iterations = tapd_list_iterations(workspace_id=workspace_id, limit=50)

    # Step 3: 模糊匹配目标迭代
    target_iteration = fuzzy_match_iteration(iterations, iteration_hint)

    if not target_iteration:
        return format_iteration_list(iterations)

    # Step 4: 拉取需求列表
    stories = tapd_list_stories(
        workspace_id=workspace_id,
        iteration_id=target_iteration.id,
        limit=200,
        fields="id,name,status,owner,priority,creator,created,modified,begin,due,size,description"
    )

    # Step 5: 表格展示
    return format_stories_table(stories, target_iteration)
```

### 模糊匹配规则

匹配优先级：名称包含 → 日期包含 → 数字匹配 → 当前周 → 月份匹配 → 最近open → 最近任意。

### 表格展示

```markdown
## 📋 迭代需求清单 - {iteration.name}

| 序号 | 需求ID | 需求名称 | 状态 | 优先级 | 负责人 | 规模 | 创建人 | 创建时间 |
|------|--------|----------|------|--------|--------|------|--------|----------|
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

**统计**: 总数{total} | 开发中{developing} | 已完成{done} | 未开始{planned} | 高优{high_priority}
```

### 自然表达对照

| 用户说 | 匹配策略 |
|---|---|
| "07/22 迭代" | 名称/日期含 07/22 |
| "本周迭代" | open + 日期含今天 |
| "Sprint 5" | 名称含 Sprint 5 |
| "7月份" | startdate 含 2026-07 |
| "最近的迭代" | 最近的 open |
| "这个迭代" | 唯一的或最近的 open |

---

## Agent 禁止行为

- ❌ 不臆造需求信息、不猜测意图、不补充未提供规则
- ❌ 不跳过需求澄清、信息不完整不生成方案
- ❌ 不创建不完整需求（无验收标准/无业务目标不创建）
- ❌ 不覆盖原始内容（增量更新、通过评论补充）

---

## 使用示例

**示例1：需求分析（含图片）**
```
用户：帮我分析这个需求，这是原型图 [图片]
Agent：1. Vision Parser → 2. 6维度理解 → 3. 完整性检查 → 4. 拆解 → 5. 验收标准 → 6. 创建记录
```

**示例2：迭代规划**
```
用户：帮我规划 Sprint 5
Agent：1. list_iterations(open) → 2. count_stories(未分配) → 3. list_stories → 4. 排序 → 5. 容量分析 → 6. 生成建议 → 7. 批量分配
```

**示例3：Bug分诊（含截图）**
```
用户：帮我分诊这个 Bug [截图]
Agent：1. Vision Parser → 2. 分析severity/priority → 3. 推荐owner → 4. 展示报告 → 5. 批量更新
```

---

## 详细参考文档

- `guides/requirement-analysis.md` - 需求分析规范
- `guides/workflows.md` - 工作流最佳实践
- `guides/templates.md` - Prompt模板
- `guides/vision-parser.md` - 图片解析规范
- `reference/mcp-tools.md` - MCP工具参考
- `reference/status-mapping.md` - 状态值映射

---

## 快速决策树

```
用户提到需求/想法？ → 需求分析流程 → 有图片？Vision Parser → 完整？拆解→验收→创建
用户提到迭代规划/Sprint？ → list_iterations + count_stories → 规划建议 → 批量分配
用户提到"迭代需求"/"Sprint需求"/"本周需求"？ → 提取标识→匹配迭代→list_stories→表格
用户提到Bug分诊？ → list_bugs(new) → 有截图？Vision Parser → 分析severity/priority/owner → 批量更新
用户提到日报/进度？ → 查昨日完成+今日计划+阻塞项 → 格式化输出
用户提到"流转"/"修改状态"/"状态改为"/"批量修改"？ → 解析→获取状态→**has_developer() 判断**→判断方向→展示→确认→执行
用户提到"搜索"/"查找"/"过滤"+需求？ → 解析条件→count→list→表格
用户提到"评论"/"讨论记录"？ → 查看list_comments / 添加create_comment
用户提到"成员"/"负责人"/"团队成员"？ → list_users→过滤→表格
```

---

**版本**：v2.7.0 (状态流转默认前置：需求需有开发人员，否则忽略)  
**最后更新**：2026-06-17  
**核心文件**：skill.md + guides/ + reference/

---

## 变更日志

### v2.7.0 (2026-06-17)
- ✅ **状态流转默认前置条件**：所有状态变更前必须 `has_developer()` 判断，description 中需匹配 `开发人员`/`负责人`/`developer`/`@名字` 标注
- ✅ **无开发人员 = 一律忽略**（不仅是复杂批量场景，单个流转、批量流转同样适用）
- ✅ **owner 字段不再作为判断依据**（owner 是历史处理人，可能已离职/转岗；改以 description 中的 `DEV_PATTERNS` 为准）
- ✅ 关键约束清单、确认清单、决策树同步更新

### v2.6.0 (2026-06-16)
- ✅ 新增复杂批量字段更新：状态流转+从description动态提取开发人+计算size/owner+无开发人忽略
- ✅ 新增触发词：规模、开发人员、处理人员、字段更新

### v2.5.0 (2026-06-16)
- ✅ 修复结构断裂、新增异常处理规范、快捷流转指令、修复日报状态值、新增搜索/评论/成员工作流

### v2.4.0 (2026-06-16)
- ✅ 新增状态流转智能判断（只能向前，禁止回退）、单个/批量需求流转工作流

### v2.3.0 (2026-06-16)
- ✅ 新增迭代需求清单智能提取（一句话指令→多步骤执行→表格展示）

### v2.2.0 (2026-06-16)
- ✅ 同步MCP需求状态、新增融合模式（一字不改输出）
4. 分析 priority（基于业务优先级）
5. 推荐 owner（基于 module）
6. 展示分诊报告
7. 用户确认后创建/更新 Bug
```


# TAPD MCP 工具完整参考

## 快速索引

| 场景 | 推荐工具 | 优先级 |
|------|---------|--------|
| 统计数据 | `tapd_count_*` | ⭐⭐⭐ 优先使用 |
| 列表查询 | `tapd_list_*` | ⭐⭐ 按需使用 |
| 详情查询 | `tapd_get_*` | ⭐ 必要时使用 |
| 创建记录 | `tapd_create_*` | 需二次确认 |
| 更新记录 | `tapd_update_*` | 需二次确认 |

---

## 一、需求管理（Story）

### tapd_count_stories
**用途**：快速统计需求数量  
**场景**：迭代容量分析、进度统计

**参数**：
- `workspace_id` (必填), `status`, `owner`, `creator`, `priority`
- `iteration_id` - "<>" 表示未分配
- `created`, `modified` - 支持 `>date`, `<date`, `date~date`

**示例**：
```python
# 统计未分配高优需求
tapd_count_stories(workspace_id="123", iteration_id="<>", priority="urgent|high")
```

### tapd_list_stories
**用途**：列出需求列表  
**场景**：查看需求池、迭代规划

**参数**：同 count_stories，额外支持 `limit`(默认30，最大200), `page`, `fields`

**示例**：
```python
tapd_list_stories(workspace_id="123", iteration_id="<>", priority="urgent|high", limit=30)
```

### tapd_get_story
**用途**：获取需求详情（含图片）  
**场景**：需求分析、任务拆解

**返回**：所有字段 + 图片内嵌在 description + 自定义字段(8个)

**特殊处理**：如有图片 → 触发 Vision Parser

### tapd_create_story
**用途**：创建需求  
**前置**：需求信息完整 + 用户已确认

**必填**：`workspace_id`, `name`  
**推荐**：`description`, `priority`, `owner`, `iteration_id`, `begin`, `due`

### tapd_update_story
**用途**：更新需求（增量）  
**原则**：只更新变化字段，不覆盖原始内容

### tapd_delete_story
**用途**：删除需求（实为status='deleted'）  
**警告**：高风险，必须二次确认

---

## 二、缺陷管理（Bug）

### Severity 级别

| 级别 | 定义 | 影响范围 |
|------|------|----------|
| **fatal** | 系统崩溃、数据丢失 | 所有用户 |
| **serious** | 核心功能部分不可用 | 大部分用户 |
| **normal** | 一般功能异常 | 部分用户 |
| **slight** | 轻微体验问题 | 少数用户 |
| **suggest** | 优化建议 | - |

### Priority 级别

| 级别 | 定义 | 处理时效 |
|------|------|----------|
| **urgent** | 线上故障 | 立即(< 2h) |
| **high** | 核心流程受影响 | 当天 |
| **medium** | 非核心功能 | 本迭代 |
| **low** | 体验优化 | 下迭代 |

### tapd_count_bugs / tapd_list_bugs / tapd_get_bug
用法同 Story，支持过滤参数：`severity`, `priority`, `current_owner`, `reporter`, `title`

### tapd_create_bug
**前置**：包含复现步骤、预期/实际结果，Severity/Priority已判断

**必填**：`workspace_id`, `title`  
**推荐**：`description`, `severity`, `priority`, `current_owner`, `module`, `platform`

### tapd_update_bug
**用途**：Bug分诊，更新 severity/priority/current_owner

---

## 三、任务管理（Task）

### tapd_count_tasks / tapd_list_tasks / tapd_get_task
用法同 Story

### tapd_create_task
**前置**：任务粒度 ≤ 2天，包含描述、负责人、验收标准

**必填**：`workspace_id`, `name`  
**推荐**：`description`, `owner`, `priority`, `iteration_id`, `begin`, `due`

---

## 四、迭代管理（Iteration）

### tapd_list_iterations
**参数**：`workspace_id`, `status`(open|done), `name`, `limit`, `page`

```python
tapd_list_iterations(workspace_id="123", status="open")
```

### tapd_get_iteration
**返回**：id, name, status, startdate, enddate, description

---

## 五、协作工具

### tapd_list_comments
**参数**：`workspace_id`, `entry_type`(story|bug|task), `entry_id`, `limit`, `page`

### tapd_create_comment
**用途**：记录讨论和变更

```python
tapd_create_comment(workspace_id="123", entry_type="story", entry_id="456", description="已评审通过")
```

### tapd_list_users
**用途**：列出工作区成员，推荐任务负责人

### tapd_list_workspaces / tapd_get_workspace
**用途**：工作区管理

---

## 六、查询语法

### 时间范围
```python
created=">2026-06-01"              # 晚于
modified="<2026-06-10"             # 早于
created="2026-06-01~2026-06-10"    # 区间
```

### 枚举 OR
```python
status="new|in_progress|done"
severity="fatal|serious"
priority="urgent|high"
```

### 模糊搜索
```python
name="登录"    # 自动转 LIKE<登录>
title="支付"
```

### 不等于
```python
iteration_id="<>"      # 未分配
iteration_id="<>123"   # 不等于123
```

### 分页
```python
tapd_list_stories(limit=30, page=1)   # 第1页
tapd_list_stories(limit=200, page=1)  # 增大单页（最大200）
```

---

## 七、工作流组合

### 迭代规划
```python
# 1. 统计未分配需求
count = tapd_count_stories(workspace_id="123", iteration_id="<>")

# 2. 拉取列表
stories = tapd_list_stories(workspace_id="123", iteration_id="<>", limit=30)

# 3. 用户确认后批量分配
for story in selected:
    tapd_update_story(workspace_id="123", story_id=story.id, iteration_id="迭代5")
```

### Bug 分诊
```python
# 1. 统计新Bug
count = tapd_count_bugs(workspace_id="123", status="new")

# 2. 拉取列表
bugs = tapd_list_bugs(workspace_id="123", status="new", limit=30)

# 3. 逐个分析（如有截图，执行Vision Parser）

# 4. 用户确认后批量更新
for bug in bugs:
    tapd_update_bug(workspace_id="123", bug_id=bug.id, severity="serious", priority="high")
```

### 需求分析（含图片）
```python
# 1. 获取详情
story = tapd_get_story(workspace_id="123", story_id="456")

# 2. 检测图片 → Vision Parser
if has_image(story.description):
    vision_result = parse_vision(story.description)

# 3. 需求理解（6维度）→ 拆解 → 创建Task
```

---

## 八、常见问题

**Q: 什么时候用 count/list/get？**  
A: 只需数量 → count；需列表 → list；需详情 → get

**Q: 如何批量操作？**  
A: 展示变更清单 → 用户确认 → 循环调用 update

**Q: 图片在哪里？**  
A: 已内嵌在 `tapd_get_*` 返回的 description 中

**Q: 如何关联 Story 和 Task？**  
A: 在 Task 的 description 中引用 Story ID

---

**版本**：v2.0.0  
**最后更新**：2026-06-12

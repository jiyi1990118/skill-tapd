# TAPD 工作流最佳实践

## 一、迭代规划流程

### 标准流程
```
获取开放迭代 → 统计未分配需求 → 拉取需求列表 → 优先级排序 → 容量分析 → 生成建议 → 用户确认 → 批量分配
```

### 详细步骤

**Step 1：获取开放迭代**
```python
iterations = tapd_list_iterations(workspace_id="123456", status="open")
# 通常选择最近的开放迭代
target_iteration = iterations[0]
```

**Step 2：统计未分配需求**
```python
count = tapd_count_stories(workspace_id="123456", iteration_id="<>")
```

**Step 3：拉取需求列表（优先级排序）**
```python
stories = tapd_list_stories(
    workspace_id="123456",
    iteration_id="<>",
    limit=50
)
# 按 priority 排序：urgent → high → medium → low
```

**Step 4：容量分析**
```
团队可用人天 = 成员数 × 迭代天数 × 效率系数

效率系数：
- 0.6-0.7：新团队、技术债多
- 0.7-0.8：成熟团队
- 0.8-0.9：高效团队

示例：
- 5 人团队，10 天迭代，效率系数 0.75
- 可用人天 = 5 × 10 × 0.75 = 37.5 人天
```

**Step 5：生成规划建议**
```markdown
## 迭代规划建议 - Sprint 5
**容量**: 37.5 人天
**待分配**: 15 个需求

### 建议纳入（按优先级）：
1. [Story-123] 用户登录功能 - urgent - 预估: 5天
2. [Story-124] 订单列表 - high - 预估: 3天
3. [Story-125] 支付对接 - high - 预估: 4天
...（累计 35 天）

### 不建议纳入：
- [Story-130] 数据统计 - low（延后到 Sprint 6）

**是否确认分配？**
```

**Step 6：批量分配**
```python
for story in selected_stories:
    tapd_update_story(
        workspace_id="123456",
        story_id=story.id,
        iteration_id="迭代5"
    )
```

---

## 二、Bug 分诊流程

### 标准流程
```
获取新 Bug → 统计概况 → 逐个分析 → 展示分诊报告 → 用户确认 → 批量更新
```

### Severity 判断标准

| Severity | 定义 | 示例 | 处理时效 |
|----------|------|------|----------|
| **fatal** | 系统崩溃、数据丢失、核心功能完全不可用 | 支付失败、数据库崩溃 | 立即（< 2h） |
| **serious** | 核心功能部分不可用，有替代方案 | 登录慢、部分订单无法查询 | 当天 |
| **normal** | 一般功能异常，不影响主流程 | 按钮样式错位、非核心页面加载慢 | 本迭代 |
| **slight** | 轻微体验问题 | 文案错误、边距不对 | 下迭代 |
| **suggest** | 优化建议，非缺陷 | 建议增加某个功能 | 待评估 |

### Priority 判断标准

| Priority | 定义 | 处理时效 |
|----------|------|----------|
| **urgent** | 线上故障，影响所有用户 | 立即修复（< 2h） |
| **high** | 影响核心流程，部分用户受影响 | 当天修复 |
| **medium** | 影响非核心功能 | 本迭代修复 |
| **low** | 体验优化 | 下迭代修复 |
| **insignificant** | 可忽略 | 待评估 |

### 详细步骤

**Step 1：统计概况**
```python
total = tapd_count_bugs(workspace_id="123456", status="new")
fatal_count = tapd_count_bugs(workspace_id="123456", status="new", severity="fatal")
serious_count = tapd_count_bugs(workspace_id="123456", status="new", severity="serious")
```

**Step 2：拉取 Bug 列表（优先 fatal/serious）**
```python
bugs = tapd_list_bugs(
    workspace_id="123456",
    status="new",
    limit=30
)
```

**Step 3：逐个分析**
```python
for bug in bugs:
    detail = tapd_get_bug(workspace_id="123456", bug_id=bug.id)
    
    # 如有截图，执行 Vision Parser
    if has_image(detail.description):
        vision_result = parse_image(detail.description)
        # 提取错误信息、环境信息
    
    # 判断 severity（基于影响范围）
    severity = judge_severity(detail.description, vision_result)
    
    # 判断 priority（基于业务优先级）
    priority = judge_priority(detail.description, severity)
    
    # 推荐 owner（基于 module）
    owner = recommend_owner(detail.module)
```

**Step 4：展示分诊报告**
```markdown
## Bug 分诊报告
**新增 Bug**: 8
**严重**: 2 | **一般**: 5 | **轻微**: 1

### 需立即处理（fatal/serious）：
1. [Bug-456] 支付失败 - 订单状态未更新
   当前: severity=normal, priority=low, 无处理人
   建议: severity=serious, priority=high, owner=李四
   原因: 影响核心交易，3 个用户反馈

2. [Bug-457] 用户信息丢失
   建议: severity=fatal, priority=urgent, owner=张三
   原因: 导致用户无法登录，所有用户受影响

### 普通 Bug：
3. [Bug-458] 按钮样式错位
   建议: severity=slight, priority=low, owner=王五

**是否应用建议？**
```

**Step 5：批量更新**
```python
for bug, recommendation in triage_results:
    tapd_update_bug(
        workspace_id="123456",
        bug_id=bug.id,
        severity=recommendation.severity,
        priority=recommendation.priority,
        current_owner=recommendation.owner
    )
```

---

## 三、日报生成流程

### 标准流程
```
计算时间范围 → 查询昨日完成 → 查询今日计划 → 识别阻塞项 → 格式化输出
```

### 详细步骤

**Step 1：计算时间范围**
```python
import datetime

today = datetime.date.today()
yesterday = today - datetime.timedelta(days=1)

yesterday_start = f"{yesterday} 00:00:00"
yesterday_end = f"{today} 00:00:00"
```

**Step 2：查询昨日完成**
```python
# 昨日完成的需求
completed_stories = tapd_list_stories(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="done"
)

# 昨日完成的任务
completed_tasks = tapd_list_tasks(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="done"
)

# 昨日解决的 Bug
resolved_bugs = tapd_list_bugs(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="resolved"
)
```

**Step 3：查询今日计划**
```python
# 今日进行中
in_progress = tapd_list_stories(
    workspace_id="123456",
    status="in_progress"
) + tapd_list_tasks(
    workspace_id="123456",
    status="in_progress"
)
```

**Step 4：识别阻塞项**
```python
# 长时间无更新的高优先级项
blocked = tapd_list_stories(
    workspace_id="123456",
    priority="urgent|high",
    status="in_progress",
    modified=f"<{yesterday_start}"  # 超过 1 天未更新
)
```

**Step 5：格式化输出**
```markdown
## 📅 Daily Standup - 2026-06-12
**项目**: XXX 项目

### ✅ 昨日完成
- [Story-123] 完成用户登录功能 (@张三)
- [Bug-456] 修复支付超时问题 (@李四)
- [Task-789] 完成接口文档 (@王五)

### 🚀 今日计划
- [Story-124] 开发订单列表页 (@张三, in_progress)
- [Task-790] Code review (@李四, in_progress)

### ⚠️ 阻塞 & 风险
- [Story-125] 支付对接 - 等待第三方接口文档（已阻塞 2 天）
- [Bug-460] Fatal 级别 bug 未分配处理人

**建议**：
1. 尽快协调第三方接口文档
2. 分配 Bug-460 给订单模块负责人
```

---

## 四、需求变更流程

### 标准流程
```
收到变更请求 → 变更评估 → 影响分析 → 决策 → 更新记录 → 同步团队
```

### 变更影响评估模板

```markdown
## 需求变更评估 - Story-102

### 变更内容
**原需求**: 支持手机号+密码登录
**变更为**: 增加微信登录

### 影响分析

#### 工作量影响
- 增加工作量: +2 天
- 新增任务: 微信 OAuth 对接

#### 风险影响
- 需要申请微信开放平台账号（耗时 1-2 天）
- 需要测试微信授权流程
- 可能影响迭代时间

#### 依赖影响
- 依赖运营提供微信 AppID/Secret
- 依赖微信开放平台审核（1-2 天）

### 决策建议

**方案 1：接受变更，延长迭代时间**
- 迭代延长 2 天
- 优点：完整实现需求
- 缺点：延迟发布

**方案 2：接受变更，砍掉低优需求**
- 移除 Story-110（数据统计，低优）
- 优点：按时发布
- 缺点：功能不完整

**方案 3：拒绝变更，留待下期**
- 本期只实现手机号登录
- 微信登录放到 Sprint 6
- 优点：风险最低
- 缺点：无法满足完整需求

### 推荐方案
**方案 3**（风险最低，按时交付）

**是否确认变更？**
```

### 变更执行

```python
# 1. 更新 Story
tapd_update_story(
    workspace_id="123456",
    story_id="102",
    description="原描述 + \n\n## 变更记录\n- 2026-06-12: 增加微信登录功能"
)

# 2. 添加评论记录变更原因
tapd_create_comment(
    workspace_id="123456",
    entry_type="story",
    entry_id="102",
    description="需求变更：增加微信登录。原因：用户反馈强烈。变更评估：增加 2 天工作量。"
)

# 3. 调整 Task（如有必要）
```

---

## 五、需求分析流程（含图片）

### 标准流程
```
用户提出需求 → 获取详情（如有 ID）→ 图片解析 → 需求理解 → 需求澄清 → 需求拆解 → 验收标准设计 → 用户确认 → 创建记录
```

### 详细步骤

**Step 1：获取需求详情（如有）**
```python
if story_id:
    story = tapd_get_story(workspace_id="123456", story_id=story_id)
else:
    story = {"description": user_input}
```

**Step 2：图片解析（如有图片）**
```python
if has_image(story.description):
    # 读取 Vision Parser 规范
    vision_spec = read_file("guides/vision-parser.md")
    
    # 执行解析
    vision_result = parse_with_vision_parser(story.description, vision_spec)
    
    # 提取结构化信息
    ui_elements = vision_result.ui_elements
    ocr_text = vision_result.ocr_text
    tables = vision_result.tables
    observed_facts = vision_result.observed_facts
```

**Step 3：需求理解（6 维度）**
```
基于用户输入 + 图片解析结果，明确：

1. 业务目标（Why）
2. 用户角色（Who）
3. 使用场景（When & Where）
4. 业务价值（Value）
5. 边界条件（Boundary）
6. 成功标准（Done）
```

**Step 4：需求澄清**
```python
# 检查完整性
missing = check_completeness(story, vision_result)

if missing:
    output_missing_checklist(missing)
    wait_for_user_input()
```

**Step 5：需求拆解**
```
Story → Module → Feature → Task

示例：
[Story-102] 用户登录功能
  ↓
模块1：用户注册
模块2：用户登录
  ↓ 模块2 拆解
功能点1：手机号+密码登录
功能点2：微信登录
  ↓ 功能点1 拆解
[Task-201] 后端：实现登录接口 - 1天
[Task-202] 前端：登录页面 - 1天
[Task-203] 测试：功能测试 - 1天
```

**Step 6：验收标准设计**
```gherkin
Given 用户已注册且未登录
When 用户输入正确的手机号和密码，点击"登录"
Then 系统验证通过，跳转到首页，显示用户昵称
```

**Step 7：用户确认并创建**
```python
# 创建 Story
story = tapd_create_story(...)

# 创建 Task
for task in tasks:
    tapd_create_task(...)
```

---

## 六、质量分析流程

### 标准流程
```
选定时间范围 → 统计 Bug 分布 → 计算趋势 → 识别问题模块 → 生成报告
```

### 详细步骤

**Step 1：统计 Bug 分布**
```python
# 按 severity 统计
fatal = tapd_count_bugs(workspace_id="123456", severity="fatal")
serious = tapd_count_bugs(workspace_id="123456", severity="serious")
normal = tapd_count_bugs(workspace_id="123456", severity="normal")

# 按 status 统计
new = tapd_count_bugs(workspace_id="123456", status="new")
in_progress = tapd_count_bugs(workspace_id="123456", status="in_progress")
resolved = tapd_count_bugs(workspace_id="123456", status="resolved")
```

**Step 2：计算趋势**
```python
# 本周新增
this_week = tapd_count_bugs(
    workspace_id="123456",
    created=">2026-06-05"
)

# 本周解决
this_week_resolved = tapd_count_bugs(
    workspace_id="123456",
    modified=">2026-06-05",
    status="resolved"
)

# 净增
net_increase = this_week - this_week_resolved
```

**Step 3：识别问题模块**
```python
# 拉取所有 Bug，按 module 分组统计
bugs = tapd_list_bugs(workspace_id="123456", status="new|in_progress", limit=200)

module_stats = {}
for bug in bugs:
    module = bug.module or "未分类"
    module_stats[module] = module_stats.get(module, 0) + 1

# 排序
top_modules = sorted(module_stats.items(), key=lambda x: x[1], reverse=True)[:3]
```

**Step 4：生成报告**
```markdown
## 质量分析报告 - 2026年6月

### 📈 Bug 趋势
- 本周新增: 15
- 本周解决: 12
- 净增: +3
- 遗留总数: 45

### 🔴 严重级别分布
- Fatal: 2 (4%)
- Serious: 8 (18%)
- Normal: 25 (56%)
- Slight: 10 (22%)

### 📦 问题模块 TOP3
1. 订单模块: 12 bugs (27%)
2. 支付模块: 8 bugs (18%)
3. 用户中心: 6 bugs (13%)

### ⚠️ 风险提示
- 支付模块有 2 个 fatal bug 超过 3 天未修复
- 订单模块 bug 密度持续上升，建议加强测试

### 💡 改进建议
1. 增加支付模块 Code Review 覆盖率
2. 补充订单模块自动化测试
3. 定期进行质量回顾会议
```

---

## 七、关键原则总结

### 1. 优先统计后详情
```python
# ✅ 正确
count = tapd_count_stories(...)
if count > 0:
    stories = tapd_list_stories(limit=30)

# ❌ 错误
stories = tapd_list_stories(limit=200)  # 直接拉取大量数据
```

### 2. 批量操作前确认
```python
# ✅ 正确
display_change_list(changes)
if user_confirms():
    for change in changes:
        tapd_update_story(...)

# ❌ 错误
for change in changes:
    tapd_update_story(...)  # 直接执行
```

### 3. 增量更新不覆盖
```python
# ✅ 正确
tapd_update_story(
    story_id="123",
    iteration_id="迭代5"  # 只更新 iteration_id
)

# ❌ 错误
tapd_update_story(
    story_id="123",
    description="新的描述"  # 覆盖用户原始内容
)
```

### 4. 重要变更记录评论
```python
# ✅ 正确
tapd_update_story(story_id="123", status="done")
tapd_create_comment(
    entry_id="123",
    description="需求已完成，已通过测试验收"
)

# ❌ 错误
tapd_update_story(story_id="123", status="done")  # 无记录
```

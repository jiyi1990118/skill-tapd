# TAPD 工作流最佳实践

## 一、迭代规划流程

```
获取开放迭代 → 统计未分配需求 → 拉取需求列表 → 优先级排序 → 容量分析 → 生成建议 → 用户确认 → 批量分配
```

**Step 1：获取开放迭代**
```python
iterations = tapd_list_iterations(workspace_id="123456", status="open")
target_iteration = iterations[0]  # 通常选最近的开放迭代
```

**Step 2：统计未分配需求**
```python
count = tapd_count_stories(workspace_id="123456", iteration_id="<>")
```

**Step 3：拉取需求列表**
```python
stories = tapd_list_stories(workspace_id="123456", iteration_id="<>", limit=50)
# 按 priority 排序：urgent → high → medium → low
```

**Step 4：容量分析**
```
团队可用人天 = 成员数 × 迭代天数 × 效率系数
效率系数：0.6-0.7（新团队）、0.7-0.8（成熟团队）、0.8-0.9（高效团队）
```

**Step 5：生成规划建议**
```markdown
## 迭代规划建议 - Sprint 5
**容量**: 37.5 人天 | **待分配**: 15 个需求

### 建议纳入（按优先级）：
1. [Story-123] 用户登录功能 - urgent - 预估: 5天
2. [Story-124] 订单列表 - high - 预估: 3天
...（累计 35 天）

### 不建议纳入：
- [Story-130] 数据统计 - low（延后到 Sprint 6）

**是否确认分配？**
```

**Step 6：批量分配**
```python
story_ids = ",".join([s.id for s in selected_stories])
tapd_batch_update_stories(workspace_id="123456", story_ids=story_ids, iteration_id="迭代5")

# 如需锁定迭代
tapd_lock_iteration(workspace_id="123456", iteration_id="迭代5")
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

**Step 5：批量更新（使用批量更新工具）**
```python
# ✅ 推荐：使用批量更新（更高效）
# 按建议分组，相同建议的bug一起更新
for recommendation, bugs in grouped_by_recommendation.items():
    bug_ids = ",".join([b.id for b in bugs])
    tapd_batch_update_bugs(
        workspace_id="123456",
        bug_ids=bug_ids,
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
# 昨日完成的需求（Story 状态：done=已完成）
completed_stories = tapd_list_stories(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="done"
)

# 昨日完成的任务（Task 状态：done=已完成）
completed_tasks = tapd_list_tasks(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="done"
)

# 昨日解决的 Bug（Bug 状态：resolved=已解决）
resolved_bugs = tapd_list_bugs(
    workspace_id="123456",
    modified=f"{yesterday_start}~{yesterday_end}",
    status="resolved"
)
```

**Step 3：查询今日计划**
```python
# 今日开发中的需求（Story 状态：developing=开发中）
# ⚠️ 注意：需求状态用 developing，不是 in_progress
in_progress_stories = tapd_list_stories(
    workspace_id="123456",
    status="developing"
)

# 今日进行中的任务（Task 状态：in_progress=进行中）
in_progress_tasks = tapd_list_tasks(
    workspace_id="123456",
    status="in_progress"
)

# 今日处理中的缺陷（Bug 状态：in_progress=处理中）
in_progress_bugs = tapd_list_bugs(
    workspace_id="123456",
    status="in_progress"
)
```

**Step 4：识别阻塞项**
```python
# 长时间无更新的高优先级需求（Story 状态：developing=开发中）
blocked_stories = tapd_list_stories(
    workspace_id="123456",
    priority="urgent|high",
    status="developing",
    modified=f"<{yesterday_start}"  # 超过 1 天未更新
)

# 长时间无更新的高优先级任务（Task 状态：in_progress=进行中）
blocked_tasks = tapd_list_tasks(
    workspace_id="123456",
    priority="urgent|high",
    status="in_progress",
    modified=f"<{yesterday_start}"
)

# 长时间未处理的新缺陷（Bug 状态：new=新建）
blocked_bugs = tapd_list_bugs(
    workspace_id="123456",
    priority="urgent|high",
    status="new",
    modified=f"<{yesterday_start}"
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
    entry_type="stories",  # 注意：使用复数形式
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

## 七、异常处理与边界情况

### 常见 API 错误及处理

**1. 401/403 权限错误**
```python
try:
    data = tapd_get_story(workspace_id="123", story_id="456")
except PermissionError as e:
    # 输出友好提示
    return """
    ❌ 权限不足

    当前应用缺少访问该项目的权限。
    请前往 TAPD 开放平台配置应用权限：
    https://open.tapd.cn/admin/4002/permission

    需要权限：需求读取、需求写入（如需修改）
    """
```

**2. 需求/迭代不存在（404）**
```python
try:
    story = tapd_get_story(workspace_id="123", story_id="999")
except NotFoundError:
    # 建议用户检查 ID
    return "❌ 未找到该需求，请检查需求 ID 是否正确。"
```

**3. 网络超时/连接失败**
```python
try:
    data = tapd_list_stories(workspace_id="123", limit=50)
except TimeoutError:
    # 重试一次
    data = tapd_list_stories(workspace_id="123", limit=50)
except ConnectionError:
    return "❌ 无法连接到 TAPD API，请检查网络或稍后重试。"
```

**4. 状态值无效（400）**
```python
try:
    tapd_update_story(workspace_id="123", story_id="456", status="invalid_status")
except BadRequestError as e:
    # 提示用户确认状态值
    return """
    ❌ 状态值无效

    错误信息：{e.info}

    可能原因：
    1. 该 workspace 没有配置此状态
    2. 状态值拼写错误

    建议：通过 `tapd_get_story` 查询一个处于目标状态的参考需求，
    确认其 `status` 字段的准确值。
    """
```

### 模糊匹配失败处理

**需求查找失败**
```python
stories = search_stories_by_name(workspace_id, hint="用户登录")
if len(stories) == 0:
    # 没找到
    return "❌ 未找到匹配的需求。请尝试：\n1. 使用更精确的关键词\n2. 提供需求 ID\n3. 提供需求 URL"
elif len(stories) > 1:
    # 找到多个，让用户选择
    return format_candidate_list(stories)
```

**迭代匹配失败**
```python
iteration = fuzzy_match_iteration(iterations, hint="07/22")
if not iteration:
    # 列出所有迭代让用户选择
    return format_iteration_list(iterations)
```

### 批量操作异常处理

```python
results = []
errors = []

for story_id in story_ids:
    try:
        tapd_update_story(workspace_id, story_id, status=target)
        results.append(f"✅ {story_id}: 更新成功")
    except Exception as e:
        errors.append(f"❌ {story_id}: {e.message}")

# 汇总输出
return f"""
更新结果：
成功：{len(results)} 个
失败：{len(errors)} 个

失败详情：
{chr(10).join(errors)}
"""
```

### 状态流转边界情况

**1. 未知状态值**
- 当前状态不在预定义的顺序列表中
- 处理：输出警告，跳过该需求，记录日志

**2. 目标状态与当前状态相同**
- 处理：跳过，输出 "已是目标状态，无需变更"

**3. 目标状态在当前状态之后（禁止回退）**
- 处理：跳过，输出说明原因

**4. 需求已被删除（status='deleted'）**
- 处理：跳过已删除需求，不执行任何操作

---

## 八、关键原则总结

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

---

## 九、迭代需求清单智能提取

### 一句话指令 → 多步骤自动执行

**核心思想**：用户只需说"帮我获取 07/22 迭代的需求清单"，AI 自动完成：迭代匹配 → 需求拉取 → 表格展示。

### 标准流程

```
提取迭代标识 → 列出所有迭代 → 模糊匹配 → 拉取需求 → 表格展示
```

### 详细步骤

**Step 1：提取用户提到的迭代标识**
```python
def extract_iteration_hint(user_input):
    """从用户输入中提取迭代标识"""
    # 日期模式：07/22、7.22、2026-07-22
    if re.search(r'(\d{1,2}[/-]\d{1,2})', user_input):
        return re.search(r'(\d{1,2}[/-]\d{1,2})', user_input).group(1)

    # Sprint 模式：Sprint 5、sprint5
    if re.search(r'[Ss]print\s*(\d+)', user_input):
        return f"Sprint {re.search(r'[Ss]print\s*(\d+)', user_input).group(1)}"

    # 时间词模式：本周、这周、当前、最近
    if any(word in user_input for word in ['本周', '这周', '当前', '这个']):
        return "current"

    # 月份模式：7月份、七月
    if re.search(r'(\d{1,2})月', user_input):
        return f"2026-{re.search(r'(\d{1,2})月', user_input).group(1).zfill(2)}"

    # 默认：取全部文本作为 hint
    return user_input.replace('迭代', '').replace('需求', '').strip()
```

**Step 2：获取迭代列表**
```python
iterations = tapd_list_iterations(
    workspace_id=workspace_id,
    limit=50  # 获取足够多的迭代用于匹配
)
```

**Step 3：模糊匹配目标迭代**
```python
def fuzzy_match_iteration(iterations, hint):
    """按优先级依次匹配"""

    # 规则1: 名称包含 hint（不区分大小写）
    for it in iterations:
        if hint.lower() in it.name.lower():
            return it

    # 规则2: 日期匹配（如 "07/22" 匹配 startdate/enddate 含 "07-22"）
    for it in iterations:
        hint_normalized = hint.replace('/', '-')
        if (hint_normalized in it.startdate or
            hint_normalized in it.enddate):
            return it

    # 规则3: 数字匹配（如 "5" 匹配 "Sprint 5"、"迭代5"）
    hint_num = re.search(r'\d+', hint)
    if hint_num:
        for it in iterations:
            it_num = re.search(r'\d+', it.name)
            if it_num and hint_num.group() == it_num.group():
                return it

    # 规则4: 当前周匹配
    if hint == "current":
        today = datetime.now().strftime('%Y-%m-%d')
        for it in iterations:
            if it.status == "open" and it.startdate <= today <= it.enddate:
                return it
        # 没有当前周的，取最近的 open 迭代
        for it in sorted(iterations, key=lambda x: x.startdate, reverse=True):
            if it.status == "open":
                return it

    # 规则5: 月份匹配
    if re.match(r'\d{4}-\d{2}', hint):
        for it in iterations:
            if hint in it.startdate or hint in it.enddate:
                return it

    # 规则6: 默认取最近的一个 open 迭代
    for it in sorted(iterations, key=lambda x: x.startdate, reverse=True):
        if it.status == "open":
            return it

    # 兜底：取最近的一个（无论状态）
    return sorted(iterations, key=lambda x: x.startdate, reverse=True)[0] if iterations else None
```

**Step 4：拉取该迭代的需求列表**
```python
stories = tapd_list_stories(
    workspace_id=workspace_id,
    iteration_id=target_iteration.id,
    limit=200,  # 尽量拉取全部
    fields="id,name,status,owner,priority,creator,created,modified,begin,due,size"
)

# 统计
stats = {
    'total': len(stories),
    'developing': sum(1 for s in stories if s.status == 'developing'),
    'done': sum(1 for s in stories if s.status == 'done'),
    'planned': sum(1 for s in stories if s.status in ['planned', 'planning']),
    'urgent_high': sum(1 for s in stories if s.priority in ['urgent', 'high']),
}
```

**Step 5：表格展示**
```markdown
## 📋 迭代需求清单 - {iteration.name}

**迭代信息**：
- ID: {iteration.id}
- 名称: {iteration.name}
- 状态: {iteration.status}
- 时间范围: {iteration.startdate} ~ {iteration.enddate}
- 需求总数: {stats['total']}

| 序号 | 需求ID | 需求名称 | 状态 | 优先级 | 负责人 | 规模 | 创建人 | 创建时间 |
|------|--------|----------|------|--------|--------|------|--------|----------|
| 1 | 11488... | 用户登录优化 | developing | high | 张三 | 3 | 李四 | 2026-06-01 |
| 2 | 11488... | 支付对接 | planned | urgent | 王五 | 5 | 李四 | 2026-06-03 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

### 统计概览
- **总数**: {stats['total']} 个
- **开发中**: {stats['developing']} 个
- **已完成**: {stats['done']} 个
- **未开始**: {stats['planned']} 个
- **高优先级**: {stats['urgent_high']} 个
```

### 用户自然表达对照

| 用户自然表达 | 推断意图 | 匹配策略 |
|---|---|---|
| "07/22 迭代" | 名字或日期含 07/22 的迭代 | 名称包含 + 日期包含 |
| "本周迭代" | 当前正在进行的迭代 | 状态=open + 日期包含今天 |
| "Sprint 5" | 第 5 个 Sprint | 名称包含 "Sprint 5" |
| "7月份" | 7 月份开始的迭代 | startdate 含 "2026-07" |
| "最近的迭代" | 最近的开放迭代 | 按 startdate 排序取第一个 open |
| "这个迭代" | 当前/唯一的开放迭代 | 取唯一的或最近的 open |
| "上周的迭代" | 上周的迭代 | 日期匹配上周范围 |
| "下一个迭代" | 即将开始的迭代 | 找 startdate > 今天的第一个 |

### 如果没找到匹配的迭代

列出所有迭代让用户选择：
```markdown
未找到与 "{hint}" 匹配的迭代。请从以下迭代中选择：

| 迭代名称 | 时间范围 | 状态 |
|----------|----------|------|
| Sprint 5 (07/22-08/04) | 2026-07-22 ~ 2026-08-04 | open |
| Sprint 4 (07/08-07/21) | 2026-07-08 ~ 2026-07-21 | done |
| Sprint 6 (08/05-08/18) | 2026-08-05 ~ 2026-08-18 | open |

请输入您想要的迭代名称：
```

---

## 十、状态流转工作流

### 核心原则

**原则 1**：只能向前流转，禁止向后回退。

**原则 2（默认前置）**：状态变更前，需求**必须**有明确的开发人员，否则忽略。

**判断逻辑**：
- 目标状态位置 ≤ 当前位置 → 跳过
- 目标状态位置 > 当前位置 → 进入下一步
- **`has_developer(description) == False` → ⚠️ 忽略，不进入确认清单**

**Workspace 48801209 状态顺序**：
```python
STATUS_ORDER = ['status_1', 'planned', 'tech_review', 'developing', 'testing', 'resolved', 'done', 'closed']
STATUS_MAP = {'新建':'status_1', '需求排期':'planned', '技术评估':'tech_review', '开发中':'developing',
              '测试中':'testing', '已解决':'resolved', '已完成':'done', '已关闭':'closed'}
```

### 场景 A：单个需求流转

**触发**："流转 [需求] 为 [状态]"

**流程**：
1. 提取需求标识（标题/ID/URL）→ `tapd_get_story` 获取需求详情（含 description）
2. 提取目标状态 → 映射为 API 值
3. **调用 `has_developer(description)` 判断是否有开发人员**
   - 无开发人员 → ⚠️ 提示"该需求无开发人员标注，按规则忽略"，**不执行**
   - 有开发人员 → 继续
4. 流转判断：目标位置 > 当前位置？
   - 目标 ≤ 当前位置 → 提示"禁止回退"，跳过
5. 展示确认 → 执行 `tapd_update_story(status=target)` → 记录评论

### 场景 B：批量迭代需求流转

**触发**："批量修改 [迭代] [状态] 为 [目标状态]"

**流程**：
1. 模糊匹配迭代 → `list_stories(iteration_id, status=filter, fields="...,description")`
2. **逐个调用 `has_developer(description)` 过滤无开发人员的需求**
3. 剩余需求逐个判断流转方向
4. 展示三类清单：✅ 将流转的 / ⚠️ 忽略(无开发人) / ❌ 将跳过的(方向不对)
5. 确认 → `batch_update_stories(story_ids, status=target)` → 记录评论

### 场景 C：批量流转 + 字段更新（规模 + 负责人）

**触发**："[迭代] [状态]→[目标]，规模=开发人数，处理人=开发人名，无开发人忽略"

**流程**：
1. 匹配迭代 → 拉取指定状态的需求（含 description）
2. 逐个处理：
   - 流转判断 → 向后/同级则跳过
   - **`has_developer(description)` 校验 → 无开发人 → ⚠️ 忽略**
   - 有开发人 → size=人数，owner=第一人
3. 展示三类清单：✅ 更新 / ⚠️ 忽略(无开发人) / ❌ 跳过(方向不对)
4. 确认 → 逐个 `update_story(status, size, owner)`（字段值不同，无法 batch）→ 记录评论

### 开发人员判断工具函数

```python
import re

DEV_PATTERNS = [
    r'开发人员[：:]\s*([一-龥\w\s、,，/|]+?)(?=\n|;|；|$|（|\(|\s{2,})',
    r'负责人[：:]\s*([一-龥\w\s、,，/|]+?)(?=\n|;|；|$|（|\(|\s{2,})',
    r'developer[s]?[：:]\s*([\w\s,，/|]+?)(?=\n|;|;|$|\(|\s{2,})',
    r'@([一-龥]{2,4})',
]

def has_developer(description: str) -> tuple[bool, list[str]]:
    """
    判断需求是否有开发人员。
    从 description 文本中匹配 DEV_PATTERNS，命中即视为有开发人员。

    返回: (是否有开发人员, 提取到的开发人列表)
    """
    devs = set()
    for pat in DEV_PATTERNS:
        for m in re.finditer(pat, description or ''):
            names = re.split(r'[、,，/|;\s]+', m.group(1).strip())
            devs.update(n.strip() for n in names if n.strip() and len(n.strip()) >= 2)
    return (len(devs) > 0, list(devs))


def filter_with_developer(stories: list) -> tuple[list, list]:
    """
    批量过滤：返回 (有开发人员的需求, 无开发人员的需求)
    """
    with_dev, without_dev = [], []
    for s in stories:
        has, _ = has_developer(s.get('description', ''))
        (with_dev if has else without_dev).append(s)
    return with_dev, without_dev
```

**重要提示**：
- ❌ **不要用 `owner` 字段判断**——owner 是历史处理人，可能是已离职/转岗人员
- ✅ **必须用 description 中的 `开发人员`/`负责人`/`@名字` 标注**
- ⚠️ `@对接人`（如 @吴栋、@王登林）会被识别为开发人员，若非实际开发人需人工事后复核
- 命中文本中提到的任何 `@名字` 都视为有开发人员，可能误判，但比漏判更安全

### 流转场景速查

| 当前 | 目标 | 是否流转 |
|---|---|---|
| 技术评估 | 需求排期 | ❌ 不流转（回退） |
| 需求排期 | 开发中 | ✅ 流转 |
| 开发中 | 开发中 | ❌ 不流转（同级） |
| 测试中 | 已完成 | ✅ 流转 |
| 已完成 | 开发中 | ❌ 不流转（回退） |

---

## 十一、需求搜索与过滤工作流

**触发**："搜索需求'用户登录'" / "查找张三负责的需求" / "高优先级未完成的需求"

**流程**：
1. 解析条件：关键词→`name=LIKE<关键词>`，负责人→`owner=`，优先级→`priority=`，状态→`status=`，时间→`created=>日期`
2. `count_stories` → `list_stories(limit=min(count,200))`
3. 表格展示

---

## 十二、评论管理工作流

**查看评论**：`list_comments(entry_type="stories", entry_id=ID)` → 列表展示
**添加评论**：`create_comment(entry_type="stories", entry_id=ID, description=文本)`

⚠️ **entry_type 用复数**：`stories` / `bug` / `tasks`

---

## 十三、成员查找工作流

**触发**："项目里有哪些成员" / "搜索名字包含'张'的成员"

**流程**：`list_users` → 按名称/昵称过滤 → 表格展示

**推荐负责人**（创建需求时未指定）：
```python
# 查询该模块下最近处理的需求，统计最常出现的负责人
recent = list_stories(module=module, limit=10)
owner = max(Counter(s.owner for s in recent).items(), key=lambda x: x[1])[0]
```

# TAPD 状态映射参考

> 不同 workspace 的状态映射可能存在差异，以下基于实际验证。

---

## Workspace 48801209 需求状态映射

基于实际 API 返回值验证：

| 中文显示 | API 状态值 | 验证依据 |
|----------|-----------|---------|
| **需求排期** | `planned` | 需求 [增加饼底价格展示](https://www.tapd.cn/tapd_fe/48801209/story/detail/1148801209001013393) 页面显示"需求排期"，API 返回 `status: "planned"` |
| 初始/新建 | `status_1` | 需求 [策略联盟-CRM异业合作增加撤回账号授权功能](https://www.tapd.cn/tapd_fe/48801209/story/detail/1148801209001018123) 初始状态为 `status_1` |
| **开发中** | `developing` | 需求 [【安全需求】关于"个人信息保护政策"的有效配置](https://www.tapd.cn/tapd_fe/48801209/story/detail/1148801209001013452) 页面显示"开发中"，API 返回 `status: "developing"` |

### 使用规范

当用户要求将需求状态改为 "需求排期" 时：

```python
# ✅ 正确
tapd_update_story(
    workspace_id="48801209",
    story_id="1148801209001018123",
    status="planned"   # ← 需求排期
)

# ❌ 错误（勿用）
tapd_update_story(
    workspace_id="48801209",
    story_id="1148801209001018123",
    status="status_2"  # ← 这不是"需求排期"！
)
```

### 如何确认其他状态值

1. 找到目标 workspace 中处于目标状态的需求
2. 使用 `tapd_get_story` 获取其 `status` 字段值
3. 记录到本文件

---

---

## 已验证状态速查表（Workspace 48801209）

| 中文状态 | API 值 | 验证需求 |
|---|---|---|
| 需求排期 | `planned` | 增加饼底价格展示 |
| 开发中 | `developing` | 【安全需求】关于"个人信息保护政策"的有效配置 |
| 初始/新建 | `status_1` | 策略联盟-CRM异业合作增加撤回账号授权功能 |

---

**版本**：v2.2.0  
**最后更新**：2026-06-16

## 通用建议

- **状态值因 workspace 而异**：不同项目模板的自定义状态值不同
- **先查后设**：修改状态前，先通过 `tapd_get_story` 确认参考需求的状态值
- **避免臆造**：不要猜测状态值（如 `status_2`、`status_3` 等），必须通过实际查询验证

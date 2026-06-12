# Bug 报告模板

```markdown
# Bug 报告 - [Bug ID]

## Bug 描述
[简洁描述问题]

## 复现步骤
1. 打开页面 [URL]
2. 输入 [数据]
3. 点击 [按钮]
4. 观察结果

## 预期结果
[应该发生什么]

## 实际结果
[实际发生了什么]

## 环境信息
- **平台**：iOS / Android / Web
- **浏览器**：Chrome 120 / Safari 17
- **版本**：v1.2.0
- **设备**：iPhone 15 / Pixel 8

## 截图
[如有截图，执行 Vision Parser 解析]

## 严重级别分析
- **影响范围**：所有用户 / 部分用户 / 特定场景
- **影响功能**：核心功能 / 非核心功能
- **建议 Severity**：fatal / serious / normal / slight

## 优先级分析
- **业务影响**：高 / 中 / 低
- **用户反馈频率**：高 / 中 / 低
- **建议 Priority**：urgent / high / medium / low

## 处理人建议
- **Module**：[模块名称]
- **建议 Owner**：[负责人]
```

## MCP 创建示例

```python
tapd_create_bug(
  workspace_id="123456",
  title="支付失败 - 订单状态未更新",
  description="""
## 复现步骤
1. 选择商品加入购物车
2. 进入结算页，选择支付方式"微信支付"
3. 点击"立即支付"，支付成功

## 预期结果
订单状态变为"已支付"，库存扣减

## 实际结果
订单状态仍为"待支付"，库存未扣减

## 环境信息
- 平台：iOS 15.0
- 浏览器：Safari
- 版本：v1.2.0
- 时间：2026-06-12 10:30:00

## 影响分析
- 影响范围：所有使用微信支付的用户
- 用户反馈：今日已有 3 个用户反馈
  """,
  severity="serious",
  priority="high",
  current_owner="李四",
  module="支付模块",
  platform="iOS",
  source="用户反馈"
)
```

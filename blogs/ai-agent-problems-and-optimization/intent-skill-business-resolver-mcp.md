---
title: "Intent、Skill、Business Resolver、Capability Router 和 MCP 到底是什么关系"
date: 2026-07-14
tags: [AI Agent, Intent, Skill, MCP, Business Resolver, Capability Router, 工程实践]
---

# Intent、Skill、Business Resolver、Capability Router 和 MCP 到底是什么关系

> 一句话导读：Intent、Skill、Business Resolver、Capability Router、MCP 不是同一层东西。Intent 判断用户要什么，Business Resolver 查询业务事实，Capability Router 决定用哪些能力，Skill 承载业务能力知识，MCP 负责连接外部系统。

在设计垂类 Agent 时，很容易把这些概念混在一起：

```text
Intent 是不是 Skill？
Business Resolver 是不是 MCP？
Capability Router 是不是 MCP？
Skill 是不是 Prompt？
```

这些概念确实有关联，但不能直接画等号。

如果把它们混在一起，系统很容易变成：

```text
一个大 prompt
+ 一堆 tools
+ 让 LLM 自己决定一切
```

这在开放式 Agent 里还能跑，但在企业垂类 Agent 里会越来越难维护。

更好的方式是先分清每一层负责什么。

---

## 一、先给结论

这几个概念的关系可以先记成这样：

```text
Intent
= 用户想做什么

Business Resolver
= 当前业务事实是什么

Capability Router
= 基于意图和业务状态，接下来需要哪些能力

Skill
= 完成某类能力所需的知识、规则、约束和流程

MCP
= 连接外部系统和工具的协议
```

所以它们不是同一个维度。

更完整的链路是：

```text
用户输入
  -> Intent 识别
  -> Business Resolver 查询业务状态
  -> Capability Router 选择能力
  -> Skill 加载业务知识和规则
  -> MCP Tool 调用外部系统
  -> LLM 组织自然语言回复
```

---

## 二、Intent：用户到底想干什么

Intent 是意图。

它回答的是：

```text
用户要做什么？
```

比如用户说：

```text
我的退款怎么还没到账？
```

Intent 识别的结果可能是：

```text
refund_status_query
```

用户说：

```text
我昨天打车能报销吗？
```

Intent 可能是：

```text
reimbursement_policy_check
```

用户说：

```text
公司今年还有多少年假？
```

Intent 可能是：

```text
annual_leave_balance_query
```

所以 Intent 更像一个分类结果，或者任务标签。

它不是执行能力本身。

---

## 三、Intent 不是 Skill

Intent 和 Skill 很容易被混在一起，因为很多时候一个意图会对应一个能力。

比如：

```text
Intent = refund_status_query
Skill = 退款查询 Skill
```

但它们不是同一个东西。

Intent 是：

```text
用户要查退款。
```

Skill 是：

```text
退款查询这件事应该怎么做。
```

一个 Skill 里可能包含：

```text
退款状态有哪些
每种退款状态如何解释
需要查询哪些系统
什么情况下要转人工
什么情况下要提示预计到账时间
什么情况下不能承诺结果
最终回复口径是什么
```

所以更准确的关系是：

```text
Intent -> 选择一个或多个 Skill
```

不是：

```text
Intent = Skill
```

举个例子，用户说：

```text
我的退款怎么还没到账，能不能帮我催一下？
```

这句话可能同时命中两个 Intent：

```text
refund_status_query
refund_urge_request
```

系统可能需要加载多个 Skill：

```text
退款查询 Skill
到账时间解释 Skill
催办工单 Skill
```

所以 Intent 是入口判断，Skill 是能力模块。

---

## 四、Skill 也不是 Prompt 的别名

很多人会把 Skill 理解成：

```text
Skill = 一段 Prompt
```

这个理解太窄了。

在企业 Agent 里，Skill 更应该是一个能力包。

它可以包含：

```text
业务规则
领域知识
工具说明
参数 Schema
输出格式
安全约束
权限要求
失败兜底
示例
小脚本
静态资源
```

比如一个“退款查询 Skill”可以长这样：

```text
refund-status-skill/
  skill.md
  policy.md
  response-style.md
  tool-schema.json
  examples.json
```

其中 `skill.md` 可能告诉模型：

```text
退款处理中不能承诺具体到账时间。
银行卡退款通常 1-3 个工作日。
超过预计时间才允许建议创建工单。
用户没有订单号时，先通过用户身份查询最近订单。
```

这里 Prompt 只是 Skill 的一部分。

真正的 Skill 是：

```text
完成某类业务能力所需的知识和约束集合。
```

---

## 五、Business Resolver：查询业务事实

Business Resolver 负责查询确定性的业务事实。

它回答的是：

```text
现在的业务状态是什么？
```

还是退款例子。

用户说：

```text
我的退款怎么还没到账？
```

Business Resolver 要查：

```text
用户是谁
用户是否登录
订单号是多少
订单是否支付
退款是否发起
退款当前状态
退款渠道是什么
预计到账时间是什么
是否已经超时
是否已有工单
```

它的输出应该是结构化状态：

```json
{
  "userId": "u123",
  "orderId": "o456",
  "paid": true,
  "refundStatus": "processing",
  "refundChannel": "bank_card",
  "estimatedArrivalDate": "2026-07-16",
  "isOverdue": false,
  "hasExistingTicket": false
}
```

这个状态不应该由 LLM 猜。

它应该由程序查数据库、查业务系统、调接口拿到。

所以 Business Resolver 是垂类 Agent 的核心层之一。

---

## 六、Business Resolver 不是 MCP

Business Resolver 经常会调用 MCP 工具，所以它容易被误解成 MCP。

但它们不是一回事。

MCP 是工具接入协议。

Business Resolver 是业务状态解析逻辑。

关系更像：

```text
Business Resolver 使用 MCP 去查业务系统。
```

比如：

```text
Business Resolver
  -> 调 order MCP 查询订单
  -> 调 payment MCP 查询支付
  -> 调 refund MCP 查询退款
  -> 调 CRM MCP 查询用户
```

MCP 只负责让外部系统变成可调用工具。

它不天然知道：

```text
退款处理中是什么意思
银行卡退款多久能到账
什么情况下应该创建工单
什么情况下应该转人工
```

这些是业务逻辑。

应该放在 Business Resolver、Policy Engine 或 Skill 里。

所以不要把 MCP 当业务层。

它更像插座。

Business Resolver 是拿着插头去查事实的人。

---

## 七、Capability Router：决定用哪些能力

Capability Router 负责根据 Intent 和 Business State 决定下一步用哪些能力。

它回答的是：

```text
基于当前情况，接下来应该调用哪些能力？
```

比如：

```text
Intent = refund_status_query
Business State = 退款处理中，银行卡渠道，未超时
```

Capability Router 可以决定：

```text
使用 refund_status_explain
使用 bank_refund_arrival_time_explain
不创建工单
不转人工
```

如果状态变成：

```text
退款已发起
银行卡渠道
已经超过预计到账时间
没有已有工单
```

Capability Router 可以决定：

```text
使用 refund_status_explain
使用 overdue_refund_explain
使用 create_support_ticket
```

它不是简单地按 Intent 找工具。

它要结合业务状态。

同样是“退款没到账”，不同状态下能力选择不同：

```text
未发起退款：解释如何申请退款
退款处理中：解释预计到账时间
退款失败：解释失败原因并引导重新申请
退款超时：创建工单或转人工
退款已到账：告知到账渠道和时间
```

这就是 Capability Router 的价值。

---

## 八、Capability Router 也不是 MCP

Capability Router 可以选择 MCP Tool，但它本身不是 MCP。

比如它可能决定：

```text
需要调用 refund.getStatus
需要调用 ticket.create
需要加载 refund-policy-skill
需要让 LLM 生成最终回复
```

其中 `refund.getStatus` 和 `ticket.create` 可能来自 MCP。

但“什么时候调用哪个能力”是路由逻辑。

MCP 只是能力入口。

Capability Router 是能力编排。

关系是：

```text
Capability Router -> 选择 Capability / Skill / MCP Tool
```

不是：

```text
Capability Router = MCP
```

---

## 九、MCP：外部系统接入协议

MCP 可以理解成 Agent 世界里的标准工具协议。

它回答的是：

```text
Agent 怎么调用外部系统？
```

比如一个企业 Agent 可以接：

```text
订单 MCP
支付 MCP
退款 MCP
CRM MCP
HR MCP
ERP MCP
知识库 MCP
工单 MCP
```

每个 MCP Server 暴露一组工具：

```text
order.getById
order.listRecent
payment.getStatus
refund.getStatus
ticket.create
hr.getAnnualLeaveBalance
```

MCP 解决的是标准化接入问题。

它不负责完整业务流程。

如果把所有 MCP 工具直接丢给 LLM 自由调用，垂类 Agent 很容易变得不可控：

```text
工具太多
模型乱选
业务状态不完整
规则判断不稳定
权限边界不清晰
排查困难
```

所以垂类 Agent 里更推荐：

```text
业务流程层决定什么时候调用 MCP。
LLM 不直接控制关键业务工具。
```

---

## 十、完整例子：退款没到账

用户输入：

```text
我的退款怎么还没到账？
```

完整链路可以是：

```text
1. Intent
   -> refund_status_query

2. Business Resolver
   -> 查用户身份
   -> 查最近退款订单
   -> 查退款状态
   -> 查退款渠道
   -> 查预计到账时间

3. Capability Router
   -> 判断退款处理中
   -> 判断未超时
   -> 选择退款状态解释能力
   -> 选择银行卡到账时间解释能力

4. Skill
   -> 加载退款查询规则
   -> 加载银行卡退款说明
   -> 加载回复约束

5. MCP Tool
   -> order.getRecent
   -> refund.getStatus
   -> payment.getRefundChannel

6. LLM
   -> 根据业务事实和 Skill 约束生成回复
```

最终回复可能是：

```text
你的订单退款已经发起，目前处于银行处理中。银行卡退款通常需要 1-3 个工作日，预计会在 7 月 16 日前到账。当前还没有超过预计时间，建议你先等待；如果超过预计时间仍未到账，我可以继续帮你创建工单。
```

这里最重要的是：

```text
退款状态不是 LLM 猜的。
到账规则不是 LLM 编的。
是否建工单不是 LLM 自由决定的。
LLM 只是把确定事实和规则组织成人话。
```

---

## 十一、和开放式 Agent 的区别

开放式 Agent，比如 Coding Agent，通常是：

```text
LLM 决定下一步
  -> Read
  -> Grep
  -> Bash
  -> Edit
  -> Test
  -> 再决定下一步
```

它面对的是未知任务，所以 LLM Planner 很重要。

但垂类 Agent 面对的是确定业务流程。

所以控制权应该从：

```text
LLM Planner
```

转向：

```text
Business State + Workflow + Capability Router
```

这就是两类 Agent 架构的根本差异。

---

## 十二、总结

最后用一张表收束：

| 概念 | 回答的问题 | 更像什么 |
| --- | --- | --- |
| Intent | 用户要做什么 | 意图分类 / 任务识别 |
| Skill | 这类能力怎么做 | 能力知识包 / 业务规则包 |
| Business Resolver | 当前业务事实是什么 | 业务状态解析层 |
| Capability Router | 接下来用哪些能力 | 能力选择和编排层 |
| MCP | 怎么连接外部系统 | 工具接入协议 |

最重要的关系是：

```text
Intent -> 选择方向
Business Resolver -> 查事实
Capability Router -> 编排能力
Skill -> 提供知识和约束
MCP -> 连接外部系统
LLM -> 理解输入和组织输出
```

所以不要把它们压扁成：

```text
Intent = Skill
Business Resolver = MCP
Capability Router = MCP
```

更好的理解是：

```text
Intent 决定用户要什么。
Business Resolver 查现在是什么状态。
Capability Router 决定该用什么能力。
Skill 提供这类能力怎么做。
MCP 负责怎么连到外部系统。
```

垂类 Agent 想稳定，关键不是让 LLM 的 Planner 越来越复杂，而是让业务状态、能力编排、工具接入和自然语言生成各自归位。

---
title: "开放式 Agent 和垂类 Agent，为什么不应该采用同一套架构"
date: 2026-07-14
tags: [AI Agent, 垂类 Agent, 开放式 Agent, LangGraph, Claude Code, MCP, Skills, 架构设计]
---

# 开放式 Agent 和垂类 Agent，为什么不应该采用同一套架构

> 一句话导读：开放式 Agent 的核心是探索未知，流程由 LLM 控制；垂类 Agent 的核心是执行业务流程，流程应该由业务状态和规则控制。

很多团队做企业 Agent 时，会自然参考 Claude Code、Cursor、Devin 这类产品：

```text
LLM 负责规划
LLM 选择工具
LLM 观察结果
LLM 决定下一步
```

这套模式在 Coding Agent 里很合理。

但如果照搬到电商客服、HR、报销、售后、金融、企业知识库这类垂直业务场景，系统往往会越做越复杂：

```text
工具越来越多
Prompt 越来越长
模型越来越难控
业务规则越来越散
排查问题越来越痛苦
```

问题不一定出在框架本身，而是两类 Agent 解决的问题根本不同。

---

## 一、开放式 Agent：探索未知

Claude Code、Cursor、Devin、通用研究 Agent 都属于开放式 Agent。

比如用户说：

```text
帮我定位这个项目为什么启动失败。
```

Agent 一开始并不知道下一步应该做什么。

它可能需要：

```text
读 README
看 package.json
查配置文件
搜索日志
运行命令
定位依赖版本
修改代码
重新运行测试
```

每一步都依赖上一步的观察结果。

所以开放式 Agent 的核心能力是：

```text
探索
规划
试错
迭代
```

典型架构是：

```text
User Goal
  -> LLM Planner
  -> Tool Use
  -> Observation
  -> Re-plan
  -> Tool Use
  -> Final Answer
```

这就是 ReAct / Tool Loop 的价值。

在这种场景里，让 LLM 控制下一步是合理的，因为任务本身就是开放的。

---

## 二、垂类 Agent：执行确定业务流程

企业里的垂类 Agent 不一样。

比如电商客服里，用户问：

```text
我的订单什么时候发货？
```

这不是一个开放探索问题。

真正需要做的是：

```text
确认用户身份
查询订单
查询支付状态
查询库存状态
查询出库状态
查询物流状态
根据业务规则回复
```

再比如报销 Agent：

```text
我昨天打车能报销吗？
```

真正需要的是：

```text
查询员工身份
查询部门规则
查询报销制度
查询订单信息
查询预算状态
根据规则判断
```

这些流程大多数是确定的。

模型真正适合做的是：

```text
理解用户自然语言
补全上下文
组织最终回复
解释复杂规则
处理开放式追问
```

而不是：

```text
自己决定要不要查权限
自己决定要不要查订单
自己决定能不能退款
自己决定能不能报销
```

垂类 Agent 的核心不是探索，而是：

```text
连接业务系统
查询业务状态
执行业务规则
生成可理解的回复
```

---

## 三、两类 Agent 最大区别：谁控制流程

很多人以为两类 Agent 的区别是：

```text
开放式 Agent 工具多
垂类 Agent 工具少
```

这个判断不够准确。

真正的区别是：

```text
系统流程由谁控制？
```

开放式 Agent：

```text
LLM 控制流程。
LLM 决定下一步要读文件、搜索、执行命令还是修改代码。
```

垂类 Agent：

```text
业务系统控制流程。
业务状态和规则决定下一步该查什么、判断什么、返回什么。
```

这是架构上的根本差异。

---

## 四、为什么企业 Agent 照搬 Planner 会变复杂

在企业业务里，有大量判断是确定性的：

```text
用户是否登录
用户是否有权限
订单是否存在
订单是否支付
退款是否发起
审批是否通过
预算是否充足
合同是否过期
```

这些不应该交给 LLM 推理。

程序查询即可。

如果全部交给 Planner，系统会出现几个问题：

```text
1. Token 消耗变高
2. 调用路径不稳定
3. 业务规则散落在 Prompt 里
4. 权限边界变模糊
5. 线上问题难复现
6. 模型升级可能影响业务流程
```

所以企业 Agent 不是 Planner 越复杂越好。

很多时候，越稳定的系统，反而越少让 LLM 参与关键业务决策。

---

## 五、企业 Agent 应该围绕 Business State 设计

垂类 Agent 里最重要的是 Business State，也就是业务状态。

比如售后场景：

```text
用户：我的退款怎么还没到账？
```

真正决定答案的是：

```text
订单是否支付
退款是否发起
退款当前状态
退款渠道
预计到账时间
是否超时
是否已有工单
```

模型不会创造新的事实。

模型只是把这些确定事实解释给用户。

所以更合理的设计是：

```text
Intent
  -> Business Resolver
  -> Capability Router
  -> Skill / Policy
  -> Tool / MCP
  -> Response Generator
```

而不是：

```text
User
  -> 一个超大 Planner
  -> 一堆 Tools
  -> 让模型自己决定所有事情
```

---

## 六、推荐的垂类 Agent 分层

### 1. Intent

Intent 负责理解用户想干什么。

比如：

```text
refund_status_query
order_shipping_query
annual_leave_query
reimbursement_policy_check
```

它回答的是：

```text
用户要做什么？
```

### 2. Business Resolver

Business Resolver 负责查询业务事实。

比如：

```text
查用户身份
查订单
查支付状态
查退款状态
查物流状态
查权限
查预算
```

它回答的是：

```text
当前业务状态是什么？
```

### 3. Capability Router

Capability Router 负责决定接下来需要哪些能力。

比如同样是退款没到账：

```text
退款未发起 -> 引导申请退款
退款处理中 -> 解释预计到账时间
退款失败 -> 解释失败原因
退款超时 -> 创建工单或转人工
退款已到账 -> 告知到账渠道和时间
```

它回答的是：

```text
基于意图和业务状态，该用哪些能力？
```

### 4. Dynamic Skills

Skill 不是 Prompt 的别名。

它更像能力模块，包含：

```text
业务规则
领域知识
工具说明
参数 Schema
回复口径
安全约束
失败兜底
```

比如：

```text
退款查询 Skill
物流解释 Skill
报销政策 Skill
年假规则 Skill
合同解读 Skill
```

### 5. Response Generator

最后再让 LLM 根据事实和规则组织自然语言回复。

这一层适合 LLM。

因为它擅长：

```text
理解上下文
组织语言
解释复杂规则
处理用户追问
把系统状态讲成人话
```

---

## 七、和 Claude Code 的关系

Claude Code 是典型开放式 Agent。

它面对的是未知代码库、未知错误、未知实现路径。

所以它需要：

```text
QueryEngine
Tool Loop
Read / Grep / Bash / Edit
TodoWrite
Compact
SubAgent
Permission
MCP
Hooks
```

它的主线是：

```text
用户目标
  -> Claude 判断下一步
  -> 调工具
  -> 看结果
  -> 再判断
```

这套模式适合代码任务。

但如果做订单客服、报销审批、HR 查询，不应该直接照搬 Claude Code 的 Planner。

正确做法是：

```text
业务流程控制关键路径
LLM 处理理解和表达
工具调用受业务状态约束
```

---

## 八、和 Claude Agent SDK 的关系

Claude Agent SDK 更像把 Claude Code 的 Agent Harness 产品化。

它适合：

```text
代码分析
自动修 bug
仓库迁移
文件操作
命令执行
开放式研究
```

因为这些任务天然需要探索。

但如果用它做垂类业务 Agent，就需要在外面加业务层：

```text
Intent
Business State
Workflow
Policy
Capability Router
```

Claude Agent SDK 可以作为某些开放问题处理器，或者作为最终表达层的一部分。

但业务事实和关键决策不应该完全交给它自由规划。

---

## 九、和 LangGraph 的关系

这篇观点非常适合用 LangGraph 来落地。

LangGraph 的强项是：

```text
显式状态
显式节点
显式边
可恢复
可中断
可人工介入
```

这很适合垂类 Agent。

比如报销 Agent 可以设计成：

```text
parse_intent
  -> resolve_user
  -> fetch_policy
  -> fetch_order
  -> check_budget
  -> decide_reimbursement
  -> generate_response
```

其中：

```text
parse_intent 可以用 LLM
generate_response 可以用 LLM
权限校验用程序
预算判断用程序
订单查询用工具
流程编排由 Graph 控制
```

所以可以简单记：

```text
开放探索任务：Claude Code / Agent SDK 更自然。
垂类业务任务：LangGraph 这类状态图更自然。
```

---

## 十、和 OpenAI Agents SDK 的关系

OpenAI Agents SDK 里有两个常见思路：

```text
Agent as Tool
Handoff
```

开放式任务里，Agent as Tool 很好用：

```text
主 Agent
  -> 调 Research Agent
  -> 调 Coding Agent
  -> 调 Review Agent
```

垂类业务里，Handoff 更像业务分流：

```text
客服 Agent
  -> 售后 Agent
  -> 物流 Agent
  -> 退款 Agent
```

但注意，转给退款 Agent 后，不代表让模型自由决定退款结果。

退款 Agent 仍然应该：

```text
查订单
查支付
查退款状态
按规则判断
再生成回复
```

Handoff 是角色切换，不是业务规则替代品。

---

## 十一、和 LangChain 的关系

LangChain 早期很多 ReAct Agent 写法是：

```text
LLM 选择工具
工具返回结果
LLM 再选择工具
```

这适合开放式任务。

但企业垂类 Agent 如果也这么做，容易变成：

```text
工具一堆
Prompt 一大段
LLM 自己挑工具
出了错再补 Prompt
```

垂类场景更应该用：

```text
Router
Chain
Structured Output
Retriever
Workflow
Policy
```

不要把所有业务逻辑都塞进一个大 ReAct Agent。

---

## 十二、和 MCP / Skills 的关系

MCP 更像外部系统接入层。

它负责把系统能力暴露给 Agent：

```text
订单 MCP
支付 MCP
退款 MCP
CRM MCP
HR MCP
ERP MCP
知识库 MCP
```

Skill 更像业务能力知识包：

```text
退款查询 Skill
物流查询 Skill
报销判断 Skill
年假解释 Skill
合同审查 Skill
```

垂类 Agent 里更推荐：

```text
Intent
  -> Business State
  -> Capability / Skill
  -> MCP Tool
  -> Response
```

而不是：

```text
LLM Planner
  -> 随便挑 MCP 工具
  -> 自己判断业务结果
```

---

## 十三、现实里往往是混合架构

开放式 Agent 和垂类 Agent 不是非黑即白。

真实系统通常是混合的：

```text
确定流程部分：
查订单、查权限、查状态、按规则判断。

开放探索部分：
解释复杂政策、处理模糊表达、总结材料、处理异常 case。
```

更稳的架构是：

```text
默认走业务状态机。
遇到开放问题，再进入 LLM Planner。
关键动作必须回到业务规则校验。
```

比如售后 Agent：

```text
用户：我的退款怎么还没到账？
  -> 走确定流程查退款状态

用户：为什么我这种情况不能退款？
  -> LLM 解释政策，但依据来自政策 Skill 和订单状态

用户：我想申诉
  -> 进入工单流程，不让模型承诺结果
```

所以垂类 Agent 不是不用 LLM。

而是不让 LLM 控制核心业务事实和关键决策。

---

## 十四、总结

开放式 Agent 和垂类 Agent 最大的区别是：

```text
开放式 Agent 面向未知任务，核心是探索。
垂类 Agent 面向确定业务流程，核心是业务状态。
```

对应的控制权也不同：

```text
开放式 Agent：
LLM 控制下一步。

垂类 Agent：
业务系统控制下一步。
```

框架选择也应该跟着变：

```text
Claude Code / Claude Agent SDK：
适合开放式代码任务、探索任务。

LangGraph：
适合垂类业务流程、状态机、审批流、可控 Agent。

OpenAI Agents SDK：
适合 Agent as Tool / Handoff，但垂类场景必须配合业务规则和 guardrails。

LangChain：
能做通用 Agent，但企业场景更推荐 router / chain / structured workflow。

MCP / Skills：
适合作为业务能力和工具接入层，不应该让 LLM 无约束自由调用。
```

最后可以记住这句：

```text
未知问题、开放探索：用 Planner / ReAct / Tool Loop。
确定流程、强业务规则：用 Business State / Workflow / Router。
混合场景：业务状态机为主，LLM Planner 只处理开放部分。
```

企业级 Agent 的稳定性，不来自一个越来越复杂的 Planner，而来自业务状态、能力编排、工具接入和自然语言生成的清晰分层。

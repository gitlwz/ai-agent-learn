# 垂类 Agent

这个目录用于沉淀垂类业务 Agent 的架构设计、业务状态建模、能力路由、Skill 设计、MCP 接入和工程落地经验。

## 学习重点

```text
Intent 如何设计
Business Resolver 如何查询业务事实
Capability Router 如何选择能力
Skill 如何组织业务知识和约束
MCP 如何接入企业系统
业务规则和 LLM 如何分工
垂类 Agent 如何做权限、审计和兜底
```

## 文章导航

| 主题 | 说明 |
| --- | --- |
| 暂无 | 后续垂类 Agent 专题文章会放在这里。 |

## Step 规范

垂类 Agent 学习路线会采用连续 step：

```text
step01 -> step02 -> step03 -> ... -> step50
```

每个 step 都基于上一个 step 增加能力，并且每个 step 目录里都必须有 `README.md`。

代码目录规范见：

[src/vertical-agent/README.md](../../src/vertical-agent/README.md)

Step 文档模板见：

[src/vertical-agent/STEP_TEMPLATE.md](../../src/vertical-agent/STEP_TEMPLATE.md)

完整 50 step 大纲见：

[src/vertical-agent/ROADMAP.md](../../src/vertical-agent/ROADMAP.md)

## 当前代码 Step

| Step | 说明 |
| --- | --- |
| [step01-minimal-runtime](../../src/vertical-agent/step01-minimal-runtime/README.md) | 建立垂类 Agent 最小运行时、请求上下文和固定回复。 |
| [step02-domain-model](../../src/vertical-agent/step02-domain-model/README.md) | 在最小 Runtime 基础上增加垂类 Agent 的领域模型。 |

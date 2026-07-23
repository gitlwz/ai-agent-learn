# AI Agent Learn

这个仓库用于沉淀 AI Agent 学习笔记、源码阅读、工程问题和面试题。

## 导航

### 垂类 Agent

目录页：[垂类 Agent](./blogs/vertical-agent/README.md)

连续 step 代码规范：[src/vertical-agent](./src/vertical-agent/README.md)

垂类 Agent 50 step 大纲：[ROADMAP.md](./src/vertical-agent/ROADMAP.md)

当前进度：[step09-rule-entity-extractor](./src/vertical-agent/step09-rule-entity-extractor/README.md)

### AI Agent 问题与优化

| 主题 | 说明 |
| --- | --- |
| [让 LLM 老实调工具：从提示词到 Agent Harness 的四层防线](./blogs/ai-agent-problems-and-optimization/llm-tool-calling-reliability.md) | 总结 LLM 工具调用可靠性的四层防线：提示词软约束、JSON Schema 硬约束、校验修复重试闭环、架构层职责分离。 |
| [大模型流式输出：为什么常用 SSE 而不是 WebSocket](./blogs/ai-agent-problems-and-optimization/llm-streaming-sse-vs-websocket.md) | 分析大模型流式输出的通信模式，解释 SSE 与 WebSocket 的适用场景、工程取舍和面试回答方式。 |
| [开放式 Agent 和垂类 Agent，为什么不应该采用同一套架构](./blogs/ai-agent-problems-and-optimization/open-agent-vs-vertical-agent-architecture.md) | 对比开放式 Agent 与垂类业务 Agent 的控制模式，并结合 Claude Code、Claude Agent SDK、LangGraph、OpenAI Agents SDK、LangChain、MCP 和 Skills 分析架构取舍。 |
| [Intent、Skill、Business Resolver、Capability Router 和 MCP 到底是什么关系](./blogs/ai-agent-problems-and-optimization/intent-skill-business-resolver-mcp.md) | 梳理垂类 Agent 中 Intent、Skill、业务状态解析、能力路由和 MCP 工具接入之间的层次关系。 |

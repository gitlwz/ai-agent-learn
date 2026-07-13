---
title: "让 LLM 老实调工具：从提示词到 Agent Harness 的四层防线"
date: 2026-07-13
tags: [AI Agent, Tool Calling, Agent Harness, JSON Schema, 工程实践]
---

# 让 LLM 老实调工具：从提示词到 Agent Harness 的四层防线

> 一句话导读：让模型稳定调用工具，不是“提示词写好一点”的问题，而是一个完整的 Agent Harness 工程问题。

工具调用可靠性不是某个具体框架的问题，而是所有 Agent 系统都会遇到的底层问题：

```text
用户说一句话
模型决定调用哪个工具
模型生成工具参数
系统执行工具
工具结果再回到模型上下文
```

看起来简单，真正上线时最容易翻车的地方就在中间两步：

```text
模型可能不调工具，直接编答案。
模型可能调了工具，但参数格式不对。
模型可能参数格式对了，但语义是错的。
模型可能自己加字段、猜日期、猜城市、猜用户意图。
```

所以这篇文章的核心观点是：

```text
LLM 输出是概率采样。
提示词只是软约束，不是强制机制。
工具调用可靠性，本质上是软件工程问题。
```

这句话非常关键。它也是我们理解 Claude Code、LangGraph、Claude Agent SDK 这类 Agent Harness 的入口。

---

## 一、为什么“提示词写清楚”不够

很多人第一次做 tool calling，都会写类似这样的系统提示词：

```text
你只能调用 get_weather。
city 必须是标准城市名。
date 只能是 today 或 tomorrow。
不能乱编参数。
```

这当然有用，但它不可靠。

因为提示词本质上是在影响模型的概率分布。它能让模型“更可能”遵守规则，但不能保证模型“一定”遵守规则。

比如用户问：

```text
北京后天天气怎么样？
```

天气工具只支持：

```text
date = today | tomorrow
```

模型可能出现几种错误：

```text
1. 不调用工具，直接回答“北京后天晴”。
2. 调工具，但传 date = "后天"。
3. 自己把后天换算成某个具体日期。
4. 编一个工具根本不支持的字段。
```

这些问题不是“模型不听话”，而是 LLM 范式本身决定的：模型在生成一个看起来合理的下一个 token，而不是执行一个强类型程序。

所以提示词只能是第一层，不能是最后一层。

---

## 二、第一层防线：提示词软约束

提示词仍然重要。只是我们要正确理解它的角色：它是软约束，是方向盘，不是刹车锁。

好的工具提示词应该像一份接口说明，而不是一句空泛要求。

不要只写：

```text
city 是城市名称。
```

应该写成：

```text
city 必须是标准城市名。
禁止传入“我所在的城市”“附近城市”这种自然语言短语。
如果用户没有提供城市，不要猜测，应该要求补充信息。
```

对于日期参数，也不要只写：

```text
date 是日期。
```

而要写成：

```text
date 只能是 today 或 tomorrow。
如果用户问后天、大后天、下周，则不要调用该工具。
```

再进一步，可以给 few-shot 示例：

```text
用户：北京明天天气怎么样？
工具调用：get_weather({ "city": "Beijing", "date": "tomorrow" })

用户：北京后天天气怎么样？
工具调用：不调用工具，说明当前工具只支持今天和明天。
```

few-shot 的价值是让模型看到边界案例。模型不是读了规则就一定遵守，但示例能明显降低出错概率。

不过，它仍然只是降低概率。

---

## 三、第二层防线：JSON Schema 硬约束

如果提示词是“讲道理”，JSON Schema 就是“设栏杆”。

工具参数不能只靠自然语言描述，而要用机器可验证的结构定义。

比如天气工具可以定义成：

```json
{
  "type": "object",
  "properties": {
    "city": {
      "type": "string",
      "description": "Standard city name, such as Beijing or Shanghai"
    },
    "date": {
      "type": "string",
      "enum": ["today", "tomorrow"]
    },
    "unit": {
      "type": "string",
      "enum": ["celsius", "fahrenheit"]
    }
  },
  "required": ["city", "date"],
  "additionalProperties": false
}
```

这里最值得注意的是：

```json
"additionalProperties": false
```

它的意思是：不允许模型自己加字段。

如果没有这条，模型可能生成：

```json
{
  "city": "Beijing",
  "date": "tomorrow",
  "reason": "用户想知道天气，所以我猜测这个参数"
}
```

这个 `reason` 字段看起来无害，但对下游系统来说就是不确定输入。

Schema 的价值是把“模型应该怎么输出”变成“程序可以验证是否合规”。

这一步之后，Agent 系统开始从提示词工程进入软件工程。

---

## 四、第三层防线：校验、修复、重试闭环

即使有 Schema，也不能完全放心。

因为模型可能输出：

```text
格式不合法的 JSON
带 Markdown 包裹的 JSON
字段类型正确但枚举值错误
参数格式正确但业务不可用
```

所以框架层要做闭环：

```text
模型输出
  -> JSON 语法校验
  -> Schema 校验
  -> 自动清洗
  -> 带错误信息重试
  -> 超过上限后降级
```

伪代码大概是：

```ts
async function callToolWithValidation(modelOutput: string) {
  for (let attempt = 0; attempt < 3; attempt++) {
    const parsed = tryParseJson(modelOutput);
    if (!parsed.ok) {
      modelOutput = await askModelToRepair(modelOutput, parsed.error);
      continue;
    }

    const valid = validateSchema(parsed.value);
    if (!valid.ok) {
      modelOutput = await askModelToRepair(modelOutput, valid.error);
      continue;
    }

    return executeTool(parsed.value);
  }

  return fallback("当前请求无法安全处理，请换一种说法。");
}
```

这里有一个非常重要的工程原则：

```text
重试必须有上限。
```

不要让模型无限修复。无限重试会带来成本、延迟和死循环风险。

一般线上系统会设置：

```text
第一次：带错误信息让同一模型修复。
第二次：使用更严格的修复提示词。
第三次：仍失败则降级或转人工。
```

这就是“兜底闭环”。

---

## 五、第四层防线：模型只决策，框架负责执行

文章最有价值的地方，是把问题上升到架构层。

不要让模型直接负责执行。

一个可靠的 Agent 系统应该分成三层：

```text
模型层：负责理解意图，决定调用哪个工具，生成候选参数。
框架层：负责校验参数、权限判断、重试、日志、调用工具。
工具层：负责真实业务能力，比如查天气、写文件、跑命令。
```

也就是：

```text
LLM 只负责“想”。
Harness 负责“卡”和“做”。
Tool 负责“业务执行”。
```

这个分层非常像 Claude Code 的工具调用链路：

```text
Claude 生成 tool_use
  -> QueryEngine 接住
  -> Tool schema 校验
  -> Permission system 权限判断
  -> Tool executor 执行 Read/Edit/Bash
  -> tool_result 回到上下文
  -> Claude 继续推理
```

真正可靠的不是模型本身，而是外面这一圈 Harness。

所以我们前面一直说：

```text
Agent = LLM + Harness + Tools + Context Loop
```

这篇文章其实是在讲其中的 Harness 为什么重要。

---

## 六、Schema 管格式，不管语义

JSON Schema 很重要，但它也有边界。

它能判断：

```text
city 是不是 string
date 是不是 today 或 tomorrow
unit 是不是 celsius 或 fahrenheit
```

但它判断不了：

```text
沪 是不是上海
魔都 是不是上海
羊城 是不是广州
帝都 是不是北京
```

比如模型输出：

```json
{
  "city": "沪",
  "date": "today"
}
```

Schema 会通过，因为 `city` 确实是字符串。

但天气 API 可能不认识 `沪`。

这就进入了参数语义验证：

```text
别名映射：沪 -> 上海
实体链接：把自然语言实体链接到标准实体 ID
领域规范化：医疗、金融、法律术语转成业务系统认识的编码
知识图谱：用结构化知识消解歧义
```

这一层不应该完全依赖 LLM。更稳的做法是放到框架层或工具层：

```text
用户输入：查一下沪的天气
模型参数：city = "沪"
框架规范化：city = "Shanghai"
工具执行：get_weather({ city: "Shanghai" })
```

这也是 Agent 工程里很容易被低估的地方：参数不是格式对了就能用，还要业务语义对。

---

## 七、和 Claude Agent SDK / LangGraph 的关系

这篇文章虽然没有专门讲 Claude Agent SDK 或 LangGraph，但它解释了这些框架为什么存在。

如果只是直接调用模型：

```text
用户 -> LLM -> 工具参数
```

那你要自己处理：

```text
工具 schema
参数校验
权限审批
重试
降级
日志
上下文回填
多轮工具循环
```

Claude Agent SDK、LangGraph、LlamaIndex、AutoGen 这类框架，本质上都在帮你做 Harness。

区别是：

```text
Claude Agent SDK 更像成品 Harness：
  Claude Code 的 loop、工具、权限、上下文管理已经封装好了。

LangGraph 更像编排框架：
  你自己定义节点、状态、边、恢复、中断和工具执行。
```

但它们的底层目标一致：

```text
不要相信模型天然可靠。
用工程系统约束模型、校验模型、补救模型。
```

---

## 八、落到我们自己的 Agent 学习项目

在我们的 `claude-learn` 教学项目里，最核心的工具调用链路其实也遵循这个方向。

简化后是：

```text
QueryEngine
  -> 调模型
  -> 识别 tool_use
  -> 找到工具定义
  -> 执行工具
  -> 把 tool_result 塞回上下文
  -> 继续下一轮
```

教学版更关注“让你看清楚 loop 是怎么转起来的”，所以有些生产能力还比较简化：

```text
Schema 校验：有骨架，但不如生产框架严格。
权限系统：有教学版实现，但没有 Claude Code 那么细。
重试策略：有错误处理，但没有完整 repair loop。
语义规范化：基本没有实现。
可观测性：有日志意识，但不是完整 tracing。
```

这正好说明一个学习路径：

```text
第一步：先实现 ReAct / tool loop，让 Agent 能调工具。
第二步：加 Tool Schema，让工具参数有契约。
第三步：加 Permission，让危险工具不能直接执行。
第四步：加校验、修复、重试，让系统能抗模型错误。
第五步：加 tracing 和语义规范化，让系统可上线、可排查。
```

如果说 ReAct 解决的是：

```text
模型如何边想边用工具？
```

那这篇文章解决的是：

```text
模型用工具时，系统如何不被它带偏？
```

---

## 九、面试里怎么讲

如果面试官问：

```text
怎么让 LLM 老老实实调工具？
```

不要只回答：

```text
写好提示词。
```

更好的回答是三层：

```text
第一，先说本质：
LLM 输出是概率采样，提示词是软约束，不是强制机制。

第二，讲防线：
提示词软约束、JSON Schema 硬约束、校验修复重试闭环、架构层职责分离。

第三，讲边界：
Schema 管格式，不管语义。复杂业务还需要实体链接、别名映射、参数规范化和 tracing。
```

这比单纯说“加 schema”更完整，因为它回答了：

```text
为什么会错？
怎么拦住格式错误？
怎么处理漏网之鱼？
怎么避免模型错误直接打到工具层？
后续工程演进在哪里？
```

---

## 十、总结

让 LLM 稳定调用工具，可以记成一句话：

```text
提示词负责引导，Schema 负责卡格式，校验重试负责兜底，Harness 负责执行隔离。
```

四层防线是：

```text
1. 提示词软约束：让模型更可能做对。
2. JSON Schema 硬约束：让错误格式过不去。
3. 校验、修复、重试：让边缘错误有恢复机会。
4. 架构分离：模型只做决策，框架负责执行。
```

最终要建立的认知是：

```text
LLM 不等于可靠执行器。
Agent Harness 才是可靠执行器。
```

这也是为什么我们学习 Claude Code 源码时，不能只盯着 prompt，而要看 QueryEngine、tool schema、permission、tool executor、context loop、compact、subagent、tracing 这些东西。

真正的 Agent 工程，不是让模型“更聪明”而已，而是让一个不稳定的概率模型，被一套稳定的软件工程系统托住。

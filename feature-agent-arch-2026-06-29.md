# Agent 架构的十字路口：AI Engineer World's Fair 2026 五场必看 Talk 深度解读

> 日期：2026年6月29日 · 旧金山 Moscone West
> 作者：AI Daily 编辑部

---

AI Engineer World's Fair 2026 今天在旧金山拉开帷幕。566 场 session、160+ 场与 Agent 相关的议题——这个行业已经从「要不要做 Agent」彻底转向了「怎么做可靠的 Agent 架构」。

今年的共识非常明确：**模型在变强，瓶颈在架构**。五场最具代表性的 Agent 架构 talk，分别从五个维度切开了这个命题。

---

## 一、架构的保质期只有 6 个月，那还建什么？

**Talk:** *Your agent architecture has a half-life of 6 months*
**演讲者:** Dan Farrelly（Inngest 联合创始人 & CTO）
**Track:** Day 3

Dan Farrelly 上来就抛出了一个让人不安的事实：过去两年，"正确的 Agent 架构" 经历了 RAG → ReAct → prompt chaining → orchestrator-workers → MCP → CLI → MCP again... 每一次行业风向转换，团队就要重写一次架构。

但 Farrelly 不是来预测下一个趋势的。他的核心观点是：**与其追逐架构范式，不如找到那些真正稳定的原始构件（primitives）**。

他指出，所有生产级 Agent 中都反复出现的几个核心要素：

- **事件驱动**：Agent 本质上是一个事件处理器，不是请求响应器
- **可恢复的执行**：Agent 执行到一半会断，架构必须天然支持断点续传
- **决策靠近代码**：将「什么时候用什么模型」的决策逻辑从配置文件挪到代码中，释放更多栈灵活性
- **执行层决定迭代速度**：好的执行层让你可以快速切换模型、重组工具链，而不动核心逻辑

「你下一个要换的不是架构，是模型。你的架构应该让你换模型像换数据库驱动一样简单。」

**核心洞见：** 与其押注某个范式，不如认真打磨执行层。范式会变，执行层不会。

---

## 二、你的 Agent 没失败，是你的 Harness 失败了

**Talk:** *Your Agent Didn't Fail. Your Harness Did.*
**演讲者:** Vinoth Govindarajan（OpenAI, Member of Technical Staff）
**Track:** Claws & Personal Agents

这可能是今年最重要的一场 Agent 架构 talk。Vinoth 直接挑战了行业默认假设：Agent 产出不对时，大家第一反应是模型问题，但大量生产故障其实发生在模型周围的 harness（鞍具/护栏系统）里。

具体来说，harness 层面最常见的失败模式包括：

1. **状态没有持久化**——Agent 跑了一半，丢了全部上下文
2. **两次运行突变了同一个 session**——并发场景下的状态污染
3. **工具调用没有返回**——hang 死在等待中
4. **审批失去了作用范围**——一次通过的权限被反复滥用
5. **内部成功没有变成用户可见的证据**——Agent 以为做完了，但没人知道

Vinoth 用 OpenClaw 作为公开案例研究，提炼出了一个可复用的生产模型。核心极简公式：

> **模型提出（propose）→ Harness 提交（commit）→ 收据证明（receipt proves）**

他提出了一套实用的「运行收据审计」（run receipt audit），每位工程师都可以应用到自己的 Agent 上：

- 什么唤醒了 Agent？
- 它继承了什么状态？
- 它使用了什么权限？
- 实际执行了什么？
- 留下了什么证据？

**核心洞见：** 模型负责聪明，harness 负责可靠。二者不可互相替代。

---

## 三、长期研究型 Agent 的记忆架构实战

**Talk:** *Memory Harnesses for Long-Running Research Agents*
**演讲者:** Stefania Druga（Sakana AI）
**Track:** Memory & Continual Learning

Sakana AI 的 Agent 需要跑几百个回合——阅读文献、运行实验、起草论文。Stefania 分享了一个反直觉的发现：**在这些长期任务中，模型很少出问题，出问题的一直是 harness。**

具体来说，长期 Agent 面临三大衰退：

- **自相矛盾**：第 80 回合的决策推翻第 10 回合的结论
- **重复劳动**：不记得已经做完的工作，重新跑一遍
- **目标漂移**：越跑越偏离最初的问题

她把这称为「**紧约束命题**」（binding-constraint thesis）：对于长周期任务来说，系统的可靠性上限不取决于模型，而取决于 harness。

Sakana AI 在实践中总结出的记忆架构模式包括：

- **三层记忆**（three-tier memory）：短期工作区 + 中期上下文池 + 长期档案库
- **渐进式披露**（progressive disclosure）：不是把所有历史一股脑塞给模型，而是按需展开
- **先召回后压缩**（recall-first compaction）：先看哪些信息被实际用到了，再决定保留什么
- **子 Agent 隔离**（sub-agent isolation）：每个子任务有自己的干净上下文空间
- **超越向量数据库的架构记忆**：不仅仅是 embedding 检索，还包括决策轨迹的结构化存储

更重要的是，她给出了一个衡量标准：**如何判断你的记忆架构真的在起作用，还是在空转。**

**核心洞见：** 长期 Agent 最大的敌人不是模型能力不足，而是上下文衰减（context rot）和目标漂移。这不是 prompt 工程能解决的问题，这是架构问题。

---

## 四、生产事故修复 Agent 的六根支柱

**Talk:** *6 Pillars of an Agentic Harness That Fixes Production Incidents*
**演讲者:** Varun Krovvidi（Resolve AI）
**Track:** Day 2

当一个生产事故在 P0 级别燃烧时，模型可以给出无数个"听起来合理"的回答，但只有一个正确答案。模型自己无法可靠地找到它。Varun 把这套工程挑战定义为 harness engineering 的新前沿。

他提出的六根支柱框架：

### 1. 模型编排（Model Orchestration）
不是单一模型搞定一切，而是多个模型各司其职：一个做推理，一个做代码定位，一个做验证。

### 2. 上下文（Context）
事故修复的上下文通常横跨日志、指标、代码变更、部署记录。Harness 必须有能力在广域上下文中精准定位相关片段。

### 3. 推理（Reasoning）
不仅仅是 LLM 的 chain-of-thought，还包括结构化的根因分析流程——哪些可能是原因，如何排除假阳性。

### 4. 行动（Actions）
Agent 读权限只是第一步。真正的生产力来自写权限：执行命令、回滚代码、修改配置。每一步都需要精细的权限控制和可撤销能力。

### 5. 学习（Learning）
每次事故修复应该成为下一次的参考。问题在于：如何让 Agent 从一次修复中真正学会，而不是过拟合单个案例。

### 6. 评估（Evals）
在事故场景下评估 Agent 质量比常规场景更难——因为正确答案只有一个。Varun 强调，更好的模型不会让这六根支柱中的任何一根消失。

**核心洞见：** Incidents are where harness engineering earns its keep. 常规场景下 model 够用，灾难场景下 harness 决定生死。

---

## 五、知识的三条路径：Prompt、Memory、Weights

**Talk:** *Prompt, Memory, Weights: The Architecture Decisions Most AI Teams Make by Accident*
**演讲者:** Anant Srivastava
**Track:** Context Engineering

绝大多数 AI 团队在做架构决策时都带着一个默认假设：prompt → RAG → fine-tuning 是一条进阶梯子。先用 prompt 做约束，不行就上 RAG，再不行就微调——仿佛每一步是上一步的「高级版本」。

Anant 的观点截然不同：**这三条路径解决的是完全不同的问题，它们不是替代关系，是互补关系。**

| 路径 | 解决的问题 | 陷阱 |
|------|-----------|------|
| **Prompt** | 塑造行为和约束 | 做的太多会撑爆上下文窗口 |
| **Memory（RAG）** | 提供当前、可引用的知识 | 做的太多会引入噪声和过时信息 |
| **Weights（微调）** | 固化专业推理和格式 | 做的太多会「记住」随时可能过时的事实 |

最常见的经典陷阱：**用微调去教模型本应通过检索获取的事实。** 你把知识烤进了权重，结果知识在发布的当天就过时了，而且你还无法引用来源。

真正的架构是**循环**的：记忆记录了 Agent 做了什么 → 这些记录成为微调的数据集 → 微调改变了值得检索的内容 → 更好的检索带来更好的记忆 —— 这个循环不断叠加。

**核心洞见：** 大多数团队的架构问题是「用错了路径」而不是「模型不够强」。把 prompt、memory、weights 看作三条独立但互联的通道，而不是一条梯子。

---

## 总结：2026 年的 Agent 架构共识

从这五场 talk 中，可以清晰看到几条行业正在形成的共识：

1. **Harness 工程是新的系统工程**——模型每年进步几个数量级，但 harness 的可靠性不会自动跟进。Harness 需要像十年前的基础设施一样被认真对待。

2. **架构的稳定性来自原始构件，而非范式**——Agent 架构的范式每半年一变，但事件驱动、持久化执行、权限控制这些原始构件会留下来。

3. **记忆是长期 Agent 的瓶颈**——当 Agent 需要跑几十上百个回合时，记忆架构的质量决定了 Agent 可持续运行的天花板。

4. **知识的三条路径必须独立设计**——prompt、memory、weights 是三种不同的架构决策，不能默认走梯度升级的路径。

5. **生产场景是最好的架构验金石**——常规对话中模型可以掩盖很多架构问题，但生产事故、长周期任务、多工具编排这些场景会毫不留情地暴露 harness 的所有缺陷。

---

## 各 Talk 链接与观看方式

大会正在进行中（6/29-7/2），以下 5 场 talk 的具体日程可以在大会日程页找到。视频录制将在会后陆续上传至 AI Engineer YouTube 频道。

| # | Talk | 演讲者 | 日程位置 | 演讲者链接 |
|---|------|--------|---------|-----------|
| 1 | 架构的保质期只有 6 个月 | Dan Farrelly | Day 3（7/1）12:05pm | [LinkedIn](https://www.linkedin.com/in/djfarrelly) |
| 2 | Your Agent Didn't Fail. Your Harness Did. | Vinoth Govindarajan | Day 2（6/30）11:10am · Claws & Personal Agents | [LinkedIn](https://www.linkedin.com/in/vinothgovindarajan/) · [Substack](https://theagentstack.substack.com/) |
| 3 | Memory Harnesses for Long-Running Research Agents | Stefania Druga | Day 3（7/1）11:40am · Memory & Continual Learning | [LinkedIn](https://www.linkedin.com/in/drugastefania/) · [Website](https://stefania11.github.io/) |
| 4 | 6 Pillars of an Agentic Harness | Varun Krovvidi | Day 2（6/30）2:50pm | — |
| 5 | Prompt, Memory, Weights | Anant Srivastava | Day 3（7/1）12:05pm · Context Engineering | [LinkedIn](https://www.linkedin.com/in/anantds) |

**观看链接：**

- **大会日程页（所有 session）：** <https://www.ai.engineer/worldsfair/schedule>
- **Sessions 开放数据（JSON）：** <https://www.ai.engineer/worldsfair/sessions.json>
- **AI Engineer YouTube（视频将在此发布）：** <https://youtube.com/@aiDotEngineer>
- **大会主站：** <https://www.ai.engineer/worldsfair>

> 注：以上 5 场 talk 分别安排在 Day 2（6/30）和 Day 3（7/1），大会期间可通过日程页找到各 session 的详细位置。录制视频将在大会后由 AI Engineer 官方 YouTube 频道发布。

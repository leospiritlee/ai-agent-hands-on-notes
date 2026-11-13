# 第3章 消息与模型层：把一次请求封装成可替换的接口

**一句话**：把 `instructions`、模型配置、`output_type` 与校验策略收进一个自己的调用封装，让后面十一章的业务代码不直接依赖任何 `SDK` 细节；这是「换模型不改业务」的唯一落点。

**为什么需要它**：第 2 章的脚本能跑，但 `Agent(...)` 散在 `main()` 里，提示词是硬编码字符串，输出是一段自由文本。这样的代码在第 4 章加循环、第 6 章加工具、第 10 章改提示词时会同时改三处，任何一次改动都无法证明只改了它该改的东西。综合报告口径把「把 LLM 调用、向量库、工具都包在自己的接口后面，让业务逻辑不依赖框架」列为选型纪律，并给出验收条件：能否在不重写编排层的前提下换掉它。官方 quickstart 也提示了这件事的边界——「Use a plain `Agent` plus `Runner` when the task mainly lives in prompts, tools, and conversation state」，任务主要在提示词、工具与会话状态里时，一个朴素封装就够了，不必先上图框架。

**核心结构**：封装分四层。输入层是 `instructions` 与消息列表，版本化存放、像代码一样被测试（综合报告口径把「像代码一样版本化、测试 prompt」列为工程底座）。契约层是 `output_type` 与 `structured output`：官方 SDK 的 `output_type` 配合 `Pydantic` 做类型校验，`platform.openai.com` 的结构化输出文档用 `json_schema` 与 `response_format` 描述同一件事，`strict: true` 让字段严格匹配 schema。执行层是 `Runner.run` 与模型配置，把供应商相关的部分压在一个函数里。失败层是校验失败时返回可行动的错误信息，而不是让模型编一个看起来合理的结果——`strict` 保证的是格式严格匹配，语义仍要靠字段名与描述，这条会在第 6 章的工具参数上再次出现。

**关键设计**：**契约先于提示词**。综合报告口径说得很直白：优先约束解码 / JSON schema，而非靠提示词祈祷格式；把结构化输出和函数调用放在工程底座里，用 `Pydantic` 定义工具 I/O schema，作为防幻觉参数的第一道防线。工单助手需要的输出形状很朴素——是否命中条款、引用的文档片段、起草的回复、是否需要人工复核——四个字段，其中「是否需要人工复核」必须非空。用 `BaseModel` 加 `Field` 写出来，比在 `instructions` 里写十遍「请务必返回 JSON」有效得多：

```python
from pydantic import BaseModel, Field

class DraftReply(BaseModel):
    matched_rule: str = Field(description="The matched refund rule")
    citation: str = Field(description="Source fragment in the rule doc")
    draft: str = Field(description="Proposed reply, never sent")
    needs_review: bool = Field(description="Whether a human must approve")
```

字段类型来自 `Pydantic`，`Field` 的描述会被读进 schema；官方结构化输出文档里 `ContactInfo(BaseModel)` 用的就是同一套写法。这里的关键是「needs_review 必须显式」，它把第 12 章的人工闸门提前变成了一个数据结构。

**拆解**：

### 统一调用封装该暴露什么、藏什么

暴露三样：run(input)、`output_type` 的校验结果、以及一次运行的 `usage` 与 trace 标识。藏起四样：供应商 SDK 的导入、模型名与默认值、密钥获取方式、重试与超时策略。综合报告口径把目录按 entry point / domain logic / agent graph / tool wrappers / RAG module / eval module 分层，本质就是让这四样只在一个地方出现。判断封装合不合格有个简单办法：把 `pip install openai-agents` 换成别的 SDK，改动是否只落在一个文件里；如果第 5 章的工具定义要跟着改，说明封装漏了一层。

### 结构化输出为什么是底座而不是美化

错误的输出在单步任务里表现为解析失败，在 Agent 里会变成错误的工具参数、错误的分支，乃至一次写操作。`strict: true` 与 `json_schema` 解决的是「格式严格匹配」，`output_type` 与 `Pydantic` 解决的是「类型能被校验」，两者都不是语义正确性的保证——官方工具使用文档反复强调，参数名与描述才是模型理解用途的地方。所以契约层要连做三件事：定义 schema、给每个字段写无歧义的名字、让校验失败返回可行动的错误信息。第 6 章会看到 `ToolError` 把同一条原则用在工具返回值上。

### 严格模式打开后，校验失败该给谁看

`strict` 打开、模型输出不合 schema 时，不要吞掉异常再让模型「再试一次」而不记录。综合报告口径要求工具返回值必须结构化、可校验，并把「校验失败返回可行动的错误信息」列为工程建议；第 11 章的 `span` 与第 10 章的回归集都依赖这次失败被如实记下来。工单助手的做法是：校验失败 → 记一条带输入摘要与 schema 版本的错误 → 返回「需要人工复核」而不是一段猜测的回复。这样即使后续模型换了版本，代码里的 needs_review 字段仍然是可断言的。

### 模型配置里哪些必须钉死

综合报告口径点了一个很隐蔽的坑：用浮动的模型版本（写 `claude-opus-4` 这类滚动标签）会让一次上游更新悄悄把评估分数打掉 15%，而代码没有任何变化；结论是「模型版本必须钉死，并纳入发布记录」。因此配置文件里至少要有：具体的模型标识、`temperature`、超时与重试上限、提示词版本号、schema 版本号。这五项在第 12 章会一起进发布记录，第 10 章的回归集才能回答「到底是哪个改动让分数变了」。

**今天怎么做**：把第 2 章的脚本重构成一个 run(input) 函数，加一个 `BaseModel` 输出契约，把模型名、`temperature`、提示词版本、schema 版本写进一处配置；跑三条真实输入，确认校验失败时会走人工复核分支。**延伸，非语料原文**：顺手给配置文件加一个 `version` 字段，后面每次改动只改这一个值，日志与评测报告就能自动对上版本。

<div class="chapend" markdown="1">

**容易误解的地方**：把 `output_type` 当成格式美化——它是防幻觉参数的第一道防线；把 `strict: true` 当成万无一失——它保证严格匹配 schema，不保证参数语义正确；把封装写成另一个框架——目标是藏在供应商差异，不是重新发明编排层；还有把模型名写成滚动标签，等于把一次上游更新变成一次没有变更记录的发布。

**记住这一句**：消息层的产出物是一个可替换的调用封装 = 版本化的 `instructions` + `output_type` 契约 + 钉死的模型配置 + 可行动的校验失败路径；契约先于提示词，校验先于执行。

</div>

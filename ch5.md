# 第5章 第一个工具：定义、注册、执行、回灌

**一句话**：先只做一个只读工具，把「定义 → 注册 → 执行 → 结果回灌」这条链路逐字段打通；`function_tool` 负责从函数签名生成 schema，`tool_use_id` 负责把结果对回那一次请求。

**为什么需要它**：综合报告口径给原型阶段的顺序是「先固定工具，再写循环，最后写提示词」，并要求工具只留 `1–3 个只读能力`。原因很实在：工具是模型唯一能改变外部世界的通道，也是唯一能把概率输出变成确定副作用的地方。官方 quickstart 里给智能体加工具的方式极简——用 `@tool` 装饰一个 Python 函数，把 `tools=[history_fun_fact]` 传给 `Agent`：

```python
import asyncio
from agents import Agent, Runner
from agents.decorators import tool

@tool
def history_fun_fact() -> str:
    """Return a short history fact."""
    return "Sharks are older than trees."

agent = Agent(
    name="History Tutor",
    instructions="Answer history questions clearly. Use history_fun_fact when it helps.",
    tools=[history_fun_fact],
)
```

逐字看这段代码会发现三件事：函数名是工具名、docstring 是描述、类型标注是参数与返回值的来源。官方把这件事叫 `docstring parsing`（在 SDK 文档里对应 `function_tool` 的自动生成 schema），而 `platform.claude.com` 的工具使用文档用 `input_schema` 把同一层契约写得更显式。工单助手的第一个工具就是 get_order_status：只读、一个参数、返回结构化的订单状态。

**核心结构**：一次工具往返有四步。第一步定义：`name`、`description`、`input_schema` 三样，字段名要无歧义，描述写给模型看而不是写给人看。第二步注册：把工具放进 `tools` 列表，或用 `MCPServer` 接外部服务；官方 MCP 文档里还有 `list_tools()` 是否缓存、`cache_tools_list=True`、`max_retry_attempts=3` 这类连接级选项。第三步执行：模型返回 `tool_use`（或 `tool calls`）后，按 schema 校验参数，再调用本地函数。第四步回灌：把结果作为 `tool_result` 追加，带上 `tool_use_id` 指回原请求，`stop_reason` 告诉 runner 这一轮为什么停。这四步里最容易偷懒的是第四步——只把结果字符串拼回上下文，短期能跑，长期无法做轨迹层评测。

**关键设计**：**`tool_use_id` 是这条链路的主键，不是可选字段**。工具使用文档里，`tool_result` 通过 `tool_use_id` 与某一次 `tool_use` 配对；`parallel_tool_calls` 打开后一轮可能返回多个调用，没有这个 id 就无法确定哪个结果属于哪个请求。综合报告口径把「工具选择／参数错误」列为第二类失败模式，要求记录参数级 trace 与 schema 校验结果，正是为了在多调用场景下还能复盘。另一条设计要点是「结果精简」：工具返回值应该是模型真正需要的高信号内容，不是下游 API 的原始响应体；综合报告口径把「结果精简」与错误语义、超时并列写进工具侧清单。

**拆解**：

### 从函数到 schema，中间发生了什么

`function_tool` 会读函数签名与 `docstring parsing` 的结果，生成给模型看的 `input_schema`；`strict: true` 则让调用严格匹配这份 schema。要亲手验证一次：把参数从 `user_id` 改成含义含糊的名字、把 docstring 删掉、把两个工具的功能写得重叠，分别看模型传进来的参数怎么漂。综合报告口径把类型化 schema、清晰描述、参数校验列为工具侧必做项；官方 `Pydantic` 的 `Field(description=...)` 让描述与类型写在同一处，比另外维护一份 JSON 更不容易漂移。注意 `strict` 保证的是格式严格匹配，语义正确仍依赖命名与描述——这两件事别混为一谈。

### 一次往返逐字段对齐

把一次真实往返打印出来，按顺序列：`stop_reason`（为什么停）、`tool_use`（工具名与参数）、`input_schema`（期望形状）、本地校验结果、`tool_result`（返回值）、`tool_use_id`（配对键）。`platform.claude.com` 的工具使用文档用 `client.messages.create`、`max_tokens`、`system`、`messages` 描述请求，用 `tool_use` / `tool_result` 描述往返；SDK 层则把这些折进 `Runner.run` 的循环。两种写法是同一条链路的不同抽象层，本章要能对着字段讲清「谁在承担失败」：参数错在校验层，工具错在执行层，回灌错在配对层。

### 只读工具为什么必须排在第一位

工单助手的 get_order_status 不写任何东西，失败重跑没有副作用。综合报告口径的动作分级要求只读操作可以直接执行，涉及资金、删除、外部发送、权限变更的操作必须审批或严格限制阈值；原型阶段先只做只读，等于把最难的权限问题推迟到工具层已经稳定之后。官方 MCP 示例里的 `ToolAnnotations` 把这件事写成了机器可读的提示：

```python
WEATHER_TOOL_ANNOTATIONS = ToolAnnotations(
    read_only_hint=True,
    destructive_hint=False,
    idempotent_hint=True,
    open_world_hint=True,
)
```

示例源码里的注释还特意说明「Hints are not guarantees; a host decides how far to trust them」——提示不是保证，最终由宿主决定信任到什么程度。这句话应该贴在每个工具注册表旁边：标注只是意图，权限必须在模型之外执行。

### 结果回灌之后，循环怎么知道该继续还是该停

把 `tool_result` 追加进上下文后重跑循环，模型要么给出 `final output`、要么再发一轮 `tool calls`、要么触发 `handoff`。第 4 章的三种终止条件在这里才开始真正工作：若工具返回的是「查无此订单」，我们希望模型直接给出受控回复或走 needs_review，而不是换个参数反复查。综合报告口径把「步骤重复 `15.7%`」列为占比最高的单一失败模式，对策就是显式终止条件与重复动作签名检测。**延伸，非语料原文**：给每个工具加一句「什么情况下应该停止调用我」写进描述，比在提示词末尾写「不要重复调用」有效。

**今天怎么做**：只写 get_order_status 一个只读工具，用 `@tool` 或 `function_tool` 注册，把一次完整往返的六个字段打印出来核对；然后把参数名改含糊、把 docstring 删掉各跑一次，记录模型传错的参数。**延伸，非语料原文**：把这三个失败样本连同期望参数存进 docs/tool-failures.md，第 10 章会直接拿它们做单测。

<div class="chapend" markdown="1">

**容易误解的地方**：以为工具就是「把现有 API 包一层」——返回值必须按模型需要裁剪并带可行动的错误信息；以为 `tool_use_id` 只是协议细节——它是多调用场景下唯一能对齐结果的主键；以为 `strict: true` 保证参数语义正确——它只保证严格匹配 schema；还有把 `read_only_hint` 当成权限保证，官方示例明说提示不是保证。

**记住这一句**：第一个工具的产出物不是能调通的函数，而是一次可逐字段核对的往返 = `input_schema` + `tool_use` + `tool_result` + `tool_use_id` + 停止理由；先把一条链路走干净，再加第二个工具。

</div>

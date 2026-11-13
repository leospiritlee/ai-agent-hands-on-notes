# 第2章 项目骨架：一次能跑通的最小调用

**一句话**：最短起点不是框架选型，而是建一个 `virtual environment`、装一个 SDK、注入密钥，然后用 `Agent` 加 `Runner.run` 跑出第一个 `final_output`；这一层跑不通，上面任何编排都只是在放大不确定性。

**为什么需要它**：官方 quickstart 的第一步就是「Create a project and `virtual environment`」，并注明「You'll only need to do this once」；紧接着是激活环境、安装 SDK、设置 `OPENAI_API_KEY`。这份顺序本身就是工程纪律：环境可复现、密钥走运行时注入、依赖可锁定，三件事都在第一次调用之前完成。综合报告口径把「把模型当应用」列为高频失败——忘了鉴权、授权、校验、日志、监控仍是自己该写的；一个连环境变量都随手硬编码的项目，后面补权限时几乎必然要重写一遍入口。本章只交一份产出物：一个能在自己机器上复现的、打印出第一行回答的脚本。

**核心结构**：骨架有四段。第一段是目录与虚拟环境：`mkdir my_project`、`python -m venv .venv`、`source .venv/bin/activate`。第二段是依赖与密钥：`pip install openai-agents`，以及 `export OPENAI_API_KEY=sk-...`（官方文档同时给了 Windows PowerShell 与 Command Prompt 的写法，按自己的平台取用）。第三段是最小 Agent：

```python
from agents import Agent

agent = Agent(
    name="History Tutor",
    instructions="You answer history questions clearly and concisely.",
)
```

第四段是把它跑起来：

```python
import asyncio
from agents import Agent, Runner

agent = Agent(
    name="History Tutor",
    instructions="You answer history questions clearly and concisely.",
)

async def main():
    result = await Runner.run(agent, "When did the Roman Empire fall?")
    print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())
```

这两段代码逐字来自官方 quickstart 与 `examples/basic/hello_world.py` 的形状。`Runner.run` 返回的是 `RunResult`，官方的说明是「Use `Runner` to execute the agent and get a `RunResult` back」；`final_output` 是最后一个 agent 的输出。别急着换名字，先让它原样跑通一次。

**关键设计**：**第一次调用要收住的是字段，不是回答内容**。跑通之后，把 `result` 里能读的东西列一遍：`final_output`、`last_agent`、`to_input_list()`，以及流式场景的 `RunResultStreaming`。官方 quickstart 明确给了第二回合的三条路：把 `result.to_input_list()` 传回 `Runner.run(...)`、挂一个 `session=...`、或用 OpenAI 服务端状态 `conversation_id` / `previous_response_id`；并给了选型表——要完全手动控制且与供应商无关的历史就用 `result.to_input_list()`，要让 SDK 替你存取历史就用 `session=...`，要服务端续接就用 `previous_response_id` 或 `conversation_id`。这三条路不是三个 API，是三种「状态归谁管」的决定，第 7 章会按这个表往下展开。先在这里把它们记进项目笔记，比第 7 章临时翻文档省事。

**拆解**：

### 目录与依赖为什么先于代码

官方文档把建项目、建 `virtual environment`、激活环境三步放在最前面，并特别注明激活要「Do this every time you start a new terminal session」。这不是教学上的啰嗦：Agent 项目最常见的不可复现，来自「我这边能跑」——本地全局装了某个版本的 SDK，换台机器就少了 `Pydantic` 的某个行为。锁依赖、写进 `requirements` 或项目清单、把密钥从代码里挪到环境变量，是第 3 章能把调用封装成可替换接口的前提。综合报告口径也把「锁依赖保证可复现」和「密钥走运行时注入」列为工程底座。

### 密钥与配置的最小可用形状

`export OPENAI_API_KEY=sk-...` 只是官方给的最简写法，它对应的原则有三条：密钥不进版本库、不进提示词、不进日志；配置集中在入口，业务代码只读配置对象；测试环境不含真实凭证与真实客户数据（这条来自综合报告口径的上线验收清单）。第 11 章的 `Tracing` 会记录一次运行的很多字段，如果密钥从一开始就没被当普通字符串到处传，后面就不用回头做一遍脱敏。**延伸，非语料原文**：现在就在 .gitignore 里写掉 .env、*.db 与运行日志目录，这个十秒钟的动作比事后清理可靠得多。

### 第一次调用到底要观察哪些字段

把 `print(result.final_output)` 换成一次结构化打印：`final_output` 看回答、`last_agent` 看是谁给的、`to_input_list()` 看这一轮完整攒下了什么、以及 `usage` 看这次花了多少 token。观察的重点是「一次往返攒下了什么」，因为第 4 章的循环本质上就是不断把新的 `tool_use` 与 `tool_result` 追加进这份结构，再送回模型。官方文档把 `RunResult` 与 `RunResultStreaming` 分开写，流式场景另有 `stream_events`；本章先不碰流式，把同步往返看清楚即可。

### 故意写坏一次，才知道骨架在挡什么

把 `instructions` 改成空、把 `name` 写成一堆无意义字符、把密钥环境变量改名后重跑，分别观察失败发生在哪一层：是配置错误、是请求被拒、还是回答变得不可控。官方建议开发者从直接使用 LLM API 开始，因为很多模式几行代码就能实现；综合报告口径把基线校准阶段的目标写成「亲手见过一次失败长什么样」。这三种失败都是确定性的，正好用来验证第 10 章的单测该断言什么。**延伸，非语料原文**：把三次报错原文连同触发条件存进 docs/first-failures.md，它会是回归集里最早的三条负样本。

**今天怎么做**：照官方 quickstart 建目录与 `virtual environment`，装 `pip install openai-agents`，设 `export OPENAI_API_KEY=sk-...`，把两段最小代码跑通并打印 `final_output`、`last_agent`、`to_input_list()`；然后故意改坏配置三次，记录报错。**延伸，非语料原文**：把能复现这一整套的命令写进项目 `README`，第 12 章打包时会直接用到。

<div class="chapend" markdown="1">

**容易误解的地方**：以为「最小骨架」就是随便跑个 demo——它同时要交出可复现的环境、不外泄的密钥和可观察的字段；以为 `Runner.run` 只是个语法糖——它返回的 `RunResult` 决定了历史怎么续、结果怎么取；以为流式越早接越好——`RunResultStreaming` 属于另一条路径，本章先要一条能逐字段检查的同步往返；还有把官方示例里的 `sk-...` 当成可用的密钥格式，它只是文档里的占位写法。

**记住这一句**：第一份可交付产出物 = `virtual environment` + 锁定的依赖 + 运行时注入的密钥 + 一次能逐字段检查的 `Runner.run` 往返；先把一次调用做对、做可观察，再谈让它自己决定下一步。

</div>

# 第7章 状态与会话：跨轮次、可恢复、可中断

**一句话**：把「第二轮还能接上」和「进程重启还能接着跑」当成两个不同的问题——前者靠 `sessions` 或 `to_input_list()` 管会话历史，后者靠 `checkpointer` 与 `thread_id` 管图状态；两者都做对，长任务才有资格谈可恢复。

**为什么需要它**：官方 quickstart 在第一次运行之后立刻给出第二回合的三条路：把 `result.to_input_list()` 传回 `Runner.run(...)`、挂一个 `session=...`、或用 OpenAI 服务端状态 `conversation_id` / `previous_response_id`；并明确写了选型依据——要完全手动控制且与供应商无关的历史用 `result.to_input_list()`，要让 SDK 替你存取历史用 `session=...`，要服务端续接用 `previous_response_id` 或 `conversation_id`。三条路不是三种写法，是三种「状态归谁管」的决定。综合报告口径把「丢失对话历史 `2.8%`」和「会话重置 `2.2%`」分别列进 MAST 的两个类别，虽然占比不高，但都属于「Agent 忠实执行了一个残缺的上下文」——这类失败在日志里往往表现为一切正常，只是答案不对。

**核心结构**：状态分三层。会话层是消息历史，`SQLiteSession` 用 `session_id` 把历史落盘，`OpenAIConversationsSession` 则走服务端会话；不想引入持久层时，`to_input_list()` 让你自己拿着列表走。运行层是一次执行的进度，官方文档里 `RunState` 用于恢复暂停的运行或被 `cancel(mode="after_turn")` 停下的运行。图层是把整个流程建成图之后的节点状态：LangGraph 用 `StateGraph`、`add_node`、`add_edge`、`add_conditional_edges` 定义图，`compile` 时挂 `checkpointer`（示例常用 `MemorySaver` 或 `InMemorySaver`），运行时用 `thread_id` 区分同一张图的不同会话。`Durable execution` 这条官方能力描述的是「Build agents that persist through failures and can run for extended periods, automatically resuming from exactly where they left off」——注意它说的是从离开的那一点继续，不是从头重放。

**关键设计**：**可恢复的前提是「状态有边界」，而不是「什么都存」**。综合报告口径给长任务的对策是状态压缩与外部记忆，并点出上下文污染的表现为「长任务中段质量下滑，忘了开头的约束」；如果把每一轮的完整工具返回、完整提示词、完整检索片段全塞进会话表，恢复得越完整，污染得也越彻底。所以工单助手的会话层只存四类东西：用户消息、最终回复、`tool_use` 的参数摘要（不是完整返回体）、以及一个指向外部存储的引用键；大对象放外部，会话里只留键。第 8 章的上下文预算正是建在这条边界上。另一条设计要点是 `interrupt`：`Human in the loop` 不是等模型自己停下来问人，而是图在指定节点挂起，人批准后 `resume`；`Command` 用来把人工决定带回图里。这样「需要人工复核」才是一个可测的状态，而不是提示词里的一句请求。

**拆解**：

### 三条历史方案各自把状态放在哪

`to_input_list()`：状态在调用方手里，最透明、最与供应商无关，代价是每次都要自己传。`session=...` 配 `SQLiteSession` 与 `session_id`：状态在本地数据库，SDK 自动读写，重启后仍在。`previous_response_id` 或 `conversation_id`：状态在服务端，最省心，但与特定供应商的会话模型绑定。官方 quickstart 那张表把这三条并列，本身就是选型清单：工单助手要能在本地复现、要能把运行台账和会话一起归档，所以选 `SQLiteSession`，同时保留 `to_input_list()` 作为导出与回放的接口。**延伸，非语料原文**：无论选哪条，都先写一个「导出整条会话为 JSON」的小函数，第 10 章的回归集会直接依赖它。

### 图状态与会话历史不要混成一个东西

会话历史是「说过什么」，图状态是「走到哪一步、每个节点留下了什么」。前者是消息列表，后者是一个带结构的 `State`（常用 `TypedDict` 描述）。LangGraph 的 `checkpointer` 在每次节点执行后写快照，`thread_id` 决定读哪一条线；`interrupt` 让流程停在某个节点等人工输入，`resume` 之后从快照继续。把这两层混在一起的典型症状是：为了恢复运行，把整段历史重新喂给模型一遍，于是成本翻倍、上下文还被旧内容占满。综合报告口径把「状态压缩与外部记忆」和「窄上下文的子 agent」并列，讲的都是别让恢复动作变成重放动作。

### 可中断为什么是功能而不是降级

工单助手的「起草但不发送」天然是一个人工闸门。用 `interrupt` 在起草节点挂起，把草稿、引用片段与 needs_review 一起交给人；批准与否通过 `Command` 回灌，图从快照继续。官方 `Human in the loop` 文档把这类机制列为内建能力，LangGraph 的 `Durable execution` 则保证挂起期间进程重启也不丢。综合报告口径把「涉及资金、删除、外部发送、权限变更的操作必须审批或严格限制阈值」写进动作分级，并要求「每级升级前明确哪些动作无监督执行、停机的触发条件、回退路径」。可中断让这条要求变成一段可执行、可测试的代码，而不是一份操作规范。

### 断点续跑怎么验证才算数

至少做三次破坏性测试：跑到一半杀掉进程再用同一 `thread_id` 恢复；在 `interrupt` 节点等上一段时间再恢复；恢复时刻意改一版提示词，看系统有没有把「状态来自旧版本」记进运行台账。综合报告口径要求「模型、提示词、工具、数据任一变更都等于一次新版本，必须整体重跑评估并记录版本组合」——恢复运行同样要带上版本组合，否则第 10 章的回归报告会说不清分数变化来自改动还是来自旧状态。官方文档里 `RunState` 支持恢复暂停运行，正是这类测试的抓手。

**今天怎么做**：把 `SQLiteSession` 接进第 6 章的封装，用固定 `session_id` 跑两轮对话并确认第二轮接得上；再挑一个长任务，用 `StateGraph` 加 `add_node`、`add_edge`、`compile` 建最小图，挂 `checkpointer` 与 `thread_id`，做一次「中途杀进程 → 恢复」的测试。**延伸，非语料原文**：把测试用的 `thread_id`、提示词版本、schema 版本一起写进测试记录，第 12 章的发布记录会用到同一组字段。

<div class="chapend" markdown="1">

**容易误解的地方**：以为挂了 `session=...` 就等于可恢复——它管的是历史，不是执行进度；把 `checkpointer` 当成日志——它存的是状态快照，运行台账仍要单独记；以为 `interrupt` 是异常——它是设计好的挂起点；还有为了「恢复得完整」把整段上下文重放一遍，恰好制造了综合报告口径说的上下文污染。

**记住这一句**：状态层的产出物 = 一条能续的会话（`SQLiteSession` + `session_id`）+ 一张能恢复的图（`checkpointer` + `thread_id`）+ 一个能挂起等人工的节点（`interrupt`）；状态要有边界，恢复才不等于重放。

</div>

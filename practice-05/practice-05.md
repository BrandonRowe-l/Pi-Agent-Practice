# 实践课5：Pi Agent 工具调用与会话恢复

> 源码仓库：`https://github.com/System-Analysis-Homework/pi-hunnu-2026`
> 本组随机标识：`river-731`（写入 `handoff.txt`，提问中未透露命令）
> 本地构建启动：`node .\packages\coding-agent\dist\cli.js`（1.5.2，DeepSeek deepseek-flash · low）

## A. 单轮交互与 read 工具调用

在项目根目录创建 `handoff.txt`，写入两行：

```text
待办：补充 README 的启动说明
启动命令：npm run demo -- --profile river-731
```

启动本地构建的 Pi，在新会话输入："请用 read 工具读取当前目录的 handoff.txt，原样告诉我待办和启动命令。"

**观察结果**：一次用户输入，本轮共 **2 次模型调用、1 次工具执行**：

1. 第 1 次模型调用返回 `read({path:"handoff.txt"})` 工具调用；
2. Agent 循环实际执行工具一次（工具记录显示 `read handoff.txt`），文件内容作为 toolResult 写回上下文；
3. 第 2 次模型调用后，Pi 原样报告两行内容，包括完整命令 `npm run demo -- --profile river-731`。工具输出还如实检测到文件开头的 UTF-8 BOM 并在回复中注明。

![A 阶段：read 工具读取 handoff.txt 并原样报告](./assets/S1-handoff-read.png)

## 图 1：单轮工具调用的代码级时序图

以实际读取 `handoff.txt` 的那一轮为例，按职责分五条生命线，箭头与注释标注真实函数/语句（行号为源码当前行号）：

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant UI as 交互与会话层<br/>interactive-mode.ts / agent-session.ts
    participant AG as Agent 执行层<br/>agent.ts / agent-loop.ts
    participant AI as 模型请求层<br/>model-runtime.ts / openai-completions.ts / DeepSeek API
    participant RD as read 工具<br/>tools/read.ts（文件系统）

    U->>UI: 输入"请用 read 工具读取 handoff.txt…"
    Note over UI: setupEditorSubmitHandler 设置 onSubmit（interactive-mode.ts:3324）
    UI->>UI: session.prompt(text,{streamingBehavior:"steer"})(:3506)
    Note over UI: AgentSession.prompt(:2175)→_preparePrompt(:2215)→_startRun(:1956)
    UI->>AG: agent.prompt(messages)(agent-session.ts:1982)
    AG->>AG: Agent.prompt(:382)→runPromptMessages(:441)→runAgentLoop(...,streamFunction)(:446)
    AG->>AI: streamAssistantResponse(:391)<br/>await streamFunction(...)(agent-loop.ts:414)
    Note over AI: model-runtime.ts:744 streamSimple→openai-completions.ts:312 stream
    AI->>AI: POST https://api.deepseek.com（OpenAI SDK, stream:true, :818）
    AI-->>AG: AssistantMessageEventStream 流式增量，含 tool_call: read({path:"handoff.txt"})
    Note over AG: 模型决定调用 read 工具，工具执行由 Agent 循环驱动
    AG->>RD: 执行 read 工具 execute(tools/read.ts:245)
    RD->>RD: fs/promises.readFile 缓冲读取(:69 defaultReadOperations)
    Note over RD: ≤2000 行且 ≤50KB 不截断（truncate.ts:11-12）；本例检测到 UTF-8 BOM
    RD-->>AG: 文件内容（待办 + 启动命令 river-731）
    AG->>AG: createToolResultMessage(:935)→emitToolResultMessage(:951)<br/>currentContext.messages.push(toolResult)
    AG->>AI: 第 2 次 streamFunction 调用（请求携带 toolResult 消息）
    AI-->>AG: 助手回复流：原样报告待办与启动命令
    AG-->>UI: 会话事件流（message_start / message_update / message_end）
    Note over UI: subscribeToAgent(:3525)；message_update(:3649) updateContent 增量渲染；message_end(:3685) 定稿
    UI-->>U: 终端显示待办与启动命令全文
```

**read 工具读取方式（按源码核对）**：默认路径用 `fs/promises.readFile`（`tools/read.ts:69`），即**缓冲区读取**，一次性读完再截断；只有走 `handle.createReadStream`（`readLocalFileChunks`，:81/:142）的路径才是流式读取，用于按行选择等场景。超限保护为 `truncate.ts` 的 `DEFAULT_MAX_LINES=2000`、`DEFAULT_MAX_BYTES=50KB`，超限执行 `truncateHead` 并附 offset 提示——本例文件仅两行，未触发截断。

**显示回复的位置**：`interactive-mode.ts:3525` `subscribeToAgent` 订阅会话事件；`message_update`（:3649）调用 `AssistantMessageComponent.updateContent` 增量渲染，`message_end`（:3685）定稿——用户看到回复就在这两处。

## B. 恢复上次会话，继续写 README

**1. 恢复原会话（-c）**：退出 Pi 后执行 `node .\packages\coding-agent\dist\cli.js -c`，终端完整恢复显示 A 阶段聊天记录（提问、`read handoff.txt` 工具记录、回复全文）：

![S2：-c 恢复后看到 A 阶段完整记录](./assets/S2-continue-history.png)

在同一会话输入指定提问（要求不调用工具）。**观察结果**：回答起草了 README 启动说明，给出完整命令 `npm run demo -- --profile river-731`，**没有调用任何工具**——命令来自已恢复的会话历史（A 阶段的 toolResult），支持它使用了恢复的历史消息：

![S3：恢复会话中回答含完整命令且未调用工具](./assets/S3-continue-answer.png)

**2. 新建会话（/new）**：在同一目录、同一模型下输入完全相同的话。**观察结果**：回答"无法起草：这次会话里没有任何'上次的待办'上下文，也没有交接单内容"，明确说明这是本会话第一条消息，**没有调用工具**，也未出现命令 `river-731`：

![S4：新会话中无法回答，缺少上下文](./assets/S4-new-session.png)

**对照结论**：恢复会话的回答包含 A 阶段完整命令且未重新读文件，有证据支持其使用了已恢复的会话历史；新会话历史为空，模型只能如实说明缺少命令。

## 图 2：恢复旧会话与新建会话的对照图

两条分支均从"A 阶段已结束（Pi 已读到并回答了启动命令）"开始：

```mermaid
flowchart TD
    START(["A 阶段已结束：Pi 已读到并回答了启动命令<br/>会话已保存为 ~/.pi/agent/sessions/&lt;cwd&gt;/*.jsonl（JSONL）"])
    START --> EXIT1["退出 Pi（/exit）"]
    START --> NEWCMD["输入 /new（interactive-mode.ts:3429 → handleClearCommand :7117）"]

    subgraph LEFT["左支：恢复旧会话（-c）"]
        C1["node cli.js -c（args.ts:111 解析 --continue）"]
        C2["SessionManager.continueRecent（session-manager.ts:1811）<br/>findMostRecentSession 加载最近会话文件"]
        C3["从 JSONL 恢复历史消息：<br/>① 用户提问 ② toolResult（handoff.txt 全文）③ 助手回复"]
        C4["用户输入：接着上次的待办，给 README 起草启动说明…"]
        C5["发给模型 = 恢复的历史(①②③) + 本次提问"]
        C6["基于已有上下文作答：报告完整命令<br/>npm run demo -- --profile river-731（实测未调用工具）"]
    end

    subgraph RIGHT["右支：新建会话（/new）"]
        N1["runtimeHost.newSession()（agent-session-runtime.ts:226<br/>→ SessionManager.newSession :1065）"]
        N2["创建聊天历史为空的新会话"]
        N3["用户输入完全相同的提问"]
        N4["发给模型 = 仅本次提问（历史中无 A 阶段 toolResult）"]
        N5["模型无 handoff.txt 上下文：回答无法起草、<br/>会话里没有上次待办/交接单内容（未调用工具）"]
    end

    EXIT1 --> C1 --> C2 --> C3 --> C4 --> C5 --> C6
    NEWCMD --> N1 --> N2 --> N3 --> N4 --> N5

    classDef leftNode fill:#dcfce7,stroke:#22c55e
    classDef rightNode fill:#fee2e2,stroke:#ef4444
    class C1,C2,C3,C4,C5,C6 leftNode
    class N1,N2,N3,N4,N5 rightNode
```

**对应源码**：会话以 JSONL 形式存放在 `~/.pi/agent/sessions/<cwd编码>/`（`session-manager.ts:587` `getDefaultSessionDir`，文件名 `<时间戳>_<id>.jsonl`）。`-c` 由 `cli/args.ts:111` 解析，经 `main.ts:480` 进入 `session-manager.ts:1811` `SessionManager.continueRecent`，用 `findMostRecentSession` 找到最近的会话文件并恢复消息；`/new` 走 `interactive-mode.ts:3429` → `handleClearCommand`（:7117）→ `runtimeHost.newSession()`（`agent-session-runtime.ts:226`）→ `SessionManager.newSession`（:1065），聊天历史为空。两种情况下发给模型的消息集合差异（是否包含 A 阶段的 toolResult）正是实测行为差异的直接原因。

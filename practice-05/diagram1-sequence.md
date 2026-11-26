%% 图1：单轮工具调用的代码级时序图（读取 handoff.txt 的那一轮）
%% 五条生命线按职责划分；箭头与注释均标注真实函数/语句（文件见 pi-hunnu-2026 仓库）
sequenceDiagram
    autonumber
    participant U as 用户
    participant UI as 交互与会话层<br/>interactive-mode.ts / agent-session.ts
    participant AG as Agent 执行层<br/>agent.ts / agent-loop.ts
    participant AI as 模型请求层<br/>model-runtime.ts / openai-completions.ts / DeepSeek API
    participant RD as read 工具<br/>tools/read.ts（文件系统）

    U->>UI: 输入"请用 read 工具读取当前目录的 handoff.txt…"
    Note over UI: setupEditorSubmitHandler<br/>interactive-mode.ts:3324 设置 onSubmit
    UI->>UI: session.prompt(text,{streamingBehavior:"steer"})<br/>interactive-mode.ts:3506
    Note over UI: AgentSession.prompt :2175<br/>→ _preparePrompt :2215<br/>→ _startRun :1956
    UI->>AG: agent.prompt(messages)<br/>agent-session.ts:1982
    AG->>AG: Agent.prompt :382<br/>→ runPromptMessages :441<br/>→ runAgentLoop(...,streamFunction) :446
    AG->>AI: streamAssistantResponse :391<br/>await streamFunction(...) agent-loop.ts:414
    Note over AI: model-runtime.ts:744 streamSimple<br/>→ openai-completions.ts:312 stream
    AI->>AI: POST https://api.deepseek.com<br/>OpenAI SDK stream:true :818
    AI-->>AG: AssistantMessageEventStream（流式增量）<br/>tool_call: read({path:"handoff.txt"})
    Note over AG: 模型决定调用 read 工具<br/>（工具执行由 Agent 循环驱动）
    AG->>RD: 执行 read 工具 execute<br/>tools/read.ts:245
    RD->>RD: fs/promises.readFile (缓冲读取)<br/>tools/read.ts:69 defaultReadOperations
    Note over RD: 文件≤2000行且≤50KB 不截断<br/>truncate.ts:11-12；本例检测到 UTF-8 BOM
    RD-->>AG: 文件内容（待办+启动命令 river-731）
    AG->>AG: createToolResultMessage :935<br/>emitToolResultMessage :951
    AG->>AG: currentContext.messages.push(toolResult)<br/>（agent-loop.ts:284 附近，实践04加过日志）
    AG->>AI: 第二次 streamFunction 调用<br/>（请求携带 toolResult 消息）
    AI-->>AG: 助手回复流：原样报告待办与启动命令
    AG-->>UI: 会话事件流（message_start/update/end）
    Note over UI: subscribeToAgent interactive-mode.ts:3525<br/>message_update :3649 updateContent 增量渲染<br/>message_end :3685 定稿
    UI-->>U: 终端显示"待办：补充 README 的启动说明<br/>启动命令：npm run demo -- --profile river-731"

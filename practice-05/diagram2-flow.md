%% 图2：恢复旧会话（-c）与新建会话（/new）的对照流程图
%% 说明两种情况下 Pi 分别把哪些聊天消息发给模型
%% A 阶段已结束：Pi 已读到并回答了启动命令（消息序列：用户提问 → toolResult(handoff.txt) → 助手回复）
flowchart TD
    START(["A 阶段已结束：Pi 已读到并回答了启动命令<br/>会话已保存为 ~/.pi/agent/sessions/&lt;cwd&gt;/*.jsonl<br/>session-manager.ts:587 getDefaultSessionDir"])

    START --> EXIT1["退出 Pi（/exit）"]
    START --> NEWCMD["输入 /new<br/>interactive-mode.ts:3429 → handleClearCommand :7117"]

    subgraph LEFT["左支：恢复旧会话（-c）"]
        C1["node cli.js -c<br/>args.ts:111 解析 --continue"]
        C2["SessionManager.continueRecent<br/>session-manager.ts:1811<br/>findMostRecentSession 找到最近会话文件"]
        C3["从 JSONL 恢复历史消息：<br/>① 用户提问<br/>② toolResult（handoff.txt 全文，含启动命令）<br/>③ 助手回复"]
        C4["用户输入：接着上次的待办，给 README 起草启动说明…"]
        C5["发给模型的消息 = 恢复的历史(①②③) + 本次提问<br/>（newSession 历史非空）"]
        C6["模型基于已有上下文作答<br/>报告完整命令 npm run demo -- --profile river-731<br/>无需重新读文件（实测未调用工具）"]
    end

    subgraph RIGHT["右支：新建会话（/new）"]
        N1["runtimeHost.newSession()<br/>agent-session-runtime.ts:226<br/>→ SessionManager.newSession :1065"]
        N2["创建聊天历史为空的新会话"]
        N3["用户输入完全相同的提问"]
        N4["发给模型的消息 = 仅本次提问<br/>（历史中无 A 阶段的 toolResult）"]
        N5["模型无 handoff.txt 上下文<br/>实测回答：无法起草，会话里没有上次待办/交接单内容<br/>（遵守不调用工具，未读文件）"]
    end

    EXIT1 --> C1 --> C2 --> C3 --> C4 --> C5 --> C6
    NEWCMD --> N1 --> N2 --> N3 --> N4 --> N5

    classDef leftNode fill:#dcfce7,stroke:#22c55e
    classDef rightNode fill:#fee2e2,stroke:#ef4444
    class C1,C2,C3,C4,C5,C6 leftNode
    class N1,N2,N3,N4,N5 rightNode

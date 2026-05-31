# Claude Code 核心机制图解

## 1. 整体架构：大脑、神经、手脚

```mermaid
graph TB
    subgraph 大脑["🧠 LLM (Claude API)"]
        direction LR
        Think["思考 + 决策<br/>只输出文字或工具调用请求<br/>不接触文件系统"]
    end

    subgraph 手脚["🛠️ 工具系统"]
        direction LR
        Bash["Bash<br/>执行命令"]
        Read["Read<br/>读文件"]
        Edit["Edit<br/>编辑文件"]
        Write["Write<br/>写文件"]
        Grep["Grep<br/>搜索内容"]
        Glob["Glob<br/>匹配文件"]
    end

    subgraph 神经["🔁 QueryEngine + query.ts"]
        Loop["while(true) 循环<br/>──心跳──"]
        State["消息历史管理<br/>──记忆──"]
        Permission["权限检查<br/>──免疫系统──"]
    end

    User["👤 用户"] -->|输入消息| State
    State -->|组装消息+工具定义| Loop
    Loop -->|POST /v1/messages<br/>system prompt + 历史 + 工具列表| Think
    Think -->|流式响应<br/>text ➜ 展示<br/>tool_use ➜ 行动| Loop
    Loop -->|检查权限后执行| 手脚
    手脚 -->|执行结果 tool_result| Loop
    Loop -->|追加到消息列表| State
    State -->|下一轮循环| Loop
    Loop -->|stop_reason=end_turn<br/>"我做完了"| User

    style 大脑 fill:#e8f5e9,stroke:#2e7d32
    style 手脚 fill:#fff3e0,stroke:#e65100
    style 神经 fill:#e3f2fd,stroke:#1565c0
```

---

## 2. query.ts 的 while(true) 死循环

```mermaid
flowchart TD
    Start(["🚀 query() 被调用"]) --> Init["初始化消息列表<br/>组装 System Prompt<br/>组装工具定义"]
    Init --> Loop{"while(true) 循环开始"}

    Loop --> Compact["① 上下文压缩<br/>━━━━━━━━━━<br/>微压缩: 删除过时工具结果<br/>自动压缩: 上下文太长时用摘要替代历史<br/>上下文坍缩: 合并旧消息为概要"]

    Compact --> CallAPI["② 调用 LLM<br/>━━━━━━━━━━<br/>POST api.anthropic.com/v1/messages<br/>带着: system prompt + 全部历史消息 + 工具定义<br/>流式读取响应"]

    CallAPI --> Parse{"③ 解析 LLM 回复"}

    Parse -->|"text 文本块"| ShowUser["展示给用户"]
    Parse -->|"tool_use 工具调用"| MarkFollow["标记 needsFollowUp = true<br/>加入待执行队列"]

    ShowUser --> CollectAll["收集完所有 chunk"]

    MarkFollow --> CollectAll

    CollectAll --> Check{"④ 有工具要执行？<br/>(needsFollowUp?)"}

    Check -->|"是"| CheckPerm["权限检查<br/>canUseTool()"]
    CheckPerm -->|allow| Execute["执行工具<br/>━━━━━━━━━━<br/>Bash ➜ execa()<br/>Read ➜ fs.readFile()<br/>Edit ➜ 替换后写入<br/>Write ➜ fs.writeFile()<br/>Grep ➜ ripgrep"]
    CheckPerm -->|deny| DenyResult["返回错误给 LLM"]
    CheckPerm -->|ask| AskUser["弹窗问用户"]
    AskUser -->|确认| Execute
    AskUser -->|拒绝| DenyResult

    Execute --> Result["构造 tool_result 消息"]
    DenyResult --> Result

    Result --> Append["追加到消息列表:<br/>...之前的消息<br/>+ assistant 消息(含 tool_use 块)<br/>+ user 消息(含 tool_result 块)"]

    Append --> Attachments["附加额外上下文:<br/>- 记忆相关内容<br/>- 技能发现<br/>- 文件变动通知<br/>- 队列中的命令"]
    Attachments --> CheckLimits["检查是否超限:<br/>- 最大回合数?<br/>- 预算上限?<br/>- Token 配额?"]
    CheckLimits -->|"未超限"| Loop

    Check -->|"否"| StopHooks["执行 Stop Hooks"]
    StopHooks --> Finished(["✅ 返回结果<br/>stop_reason = end_turn<br/>退出循环"])

    CheckLimits -->|"超限"| ErrorEnd(["❌ 错误结束<br/>max_turns / max_budget"])

    style CallAPI fill:#e8f5e9,stroke:#2e7d32
    style Execute fill:#fff3e0,stroke:#e65100
    style Loop fill:#e3f2fd,stroke:#1565c0
    style Check fill:#fce4ec,stroke:#c62828
```

---

## 3. LLM 请求/响应格式

```mermaid
sequenceDiagram
    participant CLI as Claude Code
    participant API as Anthropic API
    participant Tool as 工具系统

    CLI->>CLI: 组装 system prompt + 工具定义 + 历史消息

    CLI->>API: POST /v1/messages<br/>{<br/>  model: "claude-sonnet-4-6",<br/>  system: [...],<br/>  messages: [...],<br/>  tools: [<br/>    {name:"Bash", input_schema:{...}},<br/>    {name:"Read", input_schema:{...}},<br/>    {name:"Edit", input_schema:{...}}<br/>  ],<br/>  stream: true<br/>}

    API-->>CLI: SSE stream: event: content_block_start<br/>data: {"type":"text","text":"我来帮你..."}

    API-->>CLI: SSE stream: event: content_block_start<br/>data: {"type":"tool_use","name":"Bash",<br/>"input":{"command":"ls *.js"}}

    CLI->>CLI: 解析到 tool_use 块<br/>needsFollowUp = true

    CLI->>Tool: canUseTool("Bash", {command: "ls *.js"})
    Tool-->>CLI: allow

    CLI->>Tool: execute("Bash", {command: "ls *.js"})
    Tool->>Tool: execa("ls *.js")
    Tool-->>CLI: {content: "a.js\nb.js\nc.js"}

    CLI->>CLI: 追加 tool_result 到消息列表<br/>准备下一轮循环

    CLI->>API: POST /v1/messages<br/>{<br/>  messages: [<br/>    ...之前的消息,<br/>    {role:"assistant", content:[{type:"tool_use",...}]},<br/>    {role:"user", content:[{type:"tool_result",...}]}<br/>  ]<br/>}

    API-->>CLI: SSE stream: event: content_block_start<br/>data: {"type":"text","text":"找到了3个文件，现在重命名"}

    API-->>CLI: SSE stream: event: content_block_start<br/>data: {"type":"tool_use","name":"Bash",<br/>"input":{"command":"mv a.js a.ts && mv b.js b.ts"}}

    Note over CLI,Tool: 重复工具执行流程...

    API-->>CLI: SSE stream: event: message_delta<br/>data: {"stop_reason":"end_turn"}

    CLI->>CLI: needsFollowUp = false<br/>退出循环 ✅
```

---

## 4. 工具并发/串行编排

```mermaid
flowchart LR
    subgraph 工具分类
        A1["Read ./a.ts"]
        A2["Grep 'import' ./src/"]
        A3["Read ./b.ts"]
        B1["Edit ./a.ts"]
        B2["Bash rm temp.txt"]
        C1["Read ./c.ts"]
    end

    subgraph 第1批并发["⚡ 并发执行(只读)"]
        direction LR
        A1 & A2 & A3
    end

    subgraph 第2批串行["🔒 串行执行(写操作)"]
        direction LR
        B1 --> B2
    end

    subgraph 第3批并发["⚡ 并发执行(只读)"]
        direction LR
        C1
    end

    工具分类 --> 第1批并发
    第1批并发 --> 第2批串行
    第2批串行 --> 第3批并发

    style 第1批并发 fill:#e8f5e9,stroke:#2e7d32
    style 第2批串行 fill:#fff3e0,stroke:#e65100
    style 第3批并发 fill:#e8f5e9,stroke:#2e7d32
```

---

## 5. 完整回合示例

```mermaid
sequenceDiagram
    actor User
    participant QE as QueryEngine
    participant Query as query.ts 循环
    participant LLM as Claude API
    participant Tools as 工具系统
    participant FS as 文件系统

    User->>QE: "把所有 .js 改成 .ts"

    QE->>QE: 组装 System Prompt<br/>+ 工具列表 + 环境信息
    QE->>QE: processUserInput()<br/>处理用户输入

    QE->>Query: query(messages, systemPrompt, tools)

    rect rgb(232, 245, 233)
        Note over Query,FS: 回合 1 — 探索
        Query->>LLM: 消息: "把所有 .js 改成 .ts"<br/>工具: Bash/Read/Edit/Write...
        LLM-->>Query: text: "我先看看有哪些 js 文件"
        LLM-->>Query: tool_use: Bash("find . -name '*.js'")
        Query->>Tools: 执行 Bash
        Tools->>FS: find . -name '*.js'
        FS-->>Tools: a.js\nb.js\nc.js
        Tools-->>Query: tool_result
    end

    rect rgb(255, 243, 224)
        Note over Query,FS: 回合 2 — 行动
        Query->>LLM: 消息: [...历史 + tool_use + tool_result]
        LLM-->>Query: tool_use: Bash("for f in *.js; do mv $f ${f%.js}.ts; done")
        Query->>Tools: 权限检查 → ask
        Tools-->>User: 弹窗: "允许执行 mv 命令?"
        User-->>Tools: 确认
        Tools->>FS: 批量 mv 重命名
        FS-->>Tools: 成功
        Tools-->>Query: tool_result
    end

    rect rgb(227, 242, 253)
        Note over Query,FS: 回合 3 — 结束
        Query->>LLM: 消息: [...历史 + tool_use + tool_result]
        LLM-->>Query: text: "完成，3个文件已重命名"
        LLM-->>Query: stop_reason: "end_turn"
        Query->>Query: needsFollowUp = false<br/>退出循环
    end

    Query-->>QE: 最终结果
    QE-->>User: ✅ "完成，3个文件已重命名"
```

---

## 6. 一句话总结

```mermaid
flowchart LR
    A["LLM<br/>🧠 大脑<br/>只思考不行动"] -->|tool_use 调用请求| B["query.ts 循环<br/>🔁 心跳<br/>while(true)"]
    B -->|执行并获取结果| C["工具系统<br/>🛠️ 手脚<br/>只执行不思考"]
    C -->|tool_result 执行结果| B
    B -->|追加到消息列表<br/>再次请求| A
    A -->|end_turn 做完了| D["✅ 退出"]
```

**LLM 是大脑（只思考不行动），工具是手脚（只执行不思考），QueryEngine 是神经系统（在两者间传递信息），query.ts 里的 `while(true)` 就是心跳。**

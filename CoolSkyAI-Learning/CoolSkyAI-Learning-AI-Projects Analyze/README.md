## 分析AI开源项目来倒逼自己学习AI相关的知识点
本次要分析的AI开源项目有：
>Hermes:https://github.com/NousResearch/hermes-agent
> 
> Tinystruct: https://github.com/tinystruct
> 
> Solon: https://github.com/opensolon/solon-ai


好，我们现在正式开始。

这一轮**先不分析具体实现代码**，先把两个项目的“程序是怎么跑起来的”画出来。你作为 Java 开发者，第一件事就是建立一个非常重要的认知：

> **Hermes 和 tinystruct 的“入口”完全不是 Spring Boot 那种 `main() → Controller → Service` 模式。**

而这恰好是我们这次学习非常好的切入点。

---

# 一、第一张源码地图

先把两个项目放在一起：

```text
                    用户输入
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
     Hermes Agent              tinystruct App
       Python                    Java
          │                         │
          ▼                         ▼
   CLI / Gateway              dispatcher / HTTP
          │                         │
          ▼                         ▼
      AIAgent                 AbstractApplication
          │                         │
          ▼                         ▼
 run_conversation()             @Action
          │                         │
          ▼                         ▼
    Agent Loop                ActionRegistry
          │                         │
          ▼                         ▼
     Prompt + LLM              Java业务方法
          │
          ▼
      Tool Calling
          │
          ▼
      Tool Executor
          │
          └─────── 再回到 LLM ───────┘
```

Hermes 官方架构文档明确把 `AIAgent` 作为核心编排对象，而真正的 Agent Loop 已经拆到了 `agent/conversation_loop.py` 和多个 `agent/turn_*.py` 中。([GitHub][1])

tinystruct 则完全不同：应用继承 `AbstractApplication`，业务方法通过 `@Action` 暴露，然后由 `bin/dispatcher` 或 HTTP 服务器负责启动和路由；官方 README 明确强调应用不需要自己写 `main()`。([GitHub][2])

---

# 二、Hermes：从哪里开始？

我们先看 Hermes。

源码地图可以先记成：

```text
Hermes
│
├── CLI
│   └── hermes_cli/main.py
│
├── AIAgent
│   └── run_agent.py
│
└── Agent Core
    │
    ├── agent/agent_init.py
    │
    ├── agent/conversation_loop.py
    │
    ├── agent/turn_*.py
    │
    ├── agent/prompt_builder.py
    │
    ├── agent/context_engine.py
    │
    ├── agent/context_compressor.py
    │
    ├── agent/tool_executor.py
    │
    └── model_tools.py
```

官方文档现在明确给出了这个结构：

```text
CLI
 ↓
AIAgent
 ↓
run_conversation()
 ↓
conversation_loop.py
 ↓
turn_*.py
 ↓
LLM
 ↓
┌───────────────┐
│ text response │ → 返回
└───────────────┘

        或

┌────────────────┐
│ tool_calls     │
└───────┬────────┘
        ↓
tool_executor
        ↓
执行 Tool
        ↓
tool result
        ↓
重新进入 Agent Loop
```

这就是我们真正要学习的 **Agent Loop**。([GitHub][1])

---

# 三、Hermes 第一层：`run_agent.py`

这里先不要把它理解成：

```java
public static void main(String[] args)
```

而应该理解成：

> **AIAgent 的公共门面（Facade）**

官方文档现在明确说明：

```text
run_agent.py
    ↓
AIAgent
    ↓
agent/conversation_loop.py
```

`run_agent.py` 是对外入口/Facade，而 Agent Loop 本身已经被拆到了 `agent/conversation_loop.py` 和 `agent/turn_*.py`。([GitHub][1])

所以我们第一阶段会重点看：

```text
run_agent.py
     │
     │ AIAgent
     ▼
agent/conversation_loop.py
     │
     │ run_conversation()
     ▼
agent/turn_*.py
```

---

# 四、Hermes 最重要的一张图

这一张你建议直接记下来：

```text
User
 │
 ▼
AIAgent
 │
 ▼
run_conversation()
 │
 ├── ① 加入 User Message
 │
 ├── ② 构建 System Prompt
 │
 ├── ③ 检查 Context
 │
 ├── ④ 构建 Messages
 │
 ├── ⑤ 调用 LLM
 │
 ▼
LLM Response
 │
 ├───────────────┐
 │               │
 ▼               ▼
Text          Tool Call
 │               │
 ▼               ▼
返回用户       Tool Executor
                 │
                 ▼
             Tool Result
                 │
                 ▼
             加入 Messages
                 │
                 ▼
              再次调用 LLM
                 │
                 └──────────→
```

这张图实际上已经把我们之前列出的很多基础概念串起来了：

```text
Token
   ↓
Context Window
   ↓
Messages
   ↓
Prompt
   ↓
LLM
   ↓
Tool Calling
   ↓
Tool Result
   ↓
再次进入 Context
```

也就是说：

> **我们不是先学 Token，再学 Context，再学 Prompt……**
>
> 而是从真实 Agent Loop 中，把这些概念一个一个“逼”出来。

这正是我们之前确定的学习方法。

---

# 五、然后看 tinystruct

tinystruct 的地图完全不同。

```text
tinystruct
│
├── bin/
│   └── dispatcher
│
├── src/main/java/
│
│   ├── AbstractApplication
│   │
│   ├── @Action
│   │
│   ├── ActionRegistry
│   │
│   ├── Context
│   │
│   └── HTTP / CLI
│
└── Application
      │
      └── @Action
```

官方 README 给出的最小应用就是：

```java
public class example extends AbstractApplication {

    @Override
    public void init() {
    }

    @Override
    public String version() {
        return "1.0";
    }

    @Action("praise")
    public String praise() {
        return "Praise the Lord!";
    }
}
```

然后不是：

```java
public static void main(String[] args)
```

而是：

```bash
bin/dispatcher ...
```

来启动和路由应用。([GitHub][2])

---

# 六、所以 tinystruct 的“入口”在哪里？

这是我们第一个非常值得研究的问题。

传统 Java：

```text
main()
 ↓
Application
 ↓
Controller
 ↓
Service
 ↓
Repository
```

tinystruct：

```text
bin/dispatcher
       │
       ▼
Application
       │
       ▼
AbstractApplication
       │
       ▼
@Action
       │
       ▼
ActionRegistry
       │
       ▼
真正的方法
```

也就是说：

> **tinystruct 把传统 Java 应用里的“启动 + 路由 + 方法发现”交给了框架。**

所以我们下一步不能傻找：

```java
public static void main(String[] args)
```

因为它本来就不是这个框架的核心设计。

官方 README 也明确写了：

> No `main()` method required

并且 CLI 与 Web 共用统一的应用模型。([GitHub][2])

---

# 七、这时候 Hermes 和 tinystruct 开始产生有意思的对应关系

我们可以先做第一版映射：

| 问题         | Hermes                 | tinystruct               |
| ---------- | ---------------------- | ------------------------ |
| 程序从哪里进入？   | CLI → AIAgent          | dispatcher → Application |
| 核心对象是谁？    | `AIAgent`              | `AbstractApplication`    |
| 请求怎么进入？    | `run_conversation()`   | `@Action`                |
| 核心调度在哪里？   | Agent Loop             | ActionRegistry           |
| 业务/能力在哪里？  | Tools                  | `@Action` 方法             |
| 状态在哪里？     | Conversation / Session | Context / Session 等      |
| AI 在哪里？    | LLM Provider           | 后续接入 LLM                 |
| 循环在哪里？     | Agent Loop             | 普通请求通常是一次 Action         |
| 下一步行动是谁决定？ | LLM                    | 程序/路由                    |

这里出现了一个**非常重要的区别**：

### tinystruct

```text
用户
 ↓
Action
 ↓
执行
 ↓
返回
```

### Hermes

```text
用户
 ↓
LLM
 ↓
决定下一步
 ↓
Tool
 ↓
Tool Result
 ↓
LLM
 ↓
决定下一步
 ↓
Tool
 ↓
...
 ↓
最终答案
```

这就是为什么我们要把 Hermes 作为 **AI Agent 的核心研究对象**。

---

# 八、而 tinystruct 为什么仍然非常重要？

因为我们要研究一个问题：

> **AI Agent 最终怎么落到 Java 工程里面？**

例如：

```text
Hermes

LLM
 │
 ▼
Tool Call
 │
 ▼
Tool Executor
 │
 ▼
Tool
```

我们将来可以把它映射成：

```text
Java AI Application

LLM
 │
 ▼
Tool Call
 │
 ▼
Tool Router
 │
 ▼
tinystruct @Action
 │
 ▼
Java Method
```

于是：

```java
@Action("weather")
public String weather(String city) {
    return weatherService.query(city);
}
```

从 AI 的角度看，它可能不是简单的 Java 方法。

而是：

```text
AI 能力
   ↓
Tool
   ↓
Action
   ↓
Java Method
```

这就开始进入我们真正想学的：

> **AI + Java 应用开发**

---

# 九、第一张“源码地图”最终版

我建议你把这一张作为我们接下来分析源码的总地图：

```text
                         ┌─────────────────────┐
                         │       User          │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
              Hermes Agent                    tinystruct
                    │                               │
                    ▼                               ▼
               AIAgent                         Dispatcher
                    │                               │
                    ▼                               ▼
          run_conversation()              Application
                    │                               │
                    ▼                               ▼
            Agent Loop                  AbstractApplication
                    │                               │
        ┌───────────┼───────────┐                   │
        │           │           │                   ▼
        ▼           ▼           ▼                @Action
     Prompt       Context      LLM                  │
        │           │           │                   ▼
        │           │           ▼              Java Method
        │           │       Response
        │           │           │
        │           │      ┌────┴────┐
        │           │      │         │
        │           │      ▼         ▼
        │           │    Text      Tool Call
        │           │                │
        │           │                ▼
        │           │          Tool Executor
        │           │                │
        │           │                ▼
        │           │             Tool
        │           │                │
        │           │                ▼
        │           │           Tool Result
        │           │                │
        │           └────────────────┘
        │                    │
        └────────────────────┘
                 再次进入 Agent Loop
```

---

# 十、接下来我们真正开始“追代码”

现在地图已经有了，**下一步不要继续看架构文档了**。

我们直接进入源码。

我建议严格按照这个顺序：

```text
第 1 步
Hermes
run_agent.py
   ↓
AIAgent
   ↓
run_conversation()

第 2 步
Hermes
agent/conversation_loop.py
   ↓
找到真正的 Agent Loop

第 3 步
tinystruct
AbstractApplication.java
   ↓
理解 Java Application 生命周期

第 4 步
tinystruct
@Action
   ↓
ActionRegistry
   ↓
理解 Java 方法是怎么被发现/调用的

第 5 步
重新回 Hermes
   ↓
LLM 调用
   ↓
Tool Calling

第 6 步
把两边接起来
   ↓
Hermes Tool
       ≈
tinystruct @Action
```

而且这里马上会出现我们第一个真正的基础问题：

> **“一个用户输入的字符串，到底是怎么一步一步变成 LLM API 的 `messages` 的？”**

这个问题一旦追通，**Token、Message、System/User/Assistant、Prompt、Context Window、LLM API** 会一下子串起来。

所以我们下一轮就直接从 **Hermes `AIAgent.run_conversation()` → `agent/conversation_loop.py`** 开始，一行一行追第一条 Agent Loop。官方当前文档也明确把 `run_conversation()` 指向这里，并给出了它的实际生命周期。([GitHub][1])

[1]: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/agent-loop.md?utm_source=chatgpt.com "hermes-agent/website/docs/developer-guide/agent-loop.md at main · NousResearch/hermes-agent · GitHub"
[2]: https://github.com/tinystruct/tinystruct?utm_source=chatgpt.com "GitHub - tinystruct/tinystruct: A lightweight, modular Java application framework for web and CLI development, designed for AI integration and plugin-based architecture. Enabling developers to create robust solutions with ease for building efficient and scalable applications. · GitHub"

# Day 001：LLM 基础——Token

## 🎯 今日目标



## 📚 今日学习
### 1、Hermes的CLI中hermes_cli/main.py是入口吗？
![img_1.png](img_1.png)
如果你指的是 hermes_cli/main.py，可以把它理解成 Hermes 的 CLI（命令行客户端）入口。
![img_2.png](img_2.png)
但这里要区分两个概念：
```text
用户在终端执行命令
        │
        ▼
hermes_cli/main.py       ← CLI 入口
        │
        ▼
解析命令行参数 / 初始化
        │
        ▼
AIAgent
        │
        ▼
run_conversation()
        │
        ▼
Agent Loop
        │
        ▼
LLM ↔ Tool ↔ Context ...
```
关键点 :hermes_cli/main.py 不是 Agent Loop 本身。它更像 Java 世界里的：
```java
public static void main(String[] args) {
    // 解析启动参数
    // 创建应用
    // 启动应用
}
```
而我们真正想研究的“AI 核心”是后面的：
```text
hermes_cli/main.py
        ↓
AIAgent
        ↓
run_conversation()
        ↓
conversation_loop.py
        ↓
LLM
```
所以你现在可以形成一个非常重要的源码阅读习惯：
>入口 ≠ 核心逻辑。
入口负责“把程序启动起来”，核心逻辑负责“程序真正干什么”。

和 tinystruct 对照就更清楚了
```text
Hermes                         tinystruct
─────────────────────          ─────────────────────

CLI 命令                       CLI 命令
   │                              │
   ▼                              ▼
hermes_cli/main.py              dispatcher
   │                              │
   ▼                              ▼
AIAgent                         Application
   │                              │
   ▼                              ▼
Agent Loop                      @Action
   │                              │
   ▼                              ▼
LLM / Tools                    Java Method
```

所以我们接下来第一刀就可以切 hermes_cli/main.py：
```text
① 谁调用 main？
② main 做了什么？
③ 怎么创建 AIAgent？
④ 用户输入从哪里进入？
⑤ 从哪个方法开始进入 Agent Loop？
```
然后再顺着调用链进入 conversation_loop.py。
这会非常像我以前作为 Java 程序员排查一个项目：
```text
main()
 ↓
Controller
 ↓
Service
 ↓
核心业务
```
只不过 Hermes 是：
```text
CLI Entry
 ↓
AIAgent
 ↓
Agent Loop
 ↓
LLM
 ↓
Tool
```




### 2、一个用户输入的字符串，到底是怎么一步一步变成 LLM API 的 messages 的？
这个问题一旦追通，这些概念：
> Token、Message、System/User/Assistant、Prompt、Context Window、LLM API

会一下子串起来。
所以我们下一轮就直接从 Hermes AIAgent.run_conversation() → agent/conversation_loop.py 开始，一行一行追第一条 Agent Loop。官方当前文档也明确把 run_conversation() 指向这里，并给出了它的实际生命周期。
参考地址：https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/agent-loop.md?utm_source=chatgpt.com

#### 1. 我们今天只追这一条链
先把问题缩小：
> 用户输入 "你好"，到底是怎么变成发送给 LLM 的 messages 的？

最终我们希望看到：
```text
用户输入
   │
   ▼
"你好"
   │
   ▼
user_message
   │
   ▼
{"role": "user", "content": "你好"}
   │
   ▼
conversation history
   │
   ▼
system prompt + history
   │
   ▼
api_messages
   │
   ▼
LLM API

```
而这条链正好会把：

+ Message
+ System / User / Assistant
+ Prompt
+ Context
+ Context Window
+ Token

全部串起来。


#### 2. 第一件事：用户输入在哪里？
我们先不要急着进入 conversation_loop.py。
从项目结构来看，当前 Hermes 把 CLI 编排放在：
```jupyter
cli.py
hermes_cli/
```
![img_3.png](img_3.png)

而官方贡献指南现在明确把：
```jupyter
cli.py
```
描述为：
> HermesCLI class — interactive CLI orchestrator 翻译成中文：HermesCLI 类 — 交互式 CLI 编排器

同时：
```jupyter
run_agent.py
```
是：
> AIAgent facade  AI 代理的门面，就像Java设计模式中的fecade一样的意思

然而：
```jupyter
agent/conversation_loop.py
```
才是：
> run_conversation() — the agent turn loop 才是智能体多轮次循环执行的地方

所以我们现在应该调整之前的地图：
```text
                    CLI
                     │
                     ▼
                HermesCLI
                     │
                     ▼
                  AIAgent
                     │
                     ▼
             run_conversation()
                     │
                     ▼
          agent/conversation_loop.py
                     │
                     ▼
                LLM API
```

#### 3. 真正重要的地方：run_conversation()
现在进入：
> agent/conversation_loop.py

这个文件目前已经有 1807 行，说明 Hermes 把 Agent Loop 做得非常复杂了。官方源码顶部也直接说明：

run_conversation(agent, ...)

负责：

```text
model call
tool dispatch
retries
fallbacks
compression
post-turn hooks
```

但我们绝对不要从第一行开始读 1800 行。

这是源码学习的大忌。

我们只找：
```text
def run_conversation(...)
```
![img_4.png](img_4.png)


然后围绕这个方法建立调用链。


#### 4. 第一个非常关键的事实
官方当前 Agent Loop 文档直接给出了一个 turn【轮次】 的生命周期：
```text
run_conversation()

    ↓

1. Generate task_id 生成任务编号

    ↓

2. Append user message 追加用户消息

    ↓

3. Build / reuse system prompt

    ↓

4. Check context compression 检查上下文压缩情况

    ↓

5. Build API messages

    ↓

6. Inject temporary prompt layers 注入临时提示曾

    ↓

7. API call 调用API

    ↓

8. Parse response 解析响应的结果

    ↓

    ┌──────────────┐
    │              │
    ▼              ▼
tool_calls       text
    │              │
    ▼              ▼
执行 Tool        返回答案
    │
    ▼
tool result
    │
    └──────→ 再次进入 Loop
```

这就是我们今天真正要理解的 Agent Loop。

#### 5. 现在回答第一个问题：用户字符串什么时候变成 Message？
这里是非常关键的一步。Hermes 内部维护的是 OpenAI-compatible message format【兼容OpenAI消息格式】。官方文档明确说明：
```json
{
  "role": "system",
  "content": "..."
}
```
以及完整的消息角色：
```text
system
user
assistant
tool
```
所以，用户输入：
```text
你好
```
进入 Agent 后，并不是直接："你好"。而是直接传给模型，然后这句话被放到JSON格式报文中：
```json
{
  "role": "user",
  "content": "你好"
}
```
#### 6. 这就是我们第一个基础概念：Message
Message 是什么？
你可以把 Message 理解成：
> 一条带有身份信息的对话记录。

比如：
```json
{
  "role": "user",
  "content": "你好"
}
```
解释如下：
```text
role
  ↓
谁说的？

content
  ↓
说了什么？
```
所以我们可以推测出这样的结构：
```text
用户说：
{"role": "user", "content": "你好"}

AI说：
{"role": "assistant", "content": "你好！"}

系统说：
{"role": "system", "content": "You are an AI assistant."}
```

#### 7. 所以 System / User / Assistant 到底是什么？

现在我们应该能看到：
```text
System 系统
User 用户
Assistant 助手
```
并不是三个不同的模型。也不是三个不同的 API。而是：
```text
Message 的 role。
```

举个例子：
```json
[
  {
    "role": "system",
    "content": "You are Hermes."
  },
  {
    "role": "user",
    "content": "你好"
  },
  {
    "role": "assistant",
    "content": "你好，有什么可以帮你？"
  }
]
```
把这个报文中所有的message合并起来得到的整个东西：
```text
[
  Message,
  Message,
  Message
]
```
这就是conversation history，中文就是对话历史消息

#### 8. 这时候 Context 就出现了
假设用户连续说：
```text
User:
你好

Assistant:
你好！

User:
我是一名 Java 程序员

Assistant:
很高兴认识你。

User:
我现在想学习 AI
```
那么发送给模型的可能不是最后一句：
```text
我现在想学习 AI
```

而是：
```json
[
  {
    "role": "system",
    "content": "..."
  },
  {
    "role": "user",
    "content": "你好"
  },
  {
    "role": "assistant",
    "content": "你好！"
  },
  {
    "role": "user",
    "content": "我是一名 Java 程序员"
  },
  {
    "role": "assistant",
    "content": "很高兴认识你。"
  },
  {
    "role": "user",
    "content": "我现在想学习 AI"
  }
]
```
这一整个 Message 列表，就是我们现在要重点研究的 Context 的核心组成部分。

#### 9. 但是事情还没完
你可能会马上问：
```text
那 System Prompt 在哪里？
```
这正是下一层。
Hermes 的 Agent Loop 会：
```text
用户输入
    │
    ▼
conversation history
    │
    ├───────────────┐
    │               │
    ▼               ▼
User Message    System Prompt
    │               │
    └───────┬───────┘
            ▼
      API Messages
            │
            ▼
          LLM
```

而 System Prompt 并不是简单写死一句：
```text
You are an AI.
```
Hermes 有专门的：
```text
agent/prompt_builder.py
```
负责组装 system prompt。官方文档明确把 prompt_builder.py 定义为：
```text
System prompt assembly from memory, skills, context files, personality
从记忆、技能、上下文文件和个性中组装系统提示。
```
也就是说，System Prompt 本身可能来自多个部分。
参考文档：
```text
https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/agent-loop.md?utm_source=chatgpt.com
```

#### 10. 这就产生了一个非常重要的认知

以前我们容易把：Prompt理解成：
```text
用户输入的问题
```
其实不准确。在 Agent 系统里面：
```text
Prompt=System Prompt+Conversation History+Current User Message+
Tool Information+可能的额外 Context
```
最终才形成真正发送给模型的输入。可以画成：
```text
                 Prompt
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
     System      History      User
     Prompt                    Message
        │           │           │
        └───────────┼───────────┘
                    │
                    ▼
              API Messages
                    │
                    ▼
                   LLM
```
**这个概念非常重要。**

因为以后我们看到：
```text
Prompt Engineering  提示词工程
Prompt Template     提示模板
System Prompt       系统提示
RAG Prompt          检索增强生成的提示
Agent Prompt        代理提示
```
都会回到这里。

#### 11. 然后就是 Context Window

现在假设：
```text
System Prompt
    5,000 tokens

Conversation History
   30,000 tokens

User Message
    1,000 tokens

Tools
    8,000 tokens
```
那么：
```text
总输入 ≈ 44,000 tokens
```
如果模型 Context Window：
```text
32K
```
就爆了，超出上限了。
所以 Hermes 为什么需要：
```text
context compression
```
就很好理解了。它不是为了“优化代码”。而是因为：
> LLM 每次请求能够处理的 Token 数量存在上限。

官方当前实现会在模型调用前检查上下文压力，并在超过阈值时进行 compression；文档描述的默认流程包括压缩中间轮次、
保留最近若干消息、保持 tool call/result配对等操作。
#### 12. 现在我们终于可以回答最初的问题了
> 一个用户输入的字符串，到底是怎么一步一步变成 LLM API 的 messages 的？

当前源码结构下，可以先得到这张准确的概念地图：
```text
用户输入
   │
   │ "我想学习 AI"
   ▼
user_message
   │
   ▼
┌─────────────────────────────┐
│ Conversation History        │
│                             │
│ system                      │
│ user                        │
│ assistant                   │
│ user                        │
│ assistant                   │
│ ...                         │
│                             │
│ + 当前 user message         │
└──────────────┬──────────────┘
               │
               │
               ▼
       Context / Compression
               │
               ▼
       System Prompt Builder
               │
               ▼
       Prompt + Messages
               │
               ▼
          API Messages
               │
               ▼
             LLM
```

而 Hermes 当前的 Agent Loop 文档把第 5 步明确描述为：
> Build API messages from conversation history
> 根据对话历史构建 API 消息
























## 💡 核心概念
```text
Prompt Engineering  提示词工程
Prompt Template     提示模板
System Prompt       系统提示
RAG Prompt          检索增强生成的提示
Agent Prompt        代理提示
```


## 💻 代码实践

## 🧠 我的理解

## ❓ 遇到的问题



## 📝 今日总结

## 🔗 参考资料
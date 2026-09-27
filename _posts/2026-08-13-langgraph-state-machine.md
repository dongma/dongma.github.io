---
layout: post
title: LangGraph：从线性图到工具调用与检索增强 Agent
date: 2026-08-13 +0800
tags: ["AI"]
---



本目录按图拓扑组织练习代码：LangGraph 把 LLM 应用抽象为一张以 State 为中心、由 Node / Edge / Conditional Edge 组成的有限状态机。

示例从最简单节点线性图演进到条件分支、自循环、工具调用、ReAct 智能体与 RAG Agent，每一类文件对应一种基本控制流，便于按模式速查与复现。
<!-- more -->

> **数据约定**：`langchains/rag/chats/ArangoDB-GraphCourse_Beginners.pdf` 由 `agents/rag_agent.py` 复用，作为向量检索语料。

---

## 一、前置知识

### 1.1 有向图（Directed Graph）与状态机（State Machine）

- **图**由 `节点（Node）`与`边（Edge）`组成，本目录的"图"特指**有向图**——边有方向，表示状态转移的"前 → 后"关系。
- **状态机**：图上每个节点代表一种"状态"，状态之间由事件触发转移。`LangGraph` 把 `LLM` 应用建模成状态机，让多步骤流程**可被显式编排、可视化、可持久化**。

### 1.2 `State`：图共享的数据视图

`State` 是一个 `TypedDict`（或 `Pydantic`），描述**跨节点共享的全局状态**：

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]   # 带 reducer 的字段
    counter: int
```

- 节点的入参是整个 `state` 字典（Python 拿到的是 dict）。
- 节点返回的是 state 的"变更部分"，LangGraph 会**合并**回全局状态。
- `Annotated[..., add_messages]` 这种写法给字段挂一个 **reducer**，决定新值与旧值如何合并（本例：把消息追加到列表）。

### 1.3 `Node`：处理 state 的最小执行单元

```python
def my_node(state: AgentState) -> AgentState:
    state["final"] = "..."
    return state
```

节点就是普通 Python 函数；可以是纯计算、调 `LLM`、或调外部工具。**不强制要求同步**——本目录多数节点是 `def`；异步用 `async def`。

### 1.4 `Edge` 与 `START` / `END`

- **`START`**、`END`** 是 LangGraph 提供的**特殊伪节点**：图的入口与出口。
- **`Edge`** 是静态边：`graph.add_edge("a", "b")` 表示从 `a` 执行完必走 `b`。
- **`set_entry_point("a")`** 等价于 `add_edge(START, "a")`；`set_finish_point("a")` 等价于 `add_edge("a", END)`。

### 1.5 `Conditional Edge`：基于状态的动态路由

```python
def should_continue(state: AgentState) -> str:
    return "loop" if state["counter"] < 5 else "end"

graph.add_conditional_edges(
    "random",                  # 起点节点
    should_continue,           # 决策函数：state -> str（路由键）
    {"loop": "random", "end": END},   # 路由键 -> 目标节点
)
```

决策函数返回**字符串键**，LangGraph 在第 3 个参数里查到对应的下一节点。**键未命中会抛错**，所以所有可能的返回值都要列出来。

### 1.6 `bind_tools` 与 `ToolNode`

- `model = ChatOpenAI(...).bind_tools([add, search, ...])`：让 `LLM` 在回复里**输出 `tool_calls` 而非纯文本**。
- `ToolNode(tools)` 是 LangGraph **预置的执行节点**：碰到 `AIMessage.tool_calls` 就逐个执行对应工具，并把结果打包成 `ToolMessage` 喂回 state。
- `ToolMessage` 带 `tool_call_id`，与发起调用的 `AIMessage.tool_calls[i].id` 一一对应。

### 1.7 `messages` 的 reducer 与 `thread_id`

- 让 LangGraph 自动维护会话历史，用 `Annotated[List[BaseMessage], add_messages]` + **checkpointer**（如 `InMemorySaver`）。
- `compile(checkpointer=...)` 后，`invoke(..., config={"configurable": {"thread_id": "..."}})`：**同一 thread_id 才能复用历史**。

### 1.8 五种基本图拓扑（本目录练习的全部形态）

| 拓扑 | 形式 | 典型用途 |
| --- | --- | --- |
| **线性** | `START → A → B → END` | 一次性流水线 |
| **自循环** | `A →{loop | exit}→ A | END` | 数值生成、迭代求解 |
| **条件分支** | `A →{p1 | p2 | ...}→ B | C | ...` | 路由器、按入参分发 |
| **LLM 决策** | `LLM →{continue | end}→ tools → LLM ...` | ReAct、自规划 Agent |
| **业务结束条件** | `tools →{save⇒end | else⇒agent}→` | 多步交互（带业务条件） |

### 1.9 关键生态包

| 包 | 作用 |
| --- | --- |
| `langgraph` | 状态机本体 |
| `langgraph.constants.START/END` | 入口 / 出口伪节点 |
| `langgraph.prebuilt.ToolNode` | 自动执行 tool_calls 的节点 |
| `langgraph.checkpoint.memory.InMemorySaver` | 进程内 checkpointer |
| `langgraph.graph.add_messages` | 消息 reducer（自动追加） |

---

## 二、通用范式（贯穿所有示例）

```python
from typing import TypedDict
from langgraph.graph import StateGraph
from langgraph.constants import START, END

class AgentState(TypedDict):
    ...

def my_node(state: AgentState) -> AgentState:
    return state

graph = StateGraph(AgentState)
graph.add_node("my_node", my_node)
graph.set_entry_point("my_node")
graph.add_edge("my_node", END)
app = graph.compile()                     # 编译成可执行对象
app.invoke({"...": ...})                  # 触发图执行
```

---

## 三、目录结构与文件速查

| 文件 | 拓扑 | 一句话说明 |
| --- | --- | --- |
| `basic/hello_agent.py` | 单节点线性 | 输入名字，输出问候 |
| `basic/linear_graph.py` | 两节点线性 | 不调 LLM，纯函数流水线演示 |
| `basic/multiple_input.py` | 单节点多输入 | state 同时持有 `values` 与 `name` |
| `control_flow/loop_graph.py` | 自循环 | 生成随机数直到 `counter==5` |
| `control_flow/condition_graph.py` | 条件分支 | 按 `operation` 路由到加法 / 减法节点（骨架） |
| `agents/agent_bots.py` | 线性 + LLM | `gpt-4o` 单拍即合回答 |
| `agents/chat_bots.py` | 线性 + LLM | 手动管理 messages 列表的多轮对话 |
| `agents/react.py` | LLM 决策闭环 | 用 `add` 工具做"3+4, 31+42" |
| `agents/drafter.py` | 业务结束条件 | 写稿助手；`save` 工具出现时退出 |
| `agents/rag_agent.py` | LLM 决策 + 检索 | 自治何时检索 PDF 并生成带引用回答 |

---

## 四、模块详解

### 4.1 `basic/hello_agent.py` —— 最简单的单节点线性图

**图拓扑**：`START → greeter → END`
**演示场景**：输入一个名字，节点返回 `"Hey {name}, how is your day going?"`。

**关键代码**

```python
class AgentState(TypedDict):
    message: str

def greeting_node(state: AgentState) -> AgentState:
    state["message"] = f"Hey {state['message']}, how is your day going?"
    return state

graph = StateGraph(AgentState)
graph.add_node("greeter", greeting_node)
graph.set_entry_point("greeter")
graph.set_finish_point("greeter")
app = graph.compile()
app.invoke({"message": "Bob"})
```

---

### 4.2 `basic/linear_graph.py` —— 两节点线性图（不保留历史）

**图拓扑**：`START → first_node → second_node → END`
**演示场景**：把 `{"name":"Charlie","age":20}` 顺着节点依次加工，输出 `"Hi, Charlie. You are 20 years old."`。

**关键代码**

```python
class AgentState(TypedDict):
    name: str
    age: str
    final: str

def first_node(state):
    state["final"] = f"Hi, {state['name']}"; return state

def second_node(state):
    state["final"] += f" You are {state['age']} years old."; return state

graph = StateGraph(AgentState)
graph.add_node("first_node", first_node)
graph.add_node("second_node", second_node)
graph.set_entry_point("first_node")
graph.add_edge("first_node", "second_node")
graph.set_finish_point("second_node")
app = graph.compile()
app.invoke({"name": "Charlie", "age": 20})
```

> 与 `chat_bots.py` 的差别：本脚本**不**调 LLM，纯函数流水线；适合先理解图结构。

---

### 4.3 `agents/chat_bots.py` —— 线性图调 LLM 的多轮对话

**图拓扑**：`START → process → END`（外层 `while` 模拟多轮）
**演示场景**：`gpt-4o` 单节点应答。用户输入 → 把消息列表塞给 LLM → 把 `AIMessage` 追加到 `conversation_history`，下一轮直接复用——这种"手动管理 history"模式与 `langchains.agents` 的 `checkpointer` 思路对应。

**关键代码**

```python
class AgentState(TypedDict):
    messages: List[Union[AIMessage, HumanMessage]]

llm = ChatOpenAI(model="gpt-4o")

def process(state: AgentState) -> AgentState:
    resp = llm.invoke(state["messages"])
    state["messages"].append(AIMessage(content=resp.content))
    return state

graph = StateGraph(AgentState)
graph.add_node("process", process)
graph.add_edge(START, "process")
graph.add_edge("process", END)
agent = graph.compile()

conversation_history = []
user_input = input("请输入: ")
while user_input != "exit":
    conversation_history.append(HumanMessage(content=user_input))
    result = agent.invoke({"messages": conversation_history})
    conversation_history = result["messages"]
    user_input = input("请输入: ")
```

---

### 4.4 `basic/multiple_input.py` —— 单节点多输入字段

**图拓扑**：`START → processor → END`
**演示场景**：`AgentState` 同时持有 `values: list[int]` 与 `name: str`，节点一次性算出 `sum + 问候`，输出 `"Hi there Steve!, your sum = 10"`。

**关键代码**

```python
class AgentState(TypedDict):
    values: List[int]
    name: str
    result: str

def process_values(state: AgentState) -> AgentState:
    print(f"before => {state}")
    state["result"] = f"Hi there {state['name']}!, your sum = {sum(state['values'])}"
    return state

app.invoke({"values": [1, 2, 3, 4], "name": "Steve"})
```

> 要点：`AgentState` 可任意复杂；节点的入参是 state 的一个子集视图（实际上 Python 直接拿到整个字典）。

---

### 4.5 `control_flow/loop_graph.py` —— 自循环（条件边驱动）

**图拓扑**：`greeting → random ⇄ (条件边：loop | exit → END)`
**演示场景**：先 `greeting`，再在 `random` 节点里循环生成随机数，**直到** `counter == 5` 才退出。

**关键代码**

```python
class AgentState(TypedDict):
    name: str
    number: list[int]
    counter: int

def greeting_node(state):
    state["name"] = f"Hi, {state['name']}"
    state["counter"] = 0
    return state

def random_node(state):
    state["number"].append(random.randint(0, 10))
    state["counter"] += 1
    return state

def should_continue(state):
    return "loop" if state["counter"] < 5 else "exit"

graph = StateGraph(AgentState)
graph.add_node("greeting", greeting_node)
graph.add_node("random", random_node)
graph.add_edge("greeting", "random")
graph.add_conditional_edges(
    "random",
    should_continue,
    {"loop": "random",                # 自我闭环
     "exit": END},                    # 终止
)
graph.set_entry_point("greeting")
app = graph.compile()
app.invoke({"name": "Vaibhav", "number": [], "counter": -1})
```

> 这是 LangGraph "agent 内部循环" 的最简形式；`react.py` / `drafter.py` 的循环结构本质相同。

---

### 4.6 `control_flow/condition_graph.py` —— 条件路由（占位示例）

**图拓扑**：`START → router →{addition_operation | subtraction_operation}→ END`
**演示场景**：根据 `state["operation"]`（`"+"` / `"-"`）把请求路由到加法节点或减法节点，输出 `final_number`。

**关键代码**

```python
class AgentState(TypedDict):
    number1: int
    operation: str
    number2: int
    final_number: int

def adder(state):       state["final_number"] = state["number1"] + state["number2"]; return state
def subtractor(state):  state["final_number"] = state["number1"] - state["number2"]; return state

def decide_next_node(state):
    if   state["operation"] == "+": return "addition_operation"
    elif state["operation"] == "-": return "subtraction_operation"

graph = StateGraph(AgentState)
graph.add_node("add_node", adder)
graph.add_node("subtract_node", subtractor)
graph.add_node("router", lambda state: state)
graph.add_edge(START, "router")
graph.add_conditional_edges(
    "router",
    decide_next_node,
    {"addition_operation": "add_node",
     "subtraction_operation": "subtract_node"},
)
graph.add_edge("add_node", END)
graph.add_edge("subtract_node", END)
app = graph.compile()
app.invoke(AgentState(number1=10, operation="-", number2=5))   # → final_number=5
```

> `condition_graph.py` 是路由骨架（不调 LLM），而 `agent_bots.py` 把路由换成 LLM 决策。

---

### 4.7 `agents/agent_bots.py` —— LLM 直答型 Agent

**图拓扑**：`START → process → END`（LLM 单轮）
**演示场景**：和 `chat_bots.py` 类似——`gpt-4o` 一拍即合地回答用户输入；区别是本脚本每次都是空白的 `messages` 列表，**不**在 `while` 循环里累积历史。

**关键代码**

```python
class AgentState(TypedDict):
    messages: List[HumanMessage]

llm = ChatOpenAI(model="gpt-4o")

def process(state: AgentState) -> AgentState:
    resp = llm.invoke(state["messages"])
    print(f"\nAI resp: {resp}")
    return state

graph = StateGraph(AgentState)
graph.add_node("process", process)
graph.add_edge(START, "process")
graph.add_edge("process", END)
agent = graph.compile()

while True:
    user_input = input("请输入: ")
    if user_input == "exit": break
    agent.invoke({"messages": [HumanMessage(content=user_input)]})
```

---

### 4.8 `agents/react.py` —— ReAct 风格的工具调用循环

**图拓扑**：`START → our_agent →{continue | end}→ tools → our_agent ...`
**演示场景**：给 `gpt-4o` 一个加法工具 `add(a, b)`，让它对 `"Add 3 + 4, Add 31 + 42"` 自己决定调用几次，直到输出最终答案。

**关键代码**

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]   # 注意 add_messages reducer

@tool
def add(a: int, b: int):
    """This is an addition function that adds 2 numbers together"""
    return a + b

tools = [add]
model = ChatOpenAI(model="gpt-4o").bind_tools(tools)

def model_call(state):
    system_prompt = SystemMessage(content="You are my AI assistant ...")
    response = model.invoke([system_prompt] + state["messages"])
    return {"messages": [response]}

def should_continue(state):
    last = state["messages"][-1]
    return "continue" if last.tool_calls else "end"

graph = StateGraph(AgentState)
graph.add_node("our_agent", model_call)
graph.add_node("tools", ToolNode(tools=tools))
graph.set_entry_point("our_agent")
graph.add_conditional_edges("our_agent", should_continue,
                            {"continue": "tools", "end": END})
graph.add_edge("tools", "our_agent")
app = graph.compile()

app.stream({"messages": [HumanMessage(content="Add 3 + 4, Add 31 + 42")]},
           stream_mode="values")
```

> 关键点：`Annotated[..., add_messages]` 让 reducer 自动拼接消息；`ToolNode` 是 LangGraph 内置的执行节点。

---

### 4.9 `agents/drafter.py` —— 带状态写入的 Agent 闭环（自定义工具改全局变量）

**图拓扑**：`START → agent → tools →{continue | end}`
**演示场景**：写稿助手。"用户改一段稿" → 调 `update` 工具把 `document_content` 写进全局 → "用户保存" → 调 `save` 工具写文件并**结束**图。

**关键代码**

```python
document_content = ""

class AgentState(TypedDict):
    messages: Annotated[List[Union[AIMessage, HumanMessage]], add_messages]

@tool
def update(content: str) -> str:
    """updates the document with the provided content"""
    global document_content
    document_content = content
    return f"Document has been updated successfully! current content:\n{document_content}"

@tool
def save(filename: str) -> str:
    """Save the current document to a text file and finish the process."""
    global document_content
    if not filename.endswith('.txt'): filename = f"{filename}.txt"
    with open(filename, 'w') as f: f.write(document_content)
    return f"Document has been saved successfully to '{filename}'."

def our_agent(state):
    system_prompt = SystemMessage(content=f"""You are Drafter ...
    The current document content is:{document_content}""")
    user_message = HumanMessage(content=input("\nWhat would you like to do with the document? "))
    response = model.invoke([system_prompt] + list(state["messages"]) + [user_message])
    return {"messages": [user_message, response]}

def should_continue(state):
    """只要最近的 ToolMessage 看上去是 save 出的结果, 就 end。"""
    for message in reversed(state["messages"]):
        if (isinstance(message, ToolMessage)
            and "saved" in message.content.lower()
            and "document" in message.content.lower()):
            return "end"
    return "continue"
```

> 与 `react.py` 的差别：`should_continue` 关注 **ToolMessage 的内容**而不是有无 tool_call，更贴近业务语义。

---

### 4.10 `agents/rag_agent.py` —— LangGraph 版 RAG Agent

**图拓扑**：`START → llm →{continue | end}`，continue 分支 `→ retriever_agent → llm ...`
**演示场景**：把 `ArangoDB-GraphCourse_Beginners.pdf` 切分后落到 `langgraphs/vector/chroma`；用一个 `retriever_tool` 让 `gpt-4o` 自主决定何时检索；最后生成**带引用**的回答。

**关键代码片段 1：构建向量库（独立于图）**

```python
pages = PyPDFLoader("langchains/rag/chats/ArangoDB-GraphCourse_Beginners.pdf").load()
splits = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200).split_documents(pages)
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=QwenEmbeddings(),       # 复用 langchains.rag.chats.embeddings
    persist_directory="langgraphs/vector",
    collection_name="arango_docs",
)
retriever = vectorstore.as_retriever(search_type="similarity", search_kwargs={"k": 5})
```

**关键代码片段 2：把检索器包装成 `@tool`**

```python
def build_retriever_tool(retriever):
    @tool
    def retriever_tool(query: str) -> str:
        """Search the ArangoDB Graph Course PDF for passages relevant to the query."""
        docs = retriever.invoke(query)
        if not docs:
            return "I found no relevant information in the ArangoDB Graph Course PDF."
        return "\n\n".join(f"Document {i+1}:\n{_clean_surrogates(d.page_content)}"
                           for i, d in enumerate(docs))
    return retriever_tool
```

**关键代码片段 3：编排 LLM ↔ 工具的双节点图**

```python
def call_llm(state):
    messages = [SystemMessage(content=SYSTEM_PROMPT)] + list(state["messages"])
    return {"messages": [model.invoke(messages)]}

def take_action(state):
    tool_calls = state["messages"][-1].tool_calls
    results = []
    for t in tool_calls:
        result = tools_dict[t["name"]].invoke(t["args"].get("query", ""))
        results.append(ToolMessage(tool_call_id=t["id"], name=t["name"], content=str(result)))
    return {"messages": results}

def should_continue(state):
    last = state["messages"][-1]
    return "continue" if getattr(last, "tool_calls", None) else "end"

graph = StateGraph(AgentState)
graph.add_node("llm", call_llm)
graph.add_node("retriever_agent", take_action)
graph.add_conditional_edges("llm", should_continue,
                            {"continue": "retriever_agent", "end": END})
graph.add_edge("retriever_agent", "llm")
graph.set_entry_point("llm")
app = graph.compile()
```

---

## 五、环境变量

| 变量 | 用途 |
| --- | --- |
| `DEEPSEEK_API_KEY` | 大部分文件的 `ChatDeepSeek`（注意新版本是 `init_chat_model`） |
| `OPENAI_API_KEY` | `agents/chat_bots.py` / `agents/agent_bots.py` / `agents/drafter.py` / `agents/react.py` / `agents/rag_agent.py` 的 `gpt-4o` |
| `QWEN_API_KEY` (`DASHSCOPE_API_KEY`) | `agents/rag_agent.py` 中的 `QwenEmbeddings` |

---

## 六、学习路径建议

1. **基础拓扑**：`basic/hello_agent.py` → `basic/linear_graph.py` → `basic/multiple_input.py`
2. **控制流**：`control_flow/loop_graph.py` → `control_flow/condition_graph.py`
3. **Agent 模式**：`agents/chat_bots.py` → `agents/agent_bots.py` → `agents/react.py` → `agents/drafter.py` → `agents/rag_agent.py`


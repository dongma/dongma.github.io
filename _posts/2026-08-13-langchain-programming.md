---
layout: post
title: LangChain 编程范式
date: 2026-08-13 +0800
tags: ["AI"]
---



本目录按 LangChain 核心编程范式组织练习代码：提示工程 → 链式调用 → 会话记忆 → 自定义工具 → ReAct Agent → create_agent（新版 API）→ 端到端 RAG → 检索评估。所有示例以 DeepSeek 作为基底 LLM。

适合作为 LangChain 的入门 → 进阶实战手册：从基础组件（LLM / ChatModel / 消息类型 / PromptTemplate / Embedding / VectorStore），到四种主流应用范式（Chain / Memory / Agent / RAG），每章都配可运行的 demo。
<!-- more -->

> **数据约定**
> - `rag/Data.csv` —— 评论文本（用于 `rag/prompts.py` 邻近结构化抽取 demo；basics/prompts.py 用的是内存示例）
> - `rag/OutdoorClothingCatalog_1000.csv` —— 户外服装商品目录（用于 `rag/qa.py` / `rag/evaluation.py`）
> - `rag/chats/ArangoDB-GraphCourse_Beginners.pdf` —— 真正用来做 PDF RAG 的语料

---

## 一、前置知识

在阅读本目录代码前，建议先建立以下概念基础；其中 **RAG** 与 **Agent** 是后续讨论的两条主线。

### 1.1 `LLM` 与 `ChatModel`

- **LLM**（Large Language Model）以**字符串**为单位接收输入并产生输出。
- **ChatModel** 是其上的面向"消息"的抽象，输入改为 `List[BaseMessage]`，输出 `AIMessage`。本目录所有脚本均使用 `ChatDeepSeek(model="deepseek-chat")` 或 `ChatOpenAI(model="gpt-4o")`。
- **流式输出**：`ChatModel.stream(...)` 边产生边吐 token；`agent.stream(...)` 把每个 step 的状态增量逐个推回。

### 1.2 消息类型 `BaseMessage`

```text
SystemMessage     —— 角色设定、约束、风格
HumanMessage      —— 用户输入
AIMessage         —— 模型回复；可附 tool_calls（function-calling 决策）
ToolMessage       —— 工具执行结果；带 tool_call_id，与 AIMessage.tool_calls 一一对应
```

`messages: Annotated[List[BaseMessage], add_messages]` 这种带 reducer 的字段会自动把新消息**追加**到列表，而不是整体覆盖。

### 1.3 `PromptTemplate` 与 `ChatPromptTemplate`

- **为什么需要模板**：当 prompt 变长、需要在多处复用、或者要注入 `MessagesPlaceholder`（即"消息占位"）时，手写字符串不可维护。
- `format_messages(...)` 把模板渲染成 `[SystemMessage, HumanMessage, ...]`。

### 1.4 `Embedding` 与 `VectorStore`

- **Embedding**：把文本映射为稠密向量 `R^d`（`QwenEmbeddings` 输出 1024 维 `text-embedding-v3`）。
- **相似度度量**：本目录用 `np.dot`（等价于未归一化的余弦）。
- **VectorStore**：把向量 + `Document.page_content` + `metadata` 关联存储；`Chroma` 是常见本地选择，`InMemoryVectorStore` 则是 LangChain 内存版（无须额外服务）。

### 1.5 RAG（Retrieval-Augmented Generation）

```text
[RAG 链路]
PDF/CSV/Notion
   └─> Loader                PyMuPDF4LLMLoader / CSVLoader
        └─> Splitter         RecursiveCharacterTextSplitter
             └─> Embedder    QwenEmbeddings / OpenAIEmbeddings
                  └─> Store  Chroma(persist) / InMemoryVectorStore
                       └─> Retriever              similarity_search / SelfQueryRetriever / ContextualCompressionRetriever
                            └─> LLM               拼进 prompt 后由 ChatModel 回答
```

`rag/chats/chat_agents.py` 是这条链路的完整实践。

### 1.6 `Chain` 与 LCEL（`|` 串联）

- LCEL = LangChain Expression Language。
- 用 `prompt | llm | parser` 这种管道符串起来，从左往右把上一步的输出灌进下一步。
- `SimpleSequentialChain` 已在 `LangChain 1.0` 移除，统一用 LCEL。

### 1.7 `Tool` / `function-calling` / Agent

- **Tool**：被装饰的普通函数；`@tool` 或 `Tool(name, func, description)`。
- **function-calling**：模型在回复中**不直接给答案**，而是 `tool_calls=[{name, args}]`；客户端执行后将结果作为 `ToolMessage` 喂回。
- **ReAct（Reason+Act）**：模型反复 *思考→调工具→观察→再思考* 直到给出 `Final Answer`。`langchain_classic.agents.create_react_agent` 是这条范式的实现。
- **`create_agent` (LangChain 1.0)**：高阶封装，自动处理工具循环 + 消息管理 + 可选 `response_format`、`checkpointer`、`middleware`（重试等）。

### 1.8 会话记忆（Memory）

把会话历史以两种方式注入 prompt：

1. 手动：`history = InMemoryChatMessageHistory()`；每轮 `add_user_message / add_ai_message`，把 `history.messages` 喂给 `MessagesPlaceholder`。
2. 由 `checkpointer` 在 LangGraph 层面按 `thread_id` 自动保存/加载完整 `messages`。

### 1.9 结构化输出（Structured Output）

- `llm.with_structured_output(PydanticClass)` 让 `LLM` 通过 **tool_choice 强制**产出 Pydantic 对象。
- 注意与某些推理模型的 **thinking 模式冲突**：若模型开了"先内部思考再回答"，必须显式关掉 thinking（`extra_body={"thinking": {"type": "disabled"}}`）。

### 1.10 关键生态包

| 包 | 作用 |
| --- | --- |
| `langchain` / `langchain_core` | 抽象接口：`PromptTemplate`、`Messages`、`runnables` |
| `langchain_classic` | 经典链：`RetrievalQA`、`ConversationChain`、`MultiPromptChain` |
| `langchain_experimental` | 实验组件：`PythonREPLTool`、代码执行 Agent |
| `langchain_chroma` / `langchain_text_splitters` / `langchain_pymupdf4llm` | 向量库、文本切分、PDF 加载 |
| `langchain_deepseek` / `langchain_openai` | 各家模型 SDK 适配 |
| `langgraph.checkpoint.memory.InMemorySaver` | 进程内 checkpointer（生产用 Postgres） |

---

## 二、目录结构与文件速查

| 文件 / 子目录 | 主题 | 一句话说明 |
| --- | --- | --- |
| `basics/prompts.py` | Prompt + 结构化输出 | `ChatPromptTemplate`、翻译 / 抽取、`with_structured_output` |
| `basics/chains.py` | LCEL / 路由 | 顺序链、`response_format`、`MultiPromptChain` 多专家路由 |
| `basics/memory.py` | 会话记忆 | `InMemoryChatMessageHistory`、`ConversationBufferWindowMemory` |
| `agents/tools.py` | 自定义工具 | `@tool`、`args_schema`、`ToolRuntime[Context]` 注入 |
| `agents/agents.py` | ReAct Agent | 自定义工具、`create_react_agent`、`PythonREPLTool` 代码执行 |
| `agents/assistant.py` | `create_agent` 新版 | `checkpointer` / `middleware` / `create_deep_agent` 对比 |
| `rag/qa.py` | RAG 全流程（CSV） | 端到端：CSV → 向量库 → `RetrievalQA`（stuff 链） |
| `rag/evaluation.py` | RAG 评估 | `QAGenerateChain` 自动造 QA、`langchain.debug` 调试 |
| `rag/chats/chat_agents.py` | RAG 全流程（PDF，**核心**） | PyMuPDF4LLM + Chroma + 3 种检索器 + 3 种 QA 链 |
| `rag/chats/embeddings.py` | 自定义 Embedding | `QwenEmbeddings`：用 OpenAI SDK 调通义 embedding |
| `rag/chats/text_split.py` | 文本切分 | `CharacterTextSplitter` / `RecursiveCharacterTextSplitter` / `TokenTextSplitter` 对比 |
| `rag/chats/store/chroma` | 持久化目录 | Chroma 落盘位置，无需手工维护 |

---

## 三、模块详解

### 3.1 `basics/prompts.py` —— Prompt Template 与结构化输出

**功能：** 演示为什么要使用 `ChatPromptTemplate`（复杂提示词易复用、易组合）；演示 `agent.stream` 的流式调用；演示用 Pydantic + `with_structured_output` 让 `LLM` 直接产出 **类型化的结构化数据**。

**关键代码片段 1：风格迁移模板**

```python
TRANSLATE_TEMPLATE = ChatPromptTemplate.from_template(
    """Translate the text delimited by triple backticks into a style that is {style}.
    text: ```{text}```"""
)

def translate(text: str, style: str, llm: ChatDeepSeek) -> str:
    messages = TRANSLATE_TEMPLATE.format_messages(style=style, text=text)
    return llm.invoke(messages).content
```

**关键代码片段 2：用 Pydantic + `with_structured_output` 强制结构化抽取**

```python
class ReviewInfo(BaseModel):
    gift: bool
    delivery_days: int
    price_value: list[str]

# 注意：with_structured_output 走 tool_choice，关闭 deepseek 的 thinking 模式
struct_llm = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={"thinking": {"type": "disabled"}},
).with_structured_output(ReviewInfo)
gift_response = struct_llm.invoke(messages)   # 直接返回 ReviewInfo 对象
```

**关键代码片段 3：agent 流式输出**

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Search for AI news ..."}]},
    stream_mode="values",
):
    latest = chunk["messages"][-1]
    if latest.content:
        print(f"{'User' if isinstance(latest, HumanMessage) else 'Agent'}: {latest.content}")
    elif latest.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest.tool_calls]}")
```

---

### 3.2 `basics/chains.py` —— 顺序链与多提示路由

**功能：** 演示 LCEL 顺序链（`prompt → llm → parser → prompt → llm`）、`response_format` 强制结构化输出、`MultiPromptChain` 路由器把不同领域的问题分发到不同的"专家提示词"。

**关键代码片段 1：LCEL 顺序链（替代已移除的 `SimpleSequentialChain`）**

```python
chain = (
    prompt1
    | llm
    | RunnableLambda(extract_company_name)   # 从 LLM 输出里抽公司名
    | prompt2
    | llm
)
result = chain.invoke({"product": "colorful socks"})
```

**关键代码片段 2：`response_format` 输出 Pydantic 对象 + middleware 重试**

```python
class Answer(BaseModel):
    summary: str
    confidence: float

agent = create_agent(
    model="deepseek-chat",
    tools=[get_weather],
    response_format=Answer,                   # LLM 自动产出结构化 Answer
    checkpointer=InMemorySaver(),             # 跨调用保留会话历史
    middleware=[
        ModelRetryMiddleware(max_retries=3),
        ToolRetryMiddleware(max_retries=2),
    ],
)

# 同一 thread_id 才能复用历史；context 会透传到 tools / middleware
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
    config={"configurable": {"thread_id": str(uuid7())}},
    context=Context(user_id="user-123"),
)
```

**关键代码片段 3：4 个专家提示词 + `LLMRouterChain` 分发**

```python
# physics / math / history / computer science 四个 prompt 模板
prompt_infos = [{"name": ..., "description": ..., "prompt_template": ...}, ...]

router_chain = LLMRouterChain.from_llm(llm, router_prompt)
chain = MultiPromptChain(
    router_chain=router_chain,
    destination_chains=destination_chains,   # 4 个专家链
    default_chain=default_chain,             # 不相关问题退回 default
    verbose=True,
)
chain.invoke({"input": "什么是黑洞？"})        # → 路由到 physics
```

---

### 3.3 `basics/memory.py` —— 多套会话记忆方案

**功能：** 演示三种"会话上下文注入"实现：`InMemoryChatMessageHistory`（最底层，按 `session_id` 存）、`ConversationBufferWindowMemory(k=1)`（只保留最近 k 轮）、`ConversationChain`（封装好的链）。

**关键代码片段 1：`InMemoryChatMessageHistory` + `MessagesPlaceholder`**

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder("history"),           # 历史占位
    ("human", "{input}"),
])

store = {}
def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

def chat(session_id: str, user_input: str):
    history = get_session_history(session_id)
    prompt_value = prompt.invoke({"history": history.messages, "input": user_input})
    ai_message = llm.invoke(prompt_value)
    history.add_user_message(user_input)
    history.add_ai_message(ai_message.content)
    return ai_message
```

**关键代码片段 2：`ConversationBufferWindowMemory(k=1)`**

```python
memory = ConversationBufferWindowMemory(k=1)   # 只记最近 1 轮
memory.save_context({"input": "Hi"}, {"output": "what's up?"})
memory.save_context({"input": "Not much, just hanging"}, {"output": "cool"})

conversation = ConversationChain(llm=llm, memory=memory)
conversation.invoke({"input": "What is my name?"})   # 因 k=1 已丢失，LLM 不知道
```

---

### 3.4 `agents/tools.py` —— 自定义工具的三种姿势

**功能：** 演示 `@tool` 装饰器、`args_schema`（Pydantic 自动生成 JSON Schema）、以及 `ToolRuntime[Context]` 注入运行时上下文。

**关键代码片段 1：基础 `@tool`**

```python
@tool
def time(text: str) -> str:
    """Returns todays date, use this for any questions related to knowing todays date."""
    return str(date.today())
```

**关键代码片段 2：用 Pydantic `args_schema` 给出多参数工具**

```python
class WeatherInput(BaseModel):
    location: str
    units: Literal["celsius", "fahrenheit"] = "celsius"
    include_forecast: bool = False

@tool("get_weather_tool", args_schema=WeatherInput)
def get_weather(location: str, units: str = "celsius", include_forecast: bool = False) -> str:
    ...
```

**关键代码片段 3：`ToolRuntime` 让工具拿到 `agent.invoke(context=...)` 传入的上下文**

```python
@dataclass
class UserContext:
    user_id: str

@tool
def get_account_info(runtime: ToolRuntime[UserContext]) -> str:
    user_id = runtime.context.user_id          # 从 runtime 拿 user_id
    ...

agent = create_agent(
    model="deepseek-chat",
    tools=[get_account_info],
    context_schema=UserContext,               # 声明 context 类型
)
agent.invoke(..., context=UserContext(user_id="user123"))
```

---

### 3.5 `agents/agents.py` —— ReAct Agent 与 Python 代码执行 Agent

**功能：** 演示经典 `create_react_agent` + `AgentExecutor` 范式；自实现 `REACT_PROMPT` 规避 `hub.pull` 的弃用 warning；演示 `PythonREPLTool` 让 `LLM` 写并执行 Python 代码。

**关键代码片段 1：定义工具（`@tool` 与 `Tool` 两种风格并存）**

```python
def calculate(expression: str) -> str:
    return str(eval(expression))

@tool
def time(text: str) -> str:
    """Returns todays date ..."""
    return str(date.today())

TOOLS = [
    Tool(name="Calculator", func=calculate,
         description="Useful for math calculations. Input should be a valid Python expression."),
    Tool(name="Wikipedia", func=wikipedia_search,
         description="Useful for querying Wikipedia."),
    time,
]
```

**关键代码片段 2：内联 ReAct 提示词（替代 `hub.pull("hwchase17/react")`）**

```python
REACT_PROMPT = PromptTemplate.from_template("""Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:
Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}""")

agent = create_react_agent(llm, TOOLS, REACT_PROMPT)
executor = AgentExecutor(
    agent=agent, tools=TOOLS, verbose=True,
    handle_parsing_errors=True,               # LLM 跳出 ReAct 格式时不直接报错
)
```

**关键代码片段 3：执行 Python 代码的 Agent**

```python
from langchain_experimental.agents.agent_toolkits import create_python_agent
from langchain_experimental.tools.python.tool import PythonREPLTool

agent = create_python_agent(
    llm,
    tool=PythonREPLTool(),                    # LLM 自己写 Python 并真跑
    verbose=True,
    agent_executor_kwargs={"handle_parsing_errors": True},
)
agent.invoke({"input": "Sort these customers by last name..."})
```

---

### 3.6 `agents/assistant.py` —— `create_agent` 新版 vs `create_deep_agent`

**功能：** `LangChain 1.0` 主推的 `create_agent` 与 `deepagents.create_deep_agent` 的同题对比：`fetch_text_from_url` 从 Project Gutenberg 拉《了不起的盖茨比》，统计包含 `Gatsby` 的行数、首次出现 `Daisy` 的行号、写两句话简介。

**关键代码**

```python
@tool
def fetch_text_from_url(url: str) -> str:
    """Fetch the document from a URL."""
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 ..."})
    with urllib.request.urlopen(req, timeout=120) as resp:
        return resp.read().decode("utf-8", errors="replace")

checkpointer = InMemorySaver()
agent     = create_agent(model=model, tools=[fetch_text_from_url],
                         checkpointer=checkpointer)
deep_agent = create_deep_agent(model=model, tools=[fetch_text_from_url],
                               checkpointer=checkpointer)

# thread_id 控制会话记忆；同样任务跑两次对比两者表现
agent.invoke({...}, config={"configurable": {"thread_id": "great-gatsby-lc"}})
deep_agent.invoke({...}, config={"configurable": {"thread_id": "great-gatsby-da"}})
```

---

### 3.7 `rag/qa.py` —— RAG 全流程（CSV → 向量库 → QA）

**功能：** 用 `OutdoorClothingCatalog_1000.csv` 演示"读取数据 → `embeddings` → `VectorStore` → 检索 → 拼 prompt → LLM 回答"的两条路径：高阶 API（`VectorstoreIndexCreator`）和手写链路。

**关键代码片段 1：每行 CSV → 一个 `Document`（替代缺装的 `CSVLoader`）**

```python
def csv_to_documents(path: str) -> list[Document]:
    df = pd.read_csv(path)
    return [Document(page_content=row.to_string(),
                     metadata={"row": int(idx), "source": path})
            for idx, row in df.iterrows()]
```

**关键代码片段 2：高阶 API 一行构建索引**

```python
index = VectorstoreIndexCreator(
    vectorstore_cls=Chroma,
    embedding=qwen_embeddings,
    vectorstore_kwargs={"client": chromadb.EphemeralClient(),
                        "collection_name": "outdoor_catalog"},
).from_documents(csv_to_documents(file))
response = index.query("List all shirts with sun protection ...", llm=llm)
```

**关键代码片段 3：手写 RAG 链路（`stuff` 链：把所有召回文档塞进同一个 prompt）**

```python
db: InMemoryVectorStore = InMemoryVectorStore.from_documents(docs, embedding=qwen_embeddings)
similar_docs = db.similarity_search("Please suggest a shirt with sunblocking")

# 直接把召回片段拼进 prompt
qdocs = "".join(d.page_content for d in similar_docs)
llm.invoke(f"{qdocs} Question: {QUERY}").content

# RetrievalQA 包装版
qa = RetrievalQA.from_chain_type(llm=llm, chain_type="stuff", retriever=db.as_retriever())
```

---

### 3.8 `rag/evaluation.py` —— RAG 评估与调试

**功能：** 在 3.7 的基础上用 `QAGenerateChain` 自动批量生成 QA 对作为评测数据；通过 `langchain.debug = True` 打开 LangSmith 风格的详细日志。

**关键代码**

```python
gen_chain = QAGenerateChain.from_llm(llm)
new_examples = gen_chain.apply_and_parse(
    [{"doc": t} for t in data[:5]]           # 从前 5 条文档各造一道 QA
)
examples += new_examples

import langchain
langchain.debug = True                        # 打印检索和 prompt 细节
result = qa.run(examples[0]["query"])         # 跑第一条 QA 看效果
```

---

### 3.9 `rag/chats/chat_agents.py` —— 完整 PDF RAG 流水线（本目录核心）

**功能：** 加载 PDF → 切分 → 持久化到 Chroma → 三种检索方式对比（基础相似度 / `SelfQueryRetriever` 元数据过滤 / `ContextualCompressionRetriever` LLM 压缩）→ 三种 QA 链（裸 `RetrievationQA` / `map_reduce` / `ConversationalRetrievalChain` 带记忆）。

**关键代码片段 1：PDF 加载 + 切分 + 向量化（一次性索引并落盘）**

```python
loader = PyMuPDF4LLMLoader("./ArangoDB-GraphCourse_Beginners.pdf")
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1500, chunk_overlap=150)
splits = text_splitter.split_documents(loader.load())      # → 1305 个 chunk

Chroma.from_documents(
    documents=splits,
    persist_directory="./store/chroma",
    embedding=QwenEmbeddings(),
    collection_name="arangodb_pdf",
)
```

**关键代码片段 2：自实现 `ChromaStructuredQueryTranslator`，让 `SelfQueryRetriever` 可在中文 PDF 上工作**

```python
class ChromaStructuredQueryTranslator(Visitor):
    """把 StructuredQuery 翻译成 Chroma 的 where 语法。
    langchain-classic 1.0.x 仍依赖 langchain_community 提供 ChromaTranslator,
    但 langchain_community 与 langchain>=1.0.0 存在死锁,自实现一份。
    """
    def visit_comparison(self, comparison):
        return {comparison.attribute:
                {self._map_comparator(comparison.comparator): comparison.value}}
    def visit_structured_query(self, query):
        return query.query, {"filter": query.filter.accept(self) if query.filter else None}

retriever = SelfQueryRetriever.from_llm(
    llm=deepseek_llm,
    vectorstore=chroma_db,
    document_contents="AQL 查询语言、图遍历、Graph 概念...",
    metadata_field_info=[AttributeInfo(name="page", type="integer", ...), ...],
    structured_query_translator=ChromaStructuredQueryTranslator(),
    verbose=True,
)
```

**关键代码片段 3：`ContextualCompressionRetriever` —— 让 LLM 从召回片段里再"提取一次"**

```python
zh_extract_prompt = PromptTemplate(
    template="根据下面的问题，从上下文中**原样**提取与问题相关的内容片段。"
             "如果没有相关内容，返回 NO_OUTPUT。... 问题: {question} 上下文: {context}",
    output_parser=NoOutputParser(),
)
compressor = LLMChainExtractor.from_llm(llm=deepseek_llm, prompt=zh_extract_prompt)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)
```

**关键代码片段 4：`map_reduce` QA 链（对召回片段先 summarize，再合并）**

```python
qa_chain = RetrievalQA.from_chain_type(
    ds_model,
    retriever=vectordb.as_retriever(search_kwargs={"k": 8}),
    return_source_documents=True,
    chain_type_kwargs={
        "question_prompt": question_prompt,    # map 阶段：{question}+{context} → 摘要
        "combine_prompt": combine_prompt,      # reduce 阶段：{summaries}+{question} → 最终答案
    },
    chain_type="map_reduce",
)
```

**关键代码片段 5：`ConversationalRetrievalChain` 带记忆的多轮问答**

```python
memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)
qa = ConversationalRetrievalChain.from_llm(ds_model, retriever=vectordb.as_retriever(), memory=memory)
qa.invoke({"question": "How many path finding methods in AQL?"})
qa.invoke({"question": "How to write full path AQL?"})      # 能复用历史
qa.invoke({"question": "what's my first question in the conversation?"})
# 已知问题：该版本不会回头看历史来回答"我第一个问题是什么"
```

---

### 3.10 `rag/chats/embeddings.py` —— 自定义 `QwenEmbeddings`

**功能：** 继承 `langchain_core.embeddings.Embeddings`，通过 `OpenAI` SDK 调通义 `text-embedding-v3`（兼容 API）；遵守 Qwen 单批 ≤10 的限制分批调用；运行时用 `np.dot` 对比两条句子的余弦相似度。

**关键代码**

```python
class QwenEmbeddings(Embeddings):
    def __init__(self, model: str = "text-embedding-v3"):
        self._client = OpenAI(
            api_key=os.getenv("QWEN_API_KEY"),
            base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
        )
        self.model = model

    def embed_query(self, text):     return self._embed(text)
    def embed_documents(self, texts):
        BATCH_SIZE = 10                              # Qwen 限制
        return [e for i in range(0, len(texts), BATCH_SIZE)
                  for e in self._embed_batch(texts[i:i+BATCH_SIZE])]

# 0.90 vs 0.47 —— "i like dogs" 与 "i love canines" 远高于与 "the weather is ugly" 的相似度
print(np.dot(embeddings1, embeddings2))
```

---

### 3.11 `rag/chats/text_split.py` —— 三种切分器横向对比

**功能：** 用同一个示例文本对比 `CharacterTextSplitter` / `RecursiveCharacterTextSplitter` / `TokenTextSplitter` 在 `chunk_size` 与 `chunk_overlap` 下的产出，理解为什么 RAG 几乎都默认选 `RecursiveCharacterTextSplitter`。

**关键结论**（脚本运行后能直接看到）

| 切分器 | 行为 |
| --- | --- |
| `CharacterTextSplitter(separator=" ")` | 按单字符分隔符切，`chunk_size=26`、`overlap=4` 时英文按空格拆分 |
| `RecursiveCharacterTextSplitter(["\n\n","\n"," ",""])` | 优先按段落→行→空格→字符递归回退，最贴合自然语言结构 |
| `TokenTextSplitter(chunk_size=1)` | 按模型 token 切，`"foo bar bazzyfoo"` → `['foo',' bar',' baz','zy','foo']` |

---

## 四、环境变量

| 变量 | 用途 |
| --- | --- |
| `DEEPSEEK_API_KEY` | 几乎所有脚本的 LLM |
| `OPENAI_API_KEY` | `agents/agents.py`、`agents/assistant.py`、`rag/chats/chat_agents.py` 中 `gpt-4o` / 兼容 OpenAI 模型 |
| `QWEN_API_KEY` (`DASHSCOPE_API_KEY`) | `rag/chats/embeddings.py` 的 `QwenEmbeddings` |

---

## 五、学习路径建议

1. **入门**：`basics/prompts.py` → `basics/chains.py` → `basics/memory.py`
2. **工具与 Agent**：`agents/tools.py` → `agents/agents.py` → `agents/assistant.py`
3. **RAG**：`rag/qa.py` → `rag/chats/text_split.py` → `rag/chats/embeddings.py` → `rag/chats/chat_agents.py` → `rag/evaluation.py`


+++
title = "LangChain入门教程：从零到一构建智能应用"
lastmod = 2025-07-10T17:00:00+08:00
tags = ["langchain", "llm", "agent"]
categories = ["langchain", "llm", "agent"]
draft = false
author = "b40yd"
+++

-----

### **LangChain入门教程：从零到一构建智能应用**

**章节目录**

  * **第一章：LangChain 简介**

      * 1.1 什么是 LangChain？
      * 1.2 为什么选择 LangChain？
      * 1.3 核心概念概览

  * **第二章：环境搭建与准备**

      * 2.1 安装 LangChain 相关库
      * 2.2 准备大模型 (配置Ollama与Qwen/DeepSeek)

  * **第三章：LangChain 核心组件 (Components)**

      * 3.1 模型 I/O (Model I/O)
          * 3.1.1 与模型交互 (LLMs & Chat Models)
          * 3.1.2 提示模板 (Prompt Templates)
          * 3.1.3 输出解析器 (Output Parsers)
      * 3.2 LangChain表达式语言 (LCEL)

  * **第四章：链 (Chains) - 将组件串联起来**

      * 4.1 什么是链？
      * 4.2 使用 LCEL 构建第一个链
      * 4.3 一个示例：自动生成标准格式的JSON

  * **第五章：检索增强生成 (RAG) - 赋予模型外部知识**

      * 5.1 RAG 简介
      * 5.2 文档加载器 (Document Loaders)
      * 5.3 文本分割 (Text Splitters)
      * 5.4 向量存储与嵌入 (Vector Stores & Embeddings)
      * 5.5 检索器 (Retrievers)
      * 5.6 构建完整的 RAG 链

  * **第六章：记忆 (Memory) - 让对话持续下去**

      * 6.1 为什么需要记忆？
      * 6.2 在链中集成记忆
      * 6.3 常见的记忆类型

  * **第七章：Agents - 让模型自主思考和行动**

      * 7.1 什么是 Agent？
      * 7.2 Agent 核心概念：工具 (Tools) 和 Agent 执行器 (Agent Executor)
      * 7.3 创建一个使用搜索工具的 Agent

  * **第八章：总结与进阶**

      * 8.1 本教程回顾
      * 8.2 进阶方向：LangSmith, LangGraph, 自定义组件
      * 8.3 结语

-----

接下来，我将完善所有小节的内容。

-----

### **LangChain入门教程：从零到一构建智能应用**

欢迎来到 LangChain 的世界！本教程旨在帮助开发者快速上手，利用 LangChain 构建强大的、由语言模型驱动的应用程序。我们将从基本概念讲起，逐步深入到高级功能，如 RAG 和 Agents。

### **第一章：LangChain 简介**

#### **1.1 什么是 LangChain？**

LangChain 是一个开源框架，旨在简化使用大型语言模型（LLM）创建应用程序的过程。它提供了一套模块化的工具、组件和接口，使开发者能够将 LLM 与外部数据源、计算资源和其他 API 连接起来，构建出更复杂、更强大的应用。

#### **1.2 为什么选择 LangChain？**

  * **组件化:** LangChain 将复杂任务分解为可重用的组件（如模型、提示、索引、链、Agent），便于组合和定制。
  * **生态系统:** 它集成了数百种模型、数据源和工具，让你轻松接入不同的服务。
  * **灵活性:** 通过 LangChain 表达式语言（LCEL），你可以用非常直观的方式将组件“管道化”连接起来，构建从简单到复杂的逻辑。
  * **社区活跃:** 拥有庞大的开发者社区和丰富的用例库，可以快速找到解决方案。

#### **1.3 核心概念概览**

  * **Components:** 构建应用的基础模块，包括模型I/O、数据检索、记忆等。
  * **Chains:** 将组件按特定顺序组合起来，完成一个连贯的任务。
  * **RAG (Retrieval-Augmented Generation):** 一种让模型能够从外部知识库中检索信息并据此生成回答的强大模式。
  * **Agents:** 赋予模型使用工具（如搜索引擎、计算器）的能力，让模型可以自主决策和执行任务。
  * **LCEL (LangChain Expression Language):** 一种声明式的管道构建方式，是现代 LangChain 开发的核心。

### **第二章：环境搭建与准备**

#### **2.1 安装 LangChain 相关库**

首先，确保你已安装 Python (3.8+)。然后通过 pip 安装 LangChain 的核心库、社区库以及我们将要使用的 Ollama 集成。

```bash
pip install langchain langchain-core langchain-community langchain-experimental ollama
```

#### **2.2 准备大模型 (配置Ollama与Qwen/DeepSeek)**

我们将使用 [Ollama](https://ollama.com/) 在本地运行开源大模型，这非常方便。

1.  **安装 Ollama:** 根据你的操作系统，从官网下载并安装 Ollama。

2.  **拉取模型:** 打开终端，运行以下命令来下载 `qwen` (通义千问) 或 `deepseek-coder` 模型。你可以选择其中一个。

    ```bash
    # 拉取 Qwen 7B 聊天模型
    ollama pull qwen:7b-chat

    # 或者拉取 DeepSeek Coder 6.7B 指令模型
    ollama pull deepseek-coder:6.7b-instruct
    ```

3.  **运行 Ollama:** 确保 Ollama 服务正在后台运行。

### **第三章：LangChain 核心组件 (Components)**

这是 LangChain 的基石。我们将介绍三个最重要的组件：模型、提示模板和输出解析器。

#### **3.1 模型 I/O (Model I/O)**

##### **3.1.1 与模型交互 (LLMs & Chat Models)**

LangChain 提供了统一的接口与各类模型交互。我们使用 `ChatOllama` 来连接本地的 `qwen` 模型。

```python
from langchain_community.chat_models import ChatOllama
from langchain_core.messages import HumanMessage, SystemMessage

# 初始化模型，这里我们使用 qwen:7b-chat
# 如果你下载的是其他模型，请替换 model="qwen:7b-chat"
llm = ChatOllama(model="qwen:7b-chat")

# 准备消息
messages = [
    SystemMessage(content="你是一个专业的AI助手。"),
    HumanMessage(content="你好，请用一句话介绍一下自己。"),
]

# 调用模型
response = llm.invoke(messages)
print(response.content)
```

> **输出可能为:** 你好，我是一个由阿里巴巴通义实验室开发的大语言模型。

##### **3.1.2 提示模板 (Prompt Templates)**

硬编码提示是不灵活的。提示模板可以帮助我们动态地创建提示。

```python
from langchain_core.prompts import ChatPromptTemplate

# 创建一个模板，其中 {topic} 是一个占位符
prompt_template = ChatPromptTemplate.from_template(
    "请给我写一首关于 {topic} 的五言绝句。"
)

# 格式化模板，传入具体的值
prompt_value = prompt_template.invoke({"topic": "月亮"})
print(prompt_value)

# 我们可以直接将模板和模型连接起来
chain = prompt_template | llm
response = chain.invoke({"topic": "长城"})
print(response.content)
```

##### **3.1.3 输出解析器 (Output Parsers)**

LLM 的输出通常是字符串，但我们常常需要结构化的数据（如 JSON 或列表）。输出解析器能帮我们实现这个转换。

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser

# 默认的字符串解析器
parser = StrOutputParser()

# 构建链
chain = prompt_template | llm | parser

# 调用链，输出将是纯字符串
result = chain.invoke({"topic": "西湖"})
print(result)

# 示例：JSON 解析器
json_prompt = ChatPromptTemplate.from_template(
    "根据用户信息 `{user_input}`，提取姓名和年龄，并以JSON格式返回。JSON格式为: {format_instructions}"
)
json_parser = JsonOutputParser()

# 将解析器的格式说明注入到提示中
chain = json_prompt | llm | json_parser

response = chain.invoke({
    "user_input": "你好，我叫张三，今年25岁。",
    "format_instructions": json_parser.get_format_instructions(),
})
print(response)
```

> **输出可能为:** `{'name': '张三', 'age': 25}`

#### **3.2 LangChain表达式语言 (LCEL)**

你已经看到了 `|` 这个符号，它就是 LCEL 的核心。它允许你像 Unix 管道一样，将不同的组件以声明式的方式链接起来，非常简洁和强大。`chain = prompt | llm | parser` 就是一个典型的 LCEL 链。

### **第四章：链 (Chains) - 将组件串联起来**

#### **4.1 什么是链？**

链是 LangChain 的核心概念，它代表了一系列按顺序执行的调用（无论是对模型、工具还是自定义函数）。

#### **4.2 使用 LCEL 构建第一个链**

我们已经构建了好几个链了！LCEL 是构建链的首选方式。它的好处在于：

  * **流式处理:** 可以轻松实现服务器的流式响应。
  * **并行执行:** 可以将多个组件并行处理。
  * **可追溯性:** LangSmith 等工具可以清晰地看到链中每一步的输入输出。

#### **4.3 一个示例：自动生成标准格式的JSON**

这个例子结合了我们前面学到的所有组件，非常实用。

```python
from langchain_community.chat_models import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field

# 1. 定义我们期望的输出数据结构 (Pydantic 模型)
class Poem(BaseModel):
    title: str = Field(description="这首诗的标题")
    author: str = Field(description="这首诗的作者")
    poem_text: str = Field(description="诗的全文内容")

# 2. 初始化模型和解析器
llm = ChatOllama(model="qwen:7b-chat")
parser = JsonOutputParser(pydantic_object=Poem)

# 3. 创建提示模板，并包含格式化指令
prompt = ChatPromptTemplate.from_template(
    "请根据主题 '{topic}' 创作一首古诗。\n{format_instructions}"
)

# 4. 使用 LCEL 构建链
chain = prompt | llm | parser

# 5. 调用链
response = chain.invoke({
    "topic": "友谊",
    "format_instructions": parser.get_format_instructions()
})

print(response)
print(f"标题: {response['title']}")
```

### **第五章：检索增强生成 (RAG) - 赋予模型外部知识**

RAG 是 LangChain 最强大的应用之一。它通过“检索”外部文档来“增强”LLM 的“生成”能力，解决了模型知识陈旧和产生幻觉的问题。

#### **5.1 RAG 简介**

流程通常是：

1.  **加载 (Load):** 将外部文档（PDF, TXT, 网页等）加载进来。
2.  **分割 (Split):** 将长文档切分成小块。
3.  **存储 (Store):** 将文本块进行“嵌入”（转换成向量）并存入向量数据库。
4.  **检索 (Retrieve):** 当用户提问时，将问题也嵌入成向量，并在数据库中查找最相似的文本块。
5.  **生成 (Generate):** 将原始问题和检索到的文本块一起作为上下文，提交给 LLM 生成答案。

#### **RAG 完整代码示例**

为了运行以下代码，你需要安装额外的库：
`pip install faiss-cpu beautifulsoup4`

```python
import bs4
from langchain_community.chat_models import ChatOllama
from langchain_community.document_loaders import WebBaseLoader
from langchain_community.embeddings import OllamaEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 加载 (Load): 从网页加载数据
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()

# 2. 分割 (Split): 将文档分割成小块
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)

# 3. 存储 (Store): 创建向量数据库
# 使用Ollama的嵌入功能，也可以换成其他嵌入模型
embeddings = OllamaEmbeddings(model="qwen:7b-chat")
vectorstore = FAISS.from_documents(documents=splits, embedding=embeddings)

# 4. 检索 (Retrieve): 创建检索器
retriever = vectorstore.as_retriever()

# 5. 生成 (Generate): 创建RAG链
llm = ChatOllama(model="qwen:7b-chat")
prompt = ChatPromptTemplate.from_template("""
你是一个问答机器人。请根据下面提供的上下文来回答用户的问题。
如果上下文中没有相关信息，就说你不知道。

上下文: {context}
问题: {question}
""")

# 定义RAG链
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 调用链
question = "What is Task Decomposition?"
response = rag_chain.invoke(question)
print(response)
```

> **输出可能为:** Based on the context provided, Task Decomposition is a technique used to break down complex tasks into smaller, more manageable steps...

### **第六章：记忆 (Memory) - 让对话持续下去**

#### **6.1 为什么需要记忆？**

默认情况下，LLM 是无状态的，每次调用都是独立的。记忆组件让链能够记住之前的交互，从而实现连续对话。

#### **6.2 在链中集成记忆**

我们通常使用 `ConversationBufferMemory` 来存储对话历史。

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

llm = ChatOllama(model="qwen:7b-chat")
memory = ConversationBufferMemory()

# ConversationChain 是一个内置了记忆功能的链
conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True # 设置为True可以看到内部的提示变化
)

print(conversation.invoke("你好，我叫李华。"))
# 第二次调用，模型会记得我的名字
print(conversation.invoke("我叫什么名字？"))
```

#### **6.3 常见的记忆类型**

  * `ConversationBufferWindowMemory`: 只保留最近的 k 次对话。
  * `ConversationSummaryMemory`: 将对话历史总结后存储，节省空间。

### **第七章：Agents - 让模型自主思考和行动**

Agent 是 LangChain 中最复杂的概念。它让 LLM 具备了使用“工具”的能力，可以根据用户的输入，自主决定调用哪个工具来完成任务。

#### **7.1 什么是 Agent？**

Agent 的核心循环是：

1.  **思考 (Thought):** LLM 分析用户问题，决定下一步行动（是回答问题还是使用工具）。
2.  **行动 (Action):** 如果决定使用工具，则指定工具名称和输入。
3.  **观察 (Observation):** 执行工具并获得结果。
4.  LLM 接收观察结果，并重复此过程，直到任务完成。

#### **7.2 Agent 核心概念：工具 (Tools) 和 Agent 执行器 (Agent Executor)**

  * **Tools:** 一个函数或 API，Agent 可以调用它来获取信息或执行操作。例如：搜索引擎、计算器、数据库查询工具。
  * **Agent Executor:** 负责执行 Agent 的思考-行动循环的运行时。

#### **7.3 创建一个使用搜索工具的 Agent**

我们将使用 `DuckDuckGo` 作为搜索工具。需要安装：
`pip install duckduckgo-search`

```python
from langchain_community.tools import DuckDuckGoSearchRun
from langchain.agents import create_react_agent, AgentExecutor
from langchain import hub

# 1. 初始化模型和工具
llm = ChatOllama(model="qwen:7b-chat")
search = DuckDuckGoSearchRun()
tools = [search]

# 2. 获取预置的 Agent 提示模板
# "ReAct" 是一种流行的 Agent 思考模式 (Reasoning and Acting)
prompt = hub.pull("hwchase17/react")

# 3. 创建 Agent
agent = create_react_agent(llm, tools, prompt)

# 4. 创建 Agent 执行器
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 5. 运行 Agent
response = agent_executor.invoke({
    "input": "今天北京的天气怎么样？最高温度是多少？"
})

print(response['output'])
```

在 `verbose=True` 模式下，你可以清晰地看到 Agent 的思考过程、它如何决定使用搜索工具、以及最终如何根据搜索结果生成答案。

### **第八章：总结与进阶**

#### **8.1 本教程回顾**

我们从 LangChain 的基本概念出发，学习了如何配置本地模型，掌握了核心组件（模型、提示、解析器）和 LCEL 的使用。我们还动手实践了两个最核心的应用范式：检索增强生成（RAG）和 Agent。

#### **8.2 进阶方向**

  * **LangSmith:** 一个用于调试、监控和评估 LLM 应用的平台。它能让你清晰地看到链和 Agent 的每一步执行过程，是专业开发的必备工具。
  * **LangGraph:** LangChain 的一个扩展，用于构建更复杂的、带有循环和条件分支的 Agent，甚至可以实现多个 Agent 的协作。
  * **自定义组件:** 学习如何创建自己的 Document Loader, Tool, Parser，以满足特定的业务需求。

#### **8.3 结语**

LangChain 打开了通往智能应用新世界的大门。它将复杂的 LLM 调用抽象为简单、可组合的模块，让开发者可以更专注于应用逻辑本身。希望本教程能为你打下坚实的基础，祝你在 LangChain 的探索之路上不断创造出令人惊艳的应用！
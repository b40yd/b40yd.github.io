+++
title = "LangChain 入门指南：构建强大的大语言模型应用"
lastmod = 2025-07-10T18:00:00+08:00
tags = ["langchain", "llm", "agent"]
categories = ["langchain", "llm", "agent"]
draft = false
author = "b40yd"
+++

## **第一章：LangChain 简介与核心概念**

本章将为您介绍 LangChain 框架，包括其核心价值、应用场景、模块化架构以及构成其应用基石的关键组件。理解这些概念是掌握 LangChain 的第一步。

### **1.1 什么是 LangChain？**

LangChain 是一个强大的开源框架，旨在简化基于大语言模型（LLMs）的应用程序的开发。它提供了一套工具、组件和接口，帮助开发者从原型设计到生产部署，全面管理 LLM 应用的生命周期 1。该框架的核心在于其能够标准化 LLMs、嵌入模型和向量存储的接口，并与数百家提供商集成。它支持开发应用程序、使用 LangGraph 构建有状态的智能体，并通过 LangSmith 进行生产化（检查、监控和评估），最终通过 LangGraph Platform 进行部署 1。

最初，人们可能将 LangChain 仅仅视为一个代码库。然而，深入了解其构成会发现，它远不止于此，而是一个包含 LangSmith 和 LangGraph 的全面“生态系统” 1。这种设计表明 LangChain 的愿景超越了简单的 LLM 调用封装，而是致力于提供一个端到端的解决方案，覆盖了 LLM 应用从开发、调试、优化到部署的整个生命周期。这种“全栈”的特性是其区别于其他 LLM 工具的关键所在。对于开发者而言，这意味着选择 LangChain 不仅仅是选择了一个库，而是选择了一个集成的开发和运维环境。这显著降低了 LLM 应用生产化的门槛，但也意味着需要学习和掌握的工具集更为广泛。

### **1.2 LangChain 的核心价值与应用场景**

LangChain 的主要价值在于其模块化、可组合性以及对 LLM 应用开发流程的抽象。它允许开发者将复杂的 LLM 任务分解为可管理的步骤，并通过链式结构将这些步骤连接起来，从而构建出更复杂、更智能的应用 3。

LangChain 适用于多种 LLM 应用场景，包括但不限于：

* **聊天机器人 (Chatbots)**: 构建能够理解上下文、进行多轮对话的智能助手 5。  
* **内容摘要 (Content Summarization)**: 将大量文本（如书籍、文章、会议记录）总结成简洁的见解 5。  
* **问答系统 (Question Answering)**: 基于特定知识库提供准确答案 7。  
* **智能代理 (Intelligent Agents)**: 赋予 LLM 决策能力，使其能够选择并执行一系列工具来完成复杂任务 8。  
* **数据增强生成 (Retrieval-Augmented Generation, RAG)**: 结合外部知识库，提高 LLM 回答的准确性和时效性 7。

从框架的演进来看，LangChain 的设计体现了从简单调用到复杂决策流的发展。例如，LangChain 能够将复杂动作分解为有序的小步骤，就像洗衣流程一样，一个步骤的输出作为下一个步骤的输入，形成“链”的概念 5。在此基础上，LangChain 引入了“智能体”（Agent）的概念，其核心在于将 LLM 用作“推理引擎”，动态地决定要采取的行动序列和工具选择 8。这种从固定流程的“链”到动态决策的“代理”的演进，表明 LangChain 旨在从简单的文本生成，发展到能够进行复杂决策、与外部世界交互的智能系统。动态决策和工具使用是构建真正智能应用的关键。

### **1.3 LangChain 架构概览 (langchain-core, langchain, langchain-community, langgraph)**

LangChain 框架由多个开源库组成，每个库承担不同的职责，共同构建了一个灵活且强大的生态系统 1：

* **langchain-core**: 该库包含 LangChain 的基础抽象和 LangChain Expression Language (LCEL)。它是所有其他 LangChain 包的基础，定义了聊天模型和其他组件的核心接口 1。  
* **集成包 (例如 langchain-openai, langchain-anthropic, langchain-deepseek, langchain-qwq)**: 这些是轻量级的独立包，专门用于与特定的 LLM 提供商进行集成。它们由 LangChain 团队和集成开发者共同维护，确保了与最新模型和 API 的兼容性 1。  
* **langchain**: 这个包包含了链（Chains）、代理（Agents）和检索策略（Retrieval Strategies）等高级组件。它们构成了应用程序的认知架构，负责协调和执行复杂的 LLM 任务 1。  
* **langchain-community**: 该库汇集了由社区维护的第三方集成。它使得 LangChain 能够快速吸纳来自社区的贡献，扩展其功能和支持范围 1。  
* **langgraph**: 这是一个强大的编排框架，专门用于构建有状态、多角色的 LLM 应用程序。它通过将步骤建模为图中的边和节点来工作，支持生产级的智能体。尽管它可以独立于 LangChain 使用，但它与 LangChain 组件无缝集成，是构建复杂 LLM 工作流的关键 1。

这种细致的模块化设计，特别是将核心抽象、高级组件、社区贡献和特定集成分别打包，体现了 LangChain 高度的可扩展性和维护性。这种架构允许开发者根据需求选择性安装和使用组件，从而避免了不必要的依赖膨胀。同时，它也鼓励了社区贡献，通过 langchain-community 快速集成新的工具和模型，从而保持了框架的活力和前沿性。

### **1.4 LangChain 核心组件速览 (LLMs, Prompts, Chains, Agents, Tools, Retrievers, Memory, Callbacks)**

LangChain 的强大功能源于其精心设计的核心组件，它们可以独立使用，也可以组合起来构建复杂的 LLM 应用 10。

* **LLMs (Large Language Models)**: 在 LangChain 的语境中，LLMs 通常指代传统的文本输入-文本输出模型。它们接收一个字符串作为输入，并返回一个字符串作为输出。在 LangChain 中，这些模型通常通过 BaseLLM 接口表示 12。  
* **Chat Models (聊天模型)**: 这是更现代的语言模型，它们以消息序列作为输入，并返回聊天消息作为输出。这些模型支持为对话消息分配不同的角色（如系统、用户、助手消息），是当前主流的 LLM 交互方式 12。  
* **Prompts (提示词)**: 提示词是用于指导语言模型响应的指令和上下文。LangChain 提供了灵活的提示词模板来构建动态和可复用的提示词，从而有效地引导模型行为 14。  
* **Chains (链)**: 链用于将多个组件（如 LLM、提示词、其他链）按特定顺序连接起来，实现复杂的工作流。它们是 LangChain 的核心概念之一，用于结构化和管理多步骤任务 3。  
* **Agents (智能体)**: 智能体使用 LLM 作为推理引擎，动态决定要采取的行动序列，并调用外部工具来完成复杂任务。它们赋予 LLM 决策和执行能力 8。  
* **Tools (工具)**: 工具是智能体与外部世界交互的能力接口。它们可以是搜索、计算、API 调用等任何能够执行特定操作的函数或 API 封装 16。  
* **Retrievers (检索器)**: 检索器是从数据源（如向量数据库）检索相关文档的组件，为 LLM 提供外部知识，尤其在构建 RAG（检索增强生成）应用中发挥关键作用 7。  
* **Memory (记忆)**: 记忆系统使 LLM 应用能够记住过去的对话历史或关键信息，从而在多轮对话中保持上下文连贯性 19。  
* **Callbacks (回调)**: 回调系统允许开发者在 LLM 应用的不同阶段（如模型开始、链结束、工具错误）插入自定义逻辑，用于日志、监控、流式传输和调试等 21。

为了更清晰地理解这些核心组件，下表提供了它们的概览：

**Table 1: LangChain 核心组件概览**

| 组件名称 | 简要描述 | 在 LLM 应用中的关键作用 |
| :---- | :---- | :---- |
| LLMs (传统语言模型) | 接收字符串输入，返回字符串输出的语言模型。 | 处理简单的文本补全任务。 |
| Chat Models (聊天模型) | 接收消息序列输入，返回聊天消息输出的现代语言模型。 | 构建多轮对话、支持角色分配，是当前主流的 LLM 交互方式。 |
| Prompts (提示词) | 指导语言模型响应的指令和上下文。 | 引导模型行为，确保输出相关性和连贯性。 |
| Chains (链) | 将多个组件按特定顺序连接起来，实现复杂工作流。 | 结构化复杂任务，实现多步骤处理。 |
| Agents (智能体) | 使用 LLM 作为推理引擎，动态决定行动序列并调用工具。 | 赋予 LLM 决策能力，使其能自主完成复杂任务。 |
| Tools (工具) | 智能体与外部世界交互的能力接口。 | 扩展 LLM 能力，使其能执行外部操作（如搜索、计算）。 |
| Retrievers (检索器) | 从数据源检索相关文档，为 LLM 提供外部知识。 | 增强 LLM 知识，实现 RAG 应用，提高回答准确性。 |
| Memory (记忆) | 使 LLM 应用能够记住过去的对话历史或关键信息。 | 维持对话上下文，实现多轮对话连贯性。 |
| Callbacks (回调) | 允许在 LLM 应用不同阶段插入自定义逻辑。 | 用于日志、监控、流式传输和调试，提升可观测性。 |

## **第二章：环境搭建与模型集成**

本章将指导您完成 LangChain 的环境准备，并重点介绍如何集成和使用 Qwen3 和 DeepSeek 这两款强大的语言模型。

### **2.1 Python 环境准备与 LangChain 安装**

在开始使用 LangChain 之前，建议先准备好 Python 环境。为了避免依赖冲突，通常推荐使用 Python 3.9 或更高版本，并创建虚拟环境来管理项目依赖。

LangChain 框架本身可以通过 pip 安装。核心包是 langchain 和 langchain-core。安装命令如下：

```Bash

pip install -U langchain langchain-core
```

在 LangChain 的快速发展过程中，其 API 和内部实现可能会随版本更新而变化。例如，Python 3.10 及以下版本的异步回调可能需要手动传播 23。同时，各种集成包的安装命令通常会使用

-U 参数（例如 pip install -U langchain-qwq 24 和 pip install -qU langchain-deepseek 25），这意味着更新到最新版本。因此，开发者在选择 Python 版本和 LangChain 包版本时，应关注官方文档的兼容性说明，并定期更新以获取最新功能和修复，同时警惕潜在的向后不兼容性。这反映了 LLM 框架领域持续迭代和快速演进的特性。

### **2.2 集成 Qwen3 模型**

Qwen3 (通义千问) 是阿里云开发的大规模语言模型，LangChain 提供了对其的良好集成。

#### **2.2.1 Qwen3 API Key 配置与 langchain-qwq 安装**

要通过 API 使用 Qwen3 模型，首先需要获取 DashScope API Key。获取密钥后，需要将其配置为环境变量 DASHSCOPE_API_KEY。对于中国国内用户，可能还需要设置 DASHSCOPE_API_BASE 为国内端点，因为 langchain-qwq 默认使用国际版本 24。

安装 langchain-qwq 包的命令如下：

```Bash
pip install -U langchain-qwq
```

langchain-qwq 包提供了对 QwQ 模型和 Qwen 系列模型的全面支持，包括 Qwen3 系列模型（支持混合推理）、Qwen-Max、Qwen2.5 等。它还支持 Qwen-VL 系列视觉模型，并提供同步和异步流式传输、工具调用（支持并行执行）以及结构化输出（JSON 模式）等高级功能，同时允许直接访问模型的推理/思考内容 24。

#### **2.2.2 使用 ChatOllama 运行本地 Qwen3**

除了通过 API 调用，Qwen3 也可以在本地通过 Ollama 运行，这为开发者提供了更大的灵活性和控制权。  
首先，需要安装 Ollama 并拉取 Qwen3 模型：

```Bash
curl -fsSL https://ollama.com/install.sh | sh  
ollama pull qwen3
```

然后，安装 langchain-ollama 包：

```Bash
pip install -U langchain-ollama
```

最后，实例化 ChatOllama 模型即可使用本地部署的 Qwen3：

```Python

from langchain_ollama import ChatOllama  
llm = ChatOllama(model="qwen3")
```

#### **2.2.3 Qwen3 模型特性与 LangChain 应用示例**

Qwen3 模型在 LangChain 中通常作为 Chat Model 使用，这意味着它接收消息列表作为输入并返回消息列表作为输出，这与现代 LLM 的交互范式一致。

Qwen3 可以与 LangChain 的检索器结合，构建基于本地知识库的问答系统，即 RAG（检索增强生成）应用。这个过程涉及多个步骤：首先加载和读取文本文件，然后将文本分割成小块，接着将这些文本块和用户问题都转换为数值向量（嵌入）。随后，通过相似度搜索找到与问题最相关的文本块，并将这些匹配到的文本作为上下文，与原始问题一起组成一个完整的提示词，最后提交给 Qwen 模型生成答案 27。

以下是一个使用本地 Qwen3 构建 RAG 问答应用的简化示例：

```Python

# 示例代码片段 [27]  
from langchain_ollama import ChatOllama  
from langchain_core.prompts import ChatPromptTemplate  
from langchain.chains import RetrievalQA # 注意：新版LangChain可能推荐LCEL链  
from langchain_community.document_loaders import TextLoader  
from langchain_community.embeddings import HuggingFaceEmbeddings  
from langchain_community.vectorstores import FAISS  
from langchain.text_splitter import CharacterTextSplitter  
import os


# 1. 初始化模型  

llm_qwen = ChatOllama(model="qwen3", temperature=0.01) # 使用本地Qwen3

# 2. 准备文档和嵌入模型  

# 假设您有一个名为 'knowledge_base.txt' 的文件  
# with open('knowledge_base.txt', 'w', encoding='utf-8') as f:  
#     f.write("LangChain 是一个用于开发 LLM 应用程序的框架。它简化了 LLM 应用的整个生命周期。")  
#     f.write("nQwen3 是阿里云开发的大语言模型。")  
#     f.write("nDeepSeek 是一个以强大推理能力著称的模型。")

loader = TextLoader('knowledge_base.txt', autodetect_encoding=True)  
documents = loader.load()  
text_splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=50)  
docs = text_splitter.split_documents(documents)

# 使用HuggingFace Embeddings (需要安装 sentence-transformers)  

# pip install sentence-transformers  
embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh-v1.5") # 示例嵌入模型

vectorstore = FAISS.from_documents(docs, embeddings)  
retriever = vectorstore.as_retriever()

# 3. 构建提示词模板  
prompt_template = ChatPromptTemplate.from_messages([  
    ("system", "你是一个有用的助手，请根据提供的上下文回答问题。如果不知道答案，请说明。"),  
    ("human", "已知信息: {context}nn问题: {question}")  
])

# 4. 构建检索问答链 (在新版LangChain中，更推荐使用LCEL管道)  
# 这是一个简化的示例，实际生产中可能需要更复杂的LCEL链  

qa_chain = (  
    {"context": retriever, "question": prompt_template}

| llm_qwen  
)

# 5. 调用链  

response_langchain = qa_chain.invoke({"question": "LangChain 是什么？"})  
print(f"LangChain 是什么？ {response_langchain.content}")

response_qwen = qa_chain.invoke({"question": "Qwen3 是谁开发的？"})  
print(f"Qwen3 是谁开发的？ {response_qwen.content}")

response_deepseek = qa_chain.invoke({"question": "DeepSeek 是谁开发的？"})  
print(f"DeepSeek 是谁开发的？ {response_deepseek.content}")
```

通过 Ollama 在本地运行 Qwen3 的方法，与通过 API Key 访问 Qwen3 的方式形成了对比 24。这种差异表明了两种截然不同的部署策略。API 方式方便快捷，但依赖云服务且可能产生费用；而本地 Ollama 部署则强调了数据隐私、成本控制和离线能力的重要性。对于企业或个人开发者而言，在本地运行模型可以更好地控制数据流，避免敏感信息泄露，并可能在长期使用中降低成本，尤其是在模型推理量大的情况下。这反映了 LLM 应用开发中对灵活性和自主性的需求。

### **2.3 集成 DeepSeek 模型**

DeepSeek 模型以其强大的推理能力和对思维过程的暴露而闻名 28。LangChain 提供了对其的良好集成。

#### **2.3.1 DeepSeek API Key 配置与 langchain-deepseek 安装**

要使用 DeepSeek 模型，首先需要获取 DeepSeek API Key，并将其设置为 DEEPSEEK_API_KEY 环境变量 25。

安装 langchain-deepseek 包的命令如下：

```Bash
pip install -qU langchain-deepseek
```

#### **2.3.2 使用 ChatDeepSeek 实例化与调用**

DeepSeek 模型在 LangChain 中通过 ChatDeepSeek 类进行集成。实例化时，model 参数可以设置为 "deepseek-chat"（支持工具调用和结构化输出）或 "deepseek-reasoner"（DeepSeek-R1，不支持这些功能）25。

以下是 ChatDeepSeek 的基本调用示例：

```Python

import getpass  
import os  
from langchain_deepseek import ChatDeepSeek  
from langchain_core.messages import HumanMessage, SystemMessage

# 确保设置 DEEPSEEK_API_KEY 环境变量  
# if not os.getenv("DEEPSEEK_API_KEY"):  
#     os.environ = getpass.getpass("Enter your DeepSeek API key: ")

llm_deepseek = ChatDeepSeek(  
    model="deepseek-chat",  
    temperature=0,  
    max_tokens=None,  
    timeout=None,  
    max_retries=2,  
)

messages =  
ai_msg = llm_deepseek.invoke(messages)  
print(ai_msg.content)  
# 预期输出: J'adore la programmation.

```

#### **2.3.3 DeepSeek 模型特性与 LangChain 应用示例**

ChatDeepSeek 支持多种高级功能，包括工具调用、结构化输出、token 级流式传输、原生异步操作以及 token 使用跟踪 25。

DeepSeek 模型可以与 ChatPromptTemplate 结合，构建更灵活的翻译链：

```Python

from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([  
    ("system", "你是一个乐于助人的助手，将 {input_language} 翻译成 {output_language}。"),  
    ("human", "{input}"),  
])

chain = prompt | llm_deepseek

response = chain.invoke({  
    "input_language": "English",  
    "output_language": "German",  
    "input": "I love programming.",  
})  
print(response.content)  
# 预期输出: Ich liebe Programmieren.

```

在选择 LLM 时，开发者需要关注模型的功能集。例如，DeepSeek-Chat 支持工具调用和结构化输出，而 DeepSeek-R1 (deepseek-reasoner) 则不支持这些功能 25。这表明即使是同一提供商的不同模型，其功能集也可能存在显著差异。因此，开发者在选择 LLM 时，不仅要考虑模型的性能和成本，更要关注其是否支持 LangChain 的高级特性，如工具调用和结构化输出。这些特性对于构建复杂的智能体和需要精确输出格式的应用至关重要。盲目选择可能导致功能受限或需要额外的后处理逻辑。

为了帮助用户更好地选择模型，下表对 Qwen3 和 DeepSeek 的 LangChain 集成特性进行了对比：

**Table 2: Qwen3 与 DeepSeek 模型特性对比**

| 特性 | Qwen3 (通义千问) | DeepSeek (深度求索) |
| :---- | :---- | :---- |
| **集成包** | langchain-qwq / langchain-ollama | langchain-deepseek |
| **API Key 配置** | DASHSCOPE_API_KEY, 可选 DASHSCOPE_API_BASE | DEEPSEEK_API_KEY |
| **本地部署** | 支持 (通过 Ollama) | 支持 (通过 Ollama) |
| **LangChain 类** | ChatOllama(model="qwen3") (本地) / Qwen (自定义 LLM) | ChatDeepSeek |
| **工具调用** | 支持 (通过 langchain-qwq) | 支持 (deepseek-chat 模型) |
| **结构化输出** | 支持 (JSON 模式) | 支持 (deepseek-chat 模型) |
| **流式传输** | 支持 (同步/异步) | 支持 (token 级) |
| **视觉模型** | 支持 (Qwen-VL 系列) | 不适用 (主要为文本模型) |
| **推理能力** | 强大，支持混合推理 | 强大，R1 模型可暴露思维过程 |

## **第三章：LangChain 核心组件详解与实践**

本章将深入探讨 LangChain 的各个核心组件，并通过 Qwen3 或 DeepSeek 模型进行实际的代码演示，帮助您理解它们的内部工作原理和应用方式。

### **3.1 LLMs 与 Chat Models (语言模型与聊天模型)**

#### **3.1.1 概念区分：LLMs vs. Chat Models**

在 LangChain 中，区分 LLMs 和 Chat Models 至关重要：

* **LLMs (传统语言模型)**: 这些模型通常接收一个字符串作为输入，并返回一个字符串作为输出。它们通常代表着较早期的语言模型（例如 OpenAI 的 text-davinci-003）。尽管这些底层模型是字符串输入/输出的，LangChain 的包装器也允许它们接受消息作为输入，并在内部将其格式化为字符串，以保持接口的统一性 12。  
* **Chat Models (聊天模型)**: 这些是更现代、更主流的语言模型，它们以消息序列（而非单个字符串）作为输入，并返回聊天消息作为输出。这些模型（例如 GPT-4、Claude、Qwen3、DeepSeek）支持为对话中的每条消息分配不同的角色（如 system、user、assistant），这有助于模型更好地理解对话的上下文和意图，从而生成更准确和相关的响应 12。

值得注意的是，现代 LLM 通常通过聊天模型接口访问。LangChain 强烈推荐使用 Chat Models 进行开发，并指出许多带有 "LLM" 后缀的旧模型通常不推荐直接使用，因为它们可能无法充分利用现代模型的全部功能 13。

#### **3.1.2 BaseChatModel 接口与基本调用 (invoke, stream, batch)**

LangChain 的聊天模型实现了 BaseChatModel 接口，该接口进一步实现了 Runnable 接口。这意味着所有聊天模型都支持一套标准化的调用方法，从而提供了一致的编程体验 13。

* **invoke**: 这是与聊天模型交互的主要方法。它接收一个消息列表作为输入，并返回一个消息列表作为输出。  
* **stream**: 该方法允许您在模型生成输出时流式获取结果，从而提供更快的用户体验，尤其是在生成长文本时。  
* **batch**: 该方法允许您将多个请求批量处理，从而提高处理效率和吞吐量。

聊天模型还支持一系列通用参数，以控制模型的行为，例如：

* **temperature**: 控制模型输出的随机性。较高的值（例如 1.0）使响应更具创造性，而较低的值（例如 0.0）使其更具确定性和专注性。  
* **timeout**: 设置等待模型响应的最大时间（秒），以防止请求无限期挂起。  
* **max_tokens**: 限制响应中 token（单词和标点符号）的总数，控制输出的长度。  
* **stop**: 指定停止序列，指示模型何时停止生成 token 13。

以下是使用 DeepSeek ChatDeepSeek 进行基本调用的示例：

```Python

from langchain_deepseek import ChatDeepSeek  
from langchain_core.messages import HumanMessage, SystemMessage

llm_deepseek = ChatDeepSeek(model="deepseek-chat", temperature=0)  
messages =  
response = llm_deepseek.invoke(messages)  
print(response.content)
```

Runnable 接口是 LangChain 框架的基石，它提供了一个统一的、可组合的编程模型。所有实现 Runnable 接口的组件（包括 LLMs、Prompt Templates、Chains、Retrievers 等）都可以通过管道操作符 (|) 轻松地连接起来，形成复杂的流程。这种设计不仅简化了组件的组合，还自动支持流式传输、异步操作和批处理，极大地提高了开发效率和应用性能。理解 Runnable 接口是掌握 LangChain 灵活性的关键。

为了更好地理解聊天模型的消息交互，下表列出了 LangChain 中常用的消息类型：

**Table 3: LangChain 消息类型**

| 消息类型 | 角色 (Role) | 描述 | 示例内容 |
| :---- | :---- | :---- | :---- |
| HumanMessage | user | 代表用户输入的消息。 | HumanMessage(content="你好，世界！") |
| AIMessage | assistant | 代表 AI 模型生成的消息。 | AIMessage(content="你好！有什么可以帮助你的吗？") |
| SystemMessage | system | 提供给模型的系统指令或上下文。 | SystemMessage(content="你是一个乐于助人的助手。") |
| ToolMessage | tool | 包含工具执行结果的消息。 | ToolMessage(content="天气：晴朗，25°C", tool_call_id="call_abc123") |
| FunctionMessage | function | (已弃用，由 ToolMessage 替代) 包含函数调用结果的消息。 | FunctionMessage(content="结果：100", name="add") |

### **3.2 Prompt Templates (提示词模板)**

提示词模板在 LLM 应用开发中扮演着核心角色，它们帮助将用户输入和参数转换为语言模型可理解的指令，从而引导模型生成相关且连贯的输出 15。这些模板接收字典作为输入，其中每个键代表提示词模板中需要填充的变量，并输出一个

PromptValue 对象。PromptValue 可以直接传递给 LLM 或 ChatModel，也可以转换为字符串或消息列表。

#### **3.2.1 String PromptTemplates：构建基础提示词**

* **目的**: String PromptTemplates 用于格式化单个字符串，通常适用于简单输入和补全式模型，例如那些期望单个文本输入的模型 15。  
* **示例**: 

```Python  
  from langchain_core.prompts import PromptTemplate  
  prompt_template = PromptTemplate.from_template("讲一个关于 {topic} 的笑话。")  
  print(prompt_template.invoke({"topic": "猫"}).text)  
  # 预期输出: "讲一个关于 猫 的笑话。" (实际输出是PromptValue对象，.text获取字符串)

```

#### **3.2.2 ChatPromptTemplates：多轮对话提示词**

* **目的**: ChatPromptTemplates 用于格式化消息列表，特别适用于聊天模型。这些“模板”本身由多个消息模板组成，每个消息都有其指定的角色（如“system”、“user”或“assistant”）15。  
* **示例**:  

```Python  
  from langchain_core.prompts import ChatPromptTemplate  
  chat_prompt = ChatPromptTemplate.from_messages([  
      ("system", "你是一个乐于助人的助手。"),  
      ("user", "讲一个关于 {topic} 的笑话。")  
  ])  
  print(chat_prompt.invoke({"topic": "狗"}).messages)  
  # 预期输出:
```

#### **3.2.3 MessagesPlaceholder：动态消息注入**

* **目的**: MessagesPlaceholder 提示词模板专门用于在 ChatPromptTemplate 中动态注入消息列表，例如完整的聊天历史记录。这在需要将用户提供的消息序列插入到提示词特定位置时非常有用 15。  
* **示例**:  

```Python  
  from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder  
  from langchain_core.messages import HumanMessage, AIMessage  
  from langchain_deepseek import ChatDeepSeek

  conversation_history = [  
      HumanMessage(content="你好！"),  
      AIMessage(content="你好！有什么可以帮助你的吗？")  
  ]

  prompt_with_history = ChatPromptTemplate.from_messages([  
      ("system", "你是一个友好的聊天机器人。"),  
      MessagesPlaceholder("chat_history"), # 动态注入消息列表  
      ("user", "{question}")  
  ])

  # 使用 DeepSeek 模型结合 Prompt Template  
  llm_deepseek = ChatDeepSeek(model="deepseek-chat", temperature=0.5)

  chain = prompt_with_history | llm_deepseek

  response = chain.invoke({  
      "chat_history": conversation_history,  
      "question": "你叫什么名字？"  
  })  
  print(response.content)  
  # 预期输出: 我是一个大型语言模型，由 DeepSeek AI 训练。

```

  此外，也可以通过使用 "placeholder" 类型元组与变量名的方式实现相同的功能，而无需显式使用 MessagesPlaceholder 类 15。

LangChain 的提示词模板不仅仅是字符串格式化工具，它们更是提示词工程（Prompt Engineering）的抽象层。通过占位符和格式化规则，模板能够保持提示词的一致性，减少随机性，并提高模型响应的可预测性 14。

MessagesPlaceholder 更是将动态对话历史的注入过程抽象化，避免了手动合并和格式化消息的繁琐。这种设计将提示词的构建从硬编码变为参数化和模块化，极大地提高了提示词的可维护性、可复用性和实验效率。这对于构建可扩展的 LLM 应用至关重要，因为提示词是影响模型行为的关键因素。

### **3.3 Chains (链)**

#### **3.3.1 链的定义与作用：串联组件**

链是 LangChain 的核心概念之一。它们用于将一系列对组件（如语言模型、文档检索器、其他链）的调用编码成一个序列，并提供一个简单的接口来执行这个序列 3。链允许开发者将复杂的 LLM 任务分解为有序的、可管理的步骤，从而构建出更复杂、更智能的应用 5。

#### **3.3.2 LLMChain 示例：最简单的链**

LLMChain 是 LangChain 中最简单的链之一。它的工作原理是接收用户输入，通过 PromptTemplate 将输入格式化为特定的提示词，然后将格式化后的提示词传递给 LLM 进行处理 4。

以下是一个使用本地 Qwen3 构建 LLMChain 的示例：

```Python

from langchain_ollama import ChatOllama  
from langchain_core.prompts import PromptTemplate  
from langchain.chains import LLMChain

# 1. 初始化模型 (使用本地 Qwen3)  
llm_qwen = ChatOllama(model="qwen3", temperature=0.7)

# 2. 定义提示词模板  
template = "请用一句话总结以下文本：n{text}"  
prompt = PromptTemplate.from_template(template)

# 3. 创建 LLMChain  
llm_chain = LLMChain(llm=llm_qwen, prompt=prompt, verbose=True)

# 4. 调用链  
text_to_summarize = "LangChain 是一个用于开发由大型语言模型（LLMs）驱动的应用程序的框架。它提供了一套工具、组件和接口，帮助开发者从原型设计到生产部署，全面管理 LLM 应用的整个生命周期。"  
response = llm_chain.invoke({"text": text_to_summarize})  
print(f"原始文本: {response['text']}")  
print(f"总结: {response['text_summary']}") # LLMChain通常会返回输入和输出

```

#### **3.3.3 链的常用方法 (invoke, run, bind, with_config 等)**

链作为 Runnable 接口的实现，继承了多种强大的操作方法，这些方法提供了灵活的执行和配置选项 3。

* **invoke()**: 同步执行链。它接收一个字典作为输入，并返回一个字典作为输出，其中包含链处理后的结果 3。  
* **ainvoke()**: invoke() 的异步版本，用于在异步编程环境（如 asyncio）中执行链 3。  
* **stream()**: 流式获取链的输出。当链生成内容时，可以实时接收到部分结果，这对于构建响应迅速的用户界面非常有用 3。  
* **astream_events()**: 异步流式获取链执行过程中的详细事件。这对于调试和监控非常有用，因为它提供了关于 on_chain_start、on_chain_stream 和 on_chain_end 等事件的实时信息，包括输入、中间块和最终输出 3。  
* **batch()**: 批量处理多个输入。该方法允许您同时向链发送多个请求，从而提高处理效率和吞吐量 3。  
* **run() / arun()**: 这些是方便方法，用于同步/异步执行链。它们接收参数或关键字参数，并直接返回字符串或对象输出，而不像 invoke() 那样返回包含丰富信息的字典 3。  
* **bind()**: 该方法用于将参数绑定到 Runnable，返回一个新的 Runnable。这在链中的 Runnable 需要一个在先前 Runnable 输出或用户输入中不存在的参数时非常有用。例如，可以为 LLM 绑定 stop 序列以控制生成长度 3。  

```Python  
  from langchain_ollama import ChatOllama  
  from langchain_core.output_parsers import StrOutputParser

  llm = ChatOllama(model='llama2') # 假设本地有llama2模型  
  chain_bound = (  
      llm.bind(stop=["three"]) # 绑定停止词 "three"

| StrOutputParser()  
)  
print(chain_bound.invoke("Repeat quoted words exactly: 'One two three four five.'"))  
# 预期输出: 'One two' (在遇到"three"时停止)  
```

* **with_config()**: 该方法允许在运行时配置 Runnable 的参数。例如，可以动态调整聊天模型的 max_tokens 参数 3。  

```Python  
  from langchain_deepseek import ChatDeepSeek  
  from langchain_core.runnables import ConfigurableField

  model = ChatDeepSeek(max_tokens=50).configurable_fields(  
      max_tokens=ConfigurableField(  
          id="output_token_number",  
          name="Max tokens in the output",  
          description="The maximum number of tokens in the output",  
      )  
  )  
  # 使用默认 max_tokens=50  
  print("max_tokens_50: ", model.invoke("Tell me something about artificial intelligence in one paragraph.").content)  
  # 运行时配置 max_tokens=200  
  print("max_tokens_200: ", model.with_config(  
      configurable={"output_token_number": 200}  
      ).invoke("Tell me something about artificial intelligence in one paragraph.").content  
  )
```

* **with_fallbacks()**: 为 Runnable 添加备用机制。当原始 Runnable 失败时，它会按顺序尝试每个备用 Runnable，从而提高应用的容错性 3。  
* **with_retry()**: 创建一个新的 Runnable，当原始 Runnable 在指定异常发生时自动重试。这对于处理瞬时错误或网络波动非常有用 3。  
* **with_listeners() / with_alisteners()**: 这些方法用于绑定同步/异步生命周期监听器（on_start、on_end、on_error）。这些监听器在 Runnable 执行的不同阶段被调用，可用于日志记录、性能监控等 3。

LangChain Expression Language (LCEL) 极大地简化了链的构建，其灵感来源于 Unix 的管道操作 5。结合

Runnable 接口的丰富方法，如 bind()、with_config()、with_fallbacks() 和 with_retry()，可以看出这些不仅仅是语法糖，它们是构建健壮、可配置、容错的生产级 LLM 应用的关键。通过这些方法，开发者可以轻松地实现模型参数的动态调整、多模型 A/B 测试、错误恢复策略以及复杂的业务逻辑编排，而无需编写大量样板代码。这大大提升了开发效率和应用稳定性。

下表总结了 LangChain Chain 的常用方法：

**Table 4: LangChain Chain 常用方法**

| 方法名称 | 描述 | 关键用例 |
| :---- | :---- | :---- |
| invoke() | 同步执行链，接收字典输入，返回字典输出。 | 最常见的同步执行方式。 |
| ainvoke() | 异步执行链。 | 在异步环境中执行，提高并发性。 |
| stream() | 流式获取链的输出。 | 实时显示模型生成内容，提升用户体验。 |
| astream_events() | 异步流式获取链执行过程中的详细事件。 | 深度调试、监控链的内部运行状态。 |
| batch() | 批量处理多个输入。 | 提高处理效率，优化吞吐量。 |
| run() / arun() | 方便方法，接收参数/关键字参数，返回字符串或对象输出。 | 快速获取简单输出，不需完整字典。 |
| bind() | 绑定参数到 Runnable，返回新的 Runnable。 | 预设模型参数（如 stop 序列），创建特定行为的组件。 |
| with_config() | 在运行时配置 Runnable 的参数。 | 动态调整模型行为（如 max_tokens），实现 A/B 测试。 |
| with_fallbacks() | 为 Runnable 添加备用机制。 | 增强应用容错性，当主组件失败时尝试备用方案。 |
| with_retry() | 创建新的 Runnable，在指定异常时自动重试。 | 自动处理瞬时错误，提高应用健壮性。 |
| with_listeners() / with_alisteners() | 绑定同步/异步生命周期监听器。 | 实现自定义日志、监控和性能分析。 |

### **3.4 Agents (智能体)**

#### **3.4.1 Agent 与 Chain 的区别：决策与行动**

在 LangChain 中，智能体（Agent）与链（Chain）的核心区别在于其决策机制。在链中，一系列动作是硬编码的，执行顺序是预先确定的。而在智能体中，语言模型被用作推理引擎，动态地决定要采取哪些动作以及以何种顺序执行 8。智能体能够根据当前情况和可用工具动态地规划和执行任务，这使得它们在处理复杂、不确定或需要与外部环境交互的场景中表现出色。

#### **3.4.2 Agent 的工作原理与推理过程**

智能体通常通过一个“观察（Observation）-思考（Thought）-行动（Action）”的循环来工作。LLM 接收用户输入或来自环境的观察结果，然后进行思考，规划下一步的行动。基于其思考，LLM 选择并调用合适的工具来执行某个操作。工具的输出（例如搜索结果、计算结果）再反馈给 LLM，形成新的观察，这个循环会持续进行，直到任务完成或达到某个停止条件 9。

#### **3.4.3 AgentExecutor 及其应用**

AgentExecutor 是执行智能体的核心组件，它负责管理智能体的整个执行流程。这包括协调 LLM 的调用、执行选定的工具、处理中间步骤以及最终返回任务的完成结果 8。

以下是一个使用本地 Qwen3 构建简单工具调用智能体的示例，该智能体能够查询天气信息：

```Python

from langchain_ollama import ChatOllama  
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder  
from langchain.agents import create_tool_calling_agent, AgentExecutor  
from langchain_core.tools import tool

# 1. 定义工具  
@tool  
def get_current_weather(location: str) -> str:  
    """获取指定地点的当前天气。输入应为地点名称，例如“北京”。"""  
    if "北京" in location:  
        return "北京目前晴朗，气温25摄氏度。"  
    elif "上海" in location:  
        return "上海目前多云，气温28摄氏度。"  
    else:  
        return f"抱歉，无法获取 {location} 的天气信息。"

tools = [get_current_weather]

# 2. 初始化模型 (使用本地 Qwen3)  
llm_qwen = ChatOllama(model="qwen3", temperature=0)

# 3. 创建提示词 (包含 agent_scratchpad 用于记录思考过程)  
# MessagesPlaceholder 用于动态注入智能体的思考过程和工具输出  
prompt = ChatPromptTemplate.from_messages([  
    ("system", "你是一个有用的助手，可以访问工具。"),  
    ("human", "{input}"),  
    MessagesPlaceholder(variable_name="agent_scratchpad"), # 智能体思考过程  
])

# 4. 创建工具调用代理  
# create_tool_calling_agent 适用于支持工具调用的Chat Model  
agent = create_tool_calling_agent(llm_qwen, tools, prompt)

# 5. 创建 AgentExecutor  
# verbose=True 会打印出智能体的思考过程和工具调用  
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 6. 调用代理  
print("--- 查询北京天气 ---")  
response_beijing = agent_executor.invoke({"input": "北京的天气怎么样？"})  
print(f"最终输出: {response_beijing['output']}")

print("n--- 查询纽约天气 ---")  
response_newyork = agent_executor.invoke({"input": "纽约的天气怎么样？"})  
print(f"最终输出: {response_newyork['output']}")
```

在上述示例中，verbose=True 参数在 AgentExecutor 中被使用，这会打印出智能体的“思考过程”（thought process）和采取的行动。例如，DeepSeek R1 模型也能够“暴露其思维过程” 28。智能体通过“思维链”（Chain of Thought）或类似机制来规划和执行任务，这不仅仅是为了功能实现，更是为了提高 LLM 应用的

**可解释性**和**可调试性**。通过观察智能体的内部思考，开发者可以理解其决策逻辑，从而更好地优化提示词、工具选择或解决错误。这对于构建复杂、可靠的 AI 代理至关重要。

下表列出了 LangChain 中常见的智能体类型及其适用场景：

**Table 5: LangChain Agent 类型与适用场景**

| Agent 类型 | 描述 | 关键特性 | 常见用例 |
| :---- | :---- | :---- | :---- |
| OpenAI Functions Agent | 专为 OpenAI 微调模型设计，用于确定何时调用特殊函数。 | 支持聊天历史。 | 需要模型自动调用预定义函数（如 API 调用）。 |
| OpenAI Tools Agent | 专为 OpenAI 工具设计，用于交互和确定是否使用内置工具。 | 支持聊天历史。 | 需要模型调用内置工具（如图像生成、代码执行）。 |
| XML Agent | 适用于模型在 XML 读写和推理方面表现出色的场景。 | 支持聊天历史。 | 处理 XML 文件，与 XML 结构化数据交互。 |
| JSON Chat Agent | 适用于模型擅长读取 JSON 且应用操作 JSON 文件的场景。 | 支持聊天历史。 | 处理 JSON 数据，与 JSON 结构化数据交互。 |
| Structured Chat Agent | 适用于多输入工具。 | 支持聊天历史。 | 需要代理处理具有多个输入参数的工具。 |
| ReAct Agent | 适用于简单模型 (LLM)，基于 "Reasoning and Acting" 模式。 | 支持聊天历史。 | 适用于需要模型进行推理并采取行动的通用任务。 |
| Self-Ask with Search | 代理通过分解问题并进行搜索来回答问题。 | 通常用于问答系统。 | 需要逐步分解复杂问题并通过搜索获取信息。 |
| 9 |  |  |  |

### **3.5 Tools (工具)**

#### **3.5.1 工具的定义与作用：Agent 的外部能力**

工具是智能体与外部世界交互的接口。每个工具都有一个描述，智能体根据这个描述来理解工具的功能，并选择合适的工具来完成任务 16。工具可以是任何能够执行特定操作的函数、API 封装或外部服务。它们扩展了语言模型的能力，使其能够执行超出其训练数据范围的操作，例如进行实时搜索、执行计算或与数据库交互。

#### **3.5.2 创建自定义工具**

LangChain 提供了 @tool 装饰器，可以方便地将任何 Python 函数转换为工具 16。在定义工具时，函数的描述（docstring）至关重要，因为它会被智能体用来理解工具的功能并决定何时调用它。清晰、准确的描述能够显著提高智能体选择和使用工具的效率。

以下是创建两个简单自定义工具的示例：

```Python

from langchain_core.tools import tool

@tool  
def add(a: int, b: int) -> int:  
    """Adds two integers a and b."""  
    return a + b

@tool  
def multiply(a: int, b: int) -> int:  
    """Multiplies two integers a and b."""  
    return a * b

tools = [add, multiply]  
# 这些工具可以被 AgentExecutor 使用，让LLM能够执行加法和乘法操作。
```

#### **3.5.3 常用内置工具示例 (Python REPL, Search API 等)**

LangChain 提供了多种开箱即用的内置工具，极大地丰富了智能体的能力：

* **Python REPL Tool**: 允许智能体执行 Python 代码。这对于需要进行计算、数据处理或动态生成内容的任务非常有用 31。  
* **Search API Tool (例如 Tavily, Google Search)**: 使智能体能够执行网络搜索，获取实时信息或补充其知识库 9。  
* **Image Generation Tool (例如 DALL-E)**: 允许智能体调用图像生成服务，根据文本描述创建图像 31。  
* **Retriever Tool**: 可以将任何 LangChain 检索器转换为工具，使智能体能够查询特定的知识库或向量存储，从而实现 RAG 功能 9。

需要注意的是，赋予 LLM 外部工具能力的同时，也引入了潜在的安全漏洞。例如，PythonREPLTool 可以执行任意 Python 代码，并可能带来文件系统访问、网络请求等安全风险 31。因此，对于生产环境中的智能体，必须考虑对工具执行进行沙箱化（sandboxing）或严格的权限控制，以防止恶意或意外的代码执行。这强调了在构建智能体时，功能与安全必须并重。

### **3.6 Retrievers (检索器)**

#### **3.6.1 检索器接口与作用：获取相关文档**

检索器提供了一个统一的接口，用于从各种数据源（如向量存储、搜索引擎、传统数据库）搜索和检索与用户查询最相关的文档 7。它们接收一个查询字符串作为输入，并返回一个

Document 对象列表作为输出。每个 Document 对象包含 page_content（文档内容）和 metadata（相关元数据，如文档 ID、文件名、来源等）7。检索器是构建问答系统和 RAG（检索增强生成）工作流的核心组件。

#### **3.6.2 Vector Stores 作为检索器**

向量存储是一种强大且高效的索引和检索非结构化数据的方式。任何向量存储都可以通过调用其 as_retriever() 方法轻松转换为 LangChain 检索器 7。这使得开发者能够利用向量相似度搜索来查找与查询语义相关的文档。

以下是将向量存储转换为检索器并进行调用的示例：

```Python

# 假设 MyVectorStore 已经初始化并包含文档  
from langchain_community.vectorstores import FAISS  
from langchain_community.embeddings import HuggingFaceEmbeddings  
from langchain_community.document_loaders import TextLoader  
from langchain.text_splitter import CharacterTextSplitter

# 准备一些示例文档  
# with open('sample_docs.txt', 'w', encoding='utf-8') as f:  
#     f.write("LangChain 是一个用于开发 LLM 应用程序的框架。n")  
#     f.write("它简化了 LLM 应用的整个生命周期。n")  
#     f.write("向量存储是检索非结构化数据的重要组成部分。n")  
#     f.write("FAISS 是一个高效的相似度搜索库。")

loader = TextLoader('sample_docs.txt', autodetect_encoding=True)  
documents = loader.load()  
text_splitter = CharacterTextSplitter(chunk_size=100, chunk_overlap=0)  
docs = text_splitter.split_documents(documents)

# 初始化嵌入模型  
embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh-v1.5") # 示例嵌入模型

# 创建向量存储  
vectorstore = FAISS.from_documents(docs, embeddings)

# 将向量存储转换为检索器  
retriever = vectorstore.as_retriever()

# 调用检索器  
query = "LangChain 的主要功能是什么？"  
retrieved_docs = retriever.invoke(query)  
print(f"查询: {query}")  
print("检索到的文档内容:")  
for doc in retrieved_docs:  
    print(f"- {doc.page_content}")

```

#### **3.6.3 高级检索模式 (Ensemble Retriever, ParentDocument Retriever, Multi-Vector Retriever)**

为了应对更复杂的检索需求，LangChain 提供了多种高级检索模式：

* **Ensemble Retriever (集成检索器)**: 这种模式允许结合多个检索器，并通过加权分数（例如 Reciprocal Rank Fusion, RRF）合并搜索结果 7。这在不同检索器擅长查找不同类型相关文档的场景中特别有用，能够综合各方优势，提高检索的全面性。  

```Python  
  # 假设 bm25_retriever 和 vector_store_retriever 已初始化  
  # from langchain.retrievers import EnsembleRetriever  
  # ensemble_retriever = EnsembleRetriever(  
  #     retrievers=[bm25_retriever, vector_store_retriever],  
  #     weights=[0.5, 0.5]  
  # )  
  # docs = ensemble_retriever.invoke("您的查询")
```

* **ParentDocument Retriever**: 该检索器在索引时使用文档的小块，但在检索时返回整个父文档 7。这适用于索引时需要细粒度信息（例如为了更精确的向量嵌入），但模型在生成答案时需要更广泛上下文的场景。  
* **Multi-Vector Retriever**: 这种检索器为每个文档创建多个向量（例如，文档摘要的向量、文档中假设性问题的向量），用于索引，同时保留与原始文档的链接 7。它适用于提取的信息（如摘要或关键概念）比原始文本本身更适合索引的情况，能够从不同角度捕捉文档的语义。

LangChain 提供的这些高级检索模式表明，RAG 并非简单的“检索+生成”。为了提高 LLM 答案的质量和相关性，需要根据数据特性和应用场景精细化检索策略。不同的检索器组合、文档分块方式、以及索引内容（是原文、摘要还是假设性问题）都会显著影响 RAG 系统的性能。这表明 RAG 领域是一个持续优化的过程，需要开发者深入理解数据和模型行为，并根据具体需求选择和组合最合适的检索技术。

### **3.7 Memory (记忆)**

#### **3.7.1 记忆在对话中的重要性**

在构建多轮对话式 LLM 应用时，记忆是不可或缺的组件。记忆系统使 LLM 应用能够保留对话历史和关键信息，从而在多轮交互中保持上下文连贯性，避免重复提问或生成无关响应 19。例如，如果用户提到“我计划去巴黎旅行”，随后问“那里的天气怎么样？”，记忆系统能确保模型理解“那里”指的是巴黎 19。

#### **3.7.2 记忆的读写机制与状态存储**

LangChain 的记忆系统围绕两个基本动作构建 19：

* **读取 (Reading)**: 在链处理用户输入之前，记忆系统会从存储中读取相关上下文（例如之前的消息），并将其作为额外信息传递给 LLM，以帮助模型理解当前对话的背景。  
* **写入 (Writing)**: 在处理完成后，记忆系统会将当前的输入和 LLM 的输出写入存储，以便在未来的对话中再次使用。

记忆的状态可以存储在多种结构中，从简单的临时列表到复杂的数据库，这取决于应用的需求和持久化要求 19。

#### **3.7.3 常用记忆类型 (ConversationBufferMemory 等)**

LangChain 提供了多种记忆类型，以适应不同的应用场景：

* **ChatMessageHistory**: 这是最基础的记忆类型，用于简单地存储聊天消息历史。它是一个底层的消息列表，可以手动添加用户和 AI 的消息。

```Python  
  from langchain_core.chat_history import ChatMessageHistory  
  from langchain_core.messages import HumanMessage, AIMessage

  history = ChatMessageHistory()  
  history.add_user_message("你好！")  
  history.add_ai_message("有什么可以帮助你的吗？")  
  print(history.messages)  
  # 预期输出: [HumanMessage(content='你好！'), AIMessage(content='有什么可以帮助你的吗？')]
```
* **ConversationBufferMemory**: 这种记忆类型将聊天历史存储在一个缓冲区中，并可以在需要时加载。它通常与链结合使用，通过将聊天历史注入到提示词中，从而使 LLM 能够感知之前的对话内容 19。 

```Python  
  from langchain.memory import ConversationBufferMemory  
  from langchain_core.prompts import PromptTemplate  
  from langchain.chains import LLMChain  
  from langchain_deepseek import ChatDeepSeek

  llm_deepseek = ChatDeepSeek(model="deepseek-chat", temperature=0.7)

  template = """你是一个友好的聊天机器人，会记住之前的对话。  
  之前的对话:  
  {chat_history}  
  新的问题: {question}  
  回答:"""  
  prompt = PromptTemplate.from_template(template)

  # memory_key 必须匹配 prompt 中的占位符名称  
  memory = ConversationBufferMemory(memory_key="chat_history")

  conversation = LLMChain(  
      llm=llm_deepseek,  
      prompt=prompt,  
      verbose=True, # 打印详细的链执行过程  
      memory=memory  
  )

  # 第一次对话：用户介绍自己  
  print("--- 第一次对话 ---")  
  conversation.invoke({"question": "我叫张三。"})  
  print(f"记忆内容: {memory.load_memory_variables({})}")

  # 第二次对话：模型会记住名字  
  print("n--- 第二次对话 ---")  
  response = conversation.invoke({"question": "我叫什么名字？"})  
  print(f"模型回答: {response['text']}")  
  print(f"记忆内容: {memory.load_memory_variables({})}")
```

记忆并非一刀切的解决方案，其设计需要根据具体应用场景进行权衡。简单的缓冲区记忆适用于短对话，但对于长对话或需要复杂推理的场景，可能需要更高级的记忆类型。例如，LangChain 提供了基于向量存储的记忆 ConversationVectorStoreTokenBufferMemory 20，它能够存储大量对话历史并通过语义搜索进行检索。此外，还可以结合摘要、实体提取等技术来构建更精细的“世界模型”，以结构化地理解对话中的实体及其关系 19。选择合适的记忆策略需要权衡成本、性能和对话的复杂性。LangChain 提供了丰富的选择，但开发者需要根据具体应用场景进行决策。

### **3.8 Callbacks (回调)**

#### **3.8.1 回调系统概述与事件监听**

LangChain 提供了一个强大的回调系统，允许开发者在 LLM 应用程序的各个阶段插入自定义逻辑 21。这对于日志记录、监控、流式传输、调试以及性能分析等任务非常有用。开发者可以通过在组件调用时传递

callbacks 参数来订阅这些事件。当特定的事件（如模型开始生成、链执行结束、工具发生错误）被触发时，注册的回调处理器就会被调用 23。

#### **3.8.2 同步与异步回调处理**

回调处理器可以是同步的 (BaseCallbackHandler) 或异步的 (AsyncCallbackHandler) 23。在运行时，LangChain 会配置适当的回调管理器（例如

CallbackManager 或 AsyncCallbackManager），该管理器负责在事件触发时调用每个“已注册”回调处理器上的相应方法。这使得开发者可以根据其应用程序的架构选择合适的处理方式。

#### **3.8.3 运行时回调与构造函数回调**

在 LangChain 中，回调可以在两个主要位置进行配置 23：

* **运行时回调 (Request time callbacks)**: 这些回调在请求时作为参数传递给 Runnable 对象。它们的一个重要特性是会继承给所有子对象。例如，当一个回调处理器被传递给一个 Agent 时，它将用于与该 Agent 相关的所有回调，以及 Agent 执行过程中涉及的所有嵌套对象（如 Tools 和 LLM）。这是推荐的方式，因为它避免了手动将处理器附加到每个单独的嵌套对象 23。  
* **构造函数回调 (Constructor callbacks)**: 这些回调在对象构造时作为参数传递。它们仅限于定义它们的特定对象，不会继承给子对象。

以下是一个自定义日志回调的示例，展示了如何在运行时传递回调并观察链和模型执行的事件：

```Python

from typing import Any, Dict, List  
from langchain_deepseek import ChatDeepSeek  
from langchain_core.callbacks import BaseCallbackHandler  
from langchain_core.messages import BaseMessage  
from langchain_core.outputs import LLMResult  
from langchain_core.prompts import ChatPromptTemplate

class CustomLoggingHandler(BaseCallbackHandler):  
    def on_chat_model_start(  
        self, serialized: Dict[str, Any], messages: List], **kwargs: Any  
    ) -> None:  
        print(f"n--- Chat Model Start ---")  
        print(f"模型名称: {serialized.get('name')}")  
        print(f"输入消息: {messages}")

    def on_llm_end(self, response: LLMResult, **kwargs: Any) -> None:  
        print(f"--- LLM End ---")  
        print(f"输出: {response.generations.text}")

    def on_chain_start(  
        self, serialized: Dict[str, Any], inputs: Dict[str, Any], **kwargs: Any  
    ) -> None:  
        print(f"n--- 链 '{serialized.get('name')}' 开始 ---")  
        print(f"输入: {inputs}")

    def on_chain_end(self, outputs: Dict[str, Any], **kwargs: Any) -> None:  
        print(f"--- 链结束 ---")  
        print(f"输出: {outputs}")

callbacks = [CustomLoggingHandler()]  
llm_deepseek = ChatDeepSeek(model="deepseek-chat", temperature=0.7)  
prompt = ChatPromptTemplate.from_messages([  
    ("system", "你是一个简短回答的助手。"),  
    ("user", "什么是人工智能？")  
])  
chain = prompt | llm_deepseek

# 运行时传递回调  
print("--- 首次调用链 ---")  
response = chain.invoke({"question": "什么是人工智能？"}, config={"callbacks": callbacks})  
print(f"最终回答: {response.content}")

print("n--- 第二次调用链 (无回调) ---")  
response = chain.invoke({"question": "什么是人工智能？"})  
print(f"最终回答: {response.content}")
```

回调系统是实现 LLM 应用**可观测性**（Observability）和**运维（LLM Ops）**的关键。通过回调，开发者可以深入了解模型调用的内部机制、链的执行路径、性能瓶颈和潜在错误 6。这对于调试复杂应用、进行 A/B 测试、监控生产环境中的行为以及持续改进模型性能至关重要。例如，通过与 Chainlit 等 UI 框架的集成，以及 LangSmith 提供的调试功能，开发者可以直观地追踪应用运行轨迹 6。回调系统将 LLM 应用从“黑箱”操作转变为可分析、可优化的系统，是 LLM Ops 实践的基础。

下表提供了 LangChain Callback 事件类型的全面列表，帮助开发者理解何时可以插入自定义逻辑：

**Table 6: LangChain Callback 事件类型**

| 事件名称 | 事件触发时机 | 关联方法 (BaseCallbackHandler) |
| :---- | :---- | :---- |
| Chat model start | 聊天模型开始运行时 | on_chat_model_start |
| LLM start | 传统 LLM 开始运行时 | on_llm_start |
| LLM new token | LLM 或聊天模型发出新 token 时 (流式传输) | on_llm_new_token |
| LLM ends | LLM 或聊天模型运行结束时 | on_llm_end |
| LLM errors | LLM 或聊天模型发生错误时 | on_llm_error |
| Chain start | 链开始运行时 | on_chain_start |
| Chain end | 链结束运行时 | on_chain_end |
| Chain error | 链发生错误时 | on_chain_error |
| Tool start | 工具开始运行时 | on_tool_start |
| Tool end | 工具结束运行时 | on_tool_end |
| Tool error | 工具发生错误时 | on_tool_error |
| Agent action | 智能体采取行动时 | on_agent_action |
| Agent finish | 智能体结束运行时 | on_agent_finish |
| Retriever start | 检索器开始运行时 | on_retriever_start |
| Retriever end | 检索器结束运行时 | on_retriever_end |
| Retriever error | 检索器发生错误时 | on_retriever_error |
| Text | 任意文本运行时 | on_text |
| Retry | 重试事件发生时 | on_retry |
| 17 |  |  |

## **第四章：LangChain 应用开发进阶**

本章将介绍 LangChain 生态系统中两个重要的进阶工具：LangChain Expression Language (LCEL) 和 LangGraph，它们对于构建更复杂、更健壮的 LLM 应用至关重要。

### **4.1 LangChain Expression Language (LCEL) 简介**

LangChain Expression Language (LCEL) 是 LangChain 独有的特殊语法，它极大地简化了链的构造。其灵感来源于 Unix/Linux 中的管道（piping）概念，允许开发者以声明式的方式将不同的 LangChain 组件（即实现 Runnable 接口的对象）连接起来，形成一个数据流管道 5。

LCEL 的核心优势在于：

* **可组合性**: 任何实现 Runnable 接口的组件都可以通过直观的管道操作符 (|) 轻松组合，构建出复杂的逻辑流。  
* **流式传输**: LCEL 自动支持 token 级流式传输，这意味着模型生成的内容可以实时地传递给用户，提供更快的用户体验 3。  
* **异步支持**: LCEL 原生支持异步编程（通过 ainvoke, astream, abatch 等方法），使得开发者能够构建高并发的 LLM 应用。  
* **批处理优化**: 框架能够自动优化批处理操作，提高吞吐量，尤其是在处理大量请求时。  
* **可配置性**: 通过 bind, with_config 等方法，开发者可以在运行时动态地配置 Runnable 的行为，实现高度的灵活性和定制化 3。  
* **可观测性**: LCEL 与 LangSmith 深度集成，提供详细的追踪和调试信息，帮助开发者理解应用程序的内部运行机制。

LCEL 的出现推动了 LLM 应用开发从传统的命令式（一步步调用函数）向声明式（定义数据流管道）的范式转变。这种转变使得应用程序的逻辑更加清晰、模块化和可维护。它降低了构建复杂 LLM 应用的认知负担，并为性能优化（如自动批处理和流式传输）提供了框架级别的支持，是 LangChain 走向生产级应用的关键一步。

### **4.2 LangSmith: 调试、测试与监控 LLM 应用**

LangSmith 是 LangChain 生态系统中的一个开发者平台，专门用于追踪、调试、测试、评估和监控语言模型应用程序和智能体 1。它在 LLM 应用从原型设计到生产部署的整个生命周期中发挥着至关重要的作用。

LangSmith 的主要功能包括：

* **追踪 (Tracing)**: 记录每次 LLM 调用、链执行和工具使用，提供详细的执行路径和中间结果。这有助于开发者理解应用程序的内部行为，尤其是在复杂的多步骤任务中 6。  
* **调试 (Debugging)**: 可视化地展示应用的运行轨迹，帮助开发者快速定位问题和理解模型决策过程。  
* **评估 (Evaluation)**: 对 LLM 应用的性能进行量化评估，支持自定义评估指标，从而实现持续改进。  
* **监控 (Monitoring)**: 实时监控生产环境中的应用表现，发现异常、性能瓶颈和潜在问题，确保应用的稳定运行。

LangSmith 的出现，体现了 LLM Ops (Operations) 的重要性，即对 LLM 应用进行系统化的管理、监控和改进。随着 LLM 应用复杂度的增加，检查链或智能体内部运行情况变得至关重要 6。LangSmith 填补了 LLM 开发工具链中的关键空白，使得开发者能够像传统软件一样，对 LLM 应用进行严格的质量控制和性能优化。它将 LLM 应用从“黑箱”操作转变为可分析、可优化的系统，是 LLM 应用走向生产级部署的必备工具。

### **4.3 LangGraph: 构建有状态的多角色 LLM 应用**

LangGraph 是一个用于构建有状态、多角色 LLM 应用程序的框架。它通过将应用程序的步骤建模为图中的边和节点来工作，从而能够表示和管理复杂的、非线性的工作流 1。

LangGraph 的核心特点包括：

* **状态管理**: 允许构建具有持久状态的复杂应用程序，这对于需要维护多轮对话上下文或复杂工作流状态的场景至关重要。  
* **多角色交互**: 特别适用于需要多个智能体或角色协同工作的场景，例如一个智能体负责数据检索，另一个智能体负责内容生成，它们之间可以进行复杂的交互。  
* **生产级代理**: LangGraph 能够支持构建生产级的智能体，已被 LinkedIn、Uber、Klarna、GitLab 等公司用于其核心业务 2。  
* **与 LangChain 结合**: 尽管 LangGraph 可以独立使用，但它与 LangChain 组件无缝集成，这意味着开发者可以在 LangGraph 中利用 LangChain 的 LLM、工具、检索器等组件 2。

传统的 LangChain Chains 擅长处理线性、预定义的工作流。然而，许多复杂的 LLM 应用（如多智能体协作、复杂决策树）需要更灵活、有状态的编排能力。LangGraph 的出现标志着 LangChain 生态系统从简单的线性链条向更复杂的、基于图的工作流管理演进 1。这种演进使得开发者能够构建更接近人类思维过程的、能够进行复杂规划和交互的智能系统，从而应对更具挑战性的应用场景。

## **第五章：总结与展望**

### **5.1 LangChain 的优势与未来发展**

LangChain 作为 LLM 应用开发领域的领先框架，其优势显著：

* **模块化与可组合性**: 灵活的组件设计，使得开发者能够轻松组合和扩展功能，满足多样化的应用需求。  
* **统一接口**: 为 LLM、嵌入模型和向量存储提供标准化接口，极大地降低了与不同提供商集成的难度。  
* **丰富的集成**: 与数百家 LLM 提供商和第三方工具深度集成，为开发者提供了广泛的选择和强大的生态支持 34。  
* **生产化支持**: 通过 LangSmith 和 LangGraph 等工具，提供从原型开发到生产部署的全生命周期支持，确保 LLM 应用能够稳定、高效地运行。  
* **活跃的社区**: 拥有庞大的开发者社区和丰富的学习资源，为用户提供了强大的支持和持续的创新动力。

展望未来，LangChain 将继续在 LLM 应用开发领域发挥关键作用，尤其是在以下方面：

* **Agentic Workflows (智能体工作流)**: 框架将进一步增强智能体的决策和工具使用能力，使其能够处理更复杂的真实世界任务，实现更高级别的自主性。  
* **RAG 优化**: 持续探索更先进的检索和生成技术，以提高知识增强型 LLM 应用的准确性和鲁棒性，使其在复杂问答和信息检索场景中表现更优。  
* **多模态支持**: 随着多模态 LLM 的发展，LangChain 将持续扩展对图像、音频、视频等多种数据模态的处理能力，以适应更广泛的应用场景 13。  
* **易用性与性能**: 通过 LCEL 等工具持续提升开发体验，并优化运行时性能，使得 LLM 应用的开发更加高效，运行更加流畅。

整个报告展示了 LangChain 如何从封装 LLM 调用（LLMs, Prompts）到串联简单逻辑（Chains），再到引入动态决策（Agents, Tools），并提供外部知识（Retrievers, Memory），最终通过 LangSmith 和 LangGraph 实现生产级编排和可观测性。LangChain 的发展轨迹反映了 LLM 应用开发范式的快速演变。它从最初的“LLM 作为 API”阶段，迅速发展到“LLM 作为带有工具的推理引擎”，再到“LLM 作为复杂工作流中的有状态智能体”。这一演变趋势表明，未来的 LLM 应用将不再是简单的问答机器人，而是能够自主思考、与环境交互、具备长期记忆和复杂协作能力的智能系统，LangChain 正是推动这一趋势的关键框架之一。

### **5.2 学习资源与社区支持**

对于希望深入学习 LangChain 的开发者，以下资源将提供宝贵的帮助：

* **官方文档**: https://python.langchain.com/docs/ 是最权威和全面的学习资源。它包含了详细的指南和 API 参考 1。  
  * **教程 (Tutorials)**: 专为动手学习者设计，通过实际项目帮助用户构建特定应用，是入门的最佳途径 1。  
  * **操作指南 (How-to guides)**: 提供对常见“如何做？”问题的简洁答案，帮助用户快速完成特定任务 1。  
  * **概念指南 (Conceptual guide)**: 深入解释所有关键 LangChain 概念背后的原理，帮助用户建立扎实的理论基础 1。  
  * **API 参考 (API reference)**: 提供 LangChain Python 包中所有类和方法的完整文档，是开发时的重要参考 1。  
* **社区与贡献**: LangChain 拥有一个活跃的开发者社区，鼓励用户参与贡献，共同推动框架的发展 1。  
* **LangChain Handbook**: 社区维护的教程和示例，提供了更多实践指导和深入解析 4。  
* **LangChain Blog/Changelog**: 关注官方博客和更新日志，可以及时了解 LangChain 的最新功能、改进和发展方向 28。

#### **引用的著作**

1. Introduction | 🦜️ LangChain, [https://python.langchain.com/docs/](https://python.langchain.com/docs/)  
2. Introduction | 🦜️ Langchain, [https://js.langchain.com/docs/introduction](https://js.langchain.com/docs/introduction)  
3. Chain — LangChain documentation - Python LangChain, [https://python.langchain.com/api_reference/langchain/chains/langchain.chains.base.Chain.html](https://python.langchain.com/api_reference/langchain/chains/langchain.chains.base.Chain.html)  
4. examples/learn/generation/langchain/handbook/02-langchain-chains.ipynb at master - GitHub, [https://github.com/pinecone-io/examples/blob/master/learn/generation/langchain/handbook/02-langchain-chains.ipynb](https://github.com/pinecone-io/examples/blob/master/learn/generation/langchain/handbook/02-langchain-chains.ipynb)  
5. LangChain 101: The Beginner's Guide to Chains, Agents, Tools & More - Skillcrush, [https://skillcrush.com/blog/intro-to-langchain/](https://skillcrush.com/blog/intro-to-langchain/)  
6. Build an Agent - Python LangChain, [https://python.langchain.com/docs/tutorials/agents/](https://python.langchain.com/docs/tutorials/agents/)  
7. Retrievers | 🦜️ LangChain, [https://python.langchain.com/docs/concepts/retrievers/](https://python.langchain.com/docs/concepts/retrievers/)  
8. agents — LangChain documentation - Python LangChain, [https://python.langchain.com/api_reference/langchain/agents.html](https://python.langchain.com/api_reference/langchain/agents.html)  
9. Introducing LangChain Agents: 2024 Tutorial with Example | Bright Inventions, [https://brightinventions.pl/blog/introducing-langchain-agents-tutorial-with-example/](https://brightinventions.pl/blog/introducing-langchain-agents-tutorial-with-example/)  
10. Conceptual guide | 🦜️ LangChain - Python LangChain, [https://python.langchain.com/docs/concepts/](https://python.langchain.com/docs/concepts/)  
11. Conceptual guide - LangChain.js, [https://js.langchain.com/docs/concepts/](https://js.langchain.com/docs/concepts/)  
12. language_models — LangChain documentation, [https://python.langchain.com/api_reference/core/language_models.html](https://python.langchain.com/api_reference/core/language_models.html)  
13. Chat models - Python LangChain, [https://python.langchain.com/docs/concepts/chat_models/](https://python.langchain.com/docs/concepts/chat_models/)  
14. A Guide to Prompt Templates in LangChain - Mirascope, [https://mirascope.com/blog/langchain-prompt-template](https://mirascope.com/blog/langchain-prompt-template)  
15. Prompt Templates | 🦜️ LangChain, [https://python.langchain.com/docs/concepts/prompt_templates/](https://python.langchain.com/docs/concepts/prompt_templates/)  
16. tools — LangChain documentation - Python LangChain, [https://python.langchain.com/api_reference/core/tools.html](https://python.langchain.com/api_reference/core/tools.html)  
17. Tool — LangChain documentation, [https://python.langchain.com/api_reference/core/tools/langchain_core.tools.simple.Tool.html](https://python.langchain.com/api_reference/core/tools/langchain_core.tools.simple.Tool.html)  
18. retrievers — LangChain documentation, [https://python.langchain.com/api_reference/azure_ai/retrievers.html](https://python.langchain.com/api_reference/azure_ai/retrievers.html)  
19. Memory In Langchain -1 - Medium, [https://medium.com/@danushidk507/memory-in-langchain-1-56fda38ba1d7](https://medium.com/@danushidk507/memory-in-langchain-1-56fda38ba1d7)  
20. memory — LangChain documentation - Python LangChain, [https://python.langchain.com/api_reference/langchain/memory.html](https://python.langchain.com/api_reference/langchain/memory.html)  
21. callbacks — LangChain documentation, [https://python.langchain.com/api_reference/langchain/callbacks.html](https://python.langchain.com/api_reference/langchain/callbacks.html)  
22. Langchain Callback Handler - Chainlit, [https://docs.chainlit.io/api-reference/integrations/langchain](https://docs.chainlit.io/api-reference/integrations/langchain)  
23. Callbacks - Python LangChain, [https://python.langchain.com/docs/concepts/callbacks/](https://python.langchain.com/docs/concepts/callbacks/)  
24. langchain-qwq - PyPI, [https://pypi.org/project/langchain-qwq/](https://pypi.org/project/langchain-qwq/)  
25. ChatDeepSeek | 🦜️ LangChain, [https://python.langchain.com/docs/integrations/chat/deepseek/](https://python.langchain.com/docs/integrations/chat/deepseek/)  
26. Building a Simple AI agent using Langchain and Local Qwen3 LLM in Python - Medium, [https://medium.com/@techkamar/building-a-simple-ai-agent-using-langchain-and-local-qwen3-llm-in-python-8c721b08c130](https://medium.com/@techkamar/building-a-simple-ai-agent-using-langchain-and-local-qwen3-llm-in-python-8c721b08c130)  
27. Langchain - Qwen, [https://qwen.readthedocs.io/en/latest/framework/Langchain.html](https://qwen.readthedocs.io/en/latest/framework/Langchain.html)  
28. DeepSeek integration in LangChain, [https://changelog.langchain.com/announcements/deepseek-integration-in-langchain](https://changelog.langchain.com/announcements/deepseek-integration-in-langchain)  
29. Building a Writing Assistant with LangChain and Qwen-2.5-32B - Analytics Vidhya, [https://www.analyticsvidhya.com/blog/2025/03/writing-assistant/](https://www.analyticsvidhya.com/blog/2025/03/writing-assistant/)  
30. Getting Started with LangChain Tools | by Pelin Balci - Medium, [https://medium.com/@balci.pelin/getting-started-with-langchain-tools-3beec9e1fb95](https://medium.com/@balci.pelin/getting-started-with-langchain-tools-3beec9e1fb95)  
31. Tools | LangChain OpenTutorial - GitBook, [https://langchain-opentutorial.gitbook.io/langchain-opentutorial/15-agent/01-tools](https://langchain-opentutorial.gitbook.io/langchain-opentutorial/15-agent/01-tools)  
32. Long-term Memory: LangMem SDK Conceptual Guide - YouTube, [https://www.youtube.com/watch?v=snZI5ojuMRc](https://www.youtube.com/watch?v=snZI5ojuMRc)  
33. How to pass callbacks in at runtime - Python LangChain, [https://python.langchain.com/docs/how_to/callbacks_runtime/](https://python.langchain.com/docs/how_to/callbacks_runtime/)  
34. LLMs | 🦜️ LangChain, [https://python.langchain.com/docs/integrations/llms/](https://python.langchain.com/docs/integrations/llms/)
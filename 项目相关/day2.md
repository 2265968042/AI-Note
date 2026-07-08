### 1.项目目录结构

src/:核心源码，被其他代码import的核心模块，真正实现RAG、MCP、检索、摄取等能力。

tests/:测试，验证哥哥模块是否正常。

config/:配置文件，控制模型、向量库、检索参数等

scripts/:命令行脚本，给人从命令行直接运行的入口，用来调用src里面的能力完成摄取、查询、评估等任务。

docs/:学习和项目文档

data/:本地数据库、向量库、索引等运行数据

logs/:日志和trace

trace是指一次处理过程的详细运行记录。作用是，当答案不准时，可以回看中间过程，判断问题是出在文档、切片、检索、融合还是重排等。



### 2.项目要解决的问题和亮点

这个项目要解决的问题是实现了一个可插拔、高度模块化的RAG系统，并通过MCP协议把RAG能力暴露给Agent/AI助手使用。是一个模块化的RAG MCP Server

亮点：

- DEV_SPEC和skill来驱动开发思路

- 高度模块化、可插拔。

- 实现完整的RAG的流程。

- 实现了MCP协议。

不足：

- 项目本体不是完整的Agent应用，只是面向Agent的RAG Tool Server。Agent 端的规划、记忆、多步推理等能力需要后续扩展。

### 3.Settings

llm：配置生成答案、重排、元数据增强等调用的大语言模型，当前是qwen。

embedding：配置把文本转成向量的模型。当前是 qwen / text-embedding-v3。

vector_store：配置向量库存在哪里、用什么后端。当前是 Chroma。

retrieval：配置检索阶段取多少候选

rerank：配置是否启用重排，以及重排后保留多少结果。只要前五。

evaluation：提供了测试需要的几个指标。

observability：日志和traces

ingestion：配置文档摄取阶段。



### 4. tests

pycache：python自动生成的缓存目录。

e2e：end to end，端到端测试。从用户视角测试一整条链路能不能跑通。

fixtures：测试用的样例数据、样例文档、生成测试数据的脚本。

unit：单元测试、测试一个独立函数、类或模块是否正确。

integration:测试多个模块组合起来是否能协同合作。



### 5.总结

这个项目是一个模块化 RAG MCP Server，核心目标是把企业文档处理成可检索的知识库，并通过 MCP 协议暴露给 AI 助手或 Agent 调用。

它解决的问题是：企业内部文档多、知识分散，工程师很难快速从设备手册、SOP、质检标准、维修记录中找到准确答案。系统通过 RAG 检索相关片段，再结合大模型生成可追溯回答。

核心流程分为三部分：离线阶段对文档进行解析、切块、元数据增强、Embedding 和 BM25 索引构建；在线阶段通过 Dense Retrieval + BM25 混合检索、RRF 融合和 Rerank 找到高相关片段；对外通过 MCP Tool 暴露 query、list collections、document summary 等能力。

项目亮点是全链路可插拔，LLM、Embedding、VectorStore、Reranker 等模块都可以通过配置切换，同时提供 Trace、Dashboard 和测试体系，方便调试和评估。

后续想补的是 Agent 端能力，比如意图识别、Tool Calling 编排、多轮上下文和 ReAct 流程，把它从 RAG Tool Server 扩展成更完整的 Agent + RAG 系统。



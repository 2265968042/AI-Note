###### 1.项目整体逻辑

制造业文档：设备手册、工艺SOP、质检标准、维修记录等原始资料。

-> Ingestion Pipeline：数据摄取流水线，把原始文档读取进系统，并依次完成解析、切块、增强、向量化和入库

-> Chunk/Metadata/Image Caption：把长文档切成适合检索的小块指chunk，给chunk补充标题、摘要、标签等元数据；如果文档里面有图片，就把图片转化成文字描述，方便后续检索。

->Dense Embedding +BM25：对文档chunk建立两套检索能力；Dense Embedding负责语义相似度检索;BM25负责关键词和专业术语匹配。用户提问时，也会生成向量并提取关键词。

->Hybrid Search+RRF+Rerank：同时使用Dense和BM25检索候选结果，然后再用RRF融合两路排序，最后通过Rerank对Top-K结果重新排序，提高相关性

->MCP Tool：调用MCP工具：把知识库查询、文档列表、文档摘要等能力包装成标准工具，让Agent通过MCP协议调用。

->Agent 调用并返回答案：用户向 Agent 提问，Agent 调用 MCP Tool 查询制造业知识库，拿到相关片段和引用后，再组织成可追溯的答案返回给用户。

###### 2.mian文件逻辑

main.py 是项目入口。程序启动后先打印启动信息，然后读取 config/settings.yaml 配置文件；如果配置加载失败，就输出错误并返回非 0 状态码。配置成功后，系统根据配置初始化日志，并调用 mcp_main() 启动 MCP Server。

###### 3.pipeline文件逻辑

stage1：检查文件是否已经处理过，避免重复输入。通过函数给文件内容一个唯一的指纹。然后去history.db文件当中查看是否有这个指纹，如果有就跳过，如果没有就继续后面的处理流程。

stage2：读取pdf文档，提取文本和图片。

stage3：把长文档切成适合检索的小块chunk。

stage4：对chunk做增强，包括文本清洗/重组、元数据补充、图片描述生成。这里面有abc三个方法，a方法对chunk文本做清洗、重组、去噪，让切出来的片段更加适合检索。b方法给chunk补充title、summary、tags等信息，让后续检索和展示更加清晰。c方法用于处理图片，如果chunk当中关联了图片，就用 Vision LLM生成图片描述。

stage5：把文本变成向量，用于语义检索。同时统计关键词、词频等信息用于BM25检索。

stage6：把前面加工好的数据分别存到三个地方：向量库、BM25索引、图片索引。a方法把Dense向量和chunk信息存储到ChromaDB向量库。b方法把BM25统计信息写入关键词索引。c方法把图片路径、图片ID、所属文档等信息存储起来，方面检索结果命中图片时能返回原图。

我理解的MCP Tool的注册和调用流程是：

- ToolDefinition:是一个数据结构，用来保存一个工具的四个信息：name、description、input_schema、handler。其中handler是真正执行工具逻辑的函数。

- register_tool：把一个ToolDefinition注册进self.tools字典里面。

- get_tool_schemas：返回所有工具的schema，给MCP看。这里的schema是指name、description、input_schema

- execute_tool：异步执行工具的函数。工具名不存在报错，参数不匹配（TypeError）返回Invalid parameters，工具内部执行崩了（Exception）返回Internal server error。如果 result 是 CallToolResult：直接返回
  如果 result 是 str：包装成 TextContent 返回
  如果 result 是 list：包装成 content 返回
  其他类型：转成字符串再返回

- _register_default_tools：导入三个默认工具模块，并调用各自的register_tool函数，把工具注册进protocol_handler。三个工具是query_knowledge_hub、list_collections、get_document_summary。查询知识库、列出知识库集合、获取某个文档摘要。

- create_mcp_server：创建MCP tool，并把工具列表和工具调用者两个请求绑定到protocol_handler。使用了两个装饰器。装饰器把某个 MCP 请求类型和某个 Python 函数关联起来。
  
  tools/list 请求 -> get_tool_schemas
  tools/call 请求 -> execute_tool

我理解这个项目是一个基于 RAG + MCP 的知识库 Agent 项目。它先把企业文档处理成可检索的知识库，再通过 MCP Tool 暴露给 Agent 调用。

它解决的问题是：当企业文档很多、知识分散时，工程师很难快速找到准确内容。这个系统可以从文档中检索相关片段，并结合大模型生成回答。

核心链路是：离线阶段先把文档解析、切分成 chunk，补充元数据和图片描述，再生成向量并建立 BM25 索引；在线阶段对用户问题进行处理，通过 Dense Embedding 和 BM25 混合检索找到候选片段，再经过 RRF 融合和 Rerank 重排，最后把相关片段交给 LLM 生成答案。

和制造业结合时，可以用于设备手册查询、工艺 SOP 检索、质检标准问答、维修经验查询等场景，帮助工程师更快定位知识。

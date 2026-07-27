### Stage1：文件完整性检查

解决的核心问题：同一份文档如果已经成功处理过，下次就不要重复处理。这里说的处理就是RAG文档摄取。因为RAG的文档摄取很贵：读PDF、切chunk、调用LLM增强、调用embedding模型、写入向量库、建立BM25索引等。如果一个文档重复处理，会浪费很多时间和API成本。

整体逻辑：

先进行计算SHA256，然后判断是否该跳过这个文档。当中有个元素force，代表这次pipline是否强制重新处理文档。force默认为false，即不强制处理。当force被设为true的时候，不管文档是否处理过，都会强制重新处理。

其中SHA256：给文件内容算一个唯一指纹。

history.db，是SQLite数据库文件，用来记录历史处理结果。

should_skip判断是否要跳过。

用分块读取文件，避免大文件一次性读入内存。每读一块就update到SHA256计算器里，

最后生成整个文件的hash，用来判断文件内容是否变化。

history.db 用来持久化记录文件 hash、路径、处理状态、所属 collection、错误信息和处理时间；

### Stage2：PDF->Document

RAG的第一步不是embedding，而是先把原始资料变成干净的结构统一的文本对象。

核心：把非结构化的PDF转换成项目内统一的数据结构Document。

Document的数据结构主要由三个部分组成：id、text和metadata。

example：

        >>> doc = Document(

        ...     id="doc_abc123",

        ...     text="# Title\\n\\nContent...",

        ...     metadata={

        ...         "source_path": "data/documents/report.pdf",

        ...         "doc_type": "pdf",

        ...         "title": "Annual Report 2025"

        ...     }

        ... )

metadata是一个字典。是[str,Any]，key是字符串，value可以是任何类型。

pdfloader其中对图片的处理是单独进行的。

先判断地址是否存在，再判断是不是pdf文件。

使用哈希值来作为文档的id（先计算 PDF 文件内容的 SHA256 hash，然后取前 16 位生成 doc_id）。

定义了pdf文档的metadata主要由三个key，source_path、doc_type、doc_hash

metadata后面又加入了title和images两个key。

loader输入的是pdf文件地址，输出的是document的数据结构。

本质上是调用了markitdown的函数，pdf优先转成text_content（这是这个函数的属性），没有的话转成了字符串

先尝试在前 20 行中找 Markdown 一级标题；
如果找到了，就去掉 "# " 后返回标题；
如果没找到，就在前 10 行中找第一行非空文本作为标题；
如果还是找不到，就返回 None。

并且没有在这一步为图片生成描述，只是对图片进行了处理。

### Stage3：Chunk

核心：Document 为什么要切成 Chunk，以及项目是如何把一个 Document 变成多个 Chunk 的。

###### 1.chunk类型

```
    Example:
        >>> chunk = Chunk(
        ...     id="chunk_abc123_001",
        ...     text="## Section 1\\n\\nFirst paragraph...",
        ...     metadata={
        ...         "source_path": "data/documents/report.pdf",
        ...         "chunk_index": 0,
        ...         "page": 1
        ...     },
        ...     start_offset=0,
        ...     end_offset=150
        ... )
    """
```

chunk相比于document多了start_offset和end_offset，这两个是值chunk在document当中的起始位置和终点位置。

然后chunk和document的metadata好像还有一些不同，但是它们的类型是相同的。

不过chunk和document当中对图片的处理可能还有些不太确定。

###### 2.Documentchunker

这里面使用的spiliter也是调用的现有的包的切分的方法。

spiliter的函数的输入的document，输出是切分好的chunk列表。不过真正切分的spiliter函数输入的是deocument.text，即文本。

chunk的id和metadata是来自于另外两个函数。

chunk_id是{document.id}_{切片序号四位数}_{切片文本SHA256前8位}

chunk_id是它同时记录了来源文档、Chunk 顺序和文本内容指纹。

metadata先是复制了document的metadata。

chunk_index：当前是原文切出的第几个 Chunk
source_ref：当前 Chunk 的父 Document ID

在处理document的时候，如果遇到图片了怎么办呢？它会在原来图片的位置上面插入一段文本，插入的文本就是图片的id。这样的作用是告诉文档这里有这样一个图片。

- `Chunk` 先继承普通 metadata，但会移除整篇文档的全部 `images`。
- 然后从当前 Chunk 文本里的 `[IMAGE: 图片ID]` 找到实际引用的图片。
- 只把当前 Chunk 关联的图片重新放入 `Chunk.metadata["images"]`。

###### 3.BaseSpliter

它的作用是定义一个基类，定义的是抽象方法。这样有助于后续代码的拓展。

chunk_size和chunk_overlap要不是来自于输入的，要不是来自于配置文件，它们分别代表chunk的大小和重叠部分的大小。chunk_overlap用于避免关键上下文刚好被切在两个 Chunk 的边界。

### Stage4:Chunk增强

###### 1.BaseTransform

和上面的BaseSpliter一样，这里定义了基类和抽象方法，只定义了输入和输出。

输入是：chunks和trace，trace默认为None

输出是chunks，这是增强后的chunks。

###### 2.ChunkRefiner

PDF 转文本后，经常会有页眉页脚、HTML 标签、异常空格、过多换行等噪声需要处理。

ChunkRefiner是继承并实现 `BaseTransform` 的具体子类。

根据是否使用LLM和LLM是否可用，来选择不同的增强方法。

```
-> 规则清洗
-> 如果配置开启且 LLM 可用，再让 LLM 优化
-> 如果 LLM 失败，退回规则清洗结果
-> 返回一个新的 Chunk
```

1. 暂时抽取并保护 Markdown 代码块
2. 清除页眉、页脚、分隔线
3. 删除 HTML 注释
4. 删除 HTML 标签，但保留标签内的文字
5. 规范多余空格和过多换行
6. 恢复代码块，并清理首尾空白

“保护代码块”本质上是：**不让通用文本清洗误伤结构化内容。**

容易被误处理的内容
-> 保存到列表
-> 替换为占位符
-> 清洗其余文本
-> 用保存的原内容替换回占位符

###### 3.MetadataEnricher

MetadataEnricher主要是为了增强元数据。一般就是title、summary、tags和enriched_by

MetadataEnricher也是继承并实现 `BaseTransform` 的具体子类。

和ChunkRefiner的设计是一样的，根据LLM是否可用选择是LLM优化还是规则优化。

```
-> 规则丰富元数据
-> 如果配置开启且 LLM 可用，再让 LLM 优化
-> 如果 LLM 失败，退回规则丰富元数据结果
-> 返回一个新的 Chunk
```

按规则丰富元数据：

分别处理

- title：先查看markdown格式的标题，再按照第一行、第一句、前N个字母的顺序向下递减。

- summary：前N个句子或者前N个字母。

- tags：从英文专有名词、代码标识符、Markdown 加粗或斜体内容中抽取。对中文制造业术语的自动提取较弱

###### 4.Image_captioner

ImageCaptioner也是继承并实现 `BaseTransform` 的具体子类。

- 什么时候会启用视觉模型？视觉模型没配置时会怎样？

和ChunkRefiner的设计是一样的，根据LLM是否可用选择是LLM。如果LLM不可用的化，而是直接跳过图片描述，原样返回 Chunk。

```
ImageCaptioner：
Vision LLM 可用才生成描述
Vision LLM 不可用就跳过，不存在规则版图片描述
```

`strip()` 是同一个意思：删除字符串两侧的空格、换行。

get_caption 用了锁和缓存

```
先加锁读取缓存
-> 缓存已有该图片 ID：直接返回描述，不再调用 Vision API
-> 缓存没有：检查图片路径
-> 将图片和 prompt 发给 Vision LLM
-> 得到描述后加锁写入缓存
```

锁的作用是：图片描述可能并发生成，多个线程不能同时错误地读写同一个缓存字典。

transform()可以分成两轮来进行理解：

第一轮：准备并生成描述

1. 从所有 Chunk 的 metadata["images"] 建立：
   图片 ID -> 图片路径/页码等信息

2. 从所有 Chunk 文本中找 [IMAGE: 图片ID]

3. 只收集“确实被文本引用”的唯一图片

4. 对每张唯一图片调用一次 Vision LLM，结果写进缓存

第二轮：把描述写回对应 Chunk

1. 再次找当前 Chunk 引用的图片 ID

2. 从缓存取出图片描述

3. 在 chunk.text 的图片占位符后追加描述

4. 在 chunk.metadata["image_captions"] 中保存：
   {"id": 图片ID, "caption": 图片描述}

如果图片路径不存在、Vision LLM 调用失败，系统不会让整个摄取流程失败；它会保留原始 Chunk，只是不增加这张图的描述。这也是优雅降级。

### Stage5：Encoding编码

###### 1.DenseEncoder

将每个 Chunk 文本编码成一串浮点数向量，使语义相近的文本在向量空间中距离更近，便于之后进行语义检索。

`__init__`初始化，embedding和batch_size.

`encode()`

先判断是否有chunks。

然后取出chunks当中的text并进行遍历，有问题会报错。

按照batch_size的大小进行遍历的处理文本。调用embedding的embed方法进行处理。

```
合并结果，并保持与 Chunk 原始顺序一致
```

如果处理过程中大小有问题就报错。

```
第 i 个 Chunk -> 第 i 个向量
```

###### 2.SparseEncoder

它不生成语义向量，而是对每个 Chunk 做分词和词频统计，为后续 BM25 关键词检索建立数据基础。

`__init__`

`encode()`

校验chunk列表不为空

校验chunk.text不为空

调用`_tokenize()`分词

用counter统计每次词出现的次数

为每个Chunk输出一份BM25需要的统计数据，是一个字典。

`_tokenize()`

用的jieba的策略，对中英文混合文本进行分词，中英文都能处理。

清楚每次词两侧空白和标点符号

根据 lowercase 配置，将英文统一为小写
过滤长度小于 min_term_length 的词

制造业场景特别需要 BM25。

###### 3.BatchProcessor

```
List[Chunk]
-> 按 batch_size 分批
-> 每一批先交给 DenseEncoder
-> 再交给 SparseEncoder
-> 收集两类结果、耗时、成功数和失败数
-> 返回 BatchResult
```

`@dataclass` 是“用于创建数据类能力的装饰器

创建的数据类`BatchResult` 里面有稠密向量，关键词统计，batch计数、花费时间、成功处理的chunk数量和失败处理的chunk数量。

每batch_size大小的chunk为一batch，每个batch都是List[chunk]的格式。

`process`

检查chunks是否为空

计时

将chunks变成batch类型

分别得到每个batch的稠密向量和关键词统计

返回BatchResult类型的数据。

某一批失败时，`process()` 会记录错误、将该批 Chunk 计入 `failed_chunks`，并继续处理下一批；同时通过 `trace` 记录每批耗时、处理数量和整体汇总信息。

补：装饰器和抽象方法

`@abstractmethod` 是“用于创建抽象方法的装饰器”；`@dataclass` 是“用于创建数据类能力的装饰器

Stage 5 内部：分批 + 调用 Embedding + 生成 BM25 统计 + 合并结果
Stage 6：拿 Stage 5 已经生成好的结果存储

Stage 5 统一处理是为了保证两路编码结果对齐；Stage 6 分开存储是因为 Dense 检索和 BM25 检索需要不同的数据结构与查询方式。

### Stage6:写入向量库

###### 1.VectorUpserter

Chunk + Dense Vector
-> 生成稳定的存储 ID
-> 组装存储记录
-> 调用 VectorStore.upsert()
-> 返回已写入的 Chunk ID

它如何根据配置创建具体的向量数据库？

使用了VectorStoreFactory来创建了向量数据库，传递了设置和名字两个参数。

`upsert`

先判断向量和chunk的数量是否对的上。

按照chunk和vectors进行处理。构建records

source_path 的 SHA256 前 8 位

+ chunk_index 四位数
+ chunk.text 的 SHA256 前 8 位

向量库中存储的数据是

```
Chunk 的 Dense Embedding 向量
+ 对应 Chunk 的文本
+ 对应 metadata
+ 稳定存储 ID
```

并不存储之前相关的BM25词频统计。Stage 5 统一处理是为了保证两路编码结果对齐；Stage 6 分开存储是因为 Dense 检索和 BM25 检索需要不同的数据结构与查询方式。

###### 2.BM25Indexer

```
SparseEncoder 输出：
Chunk ID + 词频 + 文档长度
        ↓
BM25Indexer
        ↓
计算 DF、IDF
        ↓
构建 term -> postings 的倒排索引
        ↓
保存为本地 JSON 索引文件
```

`build`的四步法。

输入是 SparseEncoder 生成的“每个 Chunk 的词频统计”。

原始方式是“文档 -> 包含哪些词”：

Chunk A -> 主轴、温度、冷却、系统
Chunk B -> E201、报警、主轴、温度
Chunk C -> 更换、冷却液、冷却、系统

倒排索引反过来，变成“词 -> 出现在哪些 Chunk”：

主轴 -> Chunk A、Chunk B
温度 -> Chunk A、Chunk B
E201 -> Chunk B
冷却 -> Chunk A、Chunk C

用户搜索 `E201` 时，系统不需要扫遍所有文档；直接查倒排索引，就能立刻找到 Chunk B。

```
1. 统计整个语料库
   计算总 Chunk 数 N、平均长度 avg_doc_length、每个词出现于多少个 Chunk（df）

2. 为每个词计算 IDF
   判断该词有多稀有、多有区分度

3. 构建倒排索引
   为每个词建立 postings，记录它出现在哪些 Chunk、出现次数和 Chunk 长度

4. 保存到磁盘
   将 metadata 和完整 index 写成 JSON 文件
```

`avg_doc_length` 是整个知识库 Chunk 长度的平均值，BM25 用它作为基准，对过长或过短的 Chunk 做长度归一化，使分数更公平。

因为内存中的数据，程序重启就消失了。保存到磁盘后：第一次摄取：构建并保存 BM25 JSON 索引。后续启动：直接加载索引并查询

###### 3.ImageStorage

PDF Loader 已提取图片并保存为文件
        ↓
ImageStorage.register_image()
        ↓
在 SQLite 的 image_index 表中登记
图片 ID -> 文件路径、collection、来源文档、页码
        ↓
后续根据图片 ID 快速找到真实图片

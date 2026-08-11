# 查询与检索链路

解决的问题是：用户提出问题后，系统如何从知识库中召回最相关的Chunk。

用户问题：

- 清洗、分词、提取过滤条件

- 按照语义召回

- 按照关键词/BM25召回

- 融合两路候选

- 精细重排

- 选择Top K Chunk

- 交给LLM生成答案和引用

### 1.QueryProcessor

###### 1.ProcessedQuery

`@dataclass` 是“用于创建数据类能力的装饰器

分别保留用户问题的原始数据、当中包含的关键词和过滤条件。

当中的expand_terms是指的它预留给后续同义词扩展或查询改写。目前没有应用，只作为展示。目前项目中为空。

###### 2.query_pricessor

`@dataclass` 是“用于创建数据类能力的装饰器

QueryProcessorConfig是创建的一个数据类。

stopwords是中英文当中应该被过滤掉的。（人为的定义了两个set。）

min_keyword_length定义了关键词的最小长度

max_keywords定义了关键词的最大数量

 enable_filter_parsing：collection:xxx 这类过滤条件

###### 3.process（）

输出是ProcessedQuery类型的。

目的是把用户输入的问题变成结构化的形式。

1.检测是否为空。健壮性。

2.归一化。清除首尾空白，并将连续空格、换行、制表符压缩为一个空格。

3.分离搜索范围和真正问题。返回的分别是结构化过滤条件字典和删除过滤语法后的问题文本。

4.分词，变成列表类的。

5.关键词过滤。移除字典中过滤的词。并用最小关键词长度和关键词数量来进行过滤。

###### 4._extract_filters()

代码会识别 `key:value` 形式。默认用户不会标准输入，支持用户模糊输入的情况，但是只支持key模糊，并不支持value模糊。

普通条件，例如 `collection`，只有一个值。而 `tag` 特殊，是因为用户可能一次写多个标签。所以collection等类型是用key，value的格式。而tag是用列表的格式。

### 2.DenseRetriever

核心链路：

用户原始问题->Embedding模型生成query vector->ChromaDB查询最相似的Chunk向量->返回RetrivealResult列表

###### 1.`__init___`

有四个输入，分别是配置、embedding模型、向量库和topK的值。

###### 2.retrieve

step1：将输入的query转换成embedding型式。在线用户查询阶段，一次只处理用户一个问题。即使输入了多个问题。这多个问题也默认为一个字符串是一个问题。如果把多个问题拼成一个字符串传入，系统会把它们视为**一个混合问题**，不会自动拆成多个问题。

step2：通过第一步得到的向量在向量库中进行相似度检索，找到前k个相似度高的向量，并返回相应的检索记录。

step3：从向量库中找到的K个相似度高的向量。对这些向量对应的存储信息进行整理。

### 3.SparseRetriever

问题当中的关键词->BM25索引，得到chunk_id和BM25的分数->根据chunk_id从向量库当中取得对应的原数据。

###### 1.`__init__`

结构上和DenseRetriver的结构是完全类似的，只不过把embedding模型换成了`BM25Indexer`。

###### 2.retrieve

整体和DenseRetriver也是类似的。

关键词列表
-> 加载指定 collection 的 BM25 JSON 索引
-> BM25 根据关键词检索，返回 Top K 的 chunk_id + BM25 score
-> 使用 chunk_id 去 ChromaDB 精确取回对应 Chunk 的 text 和 metadata
-> 合并关键词分数与 Chunk 数据
-> 返回 RetrievalResult 列表

- **一份 PDF** 会被切成很多 Chunk；
- 这些 Chunk 属于某个 collection；
- **一个 collection 的所有 Chunk 合起来**，构建一份 BM25 倒排索引；

### 4.Fusion

- 它的输入是哪两路结果？
  - 向量相似度召回
  - 关键词召回

上面输入的这是两个列表。

- `rank` 是什么？

- 为什么 Dense 分数和 BM25 分数不能直接相加？
  
  因为这些分数不是归一化的分数，Dense和BM25没有进行归一化。

`fuse()`

step1:

ranking_list包含了向量召回的结果或者相似度召回的结果。list_idx是0的时候是向量相似度召回的结果。list_idx是1的时候代表BM25关键词召回的结果。

RRF 不是“分别做归一化”，而是**舍弃原始分数，按排名给予递减贡献**

rank代表的顺序，rank越小代表排名越靠前。result代表了向量召回的结果或者相似度召回的结果。

一个chunkid的混合分数=$\frac1{k+向量召回的结果排名}+\frac1{k+关键词召回的结果排名}$

并没有直接用分数相加。

排名越靠前，贡献越高；
两路都出现的 Chunk，贡献会累加。

最后根据混合排名的结果进行输出。

### 5.search

这是一个调度器，会将QueryProcessor、DenseRetriever、SparseRetriever、RRF串起来，并处理失败、过滤和最终返回。

用户问题->QueryProcessor处理问题->Dense和Sparse并行检索->任一路失败时的降级->RRF融合->metadata再过滤->返回最终Top K

Step1:

调用QueryProcessor处理用户问题。QueryProcessor 从用户问题中提取 filters+search() 参数中显式传入的 filters-> 合并为 merged_filters。

```
程序显式传入的过滤条件：用于系统控制和权限边界
```

Step2：

调用Dense和Sparse进行并行检索。入股两个都没有，返回error。

如果两个都可用，调用并行搜索函数，执行Dense和Sparse检索，并行处理。

如果只有dense可用，只调用Dense检索。如果只有Sparse可用，只调用Sparse检索。

Step3：

如果Dense和Sparse都不可用，报错。

如果Dense不可用，报错，只用sparse的结果。

如果Sparse不可用，报错，只用dense的结果。

如果两个都没有报错，但是返回的result都是空的，则融合结果也是空的。

如果两个都没有报错，返回的也不全是空值，则用Fusion进行融合。

Step5：

选择不同的检索范围，从元数据当中找到检索范围。（这里有些困惑，因为前面Dense和Sparse已经检索完了，为何这里的元数据又要看检索范围）。这里即使发现错误了，会有回滚的处理吗？

Step6：

从检索分数融合的结果当中，选择排名靠前的K个返回。

**Step 5补充：为什么检索后还要过滤？**

这是一个**兜底检查**，不是重新检索。

原因有两个：

1. 底层存储或检索器对 metadata 过滤的支持可能不完整、语法可能不同。
2. SparseRetriever 当前主要取 `collection` 传给 BM25；其他如 `doc_type`、`tags`、`source_path` 等条件，不一定能在 BM25 阶段完整过滤。

因此融合后，系统再逐个检查结果的 `metadata` 是否满足过滤条件。不满足就从结果列表移除。它没有“回滚”，因为这是只读查询，没有修改数据库。它的局限是：过滤后可能不足 `Top K` 条，系统不会自动再召回一批候选补足数量。这是你以后可以做的项目改进点



### 6.Reranker

Rerank 更准确，但通常更慢、更贵

先用快的召回方法缩小范围-> 再对少量候选精排

1.RerankConfig。带 `@dataclass` 的数据类。Reank的配置

2.RerankResult。带 `@dataclass` 的数据类。重排返回的数据类型

3.CoreReranker。`_init_`

4.rerank()

HybridSearch的候选结果

->空结果或者只有一条，直接返回

->未启用Reank，则保持原顺序，截留Top K

->转为candidates。将RetrievalResult的数据类型转换为Rerank的输入类型，即列表。列表中每个元素是字典。

->调用Reranker

->转为RetrievalResult

->截取Top K

->返回RerankResult

query是用户输入的原始问题。

如果在调用过程中失败，则返回原顺序的前Top K 个。

RRF 负责融合 Dense 与 BM25 的召回排名；
Reranker 负责结合“问题 + 候选 Chunk 文本”做更精细的相关性判断与重新排序。



当得到问题之后的操作
1.问题进行QueryProcessor，保留原问题，它保留原问题、解析过滤条件、分词并提取关键词。

2.调用向量检索和关键词检索。并且有降级处理。

3.对向量检索和关键词检索的召回结果进行融合，即search。

4.对融合结果进行metadata兜底过滤。确保结果属于指定知识库或文档范围。

4.对search召回的结果进行重排得到 Top K个结果，并有降级处理。

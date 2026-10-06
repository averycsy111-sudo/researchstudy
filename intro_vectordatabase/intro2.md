> 原文：Survey of Vector Database Management Systems by James Jie Pan · Jianguo Wang · Guoliang Li

> We start by identifying five main obstacles to vector data management, namely the vagueness of semantic similarity, large size of vectors, high cost of similarity comparison, lack of a natural partitioning that can be used for indexing, and difficulty of efficiently answering “hybrid” queries that require both attributes and vectors.

> Consequently, the modules in a VDBMS split into a query processor, which includes the query specifications, logical operators, their physical implementations, and the query optimizer; and the storage manager, which maintains the search indexes and manages the physical storage of the vectors.

这两个模块是同一个向量数据库软件内部的两大软件模块，可以运行在同一台服务器（同一套 CPU、内存、磁盘）上。

# 完整一次向量查询的端到端流程
> 
> 用户发起查询：给我找和这个查询向量最相似的Top-k向量，并且过滤元数据

## 阶段 1：【查询处理器】接管（主要占用 CPU、内存）

任务：理解请求、选检索方案、执行相似度计算

1. **解析查询**：接收查询向量和过滤条件（元数据过滤），解析请求
2. **生成逻辑执行计划**
> 
> 逻辑描述：检索向量索引 → 计算相似度 → 元数据过滤 → 返回 topk 结果
>不定义具体怎么实现
3. **查询优化器挑选最优检索策略**
优化器做决策：
   - 用哪个索引？IVF？HNSW？IVFPQ？
   - 预过滤还是后过滤？
   - 要扫描多少个聚类簇（nprobe）？
   输出最优的执行方案。

> Q：优化器为什么有决策能力？
> 向量数据库查询优化器的决策能力，来自内置的代价模型、系统统计信息、预设规则；不是 AI 大模型，而是一套基于统计和公式的代价估算计算器。
> 代价 → 预估耗时、IO 读取量、CPU 计算开销、内存占用。

> ### ① 数据库维护的**统计信息（statistics）**⭐

> 存储管理器会持续收集并维护数据集的元统计，优化器读取这些数据来估算开销（并不读取原始向量数据）：

> - 向量库总条数、向量维度
> - 索引统计：IVF 的聚类簇数量、每个簇里面向量数量；HNSW 的图节点度数；PQ 量化残差分布
> Note：这些索引统计不是查询的时候临时聚类算出来的，是建索引阶段一次性算好、之后持续维护，保存在存储管理器里的元数据。是离线训练建索引的时候就固定生成的，查询阶段不重新聚类。优化器的候选方案，只能在已经建好的索引里面挑选。
> - 元数据统计：每个过滤字段的基数、值分布
> - 硬件统计：磁盘读取延迟、内存带宽、CPU 算力

> 举例决策：
> 如果查询要做元数据过滤，优化器看统计：过滤后只剩很少向量 → 选择先过滤（预过滤）再向量检索；如果过滤条件筛不掉多少数据 → 选择先向量检索，后过滤。

> ### ② 代价模型 Cost Model

> 优化器内置一套数学公式，代价模型会分别估算：

> 1. IO 代价：需要从磁盘读多少索引页 / 向量页（存储管理器负责读，IO 开销很大）
> 2. CPU 代价：要算多少次向量距离、多少次比较、排序开销
> 3. 内存代价：中间结果占用内存大小

> ### ③ 启发式规则（heuristic rules）

> 不用算代价直接判定，快速剪掉明显很差的方案，减少计算：

> - 比如向量数量很小的时候，直接走暴力扫描，不使用 IVF/HNSW 索引
> - 高维向量，优先考虑 PQ 量化索引降低 IO
> - 某些索引不支持某种相似度度量，直接排除该方案
4. **转为物理算子，发起检索**
逻辑计划翻译成可执行的物理代码算子：
   - 查询处理器会向存储管理器发起请求：把需要的索引 / 向量数据加载到内存**

## 阶段 2：【存储管理器】响应请求（内存 + 磁盘 IO）

任务：负责数据 / 索引的存放、加载、持久化

1. 收到查询处理器的数据读取请求：“把 IVF 索引的簇中心、某些簇里面的向量编码加载进内存”
2. 存储管理器先查内存缓存：如果这部分索引 / 向量已经在内存里，直接返回给查询处理器；
3. 如果缓存没有 → 从磁盘（SSD）读取对应的数据页，加载进内存，然后交给查询处理器；
4. 同时还要管理：向量、索引在磁盘上怎么组织、数据持久化（新增向量时，把向量和索引写入磁盘保存，断电不丢失）

## 阶段 3：回到查询处理器，完成计算

拿到存储管理器送来的索引和向量数据后：

- 执行检索：比如 IVF：查询向量和簇中心算距离，选出 nprobe 个最近簇；然后在簇内做 PQ 近似距离计算
- 相似度排序、元数据过滤、截取 Top-k
- 把最终结果返回给用户

> On the other hand, modifying the index scan operator to account for attribute predicates can degrade index performance. It remains unclear how to support “hybrid” queries over both attributes and vectors in a way that is both efficient and accurate.

混合查询：同时带有向量相似度检索 + 布尔属性谓词（元数据过滤）

- 方案 A（后过滤）：先用向量索引拿出 Top-k，再做元数据布尔谓词过滤 → 容易出现结果不够 k 条（召回不足）
- 方案 B（改造索引算子，检索时直接带上属性谓词）
- 问题：
1. **索引结构原本是按照向量空间距离构建的，不是按照元数据划分**

IVF 的簇、HNSW 的图，聚类 / 建图只依据向量相似度，元数据是独立附属信息。检索时每一条候选都额外增加一次布尔判断，增加 CPU 开销。
2. **索引的局部性被破坏**

同一个簇 / 图邻居里的向量，元数据分布是杂乱的。检索过程中频繁跳过大量不满足谓词的向量，大量距离计算、内存读取变成无效工作。
3. **索引的预计算统计失效**

原本优化器依赖的簇大小、图节点度数这些索引统计，在叠加属性谓词之后，预估的代价、召回模型不准。

**没有办法同时做到【检索快 + 召回 / 结果数量准确】**，这个问题目前仍然没有清晰、完美的解决办法。

> table-based indexes such as E2LSH [49], SPANN [44], and IVFADC（即IVFPQ)[69], that are generally easy to update

核心特征：索引组织成多个独立的桶 / 倒排表（posting list）。向量被分到不同的表（桶）里面；插入、删除向量只需要修改它所属那一张表，不会改动整个索引全局结构。
> Note：SPANN（微软，十亿级磁盘 ANN 索引）倒排类混合内存 - 磁盘索引
> - 原理：分层聚类，聚类中心放内存，每个簇（分区）对应的向量列表（posting list）放在磁盘。查询时先找最近的几个簇中心，只扫描对应磁盘分区内的向量
>
> Note：FLANN
> 全称：Fast Library for Approximate Nearest Neighbors，不是单一索引算法，是一个集成多种 ANN 索引的 C++ 开源工具库
> 包含：
> 1. 随机 KD 森林（Randomized KD-Tree）：树索引，多棵随机 KD 树并行检索，适合中低维向量
> 2. 层次 K-Means 树（Hierarchical K-means Tree）：多层递归 K-means 空间划分（树状分层聚类）
> 3. LSH（局部敏感哈希）
> 4. 暴力线性扫描（Flat）
> 5.核心亮点:自带代价评估的自动选择模块 → 估算不同索引方案的构建开销、检索耗时、内存占用，自动选最优索引与超参。

# 1. KGraph（2011）

核心：NN-Descent 算法建图

目标：给每个节点找它大概的 K 个最近邻居，连成一张 K 近邻图（KNNG）。

1. 随机初始化：每个节点随便挑几个其他节点，作为初始邻居。
2. 迭代更新
循环执行：
- 取出节点 A 的所有邻居；
- 这些邻居的邻居作为 A 的候选新邻居；
- 计算 A 和这些候选之间的向量距离，如果更近，就替换掉 A 现有的较差邻居。
3. 多轮迭代后收敛，停止。最终每个节点保存 K 个近似最近邻居，形成一张单层 K 近邻图。
  
✅优点：

1. 建图简单，代码好实现，不需要复杂的边筛选规则；
2. 支持任意距离度量（L2、余弦等），不限制向量距离函数；
3. 在当年，相比暴力检索，ANN 速度提升巨大。

❌缺点

1. 图容易碎裂，出现多个不连通的子图
比如图分成两块，块之间没有边。如果检索起点刚好落在碎片里，永远跳不到全局最近点，召回不稳定，偶尔搜不到正确结果。
2. 没有智能边剪枝：邻居列表里面可能存在大量冗余边，图的边数量容易变多，检索遍历开销变大。
3. 动态更新极差：新增一个节点，要把这个点尝试和很多节点建立边；并且其他节点的邻居列表也可能要更新。删除节点，所有连向它的节点都要重新找邻居。
4. 单层图：没有高层 “快速通道”。大数据高维向量时，检索起点很关键，起点不好，搜索很慢。

---

# 2. FANNG（Fast Approximate Nearest Neighbour Graph，2016）

核心：RNG（Relative Neighborhood Graph，相对邻域图）边筛选规则

FANNG 同样是单层图，但建图逻辑和 KGraph 完全不一样。
它的重点不是 “迭代找邻居”，而是严格筛选边，只保留有价值的边，删掉冗余边，保证整张图一定连通。

RNG 规则：

两个节点 A、B 之间保留边，当且仅当：不存在任何节点 C，同时比 A、B 两者离 A 和 B 更近。
- 自动剪掉没用的边，控制总边数，不会边爆炸；
- 保证整张图一定连通，召回稳定性更好。

✅优点：

1. 图天然连通，没有碎片问题，召回很高，理论上限很强；
2. RNG 剪枝去掉无效边，减少检索时需要遍历的边；
3. 对高维图像特征向量（SIFT 这类）效果很好。

❌缺点：

1. 建图计算开销巨大
RNG 边筛选，需要大量两两向量距离计算，数据集大的时候建图极慢，成本高。
2. 单层没有分层导航
检索起点的选择依然很难。如果起点离目标区域很远，需要一步一步慢慢爬，检索速度不如带分层的 HNSW。
3. 几乎没有开源实现，学术界论文效果好，但工业没人大规模落地。
4. 同样：增删向量代价很高。一个节点变动，需要重新检查、修改很多节点之间的边。

HNSW = **FANNG 的 RNG 边筛选思想 + 多层分层导航**

2.1.1 Basic Scores

> Several similarity scores are commonly supported by VDBMSs
> Similarity is often measured via distance in practice, with values closer to 0 indicating greater similarity. Distance functions obey the metric axioms of identity
> Type	                      Score Metric	      Complexity	 Range
  Sim.（Similarity 相似度）	 Inner Prod. 内积	         O(D)	      R（全体实数）
  Sim.	                   Cosine 余弦相似度	         O(D)	      [−1,1]
  Dist.（Distance 距离）	    Minkowski 闵可夫斯基距离	O(D)	      R+（非负实数）
  Dist.	                   Mahalanobis 马氏距离	      O(D2+O(1))	R+
  Dist.	                   Hamming 汉明距离	         O(D)	      N（自然数，0,1,2…）

Definition 1 (Hamming) d(a, b) = P n i=1 δaibi

> The Hamming distance counts the number of differing dimensions between vectors a and b

Definition 2 (Inner Product) f(a, b) = P n i=1 aibi
> Note：The dot product projects a onto b and scales the result by the magnitude of b. The scaling can lead to unintuitive consequences. For example, two large identical vectors have a larger dot product compared to two small identical vectors, thus they would be considered “more similar” under this definition. If magnitude is not important, a and b can be normalized by ˆa = a/∥a∥ and bˆ = b/∥b∥ so that they
possess unit magnitudes.

Definition 3 (Cosine Similarity) f(a, b) = ⟨ˆa, bˆ⟩or f(a, b) = ∥
⟨
a
a
∥∥
,b
b
⟩
∥

Definition 4 (Minkowski) The p-order Minkowski
distance is d(a, b) = (P n
i=1 |ai − bi
|
p
)
1/p
or d(a, b) = ∥a − b∥p

Definition 5 (Mahalanobis) For any positive semidefinite matrix M, d(a, b) = p (a − b)⊤M(a − b).

> Another generalization of Euclidean distance can be
obtained by applying a linear transformation over the
vector space in order to adjust the relative proximities
of the feature vectors. The distance of two vectors in
the transformed space can be calculated using the Mahalanobis formula.

> Note: 闵可夫斯基距离是欧氏距离的广义形式；而马氏距离又是欧氏距离的另一种广义化思路，先对向量空间做线性变换，再在新空间上算欧氏距离
> 
> Aside from these basic scores, some VDBMSs also support aggregate scores for applications like multi-vector search [129]. There is also emerging work on learned scores [25,142,93], but these are not available in commercial systems

2.1.2 Aggregate Scores

**aggregate scores 聚合得分 + multi-vector search 多向量检索**
 → 一个实体对应一组多个向量。
> One way of approaching this problem is
to use an aggregate score that defines how to combine
individual scores f(x1, q). . . f(xm, q) to yield a single
value that can be compared.

例子：一张图片，拆成多个局部区域，每个区域提取一个向量；或者一段长文本，分成多个 chunk，得到一组向量。

把这一组向量各自的相似度合并、聚合（比如取最大值、平均值、加权求和）得到一个最终分数，即聚合得分。

2.1.3 Learned Scores

**learned scores 学习型相似度得分**
传统指标是固定数学公式，人工定义好的，不随数据分布自适应。而 learned scores：用机器学习模型自己学到的相似度度量。

- 优点：适配特定任务，匹配效果往往更好
- 缺点：计算开销大、推理慢，难以构建索引做 ANN 近似检索


Curse of Dimensionality

> When D grows beyond
around 10 dimensions, and when the dimensions are
independent and identically distributed, the Euclidean
distances between the two farthest and two nearest vectors approach equality as the variance nears zero

**大数定律**：
独立同分布随机变量 \(Z_1,Z_2,\dots,Z_d\)：

\(\bar Z_d=\frac1d\sum_{i=1}^d Z_i \xrightarrow{P} \mathbb E[Z]\)

样本均值收敛到期望。

→ 当 \(d\to\infty\)：

\(\frac1d\sum_{i=1}^d Z_i^2 \xrightarrow{P} \mathbb E[Z_i^2] = \text{常数}\)

\(\|X-Y\|^2 \approx d\cdot \mathbb E[Z_i^2]\)
两点之间距离平方，**几乎必然趋近于一个和 d 成正比的确定值**。

> 证明：？

方差分析\(\text{Var}(\|X-Y\|^2)=d\cdot C,\quad \text{Std}= \sqrt{dC}\)\(\frac{\text{Std}(\|X-Y\|^2)}{E[\|X-Y\|^2]} \propto \frac{1}{\sqrt{d}}\)

即随着 d 增大，距离平方的相对波动以 \(1/\sqrt{d}\) 的速度趋于 0。

> In a traditional data management system, data is manipulated directly. But in
a VDBMSs, feature vectors are proxies for the actual
entities, and they can be manipulated either directly or
indirectly. An embedding model maps real-world entities
(e.g. images) to feature vectors.

> Under direct manipulation, users freely manipulate
the values of the vectors, and maintaining the model
is the responsibility of the user. This is the case for
systems such as PASE [139] and pgvector [7].
> For indirect manipulation, vectors are hidden from
users. The vector collection appears as a collection of
entities, not vectors, and users manipulate the entities.
The VDBMS is responsible for the model（即embedding model）, which can
be user-provided.

> (c, k)-Search Queries. Most VDBMSs support “nearest neighbor” queries, where the aim is to retrieve vectors from S that are physical neighbors of q in the vector space. These queries may aim to return exact or
approximate nearest neighbors, and may also specify
the number of neighbors to return. We refer to these as
(c, k)-search queries, where c indicates the approximation degree and k is the number of neighbors.
> Out of these, most VDBMSs support the approximate
k-nearest neighbors (ANN) query, which returns k vectors from S that are within a radius, centered over q, of
c times the distance between q and its closest neighbor.

> Note: 很多工程向量库是在索引构建阶段调参间接控制近似程度，而不是让用户在查询时直接传 c。这是理论层面对查询的形式化定义，不是工程 API 参数。
> 确定性硬保证(任何数据集、查询，输出一定满足 \(dist \le c\cdot d^*\)),这类理论算法复杂度很高，工程向量库几乎不实现，只存在算法论文。

eg.:

1. 概率可证明保证:LSH

LSH是 Indyk & Motwani 当年奠基 ANN 理论的算法，专门用来实现\((c,k)\) ANN。

- 给定 c，只要哈希函数数量足够多，以很高概率返回满足 \(dist(q,p) \le c\cdot d^*\) 的候选点。不是 100% 一定成功（随机哈希带来的概率性），不是绝对硬性。
- 缺点：
  - 查询 / 建库开销大；
  - 高维 embedding 场景召回、延迟表现不如 HNSW；
  - FAiss 虽然实现了 LSH，但工业界向量数据库很少默认用它。

👉 2. HNSW/IVF：没有任何可证明的数学界。

HNSW:两个参数efConstructon,ef(efSearch)

1. efConstruction：建索引时设置
- 在建图过程中，每个节点插入时最多考察 `efConstruction` 个候选邻居,只能重建索引才能改
- 影响上限：efConstruction 越大，图结构质量越好，能达到的最优 c越好
  
2. ef（efSearch）：查询阶段参数，每次 query 可以单独改，不改动索引
- 检索时，优先队列最多维护 ef 个候选节点，沿着 HNSW 图搜索
- ef 越大：搜索遍历更多候选，召回越好，近似因子 c 越接近 1；代价是查询更慢。
- ef 越小：搜索跳的点少，速度快，但更容易漏掉真实近邻，c 变大

类似的,IVF:两个参数nlist(聚类中心总数),nprobe(查找最近nprobe个聚类桶)

即使 ef、nprobe 很大，依然存在某些查询，真实最近邻直接漏掉，返回点的距离远大于\(c\cdot d^*\)，只是经验上召回很好。

3. 暴力精确 KNN（Flat），可以看成 c=1 的特例，返回严格真实最近邻：
\(dist(q,p) \le 1\cdot d^*\)，c=1，绝对硬约束。代价：向量量大的时候查询极慢。

> Range Queries. A range query is parameterized by a
radius, r, instead of the number of neighbors to return

> Predicated Search Queries. In a predicated search
query, or “hybrid” query, each vector is associated with
a set of attribute values, and a boolean predicate over
these values must evaluate to true for each record in
the result set
> Batched Queries. For batched queries, a number of
queries are revealed to the system at the same time,
and the VDBMS can answer them in any order. These queries are especially suited to hardware accelerated query processing
> Multi-Vector Queries. Some VDBMSs also support
multi-vector search queries via aggregate scores. There are three possible sub-types: in multi-query
single-feature (MQSF) queries, the query is represented
by multiple vectors, and real-world entities are represented by single feature vectors; in multi-query multifeature (MQMF) queries, both the query and the entities are represented by multiple vectors; and in singlequery multi-feature (SQMF) queries, only the entities
are represented by multiple vectors. But so far, there is
support for MQSF and SQMF queries [11,5,12,125,13]
but no support for MQMF queries.


> Query Accuracy and Performance. The search capability of a VDBMS is assessed by evaluating query accuracy and performance.
> To evaluate accuracy, precision and recall are often
used. Precision is defined as the ratio between the number of relevant results in the result set over the size of
the result set, and recall is defined as the ratio between
the number of retrieved relevant results over all possible
relevant results.
> To evaluate performance, latency and throughput
are used. Latency is the amount of time it takes for a
VDBMS to answer a query once it is received, while
throughput is the number of queries that are answered
per unit time, often reported as queries per second.
>
> The modern belief is that even a fractional power of
N query complexity cannot be obtained unless storage
cost is worse than N O(1)DO(1)

多项式存储 Polynomial Storage:存储量是 N 和 D 的多项式函数，记作 \(poly(N,D)=N^{O(1)}D^{O(1)}\)。

HNSW / IVF可以通过细化索引,保留更多边/聚类中心来提高召回率,但这类索引仍然属于多项式存储,哪怕把它调得再精细，也无法拿到理论上的分数幂亚线性查询复杂度。只是工程平均速度变快，最坏情况依然可能接近 O (N)。

> Instead, vector indexes speed up queries by minimizing the number of comparisons. This is achieved by
partitioning S so that only a small subset is compared,
and then arranging the partitions into data structures
that can be easily traversed.
> Unlike typical attributes, vectors are not obviously
sortable nor can they be easily categorized. To achieve
high accuracy, these indexes rely on novel techniques, which we refer to as randomization, learned partitioning, and navigable partitioning. The large physical size
of vectors also leads to use of compression, namely a
technique called quantization, as well as disk resident
designs. Additionally, the need to support predicated
queries has led to special hybrid operators for indexes
> Partitioning Techniques.
– Randomization. Randomization aims to exploit probability amplification of multiple independent events,
allowing indexes to better discriminate truly similar
vectors from dissimilar ones.
– Learned Partitioning. Learning-based techniques aim
to uncover an internal structure of S so that it can
be partitioned along this structure. These techniques
can be supervised or unsupervised.
– Navigable Partitioning. Instead of fixating on absolute partitions, navigable indexes are designed so
that different regions of S can be easily traversed.
Some partitioning strategies are data independent, where
the rules are the same for any data distribution. But
the majority are data dependent. If updates to S alter its distribution, then indexes based on data dependent strategies may eventually become unbalanced over
time. In many cases, this can only be resolved by rebuilding the index.
> There are three basic structures: tables divide S into buckets containing similar
vectors; trees are a nesting of tables; graphs connect
similar vectors with virtual edges that can then be traversed.



















# 向量数据库与 Embedding 检索笔记

> 原文链接：https://zhuanlan.zhihu.com/p/27399676042，https://medium.com/@myscale/understanding-vector-indexing-a-comprehensive-guide-d1abe36ccd3c

## 1. 如何把原始数据嵌入为向量

### 1.1 不同原始数据类型转向量

> **Q：embedding 模型是怎么实现不同模态数据的跨模态搜索的？**
>
> 多模态嵌入模型将不同类型原始数据映射到同一个共享向量空间，实现跨模态相似度计算。
>
> * **图像**：原始 RGB 像素矩阵，经预处理后由视觉 Transformer/CNN 提取视觉特征，投影生成图像嵌入；
> * **文本**：字符串先分词，通过文本 Transformer 编码，投影得到文本嵌入；
> * **音频**：原始声波转为梅尔频谱，音频编码器提取特征后映射为音频嵌入；
> * **视频**：抽取关键帧提取图像特征，融合时序信息得到视频嵌入。

### 1.2 向量写入向量数据库的完整步骤（以 Chroma / FAISS / Milvus 举例）

#### ① 预处理

* **清洗**：去掉换行、多余空格、特殊符号
* **分块（Chunking）**：得到一批 `chunk_list`

> 原因：嵌入模型有最大 token 限制；块太大语义混乱，块太小丢失上下文。块重叠（overlap）
>
> 常用：512–1024 token 一块，重叠 10–20% 避免语义割裂。

#### ② 调用 Embedding 模型生成向量

embedding 模型的翻译规则是 AI 模型在海量文本里学习出来的转换逻辑，存在多种不同的 embedding 模型，建库和查询必须使用同一个。

#### ③ 保存元数据

除了向量数组，同时保存原始文本、来源、页码等 metadata，方便检索后还原原文。

```json
{
    "id": "唯一id",
    "vector": "embedding_vector",
    "metadata": {
        "original_text": "原始片段",
        "source": "论文.pdf",
        "page": 5
    }
}
```

#### ④ 批量插入向量数据库

---

## 2. 传统数据库与向量数据库

> 从本质上讲，向量数据库是针对非结构化数据的综合解决方案。与这种误解相反，它具备如今结构化/半结构化数据库管理系统中的一些用户友好特性，比如云原生、多租户和可扩展性。随着我们对本教程的深入探讨，就会发现它克服了独立向量索引的局限性，解决了可扩展性难题、集成复杂性，以及缺乏实时更新和内置安全措施等问题。

**独立向量索引**：单纯的算法库（FAISS、Annoy），储存向量，做相似度查询。

问题：

1. **可扩展性难题**：数据量变大，自己要手动分片、手动管理机器，很难横向扩展；
2. **集成复杂性**：需要手动把向量索引和业务数据库、元数据、业务逻辑拼在一起，开发工作量巨大；
3. **缺乏实时更新**：新增 / 删除向量，原生索引很难做到实时更新，经常要全量重建索引，不能频繁增删；
4. **缺少内置安全措施**：没有权限控制、访问鉴权。

👉 **向量数据库（Milvus / Qdrant / MyScale）把这些功能全部封装好。**

向量数据库是专门优化高维向量、非结构化语义检索的完整数据库产品，但传统关系型数据库依然有不可替代的优势。

### 2.1 传统关系型数据库的核心优势

1. **强 ACID 事务，适合业务交易、强一致性场景**
   多条关联行不能出现部分更新。
2. **强大的多表关联、分组、统计、复杂 SQL 聚合查询。**

> **Q：每一条记录都包含向量、元数据，为什么不能像传统数据库一样，把两个集合拿出来联合查询？**
>
> JOIN 是「两张表，通过关联字段，把行和行做笛卡尔匹配」。
>
> 向量数据库也支持跨集合查询，但只能靠应用代码做拼接。
>
> 先在集合 A 做向量检索拿到 id，拿着 id 去集合 B 做过滤查询。
>
> → 两次独立查询。不是数据库内部直接把两个集合做关联运算。当数据量巨大的时候，性能会很差。而如果硬给向量库加上完整 JOIN、ACID，会牺牲向量检索的性能，违背它的设计初衷。

⇒ 传统数据库高度结构化，向量数据库很难实现：

1. **元数据是自由的 key-value，没有强制的表结构、没有外键概念。**
   元数据可以随便新增字段，不需要提前定义表；这是它处理非结构化碎片的优势，但没有办法建立表与表之间的强关联关系。
2. **向量数据库底层存储，向量和元数据是分开存储的：**
   向量存在专门的向量存储引擎，元数据存在另一个存储。它的设计重心是快速找“向量距离近的条目”，而不是做跨集合的复杂关联运算。
3. **成熟稳定，生态极其庞大**
4. **结构化数据存储效率极高**

### 2.2 向量数据库的优势

1. **原生针对高维向量做索引优化，千万–亿级向量相似度搜索速度远快于 pgvector；**

   > pgvector 是 PostgreSQL 插件，它给关系库增加向量检索能力，但它依然受关系库架构限制，海量向量下性能不如专门向量数据库。

2. **原生支持非结构化数据的语义检索**：文本、图片的 embedding 检索；

3. **原生分布式分片、多租户**，专门为向量海量数据做架构设计；

4. **元数据灵活**，不需要强行把数据拆成固定列。

> **Note：现实工程中，两者经常搭配使用（RAG 最典型架构）**
>
> 1. **关系数据库**：存业务结构化数据：用户信息、文档元信息（文档 id、文件名、上传时间、权限），处理事务、业务统计。
> 2. **向量数据库**：只存：文本 chunk 的向量 + 少量元数据；专门做语义相似度检索。

> **Q：为什么向量数据库没有统一指令？因为太新了。→ 现实工程怎么解决“指令不统一”的痛点？**
>
> 自己写一层封装，把不同接口统一成自己内部的函数。
>
> 多语言 SDK：同一套底层 API，包装成不同编程语言的调用，底层指令是一样的。
>
> 比如 Milvus 提供 Python、Go、Java SDK。

> 在传统数据库中，我们通常查询数据库中值与查询条件完全匹配的行。而在向量数据库中，我们通过应用相似性度量来找到与查询向量最相似的向量。

> **Q：为什么向量数据库几乎不找“完全一模一样的向量”，而是找相似的？**
>
> * 想要完全一样的原文，直接用传统数据库做文本精确匹配。
> * 向量检索的价值就是处理语义近似，文字不完全相同的情况，而传统数据库无法解析文本内部语义，只能做关键词 Like 模糊匹配，没办法做语义相似度检索。

**索引**：向量数据库使用诸如 PQ、LSH 或 HNSW 等算法对向量进行索引（下文会详细介绍这些算法）。这一步将向量映射到一种数据结构，以便加快搜索速度。

**查询**：向量数据库将索引后的查询向量与数据集中的索引向量进行比较，找到最近邻向量（应用该索引使用的相似性度量）。

**后处理**：在某些情况下，向量数据库从数据集中检索最终的最近邻向量，并对其进行后处理以返回最终结果。这一步可能包括使用不同的相似性度量对最近邻向量重新排序。

在接下来的部分，我们将更详细地介绍这些算法中的每一种，并解释它们如何影响向量数据库的整体性能。

---

## 3. 索引技术

> Note:
> 1. **原始向量在底层存储本身是没有语义顺序的**，存放顺序一般就是插入的先后顺序，是无序的。
> 2. **索引是单独额外构建出来的一套导航结构**。索引不会改动原始向量的物理存放位置，索引记录的是：向量 ID、簇归属、节点连接关系（HNSW）等元信息，用来**跳过大量不需要比对的向量**。

### 3.1 扁平索引：最简单的索引

> 扁平索引之所以被称为“扁平”，是因为我们不会对输入的向量进行任何修改。由于不对向量进行近似或聚类，这些索引能产生最准确的结果。我们能获得完美的搜索质量，但这是以显著的搜索时间为代价的。
>
> 使用扁平索引时，我们引入查询向量 x_q，并将其与索引中的每个其他完整向量进行比较，计算与每个向量的距离。
>
> 在计算完所有这些距离后，我们会返回其中最近的 k 个向量作为最匹配的结果，这就是 k 近邻（kNN）搜索。

**如何加快搜索速度呢？主要有两种方法：**

1. 通过降维或减少表示向量值的比特数来减小向量大小。
2. 通过基于某些属性、相似性或距离对向量进行聚类或将其组织成树结构来缩小搜索范围，将搜索限制在最接近的聚类中，或者通过最相似的分支进行筛选。

使用这两种方法中的任何一种，都意味着我们不再进行详尽的最近邻搜索，而是进行近似最近邻（ANN）搜索，因为我们不再搜索整个高分辨率数据集。

### 3.2 向量预处理手段 / 编码技术

可以搭配下文各种索引方法使用。

#### 3.2.1 随机投影

> 随机投影的基本思想是使用随机投影矩阵将高维向量投影到低维空间。我们创建一个随机数矩阵，矩阵的大小将是我们想要的目标低维值。然后计算输入向量与矩阵的点积，得到一个投影矩阵，其维度比原始向量少，但仍然保留了它们的相似性。
>
> 当我们进行查询时，使用相同的投影矩阵将查询向量投影到低维空间。然后，将投影后的查询向量与数据库中的投影向量进行比较，找到最近邻。由于数据的维度降低了，搜索过程比在整个高维空间中搜索要快得多。
>
> 请记住，随机投影是一种近似方法，投影质量取决于投影矩阵的属性。一般来说，投影矩阵越随机，投影质量就越好。然而，生成真正的随机投影矩阵在计算上可能会很昂贵，特别是对于大型数据集。

#### 3.2.2 乘积量化 Product Quantization, PQ

属于向量量化 VQ 的一种：（常搭配 IVF 等一起使用）

**流程：**

1. 把 D 维原始长向量切成 m 段子向量（比如 128 维切成 8 段，每段 16 维）。

2. 对每一类子向量（所有向量的第 i 段）单独跑 K-means 聚类，每一段得到 k 个聚类中心，即段 i 的码书（codebook）。

   分段量化之后，这一组码字仍然绑定这个样本的编号。

   → 把高维空间拆分为多个低维子空间，避免维数灾难。

3. 码书训练完成之后，才开始给库内每一条向量做编码：

   把 X 切成 m 段子向量，对 X 的第 i 段子向量，在第 i 段专属码书里面，找距离最近的那 1 个中心，记下这个中心的索引 j_i。

   一条向量 X 最终存储的编码，就是这一串索引号：

   `[j0, j1, j2, ..., jm-1]`

   对于任意原始子向量，不存浮点数，只保存离它最近的聚类中心的编号（索引）。

   → 向量之间的区分能力下降，空间里点的重叠变多，计算相似度的时候会出现误差 → 搜索召回 / 准确率下降。

#### 3.2.3 标量量化 Scalar Quantization, SQ

- 拆分方式：按维度单独拆，每一维独立量化
- 划分依据：预先划定数值区间
- 子单元：单个数字（标量，1 维）
- 编码：每一维看落在哪个区间，输出区间编号
- 缺点：高维效果很差，因为独立逐维划分忽略维度之间的相关性，适合处理低维数据
- 优点：float被压缩为int，大幅减少内存占用且整数运算比浮点运算快

---

### 3.3 基于树的向量索引方法

> 基于树的方法对于低维数据非常有效，并且可以提供精确的最近邻搜索。然而，由于“维度诅咒”，它们在高维空间中的性能通常会下降。此外，它们需要大量内存，对于大型数据集效率较低，这会导致构建时间更长和延迟更高。

**本质：空间划分 + 层次剪枝**

**优点**

1. 构建逻辑清晰，天然支持精确最近邻搜索；
2. 内存开销相对可控；
3. 适合低维向量（图像特征、传感器数据）。

**缺点**

1. **高维灾难**：树的剪枝效率极低。高维几何会让剪枝下界变得很松，剪枝大量失效，多维度数值叠加；
2. **动态更新差**：新增 / 删除向量，树结构容易失衡，往往需要完整重建索引，不适合频繁写入的业务；
3. **召回上限不如图索引 HNSW。**

#### 精确最近邻树索引

> 不属于前文所说 ANN，带有回溯剪枝，只把不可能的情况舍去，仍是精确 KNN。

##### ① KD-Tree

1. 选一个维度。
2. 这个维度的中位数作为分割线，把数据分成左右两部分。
3. **关键剪枝**：计算查询点到分割线的距离。如果这个距离大于当前已经找到的最近点距离，说明另外一侧子树不可能存在更近点，可以直接丢弃整棵子树。
4. 递归对左右两部分继续重复，直到叶子节点存少量向量。

##### ② Ball-Tree

1. 选一个中心点，把所有向量包进一个超球；
2. 选两个相距最远的点作为两个子球中心；
3. 查询点到球心距离 − 球半径 > 当前找到的最小距离 → 这个球里面不可能存在更近的点，直接整颗球丢弃，不用遍历内部向量；
4. 递归，每个子球继续拆分。

##### ③ VP-Tree

1. 随机选一个向量作为优势点 VP；
2. 计算所有向量到这个 VP 的距离，取中位数距离；
3. 分成两组：距离 VP 小于中位数的放内圈；大于中位数放外圈；
4. 递归对内圈、外圈继续选 VP 建树。

⇒ Ball-Tree、VP-Tree 解决了 KD-Tree 会粗暴切开簇的问题。

#### 近似树索引

> 不做回溯，人为舍弃可能存在近邻的区域。

##### ④ ANNOY（Approximate Nearest Neighbors Oh Yeah）

1. 随机选两个向量，生成垂直平分超平面，即到 a 和到 b 距离完全相等的所有点构成的面。

   → Annoy 规避高维灾难的方法：用随机两点生成的斜超平面，斜向分割可以同时利用所有维度信息做划分，不会被个别无用维度限制；不做完备搜索，放弃回溯，用多棵随机树采集候选，不需要去证明分支能不能剪，所以不会出现性能断崖式下跌。

   * 所有向量 p：如果 `dist(p, a) < dist(p, b)` → 分到左子树
   * 所有向量 p：如果 `dist(p, b) < dist(p, a)` → 分到右子树

2. 递归继续随机选两个点，画分割平面；

3. 不断递归，直到叶子里向量数量少于设定阈值（比如叶子最多存 10 个点），递归停止，构建成一棵二叉树；

4. 重复上面流程，构建很多棵独立的树。

   → 单棵随机树分割很“偏”，真实最近邻很容易被切到分割面另一边，查询时直接漏掉。树越多，召回率越高，越不容易丢失真实近邻；但内存和查询耗时也变大。

**查询逻辑：**

树里面每一个内部节点都存了当初建树时随机挑选出来的一对点（a, b），以及对应的分割超平面。

1. 当前节点取出该节点保存的 a 和 b；

2. 计算查询向量 q 到 a 的距离、q 到 b 的距离；

3. 比较距离大小：

   * q 离 a 更近 → 进入左孩子；
   * q 离 b 更近 → 进入右孩子。

   → 不会访问另一侧子树，可能漏掉最近邻。

4. 来到子节点，重复上面第 1～3 步；

5. 一直走到这个节点没有子节点（叶子节点），停止。取出叶子里面存的全部向量，作为这棵树贡献的候选集；

6. 把所有树收集来的候选向量合并；

7. 在候选集合里计算距离，返回 Top-K。

> Annoy 的高效性和内存高效性使其成为处理高维数据和大型数据库的有力选择。不过，也有一些需要考虑的权衡因素。构建索引可能需要花费大量时间，特别是对于大型数据集。由于 Annoy 使用随机森林分区算法，索引无法使用新数据进行更新，必须从头重新构建。根据数据集的大小以及数据变化的频繁程度，重新训练索引的成本可能过高。

---

### 3.4 量化方法

> 量化方法在内存利用上效率较高，通过将向量压缩为紧凑的代码来实现快速搜索。但是，这种压缩可能会导致信息丢失，从而降低搜索准确性。另外，这些方法在训练阶段的计算成本较高，会增加构建时间。

**核心原理**：对原始高维浮点数向量做有损压缩，把连续取值的向量空间离散化，把量化技术包装成完整检索方案。

**查询步骤：**

1. 查询向量 q 按完全相同的量化规则切分，拿 q_i（q 的第 i 段子向量），和第 i 段码书里全部 K 个中心算距离，保存成一张距离表：

   `dis_table[i][j] = 距离 q 的 i 段子向量，第 i 段码书第 j 号中心`

2. 遍历数据库里面每一条向量的编码，算近似距离。

   数据库里面某条向量 X，存的是编码 `[j0, j1, j2, ...]`，还有 X 的 ID。

   * 取 X 编码第 0 位：j0 → 查表 `dis_table[0][j0]`
   * 取 X 编码第 1 位：j1 → 查表 `dis_table[1][j1]`
   * 全部段查表，累加，得到 `approx_dist = sum`，估算出来 q 和 X 的相似度。

3. 根据估算距离从小到大排序，选出 Top-K 候选；

4. 得到候选的 ID，可以拿到原始向量。

> **Note：**
>
> 若是纯量化编码的检索算法，只能压缩原始向量和快速估算近似距离，没有空间划分机制，仍需历遍所有向量，但大数据场景需要遍历全部向量，检索效率很低，工程上很少单独使用，通常搭配 IVF 聚类索引组成 IVFPQ 混合索引。

---

### 3.5 哈希方法

> 哈希方法速度快且相对节省内存，它将相似的向量映射到同一个哈希桶中。在处理高维数据和大规模数据集时表现良好，具有较高的吞吐量。然而，由于哈希冲突可能会导致误报和漏报，从而降低搜索结果的质量。选择合适数量的哈希函数和哈希表至关重要，因为它们会显著影响性能。

#### 3.5.1 局部敏感哈希（LSH）

> LSH 的性能范围很广，在很大程度上取决于设置的参数。搜索速度较慢时能得到较好的结果质量，而快速搜索则会导致结果质量较差。在处理高维数据时性能不佳。条形图中半填充的部分表示修改索引参数时性能的变化范围。
>
> 局部敏感哈希（LSH）的工作原理是通过一个哈希函数对向量进行处理，将相似的向量分组到同一个桶中，这个哈希函数的目的是最大化哈希冲突，而不是像通常的哈希函数那样最小化冲突。

> **这是什么意思呢？**

> 想象我们有一个 Python 字典。当我们在字典中创建一个新的键值对时，我们使用哈希函数对键进行哈希。键的这个哈希值决定了我们存储其对应值的“桶”。

> 一个类似字典的对象的典型哈希函数会试图最小化哈希冲突，目标是为每个桶只分配一个值。

> Python 字典就是一个使用典型哈希函数的哈希表的例子，这种哈希函数会最小化哈希冲突，即两个不同的对象（键）产生相同的哈希值这种情况。

> 在我们的字典中，我们希望避免这些冲突，因为这意味着多个对象会映射到同一个键上。但对于 LSH 来说，我们希望最大化哈希冲突。

> **为什么我们想要最大化冲突呢？**

> 嗯，为了进行搜索，我们使用 LSH 将相似的对象分组在一起。当我们引入一个新的查询对象（或向量）时，我们的 LSH 算法可以用来找到最匹配的组：

> LSH 的哈希函数试图最大化哈希冲突，从而对向量进行分组。

**Note：哈希函数的设计**

**标准哈希**

1. 把输入（key）转换成一串二进制，比如整数、字符串，转成固定长度比特流。
2. 打乱比特，充分混合：通过移位、异或、乘法，让输入哪怕只有一点点变化，最终哈希值也会完全变掉（雪崩效应）。
3. 最后映射到桶编号：`bucket_id = hash_value mod bucket_count`

**LSH 局部敏感哈希函数**

1. 随机生成一个高维权重向量 r（和输入向量维度一样）。

2. 随机画一个超平面，把空间切成两半。两个向量方向越接近，越大概率落在同一侧，得到相同哈希值：

   `h_r(v) = 1, r 和 v 点积 ≥ 0`

   `h_r(v) = 0, r 和 v 点积 < 0`

   * 计算向量 v 和随机向量 r 的点积；
   * 点积 ≥ 0 → 输出 1；否则输出 0。

3. 为提升效果（和 ANNOY 一样合并候选，弥补单次划分的失误）：

   * 一次性生成多个随机向量，把多个独立的哈希函数计算结果拼接，组成哈希码，作为桶 ID。即一组哈希函数 = 一个哈希表；
   * 建多个哈希表，提高召回（代价：内存上升）。

---

### 3.6 聚类方法

> 聚类方法可以通过将搜索空间缩小到特定的聚类来加快搜索操作，但搜索结果的准确性可能会受到聚类质量的影响。聚类通常是一个批处理过程，这意味着它不太适合不断添加新向量的动态数据，因为这会导致频繁的重新索引。

**核心原理**：基于 K-means 等聚类算法，把全部库向量划分成若干簇（聚类），每个簇有一个聚类中心（质心）。

1. **建索引**：将所有向量聚类，每个向量归属距离最近的簇；索引保存簇中心，同时记录每个簇内部包含哪些向量。
2. **查询阶段**：先计算查询向量和各个簇中心的距离，挑选距离最近的少数几个簇；只在选中簇的内部向量中做相似度检索，直接跳过其他全部簇，缩小搜索空间，加快检索速度。

**聚类算法：**

##### ① K-means

无监督聚类算法。无监督 = 不需要给数据打标签，自动把相似的数据归成一类。

**核心目标**：给定一堆数据，算法自动找出预设的 K 个簇中心（质心）。

**简单步骤：**

1. 随机选 K 个初始中心点；
2. **分配**：所有样本，划分给距离最近的中心点，形成 K 个簇；
3. **更新**：每个簇内所有样本求平均，得到新的簇中心；
4. 重复【分配 → 更新】直到中心点基本不再移动，收敛停止。

##### ② 基于密度的空间聚类算法（DBSCAN）

> DBSCAN 算法基于密度可达性和密度连通性的概念。它从数据集中的任意一个点开始，如果在给定半径 eps 内，该点周围至少有 minPts 个点，就会创建一个新的聚类。这里的 eps 代表 epsilon，是用户定义的输入参数，表示两个点在同一聚类中时，它们之间的最大距离；而 minPts 指的是形成一个聚类所需的最少数据点数量。
>
> 它会迭代地将 eps 半径内所有直接可达的点添加到聚类中。这个过程会一直持续，直到没有更多的点可以添加到这个聚类中。然后，算法会继续处理数据集中下一个未访问过的点，并重复上述过程，直到所有点都被访问过。
>
> DBSCAN 算法中的关键参数是 eps 和 minPts，它们分别定义了点的聚类范围和形成聚类所需的最小点密度。

> **Note：**
>
> 最后剩下始终没有被任何聚类吸纳的点，就是噪声。
>
> **参数 eps、minPts 的权衡（考点）**
>
> 1. eps 太大：距离很远的点都算邻居，很多簇合并成一大团，聚类数量变少；eps 太小：只有挨得极近才算邻居，大片点直接被判定成噪声，簇被拆碎。
> 2. minPts 越大：要求局部点的密度更高，不容易形成聚类，更多点变成噪声；minPts 越小：很低密度就能形成簇，容易把零散噪声也打包成聚类。

---


#### 3.6.1 倒排文件（IVF）索引

> 倒排文件索引（IVF）通过聚类来缩小搜索范围。它是一种非常受欢迎的索引，因为它易于使用，具有较高的搜索质量和合理的搜索速度。
>
> 它基于 Voronoi 图的概念，也称为 Dirichlet 镶嵌。
>
> 为了理解 Voronoi 图，我们需要想象将高维向量放置在二维空间中。然后在二维空间中放置一些额外的点，这些点将成为我们的“聚类”（在我们的例子中是 Voronoi 单元）质心（仍然使用的 K-means）。
>
> 然后，我们从每个质心向外扩展相同的半径。在某个时刻，每个单元圆的圆周会与另一个圆周碰撞，从而形成单元边界：
>
> 现在，每个数据点都将包含在一个单元内，并被分配给相应的质心。
>
> 但是，如果查询向量落在单元的边缘附近，就会出现一个问题：它最接近的其他数据点很可能包含在相邻的单元中。我们称之为边缘问题：
>
> 为了缓解这个问题并提高搜索质量，我们可以增加一个名为 nprobe 的值的索引参数。通过 nprobe，我们可以设置要搜索的单元数量。


### 3.6.2 IVF的多种变体

① IVFFLAT
> IVFFLAT is a simpler form of IVF. It partitions the dataset into clusters. However, within each cluster, it uses a flat structure (hence the name “FLAT”) for storing the vectors. IVFFLAT is designed to optimize the balance between search speed and accuracy.

即最简单的IVF，选出与查询向量距离最近的nprobe个中心，然后在对应的簇内部暴力历遍与查询项链的距离。

② IVFPQ

> IVFPQ is an advanced variant of IVF, which stands for Inverted File with Product Quantization. It also splits the data into clusters but each vector in a cluster is broken down into smaller vectors, and each part is encoded or compressed into a limited number of bits using product quantization.

对库内所有向量，提前做 PQ 乘积量化压缩，在选中的这 nprobe 簇里面，拿 PQ 编码做近似距离计算，根据估算距离排序，返回 TopK 候选。

③ IVFSQ

> In IVFSQ, each vector in a cluster is passed through scalar quantization. This means that each dimension of the vector is handled separately.
> In simple terms, for every dimension of a vector, we set a predefined value or range. These values or ranges help decide which cluster a vector belongs to. Each component of the vector is then matched against these predefined values to find its place in a cluster. This method of breaking down and quantizing each dimension separately makes the process more straightforward. It’s especially useful for lower-dimensional data, as it simplifies encoding and reduces the space needed for storage.


### 3.7 基于图的方法

> 基于图的方法在准确性和速度之间取得了较好的平衡。它们对高维数据很有效，并且可以提供高质量的搜索结果。但是，由于需要存储图结构，它们可能会占用大量内存，而且图的构建在计算上也很昂贵。

**核心原理**：把每一条向量作为图的节点；向量之间相似度高，就在对应节点之间建立边，构建一张近邻关系图。

1. **建索引**：遍历向量库，为每个节点和它的相似近邻节点建立连接，保存整张图的邻接关系；部分算法构建多层图，上层做远距离跳转、下层保存精细近邻关系。
2. **查询**：从图中某个入口节点出发，沿着边贪心游走，不断向距离查询向量更近的节点前进；多次迭代收敛，找到近邻候选，不需要扫描全部向量，兼顾速度与召回率。

#### 3.7.1 分层可导航小世界图（HNSW）

> Its graph-like structure takes inspiration from two different techniques: the probability skip list and Navigable Small World (NSW).

Skip List

> A skip list is an advanced data structure that combines the advantages of two traditional structures: the quick insertion capability of a linked list and the rapid retrieval characteristic of an array. It achieves this through its multi-layer architecture where the data is organized across multiple layers, with each layer containing a subset of the data points.

> Starting from the bottom layer, which contains all data points, each succeeding layer skips some points and thus has fewer data points, ultimately the topmost layer will have the smallest number of data points.
To search for a data point in a skip list, we start from the highest layer and go from left to right exploring each data point. At any point, if the queried value is greater than the current datapoint, we move back to the previous datapoint in the layer below and resume the search from left to right until we locate the exact point.

原生跳表存储一维标量 key，底层是完整有序链表，精细度从上往下递增。

Navigable Small World (NSW)

> Navigable Small World (NSW) is similar to a proximate graph where nodes are linked together based on how similar they are to each other. The greedy method is used to search for the nearest neighbor point.
We always begin with a pre-defined entry point, which connects to multiple nearby nodes. We identify which of these nodes are the closest to our query vector and move there. This process iterates until there is no node closer to the query vector than the current one, serving as the stopping condition for the algorithm.

Back to HNSW:

> So, what happens in HNSW is that we take the motivation from the skip list, and it creates layers like the skip list. But for the connection between the data points, it makes a graph-like connection between the nodes. The nodes at each layer are connected not only to the current layer nodes but also to the nodes of the lower layers. The nodes at the top are very few and intensity increases when we go down to the lower layers. The last layer contains all the data points of the database.

层级分配规则：

每插入一条新向量，随机采样一个最大层数 max_level = floor(-ln(随机0~1小数) * mL)，mL=1/ln(M)
- 概率是指数衰减：绝大多数向量只存在 Layer0；少数同时在中间层；顶层只有极少量节点（枢纽节点）

跳表也是随机决定节点最高层。

Note：如果一个节点分配最高层是 2，那它会同时存在 Layer0、Layer1、Layer2 这三层

搜索逻辑：

1. 从顶层入口点开始，在本层图上贪心遍历：不断跳到本层内离查询更近的邻居
2. 直到当前节点的所有本层邻居，都没有比它离查询更近 → 局部极小
3. 此时，才拿这个局部极小点作为入口，下降到下一层，重复贪心搜索

### 3.7.3 HNSW的变体

> In HNSWFLAT, the raw vectors are stored as they are, while in HNSWSQ, the vectors are stored in a quantized form. Apart from this key difference in data storage, the overall process and methodology of indexing and searching are the same in both HNSWFLAT and HNSWSQ.

### 3.7.4 Multi-Scale Tree Graph (MSTG) Algorithm

> Standard Inverted File Indexing (IVF) partitions a vector dataset into numerous clusters. However, a notable limitation is the substantial growth in index size for massive datasets, requiring the storage of many cluster representative vectors. The scalability of IVF is hindered by the significant memory overhead associated with this approach.
> Multi-Stage Tree Graph (MSTG) is developed by MyScale and it overcomes this through a hierarchical design. Unlike IVF which has a single layer of cluster vectors, MSTG creates multiple layers, which means less persistent centroids in the memory. For example, if a dataset needs 10,000 cluster vectors in IVF, all 10,000 have to be stored, consuming substantial memory. In MSTG, using a 2-layer hierarchy of 100 clusters per layer, only 200 vectors need storage — the 100 top-layer vectors, and their 100 direct children.
> MSTG combines the advantages of both tree and graph-based algorithms.

依旧上疏下密（树状层级聚类）
- **上层**：少量质心（训练的 k 小），代表全局大区域，覆盖范围很大，节点稀疏
- **往下**：每个上层质心管辖的区域内，再做 K-means 分出子质心，不需要把全部子层的质心一次性加载进内存；子质心覆盖的空间更小、更精细，数量变多、分布变密，与HNSW不同不是原始数据

- 单纯层次聚类树有个经典硬伤：
检索的时候，一旦顶层选的分支选错了，目标向量在另一个子分支，就再也找不回来了 → 召回率暴跌。

**MSTG 的改进：在最底层叶子节点里面维护一张近邻图。**

1. **树部分（导航层，上层所有层级）**：多层 K-means 质心构成树，用来快速粗定位，把查询快速缩小到一小块空间。
2. **图部分（叶子层）**：树的叶子节点里面存放原始向量，并且**这些原始向量之间构建近邻图**。
   - 当顺着树落到叶子区域之后，不是只在这个叶子内部暴力遍历；
   - 可以沿着图的边，**跨叶子做局部游走**，能跳到相邻叶子节点的向量，弥补树结构 “一旦走错分支就卡死” 的缺陷。
  

> It builds fast, searches fast, and remains fast and accurate under different filtered search ratios while being resource and cost-efficient.

在高选择性过滤查询下（满足条件的向量只占数据集很小一部分），MSTG 的分层树聚类能快速剪枝大量不符合过滤条件的区域，这是它的亮点。

而在无过滤、全库检索的情况下（最常规 KNN，不加任何 where 条件）：HNSW 高层长距离跳转，贪心直接逼近目标；MSTG 需要逐层和质心计算、逐层下探树分支，树导航的开销不一定占优。而且 MSTG 到底层之后，依然要跑局部近邻图，底层检索开销和 HNSW 接近。

---

## 4. 相似度度量

> **Q：之前讨论的那些索引算法都是用的距离，可以搭配不同的相似度度量使用吗？**
>
> 大多数向量近似检索索引算法理论上支持搭配多种相似度 / 距离度量，但并非可以无限制随意替换。索引的空间划分、聚类、哈希、图构建逻辑依赖度量的数学性质，因此切换相似度度量时存在明确约束。

### 4.1 基于空间划分 / 聚类的索引

IVF、DBSCAN、KD 树（Annoy）

这类索引在定义簇、核心点、质心，都是基于欧氏距离，原生默认欧氏距离。

Annoy 本质是比较向量方向 / 空间距，原生支持余弦相似度，也支持欧氏距离。

IVF / DBSCAN 如果想改成余弦相似度，需要先把向量归一化。

> **原理**：向量归一化 `||v|| = 1`，`dist_euclid` 平方 = `2 * (1 - cosθ)`。
>
> 归一化后，余弦越大，欧氏距离越小，排序结果完全一致。

### 4.2 LSH

LSH 局部敏感哈希 LSH 是按相似度度量单独设计哈希函数！不能直接混用。

### 4.3 HNSW（图索引）

HNSW 算法本身不绑定距离公式，属于通用框架。

只要你能实现一个计算两个向量相似度 / 距离的函数，就可以替换：

* 欧氏
* 余弦
* 点积

都支持。

代价：不同度量，检索性能、召回表现会变。

### 4.4 PQ

PQ 乘积量化是向量压缩，压缩后计算的是估算距离，估算器需要和你选用的相似度匹配，可以用于欧氏、归一化后的余弦。

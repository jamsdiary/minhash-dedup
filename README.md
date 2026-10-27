# minhash-dedup

用 MinHash + LSH 做大规模语料去重的小型可复现实现。

参考 FineWeb / RefinedWeb 的做法：对 CommonCrawl 类语料先做近似去重，再做质量过滤，
最后用「去重前后 benchmark 变化」量化收益。

---

## 1. 这是什么 / 解决什么问题

大模型语料里重复文档占比很高。重复数据会：

1. **浪费算力** —— 同一篇文档出现 100 次 = 100 倍 token 成本，换来几乎 0 的新信息
2. **扭曲分布** —— 高频内容被过度学习，长尾内容欠拟合
3. **污染评测** —— 测试集题目出现在训练集里，分数是假的

本项目实现的是第 1 层（效率）的解法：**近似去重**。

---

## 2. 核心原理（面试要能推出来）

### 2.1 Jaccard 相似度

两个文档的 **词集合** 的交集除以并集：

    J(A, B) = |A ∩ B| / |A ∪ B|

它是衡量「两段文本讲的是不是同一件事」的最常用指标。

### 2.2 MinHash：把集合相似度变成数字签名

对集合 A，用 k 个哈希函数各算一次最小值，组成签名 sig(A) = (h1(A), ..., hk(A))。

**关键性质（为什么它能估 Jaccard）**：

    P[ sig(A)[i] == sig(B)[i] ] = J(A, B)

因为「最小值的那个元素在 A∩B 里」的概率，恰好等于交集占并集的比例。

所以 **k 个位置上相等的比例，是 Jaccard 的无偏估计**。k=112 时标准误约 ±0.05。

### 2.3 LSH：把 O(n²) 降到近线性

签名算出来还是要比 n² 对。LSH 的招：**不逐对比，先用便宜的分桶筛出候选对**。

- 签名切 b 个 band，每 band r 个 hash
- **同一个 band 内 r 个 hash 全相等** → 进候选池
- 候选池里再用精确 Jaccard 卡阈值

候选概率：

    P[ 成为候选 ] = 1 - (1 - s^r)^b

阈值（拐点）在 s = (1/b)^(1/r)。

| 配置 | b | r | 阈值 | 出处 |
|---|---|---|---|---|
| FineWeb | 14 | 8 | 0.719 | FineWeb 博客 |
| RefinedWeb | 45 | 20 | 0.827 | RefinedWeb 论文 |
| DataTrove 默认 | 14 | 8 | ~0.72 | datatrove 源码 |

**b 和 r 怎么选**：r 越大 → 门槛越高 → 精确率高、召回低；b 越大 → 台阶越宽 → 召回高、误候选多。
生产做法：**先把 b 调大到召回够，再用精确 Jaccard 卡阈值**。

---

## 3. 设计取舍（待 D20 填充实测数据）

> 这一节是本项目的核心，面试时讲的是「为什么这样选」而不是「我做了什么」。

### 3.1 切词粒度：word vs char n-gram

| 方案 | 优点 | 缺点 | 选型 |
|---|---|---|---|
| 中文按字切分 | 中文无空格分词，实现简单 | 跨语言不通用 | 样本级用 |
| 英文按 word 切分 | 语义稳定 | 中英文混排时要分两套逻辑 | 样本级用 |
| 统一用 char n-gram | 跨语言统一、实现最简 | 中文长文本 n-gram 数量爆炸 | 备选 |

**待填**：为什么最终选了某一种，实测 recall / 耗时对比。

### 3.2 n-gram 大小

datatrove 默认 `n_grams=5`。n 越大：
- 文档集更稀疏 → Jaccard 天然偏低 → 阈值要相应调低
- 签名计算量线性上升

**待填**：n=3/5/7 的对比曲线。

### 3.3 哈希函数的实现

**待填**：选 `hashlib.blake2b`（带 seed）还是 `murmurhash`？
每个 (文档, hash_id) 需要一个独立哈希 → 用 `blake2b(data, digest_size=8, key=seed_i)` 还是
`sha1(f'{shingle}_{i}')`？**这一决定直接影响签名质量，必须实测。**

### 3.4 分桶策略

**待填**：字典分桶 vs 排序分桶 vs 磁盘分桶（大规模时内存放不下怎么办）。

### 3.5 为什么不做精确去重 / SimHash / 向量去重

| 方法 | 比什么 | 优点 | 缺点 |
|---|---|---|---|
| 精确去重 | 完全相同 | 零误判 | 改一个字就漏 |
| **MinHash + LSH** | 词集合 Jaccard | 抓「改写版重复」，近线性 | 阈值要调 |
| SimHash | 汉明距离 | 快 | 短文本不稳，参数难解释 |
| 向量去重 | 语义相似 | 抓「意思一样」 | 要跑模型，成本高 |

---

## 4. 量化结果（待 D20 填充实测数据）

> **这一节是简历上唯一能写数字的地方。** 没有数字，前面全是理论。

### 4.1 基础指标

| 指标 | 值 |
|---|---|
| 样本文档数 | 待填 |
| 语料字符数 | 待填 |
| 检出重复对数 | 待填 |
| 去重后剩余文档数 | 待填 |
| 重复率 | 待填 |
| 运行耗时 | 待填 |
| 峰值内存 | 待填 |

### 4.2 精确率 / 召回率（对人工标注的重复对）

| 阈值 | 精确率 | 召回率 | F1 |
|---|---|---|---|
| 0.6 | 待填 | 待填 | 待填 |
| 0.72 | 待填 | 待填 | 待填 |
| 0.8 | 待填 | 待填 | 待填 |

### 4.3 参数扫描

**待填**：b × r 网格扫描的候选对数量、最终重复检出数、耗时三维表。

### 4.4 对标 DataTrove

**待填**：同一份语料，本实现 vs datatrove 官方 minhash 的结果差异（应接近，说明实现正确）。

---

## 5. 仓库结构

```
minhash-dedup/
├── README.md              # 本文件
├── requirements.txt
├── notes/                 # 学习笔记：原理推导、踩坑记录
├── src/
│   ├── shingle.py         # 切 n-gram（word / char 可切换）
│   ├── signature.py       # MinHash 签名计算
│   ├── lsh.py             # 分桶 + 候选对生成
│   └── verify.py          # 精确 Jaccard 确认
├── scripts/
│   ├── run_dedup.py       # 端到端入口
│   └── eval.py            # 精确率/召回率评估
└── data/sample/
    ├── zh_sample.txt      # 中文样本（zhwiki）
    └── en_sample.txt      # 英文样本（Gutenberg）
```

---

## 6. 样本语料

| 文件 | 来源 | 说明 |
|---|---|---|
| `data/sample/zh_sample.txt` | [zhwiki dump](https://dumps.wikimedia.org/zhwiki/) | 中文维基正文，已去模板/引用/wikitext 标记 |
| `data/sample/en_sample.txt` | [Project Gutenberg](https://www.gutenberg.org/ebooks/1342) | Pride and Prejudice 分章 |

**为什么要中英文各一份**：去重最容易在「切词粒度」这一步翻车，中英文分词规则完全不同，
必须两头都验。

---

## 7. 运行方式

> D20 实现完成后补上实际命令。

---

## 8. 参考文献

- FineWeb: decanting the web's best data — https://arxiv.org/abs/2406.17557
- FineWeb 博客（MinHash 参数推导）— https://hf-mirror.com/spaces/HuggingFaceFW/blogpost-fineweb-v1
- DataTrove 源码 — https://github.com/huggingface/datatrove
- RefinedWeb — https://arxiv.org/abs/2306.01116

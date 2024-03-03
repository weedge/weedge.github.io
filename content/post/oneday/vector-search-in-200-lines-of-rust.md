---
author: "weedge"
title: "译：FANN：200行Rust实现的向量搜索"
date: 2023-09-20T10:26:23+08:00
tags: [
	"oneday","ANN","LSH","rust"
]
categories: [
	"技术",
]

---

由于 AI/ML 采用的快速进展，向量数据库无处不在。虽然它们支持复杂的人工智能/机器学习应用，但向量搜索本身从概念上来说并不难。在这篇文章中，我们将描述向量数据库如何工作，并用不到 200 行 Rust 代码构建一个简单的向量搜索库。[所有代码都可以在此 Github 存储库](https://github.com/fennel-ai/fann)中找到。我们这里使用的方法基于流行库Spotify [annoy](https://github.com/spotify/annoy)中使用的一系列称为“[局部敏感散列(Locality-sensitive_hashing)](https://en.wikipedia.org/wiki/Locality-sensitive_hashing)”的算法。本文的目标不是介绍新的算法库，而是描述向量搜索如何使用真实的代码片段工作。首先了解下什么是向量搜索。

<!--more-->

## 向量简介（又名嵌入embedding）

文档、图像、视频等复杂的非结构化数据很难在传统数据库中表示和查询，特别是如果查询意图是查找“相似”项目。那么 Youtube 如何才能选择接下来播放的最佳视频呢？或者Spotify根据您当前的歌曲自定义音乐队列？

2010 年代初人工智能的进步（从[Word2Vec](https://en.wikipedia.org/wiki/Word2vec) 和 [GloVe](https://en.wikipedia.org/wiki/GloVe) [stanfordnlp-GloVe](https://github.com/stanfordnlp/GloVe) 开始）使我们能够构建这些对象的语义表示，其中它们被表示为笛卡尔空间中的点。假设一个视频映射到点 [0.1, -5.1, 7.55]，另一个视频映射到点 [5.3, -0.1, 2.7]。这些机器学习算法的神奇之处在于，这些表示的选择能够维护语义信息——两个视频越相似，它们的向量之间的距离就越小。

请注意，这些向量（或更专业地称为嵌入）不必是 3 维的 - 它们可以并且通常位于更高维的空间中（例如 128 维或 750 维）。而且距离也不需要是欧几里德距离 - 其他形式的距离（例如点积）也可以。无论哪种方式，重要的是它们之间的距离与其相似性相对应。

现在想象一下，我们可以访问所有 Youtube 视频的此类向量。我们如何找到与给定起始视频最相似的视频？简单的。循环遍历所有视频，计算它们之间的距离并选择距离最小的视频 - 也称为查找查询视频的“最近邻居”。这实际上会起作用。不过，正如您所猜测的，线性 O(N) 扫描的成本可能太高。因此，我们需要一种更快的亚线性方法来找到任何查询视频的最近邻居。这通常是不可能的——必须付出一些代价。

事实证明，在实际情况中，我们不需要找到最近的视频- 找到足够近的视频也可以。这就是近似最近邻搜索算法（也称为向量搜索）的用武之地。目标是亚线性（理想情况下以对数时间）找到空间中任何点的足够近的最近邻。那么如何解决呢？

## 如何找到近似最近邻居？

所有向量搜索算法背后的基本思想都是相同的——进行一些预处理来识别彼此足够接近的点（有点类似于构建索引）。在查询时，使用这个“索引”来排除大片点。并在不被排除的少量点内进行线性扫描。

然而，有很多方法可以实现这个简单的想法。存在几种最先进的向量搜索算法，例如[HNSW](https://github.com/nmslib/hnswlib)（一种连接邻近顶点并通过固定入口点维护长距离边的分层图，类似skiplist）。目前存在诸如 Facebook 的[FAISS](https://github.com/facebookresearch/faiss)之类的开源项目，以及诸如[Pinecone](https://www.pinecone.io/)，[Weaviate](https://weaviate.io/)，[zilliz-Milvus-knowhere](https://github.com/zilliztech/knowhere)等高可用性向量数据库的 PaaS 产品中。

在这篇文章中，我们将在给定的“N”点上构建一个简化的向量搜索索引，如下所示：

1. 随机取 2 个任意可用向量 A 和 B。
2. 计算这两个向量之间的中点，称为 C。
3. 构建一个穿过 C 并垂直于连接 A 和 B 的线段的超平面（类似于高维中的“线”）。
4. 将所有向量分类为超平面“上方”或“下方”，将可用向量分为 2 组。
5. 对于两个组中的每一个：如果组的大小高于可配置参数“最大节点大小”，则在该组上递归调用此过程以构建子树。否则，使用所有向量（或其唯一的 ID）构建单个叶节点

因此，我们使用这个随机过程来构建一棵树，其中每个内部节点都是超平面定义，左子树是超平面“下方”的所有向量，右子树是超平面“上方”的所有向量。向量集被连续递归地分割，直到叶节点包含不超过“最大节点大小”向量。考虑下图的例子，有五点：

![img](https://github.com/weedge/mypic/raw/master/oneday/vector-search-in-200-lines-of-rust/1.png)图 1：用随机超平面分割空间 



我们随机选择向量A1=(4,2)，B1=(5,7)。它们的中点是 (4.5,4.5)，我们通过中点构建一条垂直于线 (A1, B1) 的线。该线是 x + 5y=27（用蓝色绘制），这给了我们一组 2 个向量和一组 4 个向量。假设“最大节点大小”配置为 2。我们不进一步拆分第一组，而是选择后者构建新的（A2，B2）红色超平面等等。对大型数据集进行重复分割会将超空间分割成几个不同的区域，如下所示。

![img](https://github.com/weedge/mypic/raw/master/oneday/vector-search-in-200-lines-of-rust/2.png)图 2：许多超平面后的分段空间（来自 https://t.co/K0Xlht8GwQ，作者：[**Erik Bernhardsson**](https://twitter.com/bernhardsson))



这里的每个区域代表一个叶节点，并且这里的直觉是足够接近的点很可能最终出现在同一个叶节点中。因此，给定一个查询点，我们可以在对数时间内遍历树以找到它所属的叶子，并对该叶子中的所有（少量）点运行线性扫描。这显然不是万无一失的——实际上足够近的点完全有可能被超平面分开并最终彼此相距很远。但是这个问题可以通过构建不是一棵而是许多独立的树来解决 - 这样，如果两个点足够接近，它们更有可能位于至少某些树中的同一叶节点中。在查询时，我们遍历所有树以找到相关的叶节点，对所有叶节点的所有候选节点进行并集，

好吧，理论已经足够了。让我们开始编写一些代码，首先为下面的 Rust 中的 Vector 类型定义一些实用程序，用于点积、平均、散列和平方 L2 距离。感谢 Rust 良好的类型系统，我们传播泛型类型参数 N 来强制索引中的所有向量在编译时具有相同的维度。

```rust
#[derive(Eq, PartialEq, Hash)]
pub struct HashKey<const N: usize>([u32; N]);

#[derive(Copy, Clone)]
pub struct Vector<const N: usize>(pub [f32; N]);
impl<const N: usize> Vector<N> {
    pub fn subtract_from(&self, vector: &Vector<N>) -> Vector<N> {
        let mapped = self.0.iter().zip(vector.0).map(|(a, b)| b - a);
        let coords: [f32; N] = mapped.collect::<Vec<_>>().try_into().unwrap();
        return Vector(coords);
    }
    pub fn avg(&self, vector: &Vector<N>) -> Vector<N> {
        let mapped = self.0.iter().zip(vector.0).map(|(a, b)| (a + b) / 2.0);
        let coords: [f32; N] = mapped.collect::<Vec<_>>().try_into().unwrap();
        return Vector(coords);
    }
    pub fn dot_product(&self, vector: &Vector<N>) -> f32 {
        let zipped_iter = self.0.iter().zip(vector.0);
        return zipped_iter.map(|(a, b)| a * b).sum::<f32>();
    }
    pub fn to_hashkey(&self) -> HashKey<N> {
        // f32 in Rust doesn't implement hash. We use bytes to dedup. While it
        // can't differentiate ~16M ways NaN is written, it's safe for us
        let bit_iter = self.0.iter().map(|a| a.to_bits());
        let data: [u32; N] = bit_iter.collect::<Vec<_>>().try_into().unwrap();
        return HashKey::<N>(data);
    }
    pub fn sq_euc_dis(&self, vector: &Vector<N>) -> f32 {
        let zipped_iter = self.0.iter().zip(vector.0);
        return zipped_iter.map(|(a, b)| (a - b).powi(2)).sum();
    }
}
```

构建完这些核心实用程序后，我们还可以定义超平面的外观：

```rust
struct HyperPlane<const N: usize> {
    coefficients: Vector<N>,
    constant: f32,
}

impl<const N: usize> HyperPlane<N> {
    pub fn point_is_above(&self, point: &Vector<N>) -> bool {
        self.coefficients.dot_product(point) + self.constant >= 0.0
    }
}
```

接下来，让我们重点关注生成随机超平面并构建最近邻树森林。我们应该如何表示树中的点？

我们可以直接将 D 维向量存储在叶节点内。但这会显着增加大 D 的内存碎片（主要性能损失），并且当多棵树引用相同的向量时，还会在森林中创建重复的内存。相反，我们将向量存储在全局连续位置，并在叶节点处保存“usize”大小的索引（在 64 位系统上为 8 字节，而不是 4D，其中 f32 占用 4 字节）。以下是用于表示树的内部节点和叶节点的数据类型。

```rust
enum Node<const N: usize> {
    Inner(Box<InnerNode<N>>),
    Leaf(Box<LeafNode<N>>),
}
struct LeafNode<const N: usize>(Vec<usize>);
struct InnerNode<const N: usize> {
    hyperplane: HyperPlane<N>,
    left_node: Node<N>,
    right_node: Node<N>,
}
pub struct ANNIndex<const N: usize> {
    trees: Vec<Node<N>>,
    ids: Vec<i32>,
    values: Vec<Vector<N>>,
}
```

我们如何真正找到正确的超平面？

我们对向量 A 和 B 的两个唯一索引进行采样，计算 n = A - B，并找到 A 和 B 的中点 (point_on_plane)。超平面通过系数（向量 n）和常数（n 和 point_on_plane 的点积）结构有效存储为 n(x-x0) = nx - nx0。我们可以在任何向量和 n 之间执行点积，并减去常数以将向量放置在超平面“上方”或“下方”。由于树中的内部节点保存超平面定义，而叶节点保存向量 ID，因此我们可以使用 ADT 对树进行类型检查：

```rust
impl<const N: usize> ANNIndex<N> {
    fn build_hyperplane(
        indexes: &Vec<usize>,
        all_vecs: &Vec<Vector<N>>,
    ) -> (HyperPlane<N>, Vec<usize>, Vec<usize>) {
        let sample: Vec<_> = indexes
            .choose_multiple(&mut rand::thread_rng(), 2)
            .collect();
        // cartesian eq for hyperplane n * (x - x_0) = 0
        // n (normal vector) is the coefs x_1 to x_n
        let (a, b) = (*sample[0], *sample[1]);
        let coefficients = all_vecs[a].subtract_from(&all_vecs[b]);
        let point_on_plane = all_vecs[a].avg(&all_vecs[b]);
        let constant = -coefficients.dot_product(&point_on_plane);
        let hyperplane = HyperPlane::<N> {
            coefficients,
            constant,
        };
        let (mut above, mut below) = (vec![], vec![]);
        for &id in indexes.iter() {
            if hyperplane.point_is_above(&all_vecs[id]) {
                above.push(id)
            } else {
                below.push(id)
            };
        }
        return (hyperplane, above, below);
    }
}
```

因此，我们可以定义递归过程来基于索引时间“最大节点大小”构建树：

```rust
impl<const N: usize> ANNIndex<N> {
    fn build_a_tree(
        max_size: i32,
        indexes: &Vec<usize>,
        all_vecs: &Vec<Vector<N>>,
    ) -> Node<N> {
        if indexes.len() <= (max_size as usize) {
            return Node::Leaf(Box::new(LeafNode::<N>(indexes.clone())));
        }
        let (plane, above, below) = Self::build_hyperplane(indexes, all_vecs);
        let node_above = Self::build_a_tree(max_size, &above, all_vecs);
        let node_below = Self::build_a_tree(max_size, &below, all_vecs);
        return Node::Inner(Box::new(InnerNode::<N> {
            hyperplane: plane,
            left_node: node_below,
            right_node: node_above,
        }));
    }
}   
```

请注意，在两点之间构建超平面要求这两个点是唯一的 - 即我们必须在索引之前对向量集进行重复数据删除，因为该算法不允许重复。

因此整个索引（树木的森林）可以这样构建：

```rust
impl<const N: usize> ANNIndex<N> {
    fn deduplicate(
        vectors: &Vec<Vector<N>>,
        ids: &Vec<i32>,
        dedup_vectors: &mut Vec<Vector<N>>,
        ids_of_dedup_vectors: &mut Vec<i32>,
    ) {
        let mut hashes_seen = HashSet::new();
        for i in 1..vectors.len() {
            let hash_key = vectors[i].to_hashkey();
            if !hashes_seen.contains(&hash_key) {
                hashes_seen.insert(hash_key);
                dedup_vectors.push(vectors[i]);
                ids_of_dedup_vectors.push(ids[i]);
            }
        }
    }

    pub fn build_index(
        num_trees: i32,
        max_size: i32,
        vecs: &Vec<Vector<N>>,
        vec_ids: &Vec<i32>,
    ) -> ANNIndex<N> {
        let (mut unique_vecs, mut ids) = (vec![], vec![]);
        Self::deduplicate(vecs, vec_ids, &mut unique_vecs, &mut ids);
        // Trees hold an index into the [unique_vecs] list which is not
        // necessarily its id, if duplicates existed
        let all_indexes: Vec<usize> = (0..unique_vecs.len()).collect();
        let trees: Vec<_> = (0..num_trees)
            .map(|_| Self::build_a_tree(max_size, &all_indexes, &unique_vecs))
            .collect();
        return ANNIndex::<N> {
            trees,
            ids,
            values: unique_vecs,
        };
    }
}
```

###  查询时间

一旦建立了索引，我们如何使用它来搜索单个树上输入向量的 K 个近似最近邻？在非叶节点，我们存储超平面，因此我们可以从树的根开始并询问：“这个向量是在这个超平面之上还是之下？”。这可以通过 O(D) 和点积来计算。根据响应，我们可以递归搜索左子树或右子树，直到找到叶节点。请记住，叶节点最多存储“最大节点大小”向量，这些向量位于输入向量的近似邻域中（因为它们落在所有超平面下的超空间的同一分区中，见图 1(b)）。如果该叶节点处的向量索引数量超过 K，我们现在可以按与输入向量的 L2 距离对所有这些向量进行排序，并返回最接近的 K！

假设我们的索引导致平衡树，对于维度 D、向量数量 N 和最大节点大小 M << N，搜索需要 O(Dlog(N) + DM + Mlog(M)) - 这构成了平均最差情况 log(N)  次比较超平面以查找叶节点（即树的高度）；其中每次比较都会花费 O(D) 点积，计算 O(DM) 中叶节点中所有候选向量的 L2 度量；最后对它们进行排序以返回 O(Mlog(M)) 中的前 K 个。

但是，如果我们找到的叶节点的向量少于 K 个，会发生什么情况？如果最大节点大小太小或者超平面分割相对不均匀，子树中留下的向量很少，则这是可能的。为了解决这个问题，我们可以在树搜索中添加一些简单的回溯功能。例如，如果返回的候选数不够，我们可以在内部节点进行额外的递归调用来访问另一个分支。它可能是这样的：

```rust
impl<const N: usize> ANNIndex<N> {
    fn tree_result(
        query: Vector<N>,
        n: i32,
        tree: &Node<N>,
        candidates: &mut HashSet<usize>,
    ) -> i32 {
        // take everything in node, if still needed, take from alternate subtree
        match tree {
            Node::Leaf(box_leaf) => {
                let leaf_values = &(box_leaf.0);
                let num_candidates_found = min(n as usize, leaf_values.len());
                for i in 0..num_candidates_found {
                    candidates.insert(leaf_values[i]);
                }
                return num_candidates_found as i32;
            }
            Node::Inner(inner) => {
                let above = (*inner).hyperplane.point_is_above(&query);
                let (main, backup) = match above {
                    true => (&(inner.right_node), &(inner.left_node)),
                    false => (&(inner.left_node), &(inner.right_node)),
                };
                match Self::tree_result(query, n, main, candidates) {
                    k if k < n => {
                        k + Self::tree_result(query, n - k, backup, candidates)
                    }
                    k => k,
                }
            }
        }
    }
}
```


请注意，我们可以通过在子树中存储向量总数，以及直接指向每个内部节点的所有叶节点的指针列表来进一步优化递归调用，但为了简单起见，这里不这样做。

将此搜索扩展到树木森林很简单 - 只需从所有树中独立收集前 K 个候选者，按距离对它们进行排序，然后返回总体前 K 个匹配项。请注意，更多数量的树将具有线性高的内存占用和线性缩放的搜索时间，但可以导致更好的“更接近”的邻居，因为我们跨不同的树收集候选者。

```rust
impl<const N: usize> ANNIndex<N> {
    pub fn search_approximate(
        &self,
        query: Vector<N>,
        top_k: i32,
    ) -> Vec<(i32, f32)> {
        let mut candidates = HashSet::new();
        for tree in self.trees.iter() {
            Self::tree_result(query, top_k, tree, &mut candidates);
        }
        candidates
            .into_iter()
            .map(|idx| (idx, self.values[idx].sq_euc_dis(&query)))
            .sorted_by(|a, b| a.1.partial_cmp(&b.1).unwrap())
            .take(top_k as usize)
            .map(|(idx, dis)| (self.ids[idx], dis))
            .collect()
    }
}
```


这为我们提供了 200 行 Rust 的简单向量搜索索引！

为了说明的目的，这个实现相当简单——事实上，它是如此简单，以至于我们怀疑它一定比最先进的算法差得多（尽管在更广泛的方法中是相似的）。让我们做一些基准测试来证实我们的怀疑。

可以评估算法的延迟和质量。质量通常通过召回率来衡量 - 实际最近邻（从线性扫描获得）的百分比，也是通过近似最近邻搜索获得的。有时，返回的结果在技术上并不在前 K 中，但它们非常接近实际的前 K，因此并不重要 - 为了量化这一点，我们还可以查看邻居的平均欧几里德距离，并将其与暴力平均距离进行比较强制搜索。

测量延迟很简单 - 我们可以查看执行查询所需的时间（我们通常对索引构建延迟不太感兴趣）。

所有基准测试结果均在配备 2.3 GHz 四核 Intel Core i5 处理器的单设备 CPU 上运行，使用 999,994 个 Wiki 数据 FastText 嵌入 ( [https://dl.fbaipublicfiles.com/fasttext/vectors-english/wiki-news -300d-1M.vec.zip](https://dl.fbaipublicfiles.com/fasttext/vectors-english/wiki-news-300d-1M.vec.zip) ) 300 维。我们将所有查询的“top K”设置为 20。

作为参考，我们将 FAISS HNSW 索引 (ef_search=16、ef_construction=40、max_node_size=15) 与 Rust 索引的小版本 (num_trees=3、max_node_size=15) 进行比较。我们在 Rust 中实现了详尽的搜索，而 FAISS 库有 HNSW 的 C++ 源代码。原始延迟数低，增强了近似搜索的优势：

| 算法              | 延迟 Latency | QPS     |
| ----------------- | ------------ | ------- |
| Exhaustive Search | 675.25ms     | 1.48    |
| FAISS HNSW Index  | 355.36μs     | 2814.05 |
| Custom Rust Index | 112.02μs     | 8926.98 |

两种近似最近邻方法的速度都快了三个数量级，这很好。与 HNSW 相比，我们的 Rust 实现在这个微基准测试中速度快了 3 倍。分析质量时，直观地考虑 prompt “river” 返回的前 10 个最近邻。

| Exhaustive Search |                        | FAISS HNSW Index |                        | Custom Rust Index |                        |
| ----------------- | ---------------------- | ---------------- | ---------------------- | ----------------- | ---------------------- |
| **Word**          | **Euclidean Distance** | **Word**         | **Euclidean Distance** | **Word**          | **Euclidean Distance** |
| river             | 0                      | river            | 0                      | river             | 0                      |
| River             | 1.39122                | River            | 1.39122                | creek             | 1.63744                |
| rivers            | 1.47646                | river-           | 1.58342                | river.            | 1.73224                |
| river-            | 1.58342                | swift-flowing    | 1.62413                | lake              | 1.75655                |
| swift-flowing     | 1.62413                | flood-swollen    | 1.63798                | sea               | 1.87368                |
| creek             | 1.63744                | river.The        | 1.68156                | up-river          | 1.92088                |
| flood-swollen     | 1.63798                | river-bed        | 1.68510                | shore             | 1.92266                |
| river.The         | 1.68156                | unfordable       | 1.69245                | brook             | 2.01973                |
| river-bed         | 1.68510                | River-           | 1.69512                | hill              | 2.03419                |
| unfordable        | 1.69245                | River.The        | 1.69539                | pond              | 2.04376                |

或者，考虑一下prompt  “war”。

| Exhaustive Search |                        | FAISS HNSW Index |                        | Custom Rust Index |                        |
| ----------------- | ---------------------- | ---------------- | ---------------------- | ----------------- | ---------------------- |
| **Word**          | **Euclidean Distance** | **Word**         | **Euclidean Distance** | **Word**          | **Euclidean Distance** |
| war               | 0                      | war              | 0                      | war               | 0                      |
| war--             | 1.38416                | war--            | 1.38416                | war--             | 1.38416                |
| war--a            | 1.44906                | war--a           | 1.44906                | wars              | 1.45859                |
| wars              | 1.45859                | wars             | 1.45859                | quasi-war         | 1.59712                |
| war--and          | 1.45907                | war--and         | 1.45907                | war-footing       | 1.69175                |
| war.It            | 1.46991                | war.It           | 1.46991                | check-mate        | 1.74982                |
| war.In            | 1.49632                | war.In           | 1.49632                | ill-begotten      | 1.75498                |
| unwinable         | 1.51296                | unwinable        | 1.51296                | subequent         | 1.76617                |
| war.And           | 1.51830                | war.And          | 1.51830                | humanitary        | 1.77464                |
| hostilities       | 1.54783                | Iraw             | 1.54906                | execution         | 1.77992                |

对于整个 999,994 个单词的语料库，我们还可视化了 HNSW 和自定义 Rust 索引下每个单词到其顶部 K=20 个近似邻居的平均欧几里得距离的分布：

![img](https://github.com/weedge/mypic/raw/master/oneday/vector-search-in-200-lines-of-rust/3.png)

最先进的 HNSW 指数确实提供了比我们的示例索引相对更近的邻居，平均距离和中位距离分别为 1.31576 和 1.20230（与我们的示例索引的 1.47138 和 1.35620 相比）。在随机的 10,000 大小的语料库子集上，HNSW 对前 K=20 的召回率为 58.2%，而我们的示例索引针对不同的配置（如前所述，树的数量较多）产生了不同的召回率（从 11.465% 到 23.115%）提供更高的召回率）：

| num_trees | max_node_size | Average runtime | QPS  | Recall  |
| --------- | ------------- | --------------- | ---- | ------- |
| 3         | 5             | 129.48μs        | 7723 | 0.11465 |
| 3         | 15            | 112.02μs        | 8297 | 0.11175 |
| 3         | 30            | 114.48μs        | 8735 | 0.09265 |
| 9         | 5             | 16.77ms         | 60   | 0.22095 |
| 9         | 15            | 1.54ms          | 649  | 0.20985 |
| 9         | 30            | 370.80μs        | 2697 | 0.16835 |
| 15        | 5             | 35.45ms         | 28   | 0.29825 |
| 15        | 15            | 7.34ms          | 136  | 0.28520 |
| 15        | 30            | 2.19ms          | 457  | 0.23115 |

## 为什么FANN这么快？

正如您在上面的数字中看到的，虽然 FANN 算法在质量上无法与最先进的算法竞争，但它至少相当快。为什么会这样？

老实说，当我们构建这个时，我们得意忘形并开始进行性能优化只是为了好玩。以下是一些最显着的优化：

- 将文档重复数据删除卸载到索引冷路径。通过索引而不是浮点数组引用向量可以显着加快搜索速度，因为跨树查找唯一候选者需要散列 8 字节索引（而不是 300 维 f32 数据）。
- 在将项目添加到全局候选列表之前，急切地散列并查找唯一向量，并通过递归搜索调用之间的可变引用传递数据，以避免跨堆栈帧和堆栈帧内进行复制。
- 将 N 作为通用类型参数传递，这允许将 300 维数据作为 300 长度的 f32 数组（而不是可变长度向量类型）进行类型检查，以提高缓存局部性并减少内存占用（向量具有堆上数据的附加重定向级别）。
- 我们还怀疑 Rust 编译器正在对内部操作（例如点积）进行向量化，但我们没有检查。

## 更多现实世界的考虑

此示例跳过了几个对于生产向量搜索至关重要的注意事项：(**注**：单实例 cpu指令集优化 向量矩阵如axv2，SIMD + OpenMP； 分布式数据存储扩展 RPC + 分布式一致性协议)

- 当搜索涉及多棵树时进行并行化。我们可以并行化，而不是按顺序收集候选者，因为每棵树访问不同的内存 - 每棵树可以在单独的线程上运行，其中候选者通过消息沿着通道连续发送到主进程。线程可以在索引时生成，并通过虚拟搜索（使树的部分位于缓存中）进行预热，以减少搜索开销。搜索将不再随树的数量线性缩放。
- 大型树可能不适合 RAM，需要有效的方法从磁盘读取 - 某些子图可能需要位于磁盘上，并且算法旨在允许搜索，同时最大限度地减少文件 I/O。
- 更进一步，如果树不适合实例的磁盘，我们需要跨实例分布子树，并且如果数据在本地不可用，则递归搜索调用会触发一些 RPC 请求。
- 该树涉及许多内存重定向（基于指针的树不适合 L1 缓存）。平衡树可以用数组很好地编写，但我们的树只能用随机超平面接近平衡——我们可以为树使用新的数据结构吗？
- 当新数据被动态索引时（可能需要对大树进行重新分片），上述问题的解决方案也应该适用。如果特定的索引顺序导致树高度不平衡，是否应该重新创建树？

## 结论

如果你到达这里，恭喜你！您刚刚看到了大约 200 行 Rust 中的简单向量搜索，以及我们对行星规模应用程序的向量搜索的漫谈。我们希望您喜欢阅读本文，并随时访问[https://github.com/fennel-ai/fann](https://github.com/fennel-ai/fann)的源代码。

(**注**：实验性质，运行下benchmark.sh 对比faiss hnsw了解下原理, faiss hnsw可以参数调优， LSH 可用于生产环境的库可参考[FALCONN-LIB](https://github.com/FALCONN-LIB/FALCONN)实现, 对 K，L,  T 调优，参考[Locality-Sensitive Hashing: a Primer](https://github.com/FALCONN-LIB/FALCONN/wiki/LSH-Primer), 另外一个[lsh-rs](https://github.com/ritchie46/lsh-rs) 库也可以参考，[thistle](https://github.com/bwindsor22/thistle) 则参考了[lsh-rs](https://github.com/ritchie46/lsh-rs) 和 [hnswlib-rs](https://github.com/jean-pierreBoth/hnswlib-rs)的实现，不过都不支持动态更新索引）



# reference

1. https://en.wikipedia.org/wiki/Locality-sensitive_hashing
2. https://fennel.ai/blog/vector-search-in-200-lines-of-rust/
3. https://erikbern.com/2015/09/24/nearest-neighbor-methods-vector-models-part-1
4. https://erikbern.com/2015/10/01/nearest-neighbors-and-vector-models-part-2-how-to-search-in-high-dimensional-spaces.html
5. https://erikbern.com/2016/06/02/approximate-nearest-news
6. https://www.ritchievink.com/blog/2020/04/07/sparse-neural-networks-and-hash-tables-with-locality-sensitive-hashing/ [lsh-rs](https://github.com/ritchie46/lsh-rs)
7. [arxiv paper - Thistle: A Vector Database in Rust](https://arxiv.org/pdf/2303.16780.pdf) [thistle](https://github.com/bwindsor22/thistle) 
8. https://github.com/FALCONN-LIB/FALCONN/wiki/LSH-Primer
9. [Falconn++: A Locality-sensitive Filtering Approach for Approximate Nearest Neighbor Search](https://arxiv.org/pdf/2206.01382.pdf)

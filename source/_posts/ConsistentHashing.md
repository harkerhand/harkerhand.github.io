---
title: 一致性哈希
date: 2026-03-03 14:10:20
tags: 密码学
categories: 经验分享
index_img: ./ConsistentHashing/hash.png
banner_img:
---

# 背景

假设在分布式负载均衡场景中，对于一些输入，我们希望尽可能均匀的将其分配到 n 个不同的节点。常规的哈希算法通常使用简单的哈希函数与取模来得到下标。
$$
\text{index} = hash(input) \ \% \ n
$$
但当节点扩缩容时，即 n 的大小变化时，一个显而易见的问题是。对于相同的 input，得到的 index 是不同的。

从数学上来看，即使 index 发生变化，这样的算法在负载均衡上的表现依然是优秀的。但如果节点会有类似缓存的技术来加快计算，这样的index变化就是不可接受的，它会导致**几乎所有的缓存失效**。

# 一致性哈希

一致性哈希的目的是解决分布式系统的数据分区问题：当分布式集群移除或者添加一个服务器时，必须尽可能小地改变已存在的服务请求与处理请求服务器之间的映射关系。

简单来说就是尽可能的复用缓存，尽可能对于同一个input，得到同一个index。

## 原理

一致性哈希本质上也是一种取模算法。只不过它的模数是一个极大的数，一般是 2^32 - 1。

暂且认为一致性哈希用了一个超大的桶，来规避了取模问题。现在对于相同的input，一定会得到相同的index。但这个index是用来索引节点的，现在应该如何索引？

解决方案是：对节点标记也进行哈希，将节点放入桶中。在拿到input的index后，向后遍历，直到遇到第一个节点。

在这样的前提下，想象这个超大的桶是一个环，各个节点将这个环分割，每个节点负责环上的一个弧。

- 当插入一个节点时，其影响范围只有被其切割的那一段弧。
- 当删除一个节点时，其影响范围只有其负责的那一段弧。

理论上，每次扩缩容，造成的影响只有 1/N，其中 N 是缓存总数。

## 细节

原理是这样，那实现上呢？

第一，很显然这些桶是不可能预分配内存的，太大了！但这个环，本质上也是一张（整数，节点标记）的表，同时，我们上文提到了有向后遍历的需求，那这个表就可以使用排序树（一般是红黑树）来实现，不仅解决了空间问题，还可以用二分来优化遍历的时间。

第二，当节点数量不多或者我运气很差时，节点可能会聚集在这个环的某一个区间内，造成极度的负载不均衡。一个朴素的想法是，只要节点数量够多，把这个环切分的足够细，就可以极大缓解这个不均衡的问题。但实际的节点数量就那么几个，这时候就可以采用虚拟节点的方式，我们可以将一个节点视为数十个甚至数百个虚拟节点，用这些虚拟节点来哈希做，最后放入桶中的依然是同一个节点。

第三，物理节点撞了怎么办。当虚拟节点足够多时，撞一两个其实不会太影响负载均衡的结果。还不放心的话，可以使用64位哈希。还不放心的话，就只能祈祷自己不是倒霉蛋了。

## 可能的优化

- 当节点变动不频繁的时候，可以不用排序树。直接 `vector<(hash, Node)>`，初始化结束以及增删节点时排序，查找可以使用二分，可以更好的利用CPU缓存。

## 复杂度分析

假设：

- 物理节点（个） N
- 虚拟节点（个/物理节点） V

总节点：
$$
M = N * V
$$
操作复杂度：

| 操作 | 时间复杂度 |
| ---- | ---------- |
| 查找 | O(log M)   |
| 插入 | O(V log M) |
| 删除 | O(V log M) |

# 代码实现

## C++

```c++
template<typename Node>
class ConsistentHash {
public:
    using node_ptr = std::shared_ptr<Node>;
    using hash_type = std::uint64_t;

    explicit ConsistentHash(std::size_t virtual_nodes = 100)
        : virtual_node_count_(virtual_nodes) {}

    // 添加物理节点
    void add_node(const node_ptr& node) {
        if (!node) {
            throw std::invalid_argument("node is null");
        }

        std::unique_lock lock(mutex_);

        for (std::size_t i = 0; i < virtual_node_count_; ++i) {
            auto vnode_key = make_vnode_key(*node, i);
            ring_.emplace(hash(vnode_key), node);
        }
    }

    // 删除物理节点
    void remove_node(const node_ptr& node) {
        if (!node) return;

        std::unique_lock lock(mutex_);

        for (std::size_t i = 0; i < virtual_node_count_; ++i) {
            auto vnode_key = make_vnode_key(*node, i);
            ring_.erase(hash(vnode_key));
        }
    }

    // 查找 key 对应的节点
    [[nodiscard]]
    node_ptr get_node(const std::string& key) const {
        std::shared_lock lock(mutex_);

        if (ring_.empty()) {
            throw std::runtime_error("Hash ring is empty");
        }

        auto h = hash(key);
        auto it = ring_.lower_bound(h);

        if (it == ring_.end()) {
            it = ring_.begin();  // 回环
        }

        return it->second;
    }

    [[nodiscard]]
    std::size_t node_count() const {
        std::shared_lock lock(mutex_);
        return ring_.size() / virtual_node_count_;
    }

private:
    static hash_type hash(const std::string& key) {
        return std::hash<std::string>{}(key);
    }

    static std::string make_vnode_key(const Node& node, std::size_t index) {
        return node.id() + "#" + std::to_string(index);
    }

private:
    std::size_t virtual_node_count_;
    std::map<hash_type, node_ptr> ring_;
    mutable std::shared_mutex mutex_;
};
```

## Rust

```rust
pub struct ConsistentHash<T> {
    virtual_nodes: usize,
    ring: RwLock<BTreeMap<u64, Arc<T>>>,
}

impl<T> ConsistentHash<T>
where
    T: Hash + Clone,
{
    pub fn new(virtual_nodes: usize) -> Self {
        Self {
            virtual_nodes,
            ring: RwLock::new(BTreeMap::new()),
        }
    }

    pub fn add_node(&self, node: Arc<T>) {
        let mut ring = self.ring.write().unwrap();

        for i in 0..self.virtual_nodes {
            let hash = self.hash_virtual_node(&node, i);
            ring.insert(hash, Arc::clone(&node));
        }
    }

    pub fn remove_node(&self, node: &T) {
        let mut ring = self.ring.write().unwrap();

        for i in 0..self.virtual_nodes {
            let hash = self.hash_virtual_node_value(node, i);
            ring.remove(&hash);
        }
    }

    pub fn get_node<K: Hash>(&self, key: &K) -> Option<Arc<T>> {
        let ring = self.ring.read().unwrap();

        if ring.is_empty() {
            return None;
        }

        let hash = self.hash(key);

        // 找到第一个 >= hash 的节点
        let mut iter = ring.range(hash..);

        if let Some((_, node)) = iter.next() {
            return Some(Arc::clone(node));
        }

        // 回环
        ring.values().next().map(Arc::clone)
    }

    fn hash<K: Hash>(&self, key: &K) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }

    fn hash_virtual_node(&self, node: &Arc<T>, index: usize) -> u64 {
        let mut hasher = DefaultHasher::new();
        node.hash(&mut hasher);
        index.hash(&mut hasher);
        hasher.finish()
    }

    fn hash_virtual_node_value(&self, node: &T, index: usize) -> u64 {
        let mut hasher = DefaultHasher::new();
        node.hash(&mut hasher);
        index.hash(&mut hasher);
        hasher.finish()
    }
}
```


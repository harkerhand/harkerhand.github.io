---
title: Rust数据结构源码实现
date: 2026-03-15 07:40:45
tags: 
- C++
- Rust
categories: TechMagic
index_img:
banner_img:
---



# 解析Rust部分数据结构的源码实现

需要注意的是，为了方便，对源码的一些实现有改写

## Vec

从单纯的数据结构设计上，大概如下

```rust
pub struct Vec<T, A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
pub(crate) struct RawVec<T, A: Allocator = Global> {
    inner: RawVecInner<A>,
    _marker: PhantomData<T>,
}
struct RawVecInner<A: Allocator = Global> {
    ptr: Unique<u8>,
    cap: Cap,
    alloc: A,
}
```

其中的 `_marker: PhantomData<T>` 是在**逻辑上**标注对 T 的所有权，方便编译器做一些检查；`Cap` 会根据不通平台条件编译为不同范围的usize

### new

```rust
impl<T> Vec<T> {
    pub const fn new() -> Self {
        Vec { buf: RawVec::new(), len: 0 }
    }
}
impl<T> RawVec<T, Global> {
    pub(crate) const fn new() -> Self {
        Self::new_in(Global)
    }
    pub(crate) const fn new_in(alloc: A) -> Self {
        // Check assumption made in `current_memory`
        const { assert!(T::LAYOUT.size() % T::LAYOUT.align() == 0) };
        Self { inner: RawVecInner::new_in(alloc, Alignment::of::<T>()), _marker: PhantomData }
    }
}
impl<A: Allocator> RawVecInner<A> {
    const fn new_in(alloc: A, align: Alignment) -> Self {
        let ptr = Unique::from_non_null(NonNull::without_provenance(align.as_nonzero()));
        // `cap: 0` means "unallocated". zero-sized types are ignored.
        Self { ptr, cap: ZERO_CAP, alloc }
    }
}
```

这里有一些有意思的，首先是会有一个对齐的检查，随后创建一个满足对齐的悬垂指针，所以其实还没有分配堆内存。

### with_capacity

```rust
impl<T> Vec<T> {
    pub const fn with_capacity(capacity: usize) -> Self {
        Self::with_capacity_in(capacity, Global)
    }
    pub fn with_capacity_in(capacity: usize, alloc: A) -> Self {
        Vec { buf: RawVec::with_capacity_in(capacity, alloc), len: 0 }
    }
}
impl<T> RawVec<T, Global> {
    pub(crate) fn with_capacity_in(capacity: usize, alloc: A) -> Self {
        Self {
            inner: RawVecInner::with_capacity_in(capacity, alloc, T::LAYOUT),
            _marker: PhantomData,
        }
    }
}
impl<A: Allocator> RawVecInner<A> {
    fn with_capacity_in(capacity: usize, alloc: A, elem_layout: Layout) -> Self {
        match Self::try_allocate_in(capacity, AllocInit::Uninitialized, alloc, elem_layout) {
            Ok(this) => {
                unsafe {
                    // Make it more obvious that a subsequent Vec::reserve(capacity) will not allocate.
                    hint::assert_unchecked(!this.needs_to_grow(0, capacity, elem_layout));
                }
                this
            }
            Err(err) => handle_error(err),
        }
    }
	fn try_allocate_in(
        capacity: usize,
        init: AllocInit,
        alloc: A,
        elem_layout: Layout,
    ) -> Result<Self, TryReserveError> {
        // We avoid `unwrap_or_else` here because it bloats the amount of
        // LLVM IR generated.
        let layout = match layout_array(capacity, elem_layout) {
            Ok(layout) => layout,
            Err(_) => return Err(CapacityOverflow.into()),
        };

        // Don't allocate here because `Drop` will not deallocate when `capacity` is 0.
        if layout.size() == 0 {
            return Ok(Self::new_in(alloc, elem_layout.alignment()));
        }

        let result = match init {
            AllocInit::Uninitialized => alloc.allocate(layout),
            #[cfg(not(no_global_oom_handling))]
            AllocInit::Zeroed => alloc.allocate_zeroed(layout),
        };
        let ptr = match result {
            Ok(ptr) => ptr,
            Err(_) => return Err(AllocError { layout, non_exhaustive: () }.into()),
        };

        // Allocators currently return a `NonNull<[u8]>` whose length
        // matches the size requested. If that ever changes, the capacity
        // here should change to `ptr.len() / size_of::<T>()`.
        Ok(Self {
            ptr: Unique::from(ptr.cast()),
            cap: unsafe { Cap::new_unchecked(capacity) },
            alloc,
        })
    }
}
```

前面就是一些封装，具体后面的部分：`AllocInit::Uninitialized` 规定使用非初始化的内存分配，会快一点；随后计算所需的内存大小并分配，将得到的内存地址做包装后返回。

## push

```rust
impl<T> Vec<T> {
    pub fn push(&mut self, value: T) {
        let _ = self.push_mut(value);
    }
    pub fn push_mut(&mut self, value: T) -> &mut T {
        // Inform codegen that the length does not change across grow_one().
        let len = self.len;
        // This will panic or abort if we would allocate > isize::MAX bytes
        // or if the length increment would overflow for zero-sized types.
        if len == self.buf.capacity() {
            self.buf.grow_one();
        }
        unsafe {
            let end = self.as_mut_ptr().add(len);
            ptr::write(end, value);
            self.len = len + 1;
            // SAFETY: We just wrote a value to the pointer that will live the lifetime of the reference.
            &mut *end
        }
    }
    pub const fn as_mut_ptr(&mut self) -> *mut T {
        // We shadow the slice method of the same name to avoid going through
        // `deref_mut`, which creates an intermediate reference.
        self.buf.ptr()
    }
}
impl<T> RawVec<T, Global> {
    pub(crate) const fn ptr(&self) -> *mut T {
        self.inner.ptr()
    }
    pub(crate) fn grow_one(&mut self) {
        // SAFETY: All calls on self.inner pass T::LAYOUT as the elem_layout
        unsafe { self.inner.grow_one(T::LAYOUT) }
    }
}
impl<A> RawVecInner<A> {
	unsafe fn grow_one(&mut self, elem_layout: Layout) {
        // SAFETY: Precondition passed to caller
        if let Err(err) = unsafe { self.grow_amortized(self.cap.as_inner(), 1, elem_layout) } {
            handle_error(err);
        }
    }
    unsafe fn grow_amortized(
        &mut self,
        len: usize,
        additional: usize,
        elem_layout: Layout,
    ) -> Result<(), TryReserveError> {
        // This is ensured by the calling contexts.
        debug_assert!(additional > 0);

        if elem_layout.size() == 0 {
            // Since we return a capacity of `usize::MAX` when `elem_size` is
            // 0, getting to here necessarily means the `RawVec` is overfull.
            return Err(CapacityOverflow.into());
        }

        // Nothing we can really do about these checks, sadly.
        let required_cap = len.checked_add(additional).ok_or(CapacityOverflow)?;

        // This guarantees exponential growth. The doubling cannot overflow
        // because `cap <= isize::MAX` and the type of `cap` is `usize`.
        let cap = cmp::max(self.cap.as_inner() * 2, required_cap);
        let cap = cmp::max(min_non_zero_cap(elem_layout.size()), cap);

        // SAFETY:
        // - cap >= len + additional
        // - other preconditions passed to caller
        let ptr = unsafe { self.finish_grow(cap, elem_layout)? };

        // SAFETY: `finish_grow` would have failed if `cap > isize::MAX`
        unsafe { self.set_ptr_and_cap(ptr, cap) };
        Ok(())
    }
    unsafe fn finish_grow(
        &self,
        cap: usize,
        elem_layout: Layout,
    ) -> Result<NonNull<[u8]>, TryReserveError> {
        let new_layout = layout_array(cap, elem_layout)?;

        let memory = if let Some((ptr, old_layout)) = unsafe { self.current_memory(elem_layout) } {
            // FIXME(const-hack): switch to `debug_assert_eq`
            debug_assert!(old_layout.align() == new_layout.align());
            unsafe {
                // The allocator checks for alignment equality
                hint::assert_unchecked(old_layout.align() == new_layout.align());
                self.alloc.grow(ptr, old_layout, new_layout)
            }
        } else {
            self.alloc.allocate(new_layout)
        };

        // FIXME(const-hack): switch back to `map_err`
        match memory {
            Ok(memory) => Ok(memory),
            Err(_) => Err(AllocError { layout: new_layout, non_exhaustive: () }.into()),
        }
    }
}
```

因为内存是未初始化的，所以会使用 `ptr::write` 直接向内存写数据。我们具体关注扩容逻辑，
$$
new\_cap = max(old\_cap * 2, old\_cap + needed, min_non_zero_cap)
$$
指数倍增，并且会使用类型大小来选择保底扩容大小；具体的内存分配部分，使用alloc提供的能力，尽可能的进行原地扩展（不保证）。

### get

有趣的是，rust标准库对切片实现了 `get`，而 `Vec` 又可以自动 `Deref` 为切片，所以复用了这一部分能力

### remove

读出其中的值，并批量移动内存数据

### insert

先判断扩容，然后移动，最后写入值

## LinkedList

```rust
pub struct LinkedList<T, A: Allocator = Global> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    alloc: A,
    marker: PhantomData<Box<Node<T>, A>>,
}
struct Node<T> {
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
    element: T,
}
```

普普通通毫无亮点口牙！

### new

```rust
impl<T> LinkedList<T> {
    pub const fn new() -> Self {
        LinkedList { head: None, tail: None, len: 0, alloc: Global, marker: PhantomData }
    }
}
```

太棒了，简洁易懂

### push_back

```rust
impl<T> LinkedList<T> {
    pub fn push_back(&mut self, elt: T) {
        let _ = self.push_back_mut(elt);
    }
    pub fn push_back_mut(&mut self, elt: T) -> &mut T {
        let mut node =
            Box::into_non_null_with_allocator(Box::new_in(Node::new(elt), &self.alloc)).0;
        // SAFETY: node is a unique pointer to a node in self.alloc
        unsafe {
            self.push_back_node(node);
            &mut node.as_mut().element
        }
    }
    unsafe fn push_back_node(&mut self, node: NonNull<Node<T>>) {
        // This method takes care not to create mutable references to whole nodes,
        // to maintain validity of aliasing pointers into `element`.
        unsafe {
            (*node.as_ptr()).next = None;
            (*node.as_ptr()).prev = self.tail;
            let node = Some(node);

            match self.tail {
                None => self.head = node,
                // Not creating new mutable (unique!) references overlapping `element`.
                Some(tail) => (*tail.as_ptr()).next = node,
            }

            self.tail = node;
            self.len += 1;
        }
    }
}
```

在堆上创建 `Node<T>`，随后将带有所有权的 `Box` 转化为 `NonNull`；`push_back_node` 更是极为显然的逻辑。

### pop_back

其实可以略了

## HashMap

```rust
pub struct HashMap<
    K,
    V,
    S = RandomState,
    A: Allocator = Global,
> {
    base: base::HashMap<K, V, S, A>,
}
pub struct HashMap<K, V, S = DefaultHashBuilder, A: Allocator = Global> {
    pub(crate) hash_builder: S,
    pub(crate) table: RawTable<(K, V), A>,
}
pub struct RawTable<T, A: Allocator = Global> {
    table: RawTableInner,
    alloc: A,
    // Tell dropck that we own instances of T.
    marker: PhantomData<T>,
}
struct RawTableInner {
    // Mask to get an index from a hash value. The value is one less than the
    // number of buckets in the table.
    bucket_mask: usize,
    // [Padding], T_n, ..., T1, T0, C0, C1, ...
    //                              ^ points here
    ctrl: NonNull<u8>,
    // Number of elements that can be inserted before we need to grow the table
    growth_left: usize,
    // Number of elements in the table, only really used by len()
    items: usize,
}
```

| **字段**      | **作用**       | **深度解析**                                                 |
| ------------- | -------------- | ------------------------------------------------------------ |
| `bucket_mask` | 快速取模       | 桶的数量永远是 $2^n$，所以 `hash & bucket_mask` 效果等同于 `hash % capacity`，但位运算速度极快。 |
| `growth_left` | 扩容阈值       | 记录还能插入多少元素。当它降到 0 时，触发 `rehash` 扩容，防止哈希冲突过多导致性能下降。 |
| `ctrl`        | 标记控制区起点 | 使用SIMD加速哈希匹配判断                                     |

### with_capacity

```rust
impl RawTableInner {
    fn fallible_with_capacity<A>(
        alloc: &A,
        table_layout: TableLayout,
        capacity: usize,
        fallibility: Fallibility,
    ) -> Result<Self, TryReserveError>
    where
        A: Allocator,
    {
        if capacity == 0 {
            Ok(Self::NEW)
        } else {
            // SAFETY: We checked that we could successfully allocate the new table, and then
            // initialized all control bytes with the constant `Tag::EMPTY` byte.
            unsafe {
                let buckets = capacity_to_buckets(capacity, table_layout)
                    .ok_or_else(|| fallibility.capacity_overflow())?;

                let mut result =
                    Self::new_uninitialized(alloc, table_layout, buckets, fallibility)?;
                // SAFETY: We checked that the table is allocated and therefore the table already has
                // `self.bucket_mask + 1 + Group::WIDTH` number of control bytes (see TableLayout::calculate_layout_for)
                // so writing `self.num_ctrl_bytes() == bucket_mask + 1 + Group::WIDTH` bytes is safe.
                result.ctrl_slice().fill_empty();

                Ok(result)
            }
        }
    }
    unsafe fn new_uninitialized<A>(
        alloc: &A,
        table_layout: TableLayout,
        mut buckets: usize,
        fallibility: Fallibility,
    ) -> Result<Self, TryReserveError>
    where
        A: Allocator,
    {
        debug_assert!(buckets.is_power_of_two());

        // Avoid `Option::ok_or_else` because it bloats LLVM IR.
        let (layout, mut ctrl_offset) = match table_layout.calculate_layout_for(buckets) {
            Some(lco) => lco,
            None => return Err(fallibility.capacity_overflow()),
        };

        let ptr: NonNull<u8> = match do_alloc(alloc, layout) {
            Ok(block) => {
                // The allocator can't return a value smaller than was
                // requested, so this can be != instead of >=.
                if block.len() != layout.size() {
                    // Utilize over-sized allocations.
                    let x = maximum_buckets_in(block.len(), table_layout, Group::WIDTH);
                    debug_assert!(x >= buckets);
                    // Calculate the new ctrl_offset.
                    let (oversized_layout, oversized_ctrl_offset) =
                        match table_layout.calculate_layout_for(x) {
                            Some(lco) => lco,
                            None => unsafe { hint::unreachable_unchecked() },
                        };
                    debug_assert!(oversized_layout.size() <= block.len());
                    debug_assert!(oversized_ctrl_offset >= ctrl_offset);
                    ctrl_offset = oversized_ctrl_offset;
                    buckets = x;
                }

                block.cast()
            }
            Err(_) => return Err(fallibility.alloc_err(layout)),
        };

        // SAFETY: null pointer will be caught in above check
        let ctrl = NonNull::new_unchecked(ptr.as_ptr().add(ctrl_offset));
        Ok(Self {
            ctrl,
            bucket_mask: buckets - 1,
            items: 0,
            growth_left: bucket_mask_to_capacity(buckets - 1),
        })
    }
}
```

内存刚申请下来是脏的，需要先把所有的控制字节设置为`0xFF`；在申请内存时，首先精确得到layout和控制区偏移量，随后如果分配器给出的内存超过实际申请，会进行自动扩容来利用这部分空间；最后返回的ctrl指针是控制区的起点。

### insert

```rust
impl RawTableInner {
    unsafe fn find_or_find_insert_index_inner(
        &self,
        hash: u64,
        eq: &mut dyn FnMut(usize) -> bool,
    ) -> Result<usize, usize> {
        let mut insert_index = None;
        let tag_hash = Tag::full(hash);
        let mut probe_seq = self.probe_seq(hash);
        loop {
            let group = unsafe { Group::load(self.ctrl(probe_seq.pos)) };
            for bit in group.match_tag(tag_hash) {
                let index = (probe_seq.pos + bit) & self.bucket_mask;
                if likely(eq(index)) {
                    return Ok(index);
                }
            }
            if likely(insert_index.is_none()) {
                insert_index = self.find_insert_index_in_group(&group, &probe_seq);
            }
            if let Some(insert_index) = insert_index {
                if likely(group.match_empty().any_bit_set()) {
                    unsafe {
                        return Err(self.fix_insert_index(insert_index));
                    }
                }
            }
            probe_seq.move_next(self.bucket_mask);
        }
    }
}
```

这部分是最有趣的，常规的算法会在hash后拿到桶的下标，并遍历匹配，这其中频繁的内存读取消耗巨大；而源码的算法是先通过控制位与哈希值的高七位做匹配，这可以借助SIMD实现批量匹配，只有匹配到的节点才会去尝试检查。太酷了！

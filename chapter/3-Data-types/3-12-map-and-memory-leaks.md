## 3.12 映射和内存泄漏

在 Go 中使⽤ map 时，我们应该了解⼀些关于 map 如何增⻓和缩⼩的重要特征。让我们深⼊研究这些问题，以防⽌导致内存泄漏的常⻅错误。

⾸先，要查看此问题的具体⽰例，让我们设计⼀个场景，我们将使⽤以下 map ：

```go
m := make(map[int][128]byte)
```

m 的每个值都是⼀个 128 字节的数组。

我们将执⾏以下场景：

* 分配⼀张空 map 
* 添加⼀百万个元素
* 删除所有元素并运⾏ GC

在每⼀步之后，我们想要打印堆的⼤⼩（这次使⽤ MB）来看看这个例⼦在内存⽅⾯的表现：

```go
// Init
n := 1_000_000
m := make(map[int][128]byte)
printAlloc()
// Add elements
for i := 0; i < n; i++ {
	m[i] = randBytes()
}
printAlloc()
// Remove elements
for i := 0; i < n; i++ {
	delete(m, i)
}
// End
runtime.GC()
printAlloc()
runtime.KeepAlive(m)
```

⾸先，我们分配⼀个空映射，添加⼀百万个元素，删除⼀百万个元素，然后运⾏⼀次 GC。我们还确保使⽤ `runtime.KeepAlive` 保留对 map 的引⽤，这样 map 也不会被收集。

```go
0 MB
461 MB
293 MB
```

我们能观察到什么？起初，堆⼤⼩是最⼩的。然后它在向 map 添加⼀百万个元素后显着增⻓。但是，如果我们希望在删除所有元素后堆⼤⼩会减⼩，那么这不是 Go 中 map的⼯作⽅式。最后，即使 GC 已经收集了所有元素，堆⼤⼩仍然是 293MB ⼤。所以内存缩⼩了，但不像我们预期的那样。理由是什么？

我们在上⼀节中讨论了 map 由 8 个元素的桶组成。在底层，Go map 是⼀个指向 `runtime.hmap` 结构的指针。这个结构包含多个字段，包括⼀个 `B` 字段，它给出了 map 中的桶数：

```go
type hmap struct {
 B uint8 // log_2 of # of buckets
 // (can hold up to loadFactor * 2^B items)
 // ...
}
```

添加⼀百万个元素后，B的值等于 18，即 2^18 = 262,144 个桶。那么，当我们删除⼀百万个元素时，B的值是多少？仍然是 18。因此，map 仍然包含相同数量的桶。

原因是 map 中的桶数不能缩⼩。因此，从 map 中删除元素不会影响现有存储桶的数量；它只是将存储桶中的插槽归零。⼀张 map 只能增⻓并拥有更多的桶；它永远不会缩⼩。

在前⾯的⽰例中，我们从 461 MB 变为 293 MB，因为元素已被收集，但运⾏ GC 并不会影响 map 本⾝。甚⾄额外存储桶的数量（由于溢出⽽创建的存储桶）也保持不变。

让我们退后⼀步讨论⼀下 map ⽆法缩⼩的事实何时会成为问题。

想象⼀下使⽤ `map[int][128]byte` 构建缓存。此映射为每个客⼾ ID(int) 保存⼀个 128 字节的序列。现在，假设我们要保存最后 1000 个客⼾。map ⼤⼩将保持不变，因此我们不必担⼼ map ⽆法缩⼩。

但是，现在假设我们要存储⼀⼩时的数据。同时，我们公司决定为⿊⾊星期五做⼀个⼤促销，⼀⼩时内，我们可以让数百万客⼾连接到我们的系统。在这种情况下，即使在⼏天之后，⼀旦⿊⾊星期五结束，我们的 map 将包含与⾼峰时段相同数量的存储桶。因此，它解释了为什么我们会遇到在这种情况下不会显着减少的⾼内存消耗。

那么，如果我们不想⼿动重启我们的服务来清理 map 所消耗的内存量，有什么解决⽅案呢？

⼀种解决⽅案是定期重新创建当前 map 的副本。例如，我们每⼩时构建⼀张新 map ，复制所有元素，然后发布之前的 map 。主要缺点是在复制之后直到下⼀次 GC，我们可能会在短时间内消耗两倍的当前内存。

另一种解决方案是更改映射类型以存储数组指针：`map[int]*[128]byte`。这并不能解决我们将有一个重要数量的桶。但是，每个桶条目将保留一个值的指针而不是128字节（因此，在64位系统上为8字节，32位系统上为4字节）。回到原来的场景，让我们比较一下每个步骤后每种地图类型的内存消耗：

| 步骤          | `map[int][128]byte` | `map[int]*[128]byte` |
|-------------|---------------------|----------------------|
| 分配一个空 map   | 0MB                 | 0MB                  |
| 添加一百万个元素    | 461MB               | 182MB                |
| 删除所有元素并运行GC | 293MB               | 38MB                 |

我们可以注意到，在删除所有元素之后，所需的内存量对于 `map[int]*[128]byte` 类型的重要性明显降低。此外，在这种情况下，在高峰时间，由于一些优化以减少消耗的内存，所需的内存量不太重要。

> **Note** 我们还应该注意，如果键或值超过128字节，Go不会将其直接存储在map桶中。
相反，Go将存储一个引用键或值的指针。

正如我们所见，在 map 中添加 n 个元素，然后删除所有元素意味着在内存中保持相同数量的桶。因此，我们必须记住，Go map 的大小只能增长，它的内存消耗也是如此。没有自动缩小它的策略。如果这导致高内存消耗，我们可以尝试不同的选项，例如强制重新创建映射或使用指针检查它是否可以优化它。

对于本章的最后一部分，让我们讨论 Go 中的比较值。
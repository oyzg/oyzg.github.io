---
title: "CS186笔记 RookieDB"
date: 2022-06-26T17:48:09+08:00
draft: false
categories: [数据库, 笔记]
series: [笔记]
tags: [数据库, 笔记, CS186, RookieDB]
summary: "CS186学习笔记."
---

课程地址：[CS186](https://cs186berkeley.net/)

## 缓存管理（Buffer Management）
我们在数据库中的读写操作都是在内存中的页（Page）上进行读写，缓存管理器则负责管理在内存中的页和处理来自文件和索引管理器的页请求。

由于内存空间有限，我们不可能将所有页都保存在内存中，当内存满了而我们不存在我们需要的页时，我们就需要从磁盘中读取相应的页到内存中，同时将内存中的页写入磁盘中。

在内存中，我们在缓存池（Buffer Pool）中，我们将空间划分为多个帧（Frame）来存放页，每一个帧中存放一个页。

为了有效的管理缓存池中的帧，缓存管理器在内存中分配了额外的空间用来存放一张元数据表（Metadata table），表中包含四个字段：
- Frame ID：对应唯一的内存地址
- Page ID：决定现在帧中包含的页
- Dirty Bit：用来验证当前页是否被修改
- Pin Count：用来记录当前页的请求者数量

### 处理请求
- 当请求的的页在内存中时，该帧的pin count加一，并且返回该页的内存地址
- 当请求的页不在但是buffer pool没有满时，找到这个页并读入内存，该页pin count设为1，并返回内存地址
- 当请求的页不在并且buffer pool没有空间时，就需要进行页面的置换

页面的置换策略很大程度上取决于页面访问模式，通过计算页面点击数来选择最佳策略。
Page hit就是请求的页在内存中，不需要读磁盘；
Page miss就请求的页不在内存中，需要读磁盘，会有一个额外的IO消耗，所以好的置换策略是性能的关键因素。
Hit rate就是用Page hit数除以总的请求数。

如果换出的页设置了dirty bit，需要写入磁盘来确保持久化，如果页在内存中有更新，dirty bit设置为1，写入磁盘后设置为0.

当一个请求完成后会通知磁盘管理器减少该页的pin count。

### LRU（Least Recently Used-最近最少使用）
在LRU算法中，为了便于找到最近最少使用的页，我们需要在metadata table中加一列Last Used用来记录上一次使用的时间。

LRU算法则是将pin count = 0 并且 last used最小的页换出，将需要的页换入。

在具体计算中，我们可以按照如上方法进行计算，在数据量很少时，我们可以从当前插入的页往前，找到最久没有使用的页即可。

### MRU（Most Recently Used-最近最多使用）

在直觉上我们可能无法理解MRU算法，但是MRU的适用场景是这样的：比如我们在size=3的pool中循环查找（A、B、C、D），这种情况下，使用MRU的性能更佳，平均每size次只需置换一次。

在MRU算法中则不需要添加额外的列，当需要置换时，我们只需将pin count = 0 并且最后换入的页换出即可

### Clock Ploicy

在Clock Policy中，需要添加一列Ref bit来存储a Last Used value，并且Ref bit只需要1bit，而不是多个bit，它不同于LRU中的Last Used的地方在于最近使用的页为0，其他的页为1，因此节省的时间和空间。

此外，Clock Policy还需要一个变量Clock Hand来记录考虑中的帧（这里为理解为下一次换出的页）

在初始化时，所有ref bit为1，clock hand指向第一个unpinned（pin count = 0）的帧.

Clock Policy换出页面的程序如下：
- 从0到最后循环遍历表中所有的帧，跳过pinned的帧，直到找到第一个unpinned且ref bit = 0的帧
- 如果当前遍历的帧 ref bit = 1，那么设置为0，并且将clock hand设置为下一个帧
- 找到后，移除这个页并且将ref bit设为1，将clock hand设为下一个帧

## 排序
不同与传统的排序算法，在数据库中对数据进行排序时，我们每次将页读入或者写出都是一次IO操作，因此我们对与时间复杂度的度量也不再是用O(),而是IO操作的次数。

###  Two Way External Merge Sort（二路归并算法）

为了有效的合并两个列表，我们需要保证两个列表都是有序的，而在最开始我们无法保证其是否有序，所以第一次阶段每一个列表都只有一个单独的页，我们称这个阶段为“conquer”。

接着我们就可以开始进行合并排序，我们称合并的结果为“sorted runs”，一个“sorted runs”就是一个有序的页序列。

直到我们只剩下一个“sorted runs”，排序才算是完成了。

#### 分析
在分析一个数据库算法时，最重要的指标就是这个算法消耗的IO次数。

在二路归并算法中，我们假设有n个数据页，那么每次传递数据都需要2*n次IO，读写分别消耗一次IO。

在整个排序的过程中，我们首先需要“conquer”，在这之后，我们需要log(n)次传递来进行合并，所以总共1+log(n)次传递，2n*(1+log(n))次IO操作。

缓冲页（buffer page）或缓冲帧（buffer frame）是将页存储在内存中的一个槽。在合并的过程中，我们将两个页读入内存，分别占用一个buffer frame，在合并完成后，我们还需要一个buffer frame来存放合并完成的页，我们将合并前的buffer frame叫做“input buffer”，合并后的一个叫做“output buffer”。一旦这个页写满了，我们就需要将其写入磁盘，并开始构建新的页。在二路归并排序中，我们只需要3个buffer frame（两个input buffer， 一个output buffer）。

### Full External Sort（完全外部排序）

在二路归并排序中，我们的第一个阶段只对单个页进行排序，并且整个过程中，我们只用了3个buffer frame，而在实际情况中，buffer frame的个数不止3个，所以二路归并排序没有充分利用资源，并且传递次数更多。

而完全外部排序能够充分利用buffer frame，假设有B个buffer frame，我们在“conquer”阶段，不是对单个页面进行排序了，而是对B个页进行排序，并得到一个“sorted runs”，在之后的合并阶段中，我们拿出一个buffer frame作为output buffer，剩下的B-1个作为input buffer。

#### 分析

假设我们需要对n个页进行排序，在“conquer”阶段，我们将n个页导入得到n/B个sorted runs，在后面的合并中，每一次传递都除以B-1，所以总共需要1+log_B-1(n/B)次传递，需要2n*(1+log_B-1(n/b))次IO操作。

## 哈希

在数据库中，将类似的值分组在一起叫做hash。

因为我们无法将所有的数据一次填入内存，所以我们需要构建多个不同的hash表并且将它们连接在一起。但这存在一个问题，如果同样的值存在两张hash表中，我们可以直接将两张hash表连接起来吗？
答案当然是不可以，所以我们需要确保当一个值在内存中，其他相同的值也在内存中。

在整个过程中，我们分为两个阶段，第一个阶段是“partitioning”（分片），第二个阶段是“conquer”。一个分片是用一个分片哈希函数得到值相同的集合。假设有B个buffer frame，在每次分片中，我们通过hash得到B-1个分片，也就是B-1个output buffer，因为有一个是作为input buffer。

在完成分片后，适合内存（页数小于等于B）的分片就能进入hash表构建阶段，不适合的则递归地进行分片直到所有的分片最多只有B页，并且需要使用不同的hash函数。

假设m为所需分片次数，r_i为第i次分片读入的页数，w_i为第i次分派你写出的页数，X为构建hash表的总页数，那么IO次数为m次分片所有的r_i和w_i的次数加上2X。

此外hash还存在一些重要的属性：
- r_0 = N: 第一次分片需要读入所有的页
- r_i <= w_i： 分片过程可能会创建额外的页
- w_i >= r_(i+1)： 可能有的页进入构建阶段了，不需要再次分片
- x >= N： 每次分片可能会增加页数


## Joins (连接)

参考：[知乎陈大炮笔记](https://zhuanlan.zhihu.com/p/369864030)

### 介绍

首先我们需要先了解join到底是什么，像“R INNER JOIN S ON R.name = S.name”的语句是如何执行的。完成一个Join是需要条件的，相当于把R和S两个关系根据匹配条件合并创建一个新关系，即，对于 R 中的每条记录 r_i 使用匹配条件找到 S 中的对应 s_j，并将'<r_i, s_j>'输出中作为新的行 (r的所有字段后面跟着s的所有字段)。

在下面的连接算法中，我们不考虑将新关系写入磁盘的代价，因为我们假设join的新关系在稍后执行SQL查询涉及到的另一个操作符中会用到。

### 常用两表连接算法

#### Simple Nested Loop Join 简单嵌套循环连接

简单嵌套循环连接是最简单的连接算法，假设我们有一个B页的缓存区，我们希望连接2个表，R和S，连接条件为θ，策略就是可以先获取 R 中的每条记录，然后再遍历 S 中的所有匹配项，最后产生对应匹配项。

伪代码如下：
```
for each record ri in R:
    for each record sj in S:
        if θ(ri,sj):
            yield <ri, sj>
```

这是一个糟糕的算法，我们从 R 中读取每条记录时，需要读取 S 中的每一页来寻找匹配。产生的I/O开销是 `[R]+|R|[S]` ，其中 `[R]` 是 R 中的页数， `|R|` 是 S 中的记录数。

#### Page Nested Loop Join 页嵌套循环连接
相对于简单嵌套循环，页嵌套循环不是对R中每一条记录去读S的每一页，而是对R中的每一页读S中的每一页，从而减少了读取S中每一页的次数。

```
for each page pr in R:
    for each page ps in S:
        for each record ri in pr:
            for each record sj in ps:
                if θ(ri, sj):
                    yield <ri, sj>
```

因此，页嵌套循环的IO开销为`[R]+[R][S]`。

#### Block Nested Loop Join 块嵌套循环连接
在页嵌套循环中，我们只用了3个缓存页（一个用于R，一个用于S，一个为输出缓冲区），我们想要读S的次数越少越好，所以，我们可以将B-2个缓存页用于R，这B-2个页称为Chunk，将Chunk中的每一条记录与S中的每一条记录相匹配，这样就可以进一步减少读S的次数。

这里的关键思想是，我们想利用 缓冲区 来降低 I/O 开销，所以我们可以在缓冲区里尽可能多存储 R 的页，因为我们每个 Chunk 针对 S 中页都只读一次，所以更大的 Chunk 意味着更少的 I/O。然后用 R 的每个块与 S 的每条记录进行匹配。

```
for each block of B−2 pages Br in R:
    for each page ps in S:
        for each record ri in Br:
            for each record sj in ps:
                if θ(ri,sj):
                    yield <ri, sj>
```

IO开销为`[R] + [R/(b-2)][S]`

#### Index Nested Loop Join 索引嵌套循环连接
当S上有一个对应字段的索引时，它可以非常快的在S中查找r_i的匹配，其IO开销为`[R + [R]*(在S中查找匹配记录的成本)`，其中在S中查找匹配记录的成本因索引的类型而不同。

```
for each record ri in R:
    for each record sj in S where θ(ri,sj)==true:
        yield <ri, sj>
```

#### Hash Join 哈希连接
如果R的记录小于等于B-2页，则我们可以在构建一个哈希表，如何读取S，在R的哈希表中查找匹配的记录。这称为'Naive Hash Join'朴素哈希连接，成本为`[R]+[S]`。它依赖于R能够装入内存，但这通常不太可能。此外我们需要把R构建成哈希表，这里成本也很高。

为了解决这个问题，我们可以重复地将 R 和 S 哈希到 B-1 缓冲区中，这样我们就可以得到 ≤B−2 页大小的分区，使我们能够将它们放入内存中并执行朴素哈希连接。

更具体地说，考虑每一对相关的分区 Ri 和 Si (即R的分区i和S的分区i)。如果 Ri 和 Si 都是大于 B-2 页，则将两个分区哈希成更小的分区。否则，如果 Ri 或 Si 小于等于 B-2 页，则停止分区并将较小的分区加载到内存中，以构建内存中的哈希表，并与其较大的分区执行朴素哈希连接。

这个过程被称为 Grace Hash Join 优雅哈希连接，它的I/O代价是：子节点上哈希的代价加上朴素哈希连接的代价。哈希的代价可以根据我们需要重复哈希多少个分区的次数而改变。对分区 P 进行哈希的代价包括读取 P 中的所有页所需的 I/O 以及在对分区 P 进行哈希后写入所有结果分区所需的 I/O。

Grace Hash Join 是很好，但它对键倾斜非常敏感，所以在使用这个算法时要小心。当许多记录具有相同的键时，就会发生键倾斜。例如，如果我们对一个列进行哈希，这个列的值只有 yes，虽然我们可以强行构建，但不管我们使用哪个哈希函数，它们最终都会在同一个桶中。

#### Sort-Merge Join 排序合并连接
// TODO

#### An Important Refinement 排序合并连接的一个重要改进
// TODO



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

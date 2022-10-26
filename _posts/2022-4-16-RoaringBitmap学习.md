---
layout:     post
title:      "RoaringBitmap学习"
subtitle:   "Bitmap优化"
date:       2022-4-16 10:00:00
author:     "Peakiz"
header-img: "img/post-bg-bitmap.png"
header-mask: 0.35
catalog: true
tags:
   - 数据结构
---

## 0 什么是 RoaringBitmap？

BitMap 的基本原理就是用一个bit 位来存放数据状态，适用于大规模数据，但数据状态又不是很多的情况，可以节省大量内存空间。通常被用来判断某个数据存不存在的，比如Bloom Filter等；RoaringBitmap在很多产品中都有使用，如lucene, **redis**, spark等。

在Java里面一个int类型占4个字节(32bit)，假如要对于10亿个int数据进行状态存储，如果采用数组形式来存储，需要占用的空间为：
$$
10亿*4/1024/1024/1024=4GB
$$
如果能够采用bitmap来存储,需要占用的理论最小空间为：
$$
10\_0000\_0000Bit=1\_2500\_0000byte=122070KB=119MB
$$
然而实际上，数据并非一直连续存储，bitmap的空间大小往往取决于我们想要存储的数据的最大值的位宽！比如想要存储稀疏数据[1, 10\_0000\_0000]这两个数据的状态，利用数组的话只需要两个元素即可达成（数组中存在这个元素就说明状态为存在，因此判断状态的时间复杂度为O(n)），占用空间为：4*2 = 32Byte；如果采用bitmap来存储的话就仍然需要119MB的空间大小（第n bit位为1就说明存储数值n，因此bitmap判断状态的时间复杂为O(1)）：

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/26/-170850.png" alt="image-20221026170848561" style="zoom: 40%;" />

由此可见，**bitmap适合大量连续密集的数据状态存储，不适合稀疏数据的存储。**而RoaringBitmap就是为了解决这个问题的；

RoaringBitmap的定义：

> Roaring bitmaps are compressed bitmaps. They can be hundreds of times faster.

 从上述的定义可以看出，RBM还是使用到了BM，只是在位图的的基础上进化成了高效压缩位图，从而达到了高性能以及更广泛的使用场景。**简单来说RoaringBitmap就是将每个数据进行分段分桶映射处理，一个int数据32位，可以划分为高16位和低16位，先映射高16得到第一层桶(大桶, 大桶就是普通bitmap+指针)，然后映射低16位得到第二层位置（小桶，小桶一共有3种不同类型实现)；两层结构达到压缩空桶数量的目的；**

## 1 RoaringBitmap的实现原理

以java中的RoaringBitmap实现为例：将 32bit int（无符号的）类型数据 划分为 2^16  个桶，即最多可能有65536个桶（即 container ），每个桶最多存放2^16 即65536个数值，那么所有的桶存放的数值正好就是2^32即32位无符号整数的全体值。

**在存储和查询数值时**：将数值 k 划分为高 16 位和低 16 位，取高 16 位值找到对应的桶，然后在将低 16 位值存放在相应的 Container 中（存储时如果找不到就会新建一个）

Container（小桶) 分为三类，描述可以参照图中的文字，这三类 Container 就构成了RBM存储和查询的基础，在不同场景下使用不同的 Container。

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/26/-175401.png" alt="image-20221026175400652" style="zoom:60%;" />

BM适合于大量密集数据的存储，而int数组适合于少量稀疏数据的存储。于是RBM决定各取所长，根据数据的特点在不同情况下采取不同的存储方式。

- 首先当一个桶被创建时，第一个元素一定会被放在 ArrayContainer 中存储，因为此时的数据是绝对的稀疏。
- 后续的数据被放入到该桶时，RBM会计算 ArrayContainer 的元素个数，当元素个数超过4096时，就会将 ArrayContainer 转化为 BitmapContainer 存储，而超过**4096**即是RBM认为的数据密集的阈值。

> 为什么是选择4096当作判断数据密集的阈值？

因为当元素个数为4096的时候内存数组的内存占用和bitmap的内存占用是一样大的，当小于4096的时候数组占用小，大于4096的时候，bitmap内存占用小；

- 数组占用内存为：2Byte*n/1024 (KB)   一条直线（注意此时数组最大值位2^16因此只用2Byte即可存下，即此时的ArrayContainer 其实是一个short数组)； 

- bitmap占用内存为：16bit位可以表示的最大值，即2^16 bit / 8/1024 = 8 KB

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/26/-180229.png" alt="image-20221026180228603" style="zoom:25%;" />

RunContainer 中的Run指的是行程长度压缩算法(Run Length Encoding)，对连续数据有比较好的压缩效果。它的原理是，对于连续出现的数字，只记录初始数字和后续数量。即：

- 对于数列11,12,13,14,15，它会压缩为11,4；
- 对于数列11,12,13,14,15,21,22，它会压缩为11,4,21,1；

这种压缩算法的性能和数据的连续性（紧凑性）关系极为密切，对于连续的100个short，它能从200字节压缩为4字节，但对于完全不连续的100个short，编码完之后反而会从200字节变为400字节。因此默认是不开启的；

其实RBM中常用的是ArrayContainer 和 BitmapContainer，而 RunContainer 是非常规的 Container，只有在手动调用 runOptimize() 方法的时候才会产生。调用 runOptimize() 方法时，会比较和 RunContainer 的空间占用大小，选择是否转换为RunContainer。

## 2 java中Bitmap及RoaringBitmap的使用

java中对于bitmap有专门的数据结构`java.util.BitSet`来实现，对于RoaringBitmap则需要引入相应的jar包依赖；

> BitSet的使用:

java中 BitSet 内部维护了一个long数组，初始只有一个long，所以BitSet最小的size是64，当随着存储的元素越来越多，BitSet内部会动态扩充，最终内部是由N个long来存储，这些针对操作都是透明的。

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/26/-211839.png" alt="image-20221026211838055" style="zoom:50%;" />

> RoaringBitmap的使用：

需要引入jar包：

```xml
<dependency>
    <groupId>org.roaringbitmap</groupId>
    <artifactId>RoaringBitmap</artifactId>
    <version>0.9.30</version>
</dependency>
```

类图如下：

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/26/-212125.png" alt="image-20221026212123541" style="zoom: 25%;" />

简单使用：

```java
        RoaringBitmap rr = RoaringBitmap.bitmapOf(1, 2, 3, 1000);  //print:{1,2,3,1000}
        RoaringBitmap rr2 = new RoaringBitmap();
        rr2.add(4000L, 4005L);   //r2： {4000,4001,4002,4003,4004}
        // 第三个数值是多少，索引从0开始
        int thirdvalue = rr.select(3);  // thirdvalue: 1000
        // 2这个值在第几位，如果不在是0
        int indexoftwo = rr.rank(2);  // indexoftwo: 2
        boolean c1 = rr.contains(1000); //true
        boolean c2 = rr.contains(7); // false
```

虽然RoaringBitmap是为32位的情况设计的，但对64位整数进行了扩展。为此提供了两个类: **Roaring64NavigableMap **和 **Roaring64Bitmap**：

- Roaring64NavigableMap依赖于传统的**红黑树**。键是32位整数，代表元素中最重要的32位，而树的值是32位RoaringBitmap。32位RoaringBitmap表示一组元素的最低有效位。
- 较新的Roaring64Bitmap方法依赖**ART**数据结构来保存键/值对。键由元素的最重要的48位组成，而值是16位的Roaring容器。
  - 基数树（Radix tree），又称压缩前缀树，是一种更节省空间的Trie（前缀树）。所有节点使用固定大小的数组
  - Adaptive Radix Tree（自适应基数/前缀树，ART）可变的基数树，在 Radix tree 的基础上，优化增加可变特性，其核心是优化空间的利用率。每个节点的空间大小不再相同，而是根据需要使用不同的节点类型。
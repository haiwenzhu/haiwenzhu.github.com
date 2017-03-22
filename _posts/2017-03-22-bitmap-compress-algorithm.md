---
layout: post
title: "bitmap压缩算法对比"
categories:
  - Algorithm
---

最近的项目中使用到了bitmap，简单描述一下使用场景：整体有亿级的用户，每一个用户有一个唯一的标识ID（整型），不同的用户会被打上不同的标签，系统需要支持按标签搜索用户，或者根据现有的标签组合（并集、交集），生成新的标签。

目前的解决方案是建立标签到用户的索引，一个标签对应到一组用户ID，用户ID以bitmap的格式存储，标签对应的用户是稀疏的，转换成bitmap存储空间的利用率很低，举个例子，比如某个标签对应的用户ID是[100000000, 200000000]，转换成bitmap需要ceil(100000001/8)byte数据，空间的利用率极其的低，当然这个例子很极端，在生成环境不会出现，这里只是想说明bitmap存在存储空间利用率底的场景，需要对bitmap做压缩处理。

常用的bitmap压缩算法有Oracle's BBC、WAH、Concise、Roaring。前三者都是基于run-length encoding的思路，RLE的思路是说对于重复出现的值，通过值加上重复出现的次数表示，从而到达数据的压缩，比如对于这个字符串：

`WWWWWWWWWWWWBWWWWWWWWWWWWBBBWWWWWWWWWWWWWWWWWWWWWWWWBWWWWWWWWWWWWWW`

通过RLE编码之后为：`12W1B12W3B24W1B14W`。bitmap的值只包含重复的0和1，基于ELB的压缩会是不错的选择。Roaring的压缩思路和前三者都有不同，接下来我会试着介绍WAH、Concise、Roaring各自的实现细节。

首先是WAH，WAH继承自BBC，其主要思路为：
1. 把bitmap按每31位分组，每个组称为一个block
2. 如果block既包含0又包含1，用block以1+block的内容表示
3. 如果block只包含0，则以00+n表示，其中n标识block的数量
4. 如果block只包含1， 则已01+n表示，其中n标识block的数量

![img1](http://github.tiankonguse.com/images/concise-wah.png) [图片来源][1]

n的最大可取值为2^30-1，通常情况下n都远小于这个值，因此存在空间的浪费。

Concise是对WAH算法的改进版本，把block的后30个bit位分成了两段，前5位和后25位，其中，

- 前5bit表示在第几位存在一个反转，所谓反转是指00开头的block在某一位存在一个1，或者01开头的block在某位存在一个0
- 后25个bit表示后面还有多少个block

下面这个例子，是对 {3, 5, 31–93, 1024, 1028, 1 040 187 422}数组的压缩：
![](http://github.tiankonguse.com/images/concise-concise.png) [图片来源][1]
- word#0表示0-30，{3,5}
- word#1表示31-92，{31-92}，前2bit01表示全为1，中间5bit全为0表示不存在反转，后25bit为1表示后面还有1个全为1的block，数据范围为31到31 + 31 + 31 * 1- 1
- word#2表示93-1022，{93}, 前2bit表示数据全为0，中间5bit为1标识在第1位（低93位）存在一个反转，后25bit为29标识后面还有29全为0的block，数据范围为93到93 + 31 + 31 * 29 -1
- #word3表示1023-1053，{1024,1028}
- #word4表示1054-1040187391
- #word5表示1040187392-1040187422

下面这个特殊的数列可以很好的对比WAH和Concise算法的差异，{0, 62, 124,...}，
用WAH表示为：`10000000000000000000000000000001,00000000000000000000000000000000,10000000000000000000000000000001,00000000000000000000000000000000,10000000000000000000000000000001,00000000000000000000000000000000,...`

用Concise标识为：`00000010000000000000000000000001,00000010000000000000000000000001,00000010000000000000000000000001`

在这个场景下，Concise每个数占用了32bit空间，WAH每个数占用了64bit空间。

但是基于RLE的压缩算法有以下两个明显的短板：
1. 不能很快速的check某个值是否在这个bitmap里，必须要decompress之后才能知道
2. 不能跳过某一段数据，在两个bitmap做位于运算的时候，如果一个bitmap的某一区间全为0，没有办法直接跳过另一个bitmap对应区间的数据。

Roaring算法不同于前面提到的几种，其算法思路是：
1. 对于32位的整数，每2^16个数分成一组，称为一个chunk
2. 对于一个单独的chunk，如果其包含的数超过4096个，用一个2^16bit的bitmap存储
3. 对于元素小于4096的chunk，用一个有序的数据（sorted array）保存
4. 每一个chunk上都会保存这个chunk包含的元素个数（Cardinality）

对于检索操作，比如判断某个整数x是否存在，首先x/2^16定位到属于那个chunk，如果chunk是一个bitmap，检查x mod 2^16这个bit位，如果chunk是一个数组，使用二分查找。

对于逻辑运算，也是以chunk为单位进行运算，存在bitmap vs bitmap，bitmap vs array，array vs array三种不同的场景，针对不同的场景，会采用不同的计算方法，具体细节可以参看[这里][2]，就不展开了。另外值得一提的点是，算法对逻辑运算都是in place操作的，比如两个bitmap做位或运算，会直接修改其中的一个bitmap，而不是分配一个新的bitmap。

对于前面提到的那个特殊的数列{0, 62, 124,...}，采用Roaring算法，每个数占用的空间≈16（每个数在chunk内只占用16bit，chunk本身会占用一定的空间）
WAH、Concise、Roaring算法的性能对比在这篇[paper][1]里有提及，有兴趣的同学可以看一下，从结果看，Roaring在各方面的表现都是不错的。


参考链接：
- http://github.tiankonguse.com/algorithm.html#sec-1-1-5

- https://arxiv.org/pdf/1402.6407.pdf

- https://github.com/RoaringBitmap/RoaringBitmap

[1]: http://github.tiankonguse.com/algorithm.html#sec-1-1-5

[2]: https://arxiv.org/pdf/1402.6407.pdf

[3]: https://github.com/RoaringBitmap/RoaringBitmapi


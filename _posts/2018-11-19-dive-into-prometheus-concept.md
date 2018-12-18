---
layout: post
title: "Prometheus实践之histtogram-quantile实现"
categories:
  - Microservice
---

## 前言
业界最近比较流行的一个词是数据中台，也比较诟病我司数据打通的能力，我个人比较明显的感受是大家在日常的工作中对数据的敏感性是不够的，一方面可能是我们缺少这样的数据意识，另一方面缺少相关的工具也是原因之一。当然不同的人、不同的角色对数据的义和数据的关注点是不一样，作为开发人员，对数据的关注很大一部分是集中在系统的运营数据。运营数据这块业界其实有很多的[工具](https://prometheus.io/docs/introduction/comparison/)。我个人认为一个好的运营数据工具应该具备以下几点：
1. 简单的数据上报，开发人员不需要在接入上耗费太多心智
2. 标准化的数据查询，最好是有类SQL化的查询语言
3. 多样化的数据呈现，包括可视化图表、pushmail等
目前开源的工具里，Prometheus是微服务架构下被选择比较多的工具之一，我认为它也比较符合我上面的提到的几点，最近也有尝试部署和使用，把过程中一些自己觉得比较难理解的点记录以下。

## Histogram和Summary的区别
Prometheus提供了四种[数据类型](https://prometheus.io/docs/concepts/metric_types/)，其中Histogram和Summary是比较复杂和难理解的。
先看一个histogram的数据示例：![hisgoram sample](https://i.imgur.com/KcPTT4p.png)其中，test_histogram是数据指标的名称，一个histogram的数据指标会包含多个数据维度：sum、count、bucket，其中sum是所有数据的和，count是数据个数，bucket是小于等于该bucket的数据个数。buckets可以理解为是对数据指标值域的一个划分，划分的依据应该基于数据值的分布，比如某个指标99%的值都是小于100的，bucket的划分应该集中在100以内。bucket的划分会影响到quantaile计算的精度，这块在后面histogram_quantile这个函数的解释时会提到。
Summary和histogram比较类似，两者的主要区别在于Summary的quantile计算是在数据上报的时候就已经计算好的，需要在定义数据指标的时候就指定quantile的值，因为是数据上报计算的quantile，所以不支持包含数据过滤和聚合的quantile计算，文档里有详细的histogram和quantile的[区别](https://prometheus.io/docs/practices/histograms/)。

## `histogram_quantile`实现
先解释以下`quantile`，`quantile`的中文翻译为`分位数`，可以理解为分割数据的点，比如`quantile=0.9`对应的是数据集中小于其占比为90%的值，如果该数据是页面响应时间，则页面90%的响应时间都在`quantile=0.9`之内。
`histogram_quantile(φ float, b instant-vector)`用于计算history数据指标一段时间内的quantile值，参数b一定是包含le这个label的，不包含就无从计算quantile。
我一开始看到这个函数比较疑惑的一点在于每个bucket只记录的小于等于该bucket的数据个数，没有记录具体的数据值，要计算具体的quantile应该是需要知道具体每一个数的数值才行的。实际上需要明白quantile对应的值是一个预估值，计算结果是基于bucket内的数据是线性分布的。下面的这段伪代码是histogram_quantile的实现：
```
histogram_quantile(quantile, buckets)
    sortByUpperbound(buckets) // sort buckets asc
    rank = quantile*buckets[-1].count
    index = 0 //index of bucket which contain rank
    for i in buckets {
        if buckets[i].upperBound > rank
            index = i
            break
    if index == len(buckets)
        return buckets[-2].upperBound
    if index == 0 && buckets[0].upperBound <= 0
        return buckets[0].upperBound

    lowerBound = 0
    upperBound = buckets[index].upperBound
    count = buckets[index].count
    if index > 0
        lowerBound = buckets[index-1].upperBound
        count -= buckets[index-1].count
        rank -= buckets[index-1].count
    return lowerBound + (upperBound-lowerBound)*(rank/count) //bucket内线性分布
```
上面这段代码省去了一些边界条件的检查，详细的代码可以看[这里](https://github.com/prometheus/prometheus/blob/master/promql/quantile.go)

## 客户端
基于php的[client library](`https://github.com/Jimdo/prometheus_client_php)支持三种模式的客户端数据存储，`APC`、`Redis`和`In-memory`。对于通常的web应用，比较适合的应该是Redis，Redis的实现里不支持database的选择，有人发起了[pull request](https://github.com/Jimdo/prometheus_client_php/pull/89)，不过还是open状态，没有被接受。

## 最后
Prometheus最近刚尝试使用，这篇文章只是尝试解释一些过程中困惑的点，上面提到的几个问题在文档里都有提到，想尝试的同学[官方文档](https://prometheus.io/docs/introduction/overview/)是最好的开始，HAVE FUN~。
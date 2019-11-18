---
layout:     post
title:      "HBase Timestamp 与幂等性"
subtitle:   "HBase Timestamp 与幂等性"
date:       2019-03-29 23:09:41
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - HBase
---

HBase 的数据模型包括表（Table）、行（Row）、列族（Column Family）、列限定符（Column Qualifier）、单元格（Cells）、时间戳（Timestamp），其中单元格是行与列的交叉点，用来存储数据值，而 timestamp 则是每个值的版本号标识。默认情况下，timestamp 的值是更新数据时的当前时间戳，由系统自动更新，并不太会被关注，但是在实际的项目中，如果能合理的使用 timestamp 却能带来巨大的作用。本文主要介绍利用 timestamp 来保证数据的幂等性更新实现。

![image](/img/2019-03-29-hbase-timestamp/1.png)

## 一、幂等性更新

HBase 的更新或删除不会去覆盖一个值，它只会在后面追加写，直到 major_compaction 时，才会丢弃过期的数据。而判断数据新旧的方式则主要是依赖于单元格中的 timestamp 的值，它的值越大则证明该单元格的值越新。如果表的版本号设置为 1，在删除过期数据时，在相同 rowkey、Family:Qualifier 下的单元格，仅会保留最新的单元格的值。

![image](/img/2019-03-29-hbase-timestamp/2.png)

所以利用 timestamp 能很好的实现数据的幂等性更新，即不管写入的先后顺序是否符合预期，只要 timestamp 的值是固定的，那么最终的结果也将是符合预期的。这在分布式系统中十分有效，因为这不仅不需要保证数据的更新顺序，也让多线程的数据更新变得简单。

![image](/img/2019-03-29-hbase-timestamp/3.png)

#### Java API

在 Java 中，可以通过调用以下 API 构建 Put 对象：

```java
org.apache.hadoop.hbase.client.Put#addColumn(family, qualifier, ts, value)
```

为了更新操作不会覆盖删除操作，可以通过以下 API 来构建 Delete 对象：

```
org.apache.hadoop.hbase.client.Delete#addColumns(family, qualifier, timestamp);
```

需要注意的是，这里是`addColumns`而不是`addColumn`方法，`addColumn`方法只会删除最近的版本。

## 二、Timestamp 与 TTL

Timestamp 不仅可以作为版本标识，同样的，它也是作为数据是否超时的一个依据。如果你在表的属性中设置了 TTL 的值，那么数据是否超时的公式应该是这样的：


```math
isTimeout = (currentTimestamp - timestamp > TTL * 1000)
```

当数据超时后，你将无法直接获取到这个值，并且它将最终会被删除。 得知这个特点有助于我们在手动设置 timestamp 时做一些全面的分析，因为不是所有的数据都能得到它的时间戳。比如最近在做 MSSQL 数据同步时，以 CT 方式监听数据变更时，它是不带时间的，仅有事务的更新版本（从 1 开始递增的数字），如果以事务的版本号作为 HBase 数据的 timestamp，那么在遇到 TTL 时，可能会有明明写入成功却不能获取到值的情形。所以在实际场景中，需要考虑这样的情况。

以上。
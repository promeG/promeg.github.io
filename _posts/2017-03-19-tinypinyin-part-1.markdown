---
layout:     post
title:      "打造最好的Java拼音库-TinyPinyin(1)"
subtitle:   "单字符转拼音的极致优化"
date:       2017-03-18 12:52:36 +0800
author:     "PromeG"
header-img: "img/home-bg.jpg"
tags:
    - TinyPinyin
    - Java
    - Android
---

汉字转拼音是一个开发中经常使用的功能。其中，[pinyin4j](https://sourceforge.net/projects/pinyin4j/)是应用较广的Java汉字转拼音库。然而，此库有不少的缺点：

1. Jar文件较大，205KB；
2. Pinyin4J的PinyinHelper.toHanyuPinyinStringArray 在第一次调用时耗时非常长（~2000ms）；
3. 功能臃肿，许多情况下我们不需要声调、方言；
4. 无法添加自定义词典，进而无法有效处理多音字；
5. 内存占用太高。

为了解决上述问题，我开发了[TinyPinyin](https://github.com/promeG/TinyPinyin)，旨在提供最好的Java和Android拼音库。经过一段时间的开发，目前该项目已迈入2.x版本，基本的功能已经完整，项目架构也已定型，有着相当好的运行速度及很低的内存占用，也在不少项目中实际得到了应用。

在TinyPinyin的开发中遇到了不少有趣的问题，汇总为几篇文章，与大家分享 :)

## 1. TinyPinyin简要介绍

TinyPinyin是一个适用于Java和Android的快速、低内存占用的汉字转拼音库。

其特性包括：

* 支持基于词典的多音字处理，支持简体中文、繁体中文；

* 极速的执行效率(Pinyin4J的4~16倍)；

* 很低的内存占用（不添加词典时小于30KB）。

### 1.1 简洁的API

#### 汉字转拼音API

```java
/**
 * 如果c为汉字，则返回大写拼音；如果c不是汉字，则返回String.valueOf(c)
 */
String Pinyin.toPinyin(char c)

/**
 * c为汉字，则返回true，否则返回false
 */
boolean Pinyin.isChinese(char c)

/**
 * 将输入字符串转为拼音，转换过程中会使用之前设置的用户词典，以字符为单位插入分隔符
 */
String toPinyin(String str, String separator)
```

#### 词典API

TinyPinyin基于词典加入了对多音字的支持。

```java
// 添加中文城市词典
Pinyin.init(Pinyin.newConfig().with(CnCityDict.getInstance());

// 添加自定义词典
Pinyin.init(Pinyin.newConfig()
            .with(new PinyinMapDict() {
                @Override
                public Map<String, String[]> mapping() {
                    HashMap<String, String[]> map = new HashMap<String, String[]>();
                    map.put("重庆",  new String[]{"CHONG", "QING"});
                    return map;
                }
            }));
```

### 1.2 极速的执行效率

使用[JMH](http://openjdk.java.net/projects/code-tools/jmh/)工具得到bechmark，对比TinyPinyin和Pinyin4J的运行速度。

性能测试结果简要说明：单个字符转拼音的速度是Pinyin4j的**四倍**，添加字典后字符串转拼音的速度是Pinyin4j的**16倍**。

从性能测试的结果来看，TinyPinyin显著优于应用广泛的Pinyin4j。特别的是，添加了含有近15000个词的大词典后，TinyPinyin的速度竟比不支持多音字处理的Pinyin4j的速度快的更多，到底是如何做到的呢？

接下来，本文将详细介绍TinyPinyin在速度和内存方面的详细设计。

附：性能测试的详细结果：

Benchmark | Mode  | Samples | Score |  Unit
-------------------------- | --- | ----- | ---- | ----
TinyPinyin_Init_With_Large_Dict（初始化大词典）| thrpt | 200 | 66.131 | ops/s
TinyPinyin_Init_With_Small_Dict（初始化小词典）  | thrpt | 200 | 35408.045 | ops/s
TinyPinyin_StringToPinyin_With_Large_Dict（添加大词典后进行String转拼音） | thrpt | 200 | 16.268 | ops/ms
Pinyin4j_StringToPinyin（Pinyin4j的String转拼音） | thrpt | 200 | 1.033 | ops/ms
TinyPinyin_CharToPinyin（字符转拼音） | thrpt | 200 | 14.285 | ops/us
Pinyin4j_CharToPinyin（Pinyin4j的字符转拼音）| thrpt | 200 | 4.460 | ops/us
TinyPinyin_IsChinese（字符是否为汉字） | thrpt | 200 | 15.552 | ops/us
Pinyin4j_IsChinese（Pinyin4j的字符是否为汉字） | thrpt | 200 | 4.432 | ops/us


## 2. 单字符转拼音的极致优化

对于单字符转拼音来说，要解决两个问题：

* 判断传入的字符是否为汉字

* 如果是汉字，则返回它的拼音

在具体解决问题前，首先要深入了解问题本身。最直观的单字符转拼音方案是维护一张巨大的映射表，存储每一个中文字符对应的拼音，如“中”对应“ZHONG”。那么中文字符和拼音共多少个呢？经过简单的统计分析，发现中文字符有如下特征：

* 中文字符共有20378个

* 中文字符除了12295外，均分布在 19968 ~ 40869 之间 (Unicode的4E00 ~ 9FA5)，并非连续分布，此范围内夹杂了524个非中文字符

* 拼音共有407个（不包含声调）

根据中文字符和拼音的特征，便可以设计如下的字符转拼音方案：

* 预先构建19968 ~ 40869的映射表，将每一个char映射为一个拼音（是中文字符）或null（不是中文字符）

* 判断传入的字符是否为12295，是则返回其拼音

* 判断传入的字符是否处于19968 ~ 40869之间，不属于则判定不是中文字符；属于的话则查预先构建的映射表，根据查到的值判断是否为中文，并返回相应的结果。

上述方案采用了查表的方法转换拼音，因此速度很快。然而，映射表的构建往往占据较大的内存，因此需设法降低映射表的空间占用。下文将具体阐述TinyPinyin所做的优化。

### 2.1 拼音映射表原始方案

最naive的拼音映射表的结构是：

```java
char --> String // 字符 --> 拼音，如 20013(中) --> "ZHONG"
```

此方案的劣势非常明显：中文字符共有两万多个，但拼音共有407个，为每个中文字符都分配一个String对象过于浪费空间。因此，需加以优化。

### 2.2 拼音映射表初步优化

之前统计发现拼音共有407个，那么我们可以分配一个静态的数组保存这407个拼音：

```java
static final String[] PINYIN_TABLE = new String[]{"A", "AI", ...
```

然后以拼音在数组中的位置作为此拼音的编码，如"A"对应的编码为0，"AI"的编码为1。拼音映射表便只需存储char对应的拼音编码即可，无需存储拼音本身，大幅降低了内存消耗。

需要注意的是，拼音共407个，因此至少需要9位来表示一个拼音。Java中byte为8位，short为16位，可采用short来表示一个拼音。

优化后的映射表如下：
```java
char --> short // 字符 --> 拼音编码
```

内存占用为：short[21000]存储映射表，共占用42KB内存，存编码的方式比直接存拼音占用空间要小很多。

然而，我们注意到，编码使用9位就足够了，使用short造成了较大的浪费，每个拼音编码浪费了16 - 9 = 7位，也就是说，理想情况下我们可以将存储所有汉字拼音的42KB内存优化到 42*9/16 = 24KB。

那么如何实现呢？请见下一步优化。

### 2.3 拼音映射表终极优化

思路是使用byte[21000]存储每个汉字的低8位拼音编码，另外采用byte[21000/8]来存储每个汉字第9位（最高位）的编码，每个byte可存储8个汉字的第9位编码。

共耗用内存21KB + 3KB = 24KB，整整降低了42.8%的内存占用。

当然，由于每个编码分为两部分存储，因此解码过程稍微复杂一些，不过采用位运算即可快速的计算出真实的编码：

```java
// 计算出真实的编码
short decodeIndex(byte[] paddings, byte[] indexes, int offset) {
  int index1 = offset / 8;
  int index2 = offset % 8;
  short realIndex = (short) (indexes[offset] & 0xff);

  if ((paddings[index1] & PinyinData.BIT_MASKS[index2]) != 0) {
    realIndex = (short) (realIndex | PinyinData.PADDING_MASK);
  }
  return realIndex;
}
```

## 3 小结

TinyPinyin的单字符转拼音功能就介绍到这里，从上述过程可以看到，转拼音虽然是一个很简单的功能，但想要做到极致却不容易。经过优化，TinyPinyin的内存占用得到了极大的降低，且单字符转拼音的速度达到了Pinyin4j的四倍。

下篇文章将介绍多音字的快速处理方案。还记得前文的性能测试吗？添加了含有近15000个词的大词典后，能够处理多音字的TinyPinyin的速度，竟比不支持多音字处理的Pinyin4j的速度快的更多（16倍），具体做了哪些优化呢？请移步下篇文章：TinyPinyin之多音字快速处理方案。

---
layout:     post
title:      "打造最好的Java拼音库TinyPinyin（二）：多音字快速处理方案"
subtitle:   ""
date:       2017-03-20 08:52:36 +0800
author:     "PromeG"
header-img: "img/home-bg.jpg"
tags:
    - TinyPinyin
    - Java
    - Android
---

上篇文章[打造最好的Java拼音库TinyPinyin（一）：单字符转拼音的极致优化](http://promeg.me/2017/03/18/tinypinyin-part-1/)，介绍了单字符转拼音的设计方案，本文将介绍TinyPinyin项目中的多音字处理功能。

## 1. 多音字快速处理方案概览

多音字处理是汉字转拼音库的一个重要特性。多音字的识别是基于词典实现的，这是由于绝大部分情况下，一个多音字到底该取哪个拼音，是由其所处的词决定的。例如，对于“重”字，在“重要”一词中应读“ZHONG”，在“重庆”一词中应读“CHONG”。

TinyPinyin中对应的API如下：

```java
// 添加中文城市词典
Pinyin.init(Pinyin.newConfig().with(CnCityDict.getInstance());


/**
 * 将输入字符串转为拼音，转换过程中会使用之前设置的用户词典，以字符为单位插入分隔符
 *
 * 例: "hello:中国"  在separator为","时，输出： "h,e,l,l,o,:,ZHONG,GUO,!"
 *
 * @param str  输入字符串
 * @param separator 分隔符
 * @return 中文转为拼音的字符串
 */
String toPinyin(String str, String separator);
```

在TinyPinyin中，对多音字的处理也是基于词典实现的，步骤如下图所示：

![TinyPinyin1.png](/img/201703/TinyPinyin1.png)

* 向TinyPinyin添加词典
* 传入待转为拼音的字符串
* 根据词典，对字符串进行中文分词
* 单独将分词得到的各个词或字符转为拼音，拼接后返回结果

在整个过程中，最为核心的部分便是分词了。下面具体介绍分词的处理。

## 2 TinyPinyin分词方案

基于词典的分词，本质上可分解为两个问题：

* 一是**多串匹配**问题。即给定一些模式串（字典中的词），在一段正文中找到所有模式串出现的位置，注意匹配可能有重叠，如"中国人民"可匹配出：["中国", "中国人", "人民"]
* 二是从匹配到的所有模式串集合中，按照一定的规则挑选出相互没有重叠的模式串子集，以此来得到分词结果，如上例中可挑选出的两种分词结果为：["中国", "人民"]和["中国人", "民"]

### 2.1 多串匹配算法

TinyPinyin选用了Aho–Corasick算法实现了多串匹配。Aho-Corasick算法简称AC算法，通过将模式串预处理为确定有限状态自动机，扫描文本一遍就能结束。其复杂度为O(n)，即与模式串的数量和长度无关，非常适合应用在基于词典的分词场景。

网上有很多对AC算法原理的介绍，这里不再展开，需要注意的是，AC算法的Java实现有两个流行的版本：[AC算法Java实现](https://github.com/robert-bor/aho-corasick) 和 [双数组Trie树(DoubleArrayTrie)Java实现](https://github.com/hankcs/AhoCorasickDoubleArrayTrie)。

后者声称在降低内存占用的情况下，速度能够提升很多，因此TinyPinyin首先集成了此库作为AC算法的实现。然而，集成后实际使用JProfiler监测发现，[双数组Trie树(DoubleArrayTrie)Java实现](https://github.com/hankcs/AhoCorasickDoubleArrayTrie)占用的内存很高，初步分析后发现，[AhoCorasickDoubleArrayTrie.loseWeight()](https://github.com/hankcs/AhoCorasickDoubleArrayTrie/blob/master/src/main/java/com/hankcs/algorithm/AhoCorasickDoubleArrayTrie.java)中有一些神奇的代码：

```java
/**
 * free the unnecessary memory
 */
private void loseWeight()
{
    int nbase[] = new int[size + 65535];
    System.arraycopy(base, 0, nbase, 0, size);
    base = nbase;

    int ncheck[] = new int[size + 65535];
    System.arraycopy(check, 0, ncheck, 0, size);
    check = ncheck;
}
```

从代码中可以看到，即使是一个空词典，也至少会分配两个 int[65535]，共512KB，内存占用实在太高，因此TinyPinyin最终选用了[Trie树(DoubleArrayTrie)Java实现](https://github.com/hankcs/AhoCorasickDoubleArrayTrie)。

### 2.2 分词选择器

分词选择器的作用是，从匹配到的所有模式串集合中，按照一定的规则挑选出相互**没有重叠**的模式串子集，以此来得到分词结果。

如上例中可挑选出的两种分词结果为：["中国", "人民"]和["中国人", "民"]。常见的分词选择算法包括：正向最大匹配算法、逆向最大匹配算法、双向最大匹配算法等。

TinyPinyin选择了正向最大匹配算法，其基本思路是：从句子左端开始，不断匹配最长的词（组不了词的单字则单独划开），直到把句子划分完。算法的思想简单直接：人在阅读时是从左往右逐字读入的，正向最大匹配法是与人的习惯相符的。

算法的具体实现请见[ForwardLongestSelector](https://github.com/promeG/TinyPinyin/blob/master/lib/src/main/java/com/github/promeg/pinyinhelper/ForwardLongestSelector.java)。算法的输入是匹配到的所有模式串集合（相互之间可能存在重叠），要求输出一个符合最大正向匹配原则的相互没有重叠的模式串子集。TinyPinyin用10行代码就实现了此算法，感兴趣的可以看一下 :-)

## 3. 多音字处理性能

我们旨在提供最好的汉字转拼音库，因此TinyPinyin很看重运行速度和内存占用。在多音字处理方面，我们关心以下几点：

* 词典的初始化耗时多久？
* 词典的内存占用如何？
* 分词速度如何？

性能测试的结果：

Benchmark | Mode  | Samples | Score |  Unit
-------------------------- | --- | ----- | ---- | ----
TinyPinyin_Init_With_Large_Dict（初始化大词典）| thrpt | 200 | 66.131 | ops/s
TinyPinyin_Init_With_Small_Dict（初始化小词典）  | thrpt | 200 | 35408.045 | ops/s
TinyPinyin_StringToPinyin_With_Large_Dict（添加大词典后进行String转拼音） | thrpt | 200 | 16.268 | ops/ms
Pinyin4j_StringToPinyin（Pinyin4j的String转拼音） | thrpt | 200 | 1.033 | ops/ms


我们选择了2个词典进行初始化的性能测试，分别是含有15000个词的大词典，和含有几十个词的小词典。

从结果中可以看到，初始化大词典约耗时15ms，初始化小词典仅需0.03ms。可以说初始化本身所需时间非常的少。

而在内存占用方面，使用中文城市词典时，额外消耗约43KB内存。

最后，添加了含有近15000个词的大词典后，TinyPinyin的速度比不支持多音字处理的Pinyin4j的速度快16倍，可以说执行速度非常的快，达到了TinyPinyin极致速度、极致空间占用的目标。

## 3 小结

TinyPinyin的多音字支持就介绍到这里，下篇文章将分享TinyPinyin的API设计。

# 4. API设计探讨

# 4. 单元测试设计

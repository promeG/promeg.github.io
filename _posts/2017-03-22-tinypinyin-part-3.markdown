---
layout:     post
title:      "打造最好的Java拼音库TinyPinyin（三）：API设计和测试实践"
subtitle:   ""
date:       2017-03-22 08:52:36 +0800
author:     "PromeG"
header-img: "img/home-bg.jpg"
tags:
    - TinyPinyin
    - Java
    - Android
---

之前的两篇文章[打造最好的Java拼音库TinyPinyin（一）：单字符转拼音的极致优化](http://promeg.io/2017/03/18/tinypinyin-part-1/) 和 [打造最好的Java拼音库TinyPinyin（二）：多音字快速处理方案](http://promeg.io/2017/03/20/tinypinyin-part-2/)，详细介绍了单字符转拼音、多音字的处理这两个具体功能的高效实现细节，本文是TinyPinyin系列的完结篇，将分享TinyPinyin项目在API设计上的思考，以及测试实践。

## 1. 汉字转拼音API设计

### 1.1 字符处理接口

TinyPinyin的汉字转拼音API非常简洁：

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

优秀的API的设计应满足**正交性**和**完备性**。从正交性的角度来看，Pinyin.toPinyin(char c)和Pinyin.isChinese(char c)是正交的，但String Pinyin.toPinyin(char c)与String toPinyin(String str, String separator)严格意义上并不是正交的。

之所以这么做的原因，在于Pinyin.isChinese(char c)接口无法支持多音字的处理，因为绝大部分情况下，一个多音字到底该取哪个拼音，是由其所处的词决定的，单个字符无法确定多音字的读音。在这里，功能实现的优先级要大于想要遵循的设计范式。

### 1.2 词典设置接口

初始化TinyPinyin时，添加词典的接口如下：

```java
Config config = Pinyin.newConfig()
                  .with(dict_1)
                  .with(....) // 可以继续添加多个词典
                  .with(dict_n);

Pinyin.init(config);
```

还有一个需求是，在TinyPinyin初始化之后，想再追加一些词典，实现此功能的接口是：

```java
Pinyin.add(other_dict); // 向Pinyin中追加词典
```

然而，每次新添加词典都会触发一次完整的AC算法构建过程，因此若有多个词典，推荐使用性能更优的Pinyin.init(Config)接口。

## 2. 词典API设计

拼音转换的接口较为容易，词典API的设计就没那么简单了。

### 2.1 基础词典：PinyinDict

设计具体的词典之前，我们需要思考**汉字转拼音词典**究竟是什么。

对汉字转拼音词典来说，它本质上包含了一组词的集合，以及集合中每个词的拼音。词的集合可以用Set<String>来表示，每个词和它的拼音之间的映射关系是：

```java
  词(String) --> 拼音(String[]) // 如："重庆" --> ["CHONG", "QING"]
```

因此，词典可由两个API组成：返回词典所有词的 Set<String> words() 和  将词转为拼音的 String[] toPinyin(String word)。这里有个约定，toPinyin接口应保证对words中的所有词，toPinyin(String)均返回非null的结果。

```java
/**
 * 字典接口
 */
public interface PinyinDict {

    /**
     * 字典所包含的所有词
     */
    Set<String> words();

    /**
     * 将词转换为拼音，应保证对words中的所有词，toPinyin(String)均返回非null的结果
     */
    String[] toPinyin(String word);
}
```

这两个接口是不是看起来很熟悉？没错，这两个接口加起来，便成为了一个Map：Map<String, String[]>。

问题来了，为什么词典接口不做成直接返回一个Map<String, String[]>呢？

原因在于，如果直接返回Map，则相当于限定了词典只能采用Map这一种数据结构来实现，而在设计词典模型时，不应限制词典具体的实现方式。

例如，当词典非常大时，把整个词典加载到内存中的Map便不合适了，更好的做法应该是在执行 toPinyin(String) 时，从文件或数据库中读取相应的拼音，降低内存占用。这时我们拆分出的接口便体现出优势了：这两个接口是支持流处理的！因此词典既可以放在内存中，也可以放到文件数据中，甚至可以通过网络接口获取拼音转换结果。

### 2.2 便捷词典：PinyinMapDict

上文描述的基础词典PinyinDict较为简洁。然而，用户使用PinyinDict实现自定义词典时却很复杂：

```java
final Map<String, String[]> map = new HashMap();
map.put("重庆", new String[]{"CHONG", "QING"});

PinyinDict dict = new PinyinDict() {
    @Override
    public Set<String> words() {
        return map.keySet();
    }

    @Override
    public String[] toPinyin(String word) {
        return map.get(word);
    }
};
```

为了便于更好的创建自定义词典，TinyPinyin提供了对基础词典的封装：PinyinMapDict

```java
/**
 * 基于java.util.Map的字典实现，利于添加自定义字典
 */
public abstract class PinyinMapDict implements PinyinDict {

    /**
     * Key为字典的词，Value为该词所对应的拼音
     *
     * @return 包含词和对应拼音的 {@link java.util.Map}
     */
    public abstract Map<String, String[]> mapping();


    @Override
    public Set<String> words() {
        return mapping() != null ? mapping().keySet() : null;
    }

    @Override
    public String[] toPinyin(String word) {
        return mapping() != null ? mapping().get(word) : null;
    }
}
```

这样一来，用户添加自定义词典便非常简洁了：

```java
PinyinDict dict = new PinyinMapDict() {
    @Override
    public Map<String, String[]> mapping() {
        Map<String, String[]> map = new HashMap();
        map.put("重庆", new String[]{"CHONG", "QING"});
        return map;
    }
}
```

### 2.3 Android词典：AndroidAssetDict

为了提升效率，TinyPinyin专门为Android平台的词典设计了一个辅助类：[AndroidAssetDict](https://github.com/promeG/TinyPinyin/blob/master/tinypinyin-android-asset-lexicons/src/main/java/com/github/promeg/tinypinyin/android/asset/lexicons/AndroidAssetDict.java)。

这是由于Android代码访问JAR文件中的资源非常低效（[参考](http://blog.danlew.net/2013/08/20/joda_time_s_memory_issue_in_android/)）。因此，AndroidAssetDict采用了将字典文件存入asset中的方式提升访问效率。

AndroidAssetDict的使用非常简单，大家可参考[tinypinyin-lexicons-android-cncity](https://github.com/promeG/TinyPinyin/tree/master/tinypinyin-lexicons-android-cncity)这个子项目，只需要重写String assetFileName()这一个方法即可。当然，词典文件的格式需要与[示例](https://github.com/promeG/TinyPinyin/blob/master/tinypinyin-lexicons-android-cncity/src/main/assets/cncity.txt)保持一致。

## 3 测试实战


TinyPinyin项目中，功能代码和测试代码的比例是10：6，测试的覆盖率还是很高的。另外，为了评估性能，也添加了一些基于jmh工具的性能测试。


### 3.1 单元测试

单元测试在TinyPinyin中扮演了非常重要的角色，下面介绍一个核心的测试：单字符转拼音测试。

既然TinyPinyin之前已经有了Pinyin4J这个库，那就以它作为基准，确保对所有的字符（Character.MAX_VALUE ~ Character.MIN_VALUE），TinyPinyin与Pinyin4J有相同的返回结果。这样便保证了单字符转拼音功能的正确性，该部分测试如下：

```java
@Test
public void test_toPinyin_char() {
    char[] allChars = allChars();
    int chineseCount = 0;

    for (int i = 0; i < allChars.length; i++) {
        char targetChar = allChars[i];
        String[] pinyins = PinyinHelper.toHanyuPinyinStringArray(targetChar, format);
        if (pinyins != null && pinyins.length > 0) {
            // is chinese
            chineseCount++;
            assertThat(Pinyin.toPinyin(targetChar), equalTo(pinyins[0]));
        } else {
            // not chinese
            assertThat(Pinyin.toPinyin(targetChar), equalTo(String.valueOf(targetChar)));
        }
    }

    int expectedChineseCount = 20378;

    assertThat(chineseCount, is(expectedChineseCount));
}
```

### 3.2 性能测试

性能测试并不是一件容易的事情，在借助专业的工具（如jmh）外，还需要精心设计测试的输入、初始化等过程，如果这些因素做的不好，则可能会得到错误的性能测试结果。

下面介绍添加大字典后，字符串转拼音API的性能测试的输入选择。

字符串转拼音API的输入是一个字符串，那么我们应该选什么样的字符串呢？

首先，不能使用随机生成的字符串，这是由于随机生成的字符串中几乎不会出现词典中的词，那么在执行过程中便不会触发词典匹配，性能测试无效。

其次，不能使用过短的字符串，过短的字符串测试效果不明显；也不能在所有的执行轮次中选用同一个字符串，测试结果不精确。

那TinyPinyin是怎么选择输入的呢？找了一本很棒的小说《刀锋》的txt文档，每轮运行前，从txt文档中随机抽取1000个连续字符，作为输入字符串，完美解决了上述问题。具体代码请见PinyinDictBenchmark2。

## 4. 总结

本系列介绍了TinyPinyin单字符转拼音、多音字的处理这两个具体功能的高效实现细节，以及API设计上的思考和在测试方面的实战。从TinyPinyin开发过程可以看到，即使是一个功能非常简单的库，想做到极致也很不容易。

希望大家喜欢，欢迎讨论！

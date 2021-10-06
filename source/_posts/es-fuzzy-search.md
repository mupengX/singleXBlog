title: ES模糊搜索

date: 2021-10-05 16:12:18

categories: search

tags: 搜索

---

## 前言
提到全文搜索ES是一款比较被人熟知的开源搜索引擎，很多公司会基于ES来做全文搜索相关的业务。例如，在做内容相关的业务时常会有类似的运营需求：根据某个关键词搜索出所有包含该关键词的内容。如果量级比较小使用SQL的`LIKE`语句便可以解决，如果量级比较大的话一般需要通过搜索引擎来处理，但中文搜索往往因为分词问题会影响到最终的搜索效果。不同的业务对搜索结果的准确率和召回率各有不同的侧重，在根据某关键词搜索出所有内容案例中更加重视召回率，在保证召回率的基础上尽可能的提升准确率。

##  可用于模糊搜索的几种方式
首选简单讲一下对于中文内容ES建立倒排索引的过程：一篇文章送入ES后，ES首先会对内容进行分词，每个词被称为一个term，每个term会建立一个倒排表，倒排表为包含该term的doc id以及该term在doc中的位置等信息的 list。其中分词是指将一个句子分成多个短语，例如：`人民大道路面积水` 可被分词为`人民`、`大道`、`路面`、`面积`、`积水`等。常见的ES分词器有：standard analyzer、IK、ngram分词器、结巴分词器等。分词效果的好坏会直接影响到最终的搜索效果，例如：根据关键词`东方木子`进行内容搜索，很有可能因为在建立索引时被分词为`东方`、`方木`、`木子`等而导致内容无法被召回

ES提供的搜索方式大致可分为：基于term的精准匹配搜索(Term-level queries)和基于分词的全文搜索(Full-text queries)

基于term的精准匹配搜索过程：将输入的query当做term不进行分词，直接查找倒排表中包含该term的doc，并将结果返回

基于分词的全文搜索过程：针对输入的query进行分词，按照得到的分词term结果在倒排表中查找包含这些term的doc，按照相关性进行打分，最终按照得分降序排序返回结果

### 基于Term的几种模糊搜素方式

#### Fuzzy搜索

官方文档描述：
> Returns documents that contain terms similar to the search term, as measured by a Levenshtein edit distance.

该搜索方式返回所有包含的term与query相似的doc，term与query的相似性以Levenshtein编辑距离来度量。例如：输入`邓子棋`可以搜索出包含`邓紫棋`的文章，起到了一个纠错的作用。我们假设根据关键词搜索的场景下关键词几乎不会被输入错误，因此Fuzzy搜索并不能提升召回率反而会召回一些不应该召回的内容降低了搜索的准确率

#### wildcard通配符匹配

官方文档描述：

> Returns documents that contain terms matching a wildcard pattern

通配符匹配搜索从query形式上看是最接近SQL中`LIKE`语句的搜索方式，在SQL中的` LIKE %东方木子%`可以在ES中写成

```json
GET /myindex/article/_search
{
    "query": {
        "wildcard": {
            "content": "*东方木子*" 
        }
    }
}
```

但从实际效果来看并非如此，就像官方文档上说的这种查询方式返回的是所包含的term中能匹配上该搜索条件的所有doc。还是以`东方木子`为例，如果在索引中分词为`东方`和`方木`则上面的查询语句没有办法匹配到该doc，除非写成`*东*方*木*子*`。

另外该搜索模式在某些场景下资源开销比较大会影响性能，官方 提示如下：

> Avoid beginning patterns with `*` or `?`. This can increase the iterations needed to find matching terms and slow search performance.

中文含义是：避免以*或？开头的模式，这会增加查找匹配项所需的迭代次数并降低搜索性能，并且如果没有开启允许开销较大的查询设置则ES不会执行该查询：

> Wildcard queries will not be executed if [`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) is set to false.

Regexp正则表达式搜索与Wildcard原理类似不再赘述

### 基于分词的全文模糊搜索模式

#### match搜索

官方文档对该搜索方式的描述：

> Returns documents that match a provided text, number, date or boolean value. The provided text is analyzed before matching.

match搜搜是最常见的ES搜索模式，它首先将query进行分词得到term集合，然后找到包含这些term的doc并进行相关性打分，最终返回结果。在根据关键词搜索内容的场景下该模式会出现召回率低的问题。其中一个主要原因是：相同的短语包含在文章内进行分词和当做query进行分词得到的结果有差异，因为所处的上下文环境不一样。例如：实际在doc中包含`东方木子`的语句可能为`这一房东方木子`，最终被分词为：`房东`、`方木`进而导致无法召回。（上述仅为举例，实际情况要看使用的分词器，例如IK分词器有ik_max_word和ik_smart两种模式。ik_max_word为最细粒度的分词方式，可能会切出`房东`、`东方`、`方木`、`木子`等多个term；ik_smart的分词粒度相对粗一些可能切出：`房东`、`方木`两个term）。相比于召回率问题更重要的是该模式下会将其他仅包含`东方`term的doc给召回，这些doc并不包含`东方木子`这个短语导致准确率下降。

#### match_phrase短语匹配

官方文档描述：

> Returns documents that contain terms matching a wildcard pattern

该搜索模式下首先对query进行分词，然后根据分词后的term进行doc查找。与 match搜索不同的是match_phrase**找到的是包含所有term且term间的位置与搜索词项相同的doc**

使用该搜索模式虽然也会遇到与match搜索类似的问题，但因为该搜索会考虑term间位置的关系因此准确率上会有一定的提升

##### 配合NGram分词提升召回率

官方文档介绍：

> The `ngram` tokenizer first breaks text down into words whenever it encounters one of a list of specified characters, then it emits [N-grams](https://en.wikipedia.org/wiki/N-gram) of each word of the specified length.

以gram为单位可配置min_gram最小词元长度和max_gram最大词元长度。例如：

一个单词 quick，5 种长度下的 ngram

```
ngram length=1，q u i c k
ngram length=2，qu ui ic ck
ngram length=3，qui uic ick
ngram length=4，quic uick
ngram length=5，quick
```

还有一种形式叫做edge ngram：

```
首字母后进行 ngram
q
qu
qui
quic
quick
```

类似前缀索引的效果，可以应用于搜索suggest场景

NGram分词搭配使用match查询只要任何一个字匹配上就会被召回，错误率太高；搭配使用match_phrase查询既可以减少因对doc和query进行智能分词结果有差异导致的召回率低问题，同时加上match_phrase的slop偏移量参数配置可提升搜索的准确率

##### 混合使用match和match_phrase提高召回率和准确率

单独使用match搜索存在召回率高但准确率低的问题，而单独使用match_phrase存在准确率高但召回率低的问题。如果二者混合一起使用可以一定程度上既提升召回率又提升准确率：

```json
GET /myindex/article/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "content": "东方木子"
        }}
      ],
      "should": [
        {"match_phrase": {
          "content": {
            "query": "东方木子",
             "slop": 2
          }
        }}
      ]
    }
  }
}
```

但这种混用的方式不一定适合搜索出所有包含某关键词内容的场景

## 其他的方案

回到文章开始提到的场景：搜索出所有包含某关键词内容。如果我们就是要以超高的准确率找到包含某关键词的全部内容那这就变成了一个关键词匹配的场景，在这个场景下除了使用ES之外还有没有其他的方案呢？

这种场景实际转变为了字符串匹配的问题，字符串匹配问题业内已经有比较成熟的算法，那么我们可以使用MR或者Spark跑一个离线任务对每一篇文章进行字符串匹配计算将匹配上的文章返回，但这种方式不能满足请求即时响应。

如果要支持输入的关键词纠错查询或者支持?、*正则匹配呢？这种情况下如果用一个正则表达式去匹配全文内容会比较耗费资源，一种办法是先将doc按照标点符号进行断句，然后针对每一个句子进行正则匹配，这样做的前提假设是关键词不会有跨句出现的情况。

## 最后

以上即为在中文搜索中实现类似SQL `LIKE`模糊匹配的几种方式，中文搜索比英文搜索复杂的一个地方在于中文的分词问题，如果分词比较准确则能够很大的提升搜索效果，当然除了分词搜索本身还有许多难题要解决，例如query理解等，这不在本文讨论范围之内。

## 参考

[Elastic Search官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
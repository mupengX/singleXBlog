title: Rank的概率模型
date: 2017-05-13 11:28:54
categories: search
tags:
mathjax: true
---

## Rank是干啥的

Rank是搜索引擎中的精髓模块，Rank所做的事情就是根据用户的query对所有的doc(常见的如网页)进行打分排序。打分排序的依据是doc与query的相关性，而相关性怎么计算呢？把搜索这个动作可拆为两件事情：

1. 用户给出一个query
2. 某个doc满足用户的搜索需求

rank排序依据可以用一个条件概率来表示: $P(d_{i}|q)$其中q是用户给定的一个查询，$d_i$代表doc i满足用户的需求。这个条件概率的描述是：**在给定一个用户查询时，一个特定的doc满足用户需求的概率**

rank要做的就是对所有的doc按照$P(d_i|q)$值的大小进行排序。根据贝叶斯公式可以得到：
$$P(d_i|q)=\frac{P(q|d_i)P(d_i)}{P(q)}\tag{1}$$

这个公式告诉我们：
1. 不同的doc在同一个query查询下的值是可以比较的
2. $P(d_i)$是一个与查询无关的值，这个值可以理解为doc满足用户query的能力，可以在不同的doc间进行比较
3. $P(q|d_i)$可以理解为已知一个doc满足了用户的一些query，那么用户查询是q的概率是多少。就好比站长查看用户是从搜索引擎中的哪些query引到自己的网站。这个值反映的是一个doc对不同的query满足程度的比较，非doc间可比较。该值难以直接计算，它将一个已知query求条件概率的线上计算问题变成了一个已知doc求条件概率的线下计算问题。
4. $P(q)$是指一个特定查询出现的概率。该值较大时代表一个热门query，(1)式中的分子必须较大才能满足用户需求。如果该值较小，(1)式中分子无需太大也能满足用户的需求。

## PageRank

传统的TF*IDF算法，TF代表词频，IDF代表逆向文档频率，它主要思想为：
1. 某个词或短语在一篇文章中出现的次数越多，越相关。
2. 整个文档集合中包含某个词的文档数量越少，这个词越重要。

可见主要是针对(1)式中的$P(q|d_i)$，对$P(d_i)$考虑的比较少。而$P(d_i)$是一个查询无关的值，理解为在完全不知道用户查询的情况下，一个doc满足用户需求的情况，也可以理解为无查询推荐。它表示，即使在不知道用户想要什么，或者对用户的需求分析错的情况下，仍然可以尽量给出满足用户需求的doc。对于$P(d_i)$，google 利用一个随机浏览模型（即pagerank）进行计算，即认为这个值近似等于一个不断随着超链随机浏览的人看到网页i的概率。这个值的加入显著地提升了搜索质量。
由此可以窥见，推荐与搜索本来就是不分家的，有推荐的搜索才是真正的搜索，才体现出搜索的“情感”(query解析、排序、页面展示)，搜索引擎的难点(用户意图理解、有价值信息的展示)。
目前由于链接作弊等原因PageRank与真正想要值的差距已经越来越大了，渐渐的被其他算法所替代。

## 页面因子

我们认为对于一个doc先验的满足用户需求的概率由很多因素决定，如：页面质量、被其他页面链接数等。我们可以将其抽象为一系列页面因子组成的向量，对于一个doc w=< X1...Xn >,令S为用户得到满足的事件。
$$p(d)=p(s|w)=p(s|x_1,x_2,...x_n)=\frac{p(x_1..x_n|s)p(s)}{p(x_1..x_n)}$$
假设各页面因子是相互独立的
$$p(s|< x_1,..x_n >）=\frac{p(x_1|s)p(x_2|s)...p(x_n|s)p(s)}{p(x_1)...p(x_n)})$$
其中p(xi|s)是对于满足用户需求的某一个页面因子的值，例如：用户点击数等。
在(1)式中如果p(di)=0，无论其他值如何这个doc满足用户需求的概率都是0，这种doc就是属于垃圾doc，没必要参与rank。

## query分析
一个query可以被拆分为多个term，那么(1)式中p(q|di)的线下计算可以拆分为p(t|di)的计算。通过对query的分解变换将其转换为好计算的近似值，一个query可以分解为q=< t1..tn >,分解方式可以为：
1. 完全不分解：$$p(q|d_i)=p(< t_1..t_n >|d_i)\tag2$$
2. 分解为相互独立的term：$$p(q|d_i)=p(t_1|d_i)..p(t_n|d_i)$$
3. 每个term都只跟前一个term相关:
$$p(q|di)=p(t1|di)p(t2|di,t1)p(t3|di,t2)...p(tn|di,t_{n-1})$$
4. 查询的term必须按照一定顺序出现


## 多样性
如果一个doc可以满足多个query，那么：$\sum_jp(q_j|d_i)=1$ 实际情况中我们要根据query来分析用户的真正查询意图，同一个字符串可能对应多个用户需求。例如：对于“人人”这个查询，可能对应两种需求：人人网和人人车。假设其中对人人车的需求要高于对人人网的需求，(1)式中的排序会偏袒热门需求，因为对于冷门需求(1)式中对分子的权值要求比较低，如果不考虑“人人”对两种需求各自出现的概率那么满足“人人网”的doc在排序处于不利的位置。针对多样性，可以参考以下公式，其中$q_s$代表查询字符串：
$$p(d_i|q_s)=p(q_1|q_s)p(d_i|q_1)+P(q_2|q_s)p(d_i|q_2)=p(q_1|q_s)\frac{p(q_1|d_i)}{p(q_1)}+p(q_2|q_s)\frac{p(q_2|d_i)p(d_i)}{p(q_2)}$$

## 其他
在搜索引擎中可以引入其他的因素来扩展这个概率模型，比如引入时效性、权威性、引入地域等各种用户个性化信息。
另外目前各大搜索引擎希望在第一页(前10条)或者前3条结果中来满足用户需求，而不仅仅局限于某一个doc满足用户需求概率的最大化。
除了返回网页链接外，还可以通过增加返回结果类型的方式来满足用户的需求，例如：返回阿拉丁，框应用等。

## 总结
该模型解释了rank的排序依据以及如何将其分解为子目标进行计算，对于不好计算的变量，可以通过将其与好算的变量关联起来或者想办法近似的计算。近似计算就会用到一些假设，如果结果不对那么之前的假设就可能有问题。

**各种近似的计算都是对理论模型的逼近，理论模型存在的意义之一在于，它让我们的思考系统化，告诉我们在逼近什么，努力的方向在哪，离目标还有多远。想尽一切办法，越来越精确地逼近理论值，这就是进步。**



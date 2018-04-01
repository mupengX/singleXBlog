title: 会话管理(Dialog Management)

date: 2018-04-01 16:24:12

categories: NLP

tags: AI

toc: true

---
## 基本概念

会话管理(以下简称DM)是人机多轮对话系统的核心部分。它主要的功能包括：

- 对话状态维护(dialog state tracking, DST)
 - 维护更新会话状态。当前的会话状态依赖于之前的系统状态和之前的系统响应以及当前时刻的用户输入。
- 产生决策(dialog strategy)
 - 根据DST中会话状态做出系统决策，决定下一步做什么。
- 与其他后端系统交互 

下图是一个经典的对话系统结构图，可以看到DM的位置，在NLU(Linguistic Analysis)之后在NLG(Text Generation)之前，简单来说DM控制着人机对话的过程和走向，根据对话上下文信息来决定此刻系统对用户的输入做出的响应。

![Architecture of dialog system](\img\DM.svg)

在展开描述DM之前先简单说一下NLU(Natural Language Understanding)。NLU主要的工作是分析用户说的话并确定这句话的意图(Intent)是什么。同一个意图通常会有多种表达方式，例如：“今天天气怎么样”、“今天会下雨吗？”、“我想查询一下天气”等等，这些都是在表达同一个意图：查询天气。意图不应该关心句子的某些具体细节，例如：“告诉我去车站怎么走”与“怎么去最近的酒店”所表达的意图都是查询路线，不同的是二者的目的地不同，所以意图应该跟这句话所触发的Action相对应。而至于具体的细节，例如目的地，被当做意图的slot来填充。

## 实现方法

DM的实现方式有多种，每一种都有自己的优势和劣势，下面简单介绍几种。

### Structure-based Approaches

本质上是一种关键词匹配的做法，例如ELIZA和AIML（人工智能标记语言），通常是通过捕捉用户输入语句的关键词和短语来进行回应，通过增加变量并按topic来组织规则等方法能够一定程度上支持简单的多轮对话。

### FSM-based Approaches

这种方法通常将对话建模成一棵树或者有限状态机。系统根据用户的输入在有限的状态集合内进行状态跳转并选择下一步输出，如果从开始节点走到了终止节点则任务就完成了。

该方法的特点是：

- 提前设定好对话流程并由系统主导
- 建模简单，适用于简单任务
- 将用户的回答限定在有限的集合内
- 表达能力有限，灵活性不高

### Frame-based Approaches

在以完成任务为导向的对话系统中，Frame-based的方法是一种较为合适的方法。该方法将对话建模成一个填槽(slot filling)的过程。槽是多轮对话过程中将初步用户意图转化为明确用户指令所需要补全的信息。一个槽与一件事情的处理中所需要获取的一种信息相对应。例如上文提到的：“告诉我去车站怎么走”，其中目的地是一个槽位，“车站”是该槽位所填充的值。槽位又可细分为，平行槽位、依赖槽位、单值槽位和多值槽位等。另外，为了引导用户填充槽位会有对应的澄清话术。例如：“目的地”对应的澄清话术是“您想从哪出发呢？”，“出发时间”对应的澄清话术是“您想什么时间出发呢？”
该方法的特点是：
- 支持混合主导型系统，用户和系统都可以获取对话的主导权
- 输入相对灵活，用户回答可以包含一个或多个槽位信息
- 对槽位提取准确度的要求比较高
- 适用于相对复杂的多轮对话

### Agenda + Frame(CMU Communicator)

Agenda + Frame(CMU Communicator)对Frame-based的方法进行了改进，增加了层次的概念支持话题切换、回退等操作。

该方法首先将会话构建成一棵树，这棵树是动态的，可以在会话过程中对树进行添加子树或挪动子树等操作。树上的每一个节点对应一个handler，handler有优先级，此外还构建了任务计划，该计划包含了话题的有序列表。

从左到右、深度优先遍历这颗树生成handler的顺序。当用户输入时，系统按照顺序调用每个 handler，每个 handler 尝试解释并回应用户输入。如果handler捕获到信息就把信息标记为consumed，这保证了一个 information item只能被一个handler消费。如果用户输入不被 handler处理，那么系统将会进入output pass，每个handler都有机会产生自己的 prompt。

handler可以通过返回码声明自己为当前对话焦点，这个handler就被提升到树的顶端。同时这里使用sub-tree promotion 的方法保留特定主题的上下文，handler被提升到兄弟节点中最左边的节点，父节点同样以此方式提升。

## DM设计

刚开始要做DM的时候可能比较容易想到状态机、马尔科夫决策过程、end2end等方法，但实际操作起来去发现问题多多。

### 模型化困难

- 训练数据来源不容易找到及标准成本很高
- 在产品策略发生变化时需要重新标注语料
- 如何将与DM交互的其他service结果融入模型

个人目前的做法是NLU是单独实现好的，那么DM要做的是多轮会话topic切换(场景打断与恢复)，基于context的语义信息筛选，多轮对话的容错，与其他service交互以及各下游服务结果的rank等。

人与人之间的对话是及其复杂的，想用一个模型来刻画描述这个过程十分困难，无论在理论研究还是在工程实践领域仍有大量的工作要做，相信将来人机对话的体验会越来越好。

**参考链接**

[NLU vs. Dialog Management: To Whom am I Speaking?](https://www.inf.uni-hamburg.de/en/inst/ab/lt/publications/2016-schnelleetal-workshop-smartobjects.pdf)

[AN AGENDA-BASED DIALOG MANAGEMENT
ARCHITECTURE FOR SPOKEN LANGUAGE SYSTEMS](http://www.cs.cmu.edu/~xw/asru99-agenda.pdf)

[Dialog management](https://tutorials.botsfloor.com/dialog-management-799c20a39aad)

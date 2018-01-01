title: 关于chatbot

toc: true

date: 2017-12-16 23:40:59

categories: 此刻我在想什么

tags:

---

### 什么是Chatbot

Chatbot，宽泛的来讲叫做聊天机器人也有人叫他对话机器人，从16年开始Chatbot发展的比较火热成为大家关注的焦点，国外的有微软的小冰、Amazon的Alex，国内有百度的度秘等。如果说上述这些你还比较陌生的话，那么提到iPhone、iPad以及Mac等系统自带的Siri你应该就比较熟悉了。Chatbot的最大的优势在于他的对话式的交互方式，通过语言对话来交流已经发展成为人类的一种本能，通过像人与人之间交流的方式来与软件、机器等进行交互可以大大降低用户学习使用成本，这也是Chatbot这些年比较火热的一个原因。Chatbot常见的形态有：个人助手、家人陪伴、客服问答等。


### 如何实现一个Chatbot

目前对于Chatbot的实现方案有两种：一种是基于IR，也就是基于检索的方式；另一种是基于机器学习生成式的模式。

#### 基于检索的实现方式
用检索的方式来实现Chatbot的前提是拥有较多的问答对来构建知识库，针对用户的输入在知识库中找到能够回答用户问题的答案。其基本流程是：
`输入->明确用户意图->识别实体->retrieval->matching->ranking->response`

其中matching和ranking阶段需要好的模型来保证最后输出的准确性。这种实现方式得到的Chatbot更偏向于问答场景的对话，当然了，从某种角度来看人与人之间的交流也可以看做是一问一答式的对话。问答类的对话对于有明确答案的事实类问题是比较容易做到的，难的是如何回答好“为什么”、“怎么做”这一类的问题。

值得一提的是，吴军博士在《智能时代》里提到在12年他回到Google后Google希望他能做一些跟机器智能相关的事情，经过一段时间的调查他们决定用Google强大的搜索引擎来尝试着回答一些比较复杂的问题。当时的做法是：首先根据网页来确定用户的哪些问题Google可以回答哪些问题Google回答不了，缩小要解决问题的范围，使问题得到解决变得可能。然后通过机器学习的方法来把问题和网页来做匹配，类似于上述的learn to matching的过程。最终通过NLP的技术来合成最后的答案。经过吴军博士及其团队的努力使得计算机能够回答30%的复杂问题，并通过评测计算机的回答跟人的回答已经相当接近甚至很难区分哪个是机器回答的哪个是人回答的，算是一定程度上通过了图灵测试。

#### 基于生成式的实现方式

基于生成式的方式主要得益于近些年深度学习在NLP方面长足的发展，首先需要收集大量的对话、问答语料，然后将语料中的文字转成向量，在这些向量矩阵上做卷积以及GRU操作得到最终的答案，是一种sequence2sequence的方案。这种通过分析和学习语言元素的组成进而组合生成回答的方式相比基于IR的方式的好处是不需要构建和维护大规模的知识库，可以理解为通过不断的学习和训练的方法将知识库消化掉了，它比较灵活也更像是人的思考模式，同时它的缺点是结果不具有较强的可读性，有的时候会产生一些常人不能理解的句子，类似于出现之前报道的Facebook的两个机器人用人类看不懂的方式来对话的这种情况。

### Alexa和度秘是怎么做的

严格意义上来说Alexa和度秘并非单纯只采用上述两种方案中的某一种。相对于宽泛的聊天而言，Alexa和度秘更偏向于task oriented的对话系统，聊天只是其中的一个功能，更多的是在一问一答的过程中完成用户的指令、向用户提供服务或者与用户闲聊。

为了更好的服务好用户，度秘将提供不同垂类服务的模块分别作为单独的bot，Alexa称之为skill。当用户与其交互时通过分析用户的意图调用不同的bot来满足用户的需求，这当中除了涉及到意图识别还涉及到多轮对话的管理和bot之间的调度。从大的层面上来看大家所采用的技术都差不多，细节上的差别则是针对各自产品面向的场景不同所采用的训练语料也有所不同。当然，如果用户问到了bot无法回答的问题度秘也会通过检索的方式来尝试回答(暂不确定同样的情况下Alexa是否也采用检索的方式来尝试回答)。所以从工业的角度出发一定是结合学术界的多种方法来完成现阶段的产品。过于宽泛的聊天并不好控制，最糟糕的就是出现Chatbot胡乱回答，驴唇不对马嘴甚至是南辕北辙的情况，缩小业务范围聚焦在某一个领域反而问题比较容易得到解决。


### 未来
目前深度学习和NLP并没有那么完美，Chatbot看起来也还是有点傻傻的甚至有些“人工智障”，但是对话的交互方式必将带来变革，通过对话式的交互方式我们再也不需要去教用户如何来使用我们的产品。有些人也可能觉得这是人工智能的泡沫，但不是还有那么一句话么：No beer no bubbles.
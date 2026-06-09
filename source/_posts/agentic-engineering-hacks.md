title:  一个 90 万阅读的技术帖，藏着 Agentic Engineering 的 22 条心法

date: 2026-06-08 16:03:13

categories: AI

tags: AI

------

三个月前，Matt Van Horn 的《我知道的所有 Claude Code 技巧》在 X 上爆火。就在上周，他发了新帖《Every Agentic Engineering Hack I Know》，很快被 90 万人看过。

Matt 是谁？last30days（2.7 万星）、Printing Press（4000+ 星）、Agent Cookie 的作者。今年他成了 Python、Go、OpenClaw 等顶级开源项目的核心贡献者。他说自己从高中之后就没发布过别人觉得有价值的软件，直到今年。

帖子分享了 22 条心法，我按主题整理了一下，挑了几条最有感触的聊一聊。

---

## 一、先规划，再动手

Matt 的第一条规则是：有想法立刻写 plan.md，不是"让我想想"，不是"先写代码"。

但他紧接着说了第二条：**别读 plan.md。**

这两条放在一起很有意思。他的逻辑是：强制写 plan 让 Agent 不偷懒——它会去研究、承诺方案、写下验收标准，然后真的去达成。有计划的 Agent 交付完成的工作，没有计划的 Agent 抄近路、提前停。

他原话是：**"The plan is the leash."**

**胖虎说：**

这个思路和我之前写的 Agent Spec 有点类似，但本质不同。Agent Spec 是把"行为的定义"从代码里抽出来单独管理，更像一份合同。而 Matt 的 plan.md 是工作流里的一个 checkpoint——`/ce-plan` 写完，`/ce-work` 执行，session 挂了也不怕，plan 还在。

他的观点更激进：**plan 是给 Agent 看的，不是给你看的。** 你只需要问一句 TLDR，然后让它去干活。这个"信任但验证"的度，值得琢磨。

> 其他相关条目：非工程工作（战略文档、产品 spec、竞品分析）也可以用同一条 loop；先让 Agent 规划怎么规划，再做正事。

---

## 二、怎么把活高效喂给 Agent

Matt 的做法是同时开 4-6 个 cmux 会话，每个跑不同任务：一个写 plan、一个 build、一个做研究、一个修 bug。一个窗口在研究时，切到另一个窗口执行，变成多线程调度 Agent。

他还提到一个细节：**新终端标签页默认直达 Claude Code**，不需要 cd、不需要敲命令。当启动一个新 session 的成本降到一次快捷键，你会开更多 session。

**胖虎说：**

这个"降低摩擦"的思路值得借鉴。很多时候我们不是不想用工具，是启动成本太高。把默认入口改了、把权限跳过了、把远程控制打开了——这些小事累积起来，就是效率差距。

> 其他相关条目：语音做主输入（Monologue/Wispr Flow）、跳过权限确认（"YOLO，GitHub 会帮我兜底"）、远程控制 + 邮箱入口（给 Agent 一个邮箱地址，发邮件就是开任务）。

---

## 三、Agent 强不强，看你喂了多少上下文

Matt 提到一个很实用的做法：**跑 `/ce-plan` 之前，先跑 `/last30days` 做研究。**

这是他自己的开源项目。举个例子：在 Vercel 的 agent-browser 和 Playwright 之间做选择，他没先读文档，直接跑：

```
/last30days Vercel agent browser vs Playwright
```

几分钟内 Reddit、X、YouTube、HN 等平台的讨论被并行抓回来。结果是 agent-browser 每次调用吃掉的上下文更少，而 Playwright 光工具定义就灌进去几千 token。再把这批结果喂给 `/ce-plan`，做出来的 plan 直接站在社区最近 30 天的真实经验上，而不是模型六个月前的训练数据。

他的另一个做法是：**会议别自己总结，原始 transcript 直接扔进去。**

用 Granola 全程录音，结束后不整理，直接把完整 raw transcript 丢给 Claude Code。原始对话里的跑题、停顿、插科打诨，模型会自己判断哪些留下、哪些忽略——不需要你先替它总结一遍。

他还说：**同时跑多个 Agent 时，你负责给信号。**

原文标题叫 Human Signal。Agent 提供产量，人提供品味、方向和判断。你要做的是不断给出反馈——"第二版更接近了，但把第一版的语言拿回来""先处理最大的风险""这个方向不对"。

原话是：**"Agents supply volume, you supply taste."**

> 其他相关条目：笔记做成 Agent 的知识库（找一个带 CLI 的笔记系统，让 Agent 能读进去）。

---

## 四、让 Agent 走出终端

Matt 的两个项目很有意思：

**Printing Press** 把真实世界的服务包装成 CLI，让 Agent 能直接调用——比如给 Tesla 预热、在 Instacart 下单。配套的 **Agent Cookie** 把浏览器 session 交给 CLI，让 Agent 带着你的登录态行动。

他的判断是：**工作方式会不会变，很大程度取决于能不能把自己每天在用的服务印成 Agent 可调用的接口。**

他还提到一个很实用的协作方式：**plan.md 给 Agent 看，Proof 给同事看。**

plan.md 在终端里好用，直接拿给同事看不顺手。把 plan 或 spec 丢进 Proof，生成一个链接，同事像看文档一样阅读和 inline 评论，评论还能再流回 Agent 的工作流。比把 Markdown 粘到 Slack 里强。

> 其他相关条目：任何做超过两次的事写成 skill；开源贡献也走同一条 loop。

---

## 五、最后一条提醒

帖子最后，Matt 坦诚地说：**AI Psychosis 是真实存在的。**

Agent 本应帮我们干活，但每个朋友都比以前更拼命。他把这套 loop 形容成反馈极快的视频游戏——你说一句，东西就长出来；改一句，结果又更好。

**"Building with agents is the greatest video game ever made."**

风险不是做出来的东西没人用，而是沉浸在 build 的兴奋里，把身边的人和真正重要的关系弄丢了。

**胖虎说：**

22 条"怎么更高效"的 hack 之后，最后一条是"小心别把自己烧没了"。这种坦诚很难得。

效率提升的尽头不是"做更多"，而是"想清楚做什么"。这个道理在 AI 时代反而更值得记住。

---

## 写在最后

看完这 22 条，我觉得最值得记住的是这几点：

1. **Plan-first 不是新东西，但 Agent 让它变得真正可行**——以前 plan 和 code 是脱节的，现在 plan 直接驱动执行
2. **拉开差距的往往是上下文**——截图、issue、会议录音、过去的笔记，这些东西能稳定流进 Agent，效果完全不一样
3. **人的位置在上移**——你做的越来越像调度：给信号、给判断、给品味、给取舍
4. **越能 build，越要小心 build 带来的成瘾感**

---

原文地址：https://x.com/mvanhorn/status/2061877533885473181

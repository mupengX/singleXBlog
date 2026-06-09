title: ADK Go：Go 开发者终于等来了自己的 AI 智能体框架

date: 2026-05-11 19:55:43

categories: AI

tags: AI

------

## 一、引言

过去一年，AI 智能体的热潮席卷全球。但如果你是 Go 开发者，心里多半是既兴奋又有点不是滋味。

兴奋的是，AI 的能力确实让人大开眼界；不是滋味的是，想用 Go 构建一个真正能上生产的 AI 应用，却发现满世界都是 Python 的炼丹教程。调 prompt 像念咒，结果时好时坏。单元测试怎么写？版本怎么管理？怎么和现有 Go 微服务优雅集成？一堆问题让人头大。

这种感觉，就像一个习惯了精密图纸和标准化流程的机械工程师，突然被扔进了满是玄学咒语的炼丹房。

直到 Google 开源了 **ADK-Go（Agent Development Kit for Go）**，并且迅速霸榜 GitHub Go 趋势榜一周多，我才意识到：转机来了。

---

## 二、为什么 Go 开发者需要 ADK？

先说 Go 本身的优势，这不是废话——它直接决定了 ADK Go 的价值天花板：

- **高并发**：Goroutine 仅需几千字节内存，可处理数千个并发工具调用
- **类型安全**：静态类型消除了 LLM 输出解析的运行时错误
- **云原生**：天然适配容器化和微服务部署

但更重要的是 ADK Go 的**设计理念**：它选择了「代码优先」，而不是拖拽配置、YAML 低代码那套。

所有逻辑用 Go 代码定义，版本控制、代码审查、单元测试一应俱全，和现有 Go 项目无缝集成。这才是 Gopher 应该有的工作方式。

---

## 三、核心概念：5 分钟搞懂架构

ADK Go 采用**事件驱动的运行时架构**，一句话说清楚：

```
用户请求 → Runner → 事件循环 → 模型调用 → 工具执行 → 返回结果
```

**四个核心组件**：

| 组件 | 是什么 |
|------|------|
| **Runner** | 系统入口，管理整个执行生命周期 |
| **Agent** | 执行特定任务的智能体 |
| **Tool** | 智能体能调用的外部能力（函数、API 等） |
| **Session** | 管理对话上下文和状态 |

**三种智能体类型**：
- **LLM Agent**：最常用，背后是大语言模型
- **Workflow Agent**：控制执行流程（顺序 / 并行 / 循环）
- **Multi-Agent**：多个智能体协作，各司其职

---

## 四、动手写第一个智能体

理论说够了，直接上代码。以「天气查询智能体」为例：

### 环境准备

```bash
mkdir my_agent && cd my_agent
go mod init my_agent
go get google.golang.org/adk
export GOOGLE_API_KEY="your_api_key"
```

### 核心代码

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "google.golang.org/adk/agent/llmagent"
    "google.golang.org/adk/model/gemini"
    "google.golang.org/adk/runner"
    "google.golang.org/adk/tool"
    "google.golang.org/adk/tool/geminitool"
    "google.golang.org/genai"
)

func main() {
    ctx := context.Background()

    // 1. 初始化模型
    model, err := gemini.NewModel(ctx, "gemini-2.5-flash", &genai.ClientConfig{
        APIKey: os.Getenv("GOOGLE_API_KEY"),
    })
    if err != nil {
        log.Fatalf("Failed to create model: %v", err)
    }

    // 2. 创建智能体
    agent, err := llmagent.New(llmagent.Config{
        Name:        "weather_agent",
        Model:       model,
        Description: "回答城市天气问题的助手",
        Instruction: "你是一个天气助手，使用 Google Search 获取实时天气信息。",
        Tools:       []tool.Tool{geminitool.GoogleSearch{}},
    })
    if err != nil {
        log.Fatalf("Failed to create agent: %v", err)
    }

    // 3. 创建运行器并执行
    r, err := runner.New(runner.Config{Agent: agent})
    if err != nil {
        log.Fatalf("Failed to create runner: %v", err)
    }

    // 4. 发起对话
    result, err := r.Run(ctx, "北京今天天气怎么样？")
    if err != nil {
        log.Fatalf("Failed to run: %v", err)
    }
    fmt.Println(result)
}
```

三步：初始化模型 → 创建智能体 → 跑起来。和写普通 Go 代码没什么两样。

---

## 五、生产级特性：它凭什么说自己不是玩具？

光能跑 demo 不算本事，ADK Go 真正有价值的是这几个生产级能力：

**① 原生 OpenTelemetry 追踪**

每个模型调用、每次工具执行都有结构化追踪，可以可视化智能体的"思维链"，调试复杂逻辑不再靠 print 大法。

```go
telemetryProviders, _ := telemetry.New(ctx, telemetry.WithOtelToCloud(true))
defer telemetryProviders.Shutdown(ctx)
```

**② Retry and Reflect 插件**

工具调用失败？不用手动处理，插件会把错误反馈给模型，让它自动调整参数重试。某种意义上是"自愈"能力。

```go
r, _ := runner.New(runner.Config{
    Agent: myAgent,
    PluginConfig: runner.PluginConfig{
        Plugins: []*plugin.Plugin{
            retryandreflect.MustNew(retryandreflect.WithMaxRetries(3)),
        },
    },
})
```

**③ Human-in-the-Loop**

敏感操作加一行 `RequireConfirmation: true`，执行前强制等人工确认，生产环境的救命稻草。

---

## 六、说实话：它的局限在哪？

我不想写成一篇广告软文，所以必须把坑也说清楚：

- **生态还嫩**：Python 版本的第三方工具丰富度碾压 Go 版本，遇到冷门需求大概率要自己造轮子
- **主要优化 Gemini**：虽然号称"模型无关"，但用其他模型可能踩坑，建议先验证兼容性
- **学习曲线存在**：代码优先的理念对 Go 开发者友好，但对 Python 背景的 AI 玩家反而增加了门槛
- **Google 绑定风险**：深度集成 GCP，如果你有多云需求，提前想好迁移成本

---

## 七、什么人应该用它？

**适合你，如果：**
- 团队技术栈是 Go，想少踩迁移坑
- 要构建高并发的生产级 AI 服务
- 需要与现有 Go 微服务集成

**暂时不适合，如果：**
- 快速验证想法（Python 原型更快）
- 需要大量第三方工具集成（LangChain 生态更丰富）
- 团队完全不懂 Go

---

## 结语

ADK Go 不是银弹，但对 Go 开发者来说，它是目前最工程化、最有生产诚意的 AI 智能体框架。

炼丹房的大门终于向 Gopher 打开了，而且这次进去，你会发现里面装了空调、贴了施工图纸，还备了灭火器。

**本周行动建议**：花 30 分钟跑通上面那个天气 demo，感受一下「用写普通 Go 代码的方式写 AI 应用」是什么感觉。

---

**参考资源：**
- 官方文档：https://adk.dev/
- GitHub 仓库：https://github.com/google/adk-go
- Go 示例：https://github.com/google/adk-samples/tree/main/go
- Codelabs 教程：https://codelabs.developers.google.com/codelabs/agent-starter-pack-golang

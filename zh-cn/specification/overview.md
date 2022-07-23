<!--- Hugo front matter used to generate the website version of this page:
weight: 1
--->

# 概述

<summary>目录</summary>

<!-- toc -->

- [OpenTelemetry 客户端架构](#opentelemetry-client-architecture)
    - [API](#api)
    - [SDK](#sdk)
    - [语义约定](#semantic-conventions)
    - [贡献包](#contrib-packages)
    - [版本控制和稳定性](#versioning-and-stability)
- [跟踪信号](#tracing-signal)
    - [痕迹](#traces)
    - [跨度](#spans)
    - [跨度上下文](#spancontext)
    - [跨度之间的链接](#links-between-spans)
- [公制信号](#metric-signal)
    - [记录原始测量值](#recording-raw-measurements)
        - [措施](#measure)
        - [测量](#measurement)
    - [使用预定义的聚合记录指标](#recording-metrics-with-predefined-aggregation)
    - [Metrics 数据模型和 SDK](#metrics-data-model-and-sdk)
- [日志信号](#log-signal)
    - [数据模型](#data-model)
- [行李信号](#baggage-signal)
- [资源](#resources)
- [上下文传播](#context-propagation)
- [传播者](#propagators)
- [集电极](#collector)
- [仪器库](#instrumentation-libraries)

<!-- tocstop -->




本文档概述了 OpenTelemetry 项目并定义了重要的基本术语。

其他术语定义可在[词汇表](glossary.md)中找到。

## OpenTelemetry 客户端架构

![横切关注点](../internal/img/architecture.png)

在最高架构级别，OpenTelemetry 客户端被组织成[**信号**](glossary.md#signals)。每个信号都提供了一种特殊形式的可观察性。例如，跟踪、指标和行李是三个独立的信号。信号共享一个公共子系统——**上下文传播**——但它们彼此独立运行。

Each signal provides a mechanism for software to describe itself. A codebase, such as web framework or a database client, takes a dependency on various signals in order to describe itself. OpenTelemetry instrumentation code can then be mixed into the other code within that codebase. This makes OpenTelemetry a [**cross-cutting concern**](https://en.wikipedia.org/wiki/Cross-cutting_concern) - a piece of software which is mixed into many other pieces of software in order to provide value. Cross-cutting concerns, by their very nature, violate a core design principle – separation of concerns. As a result, OpenTelemetry client design requires extra care and attention to avoid creating issues for the codebases which depend upon these cross-cutting APIs.

OpenTelemetry 客户端旨在将每个信号中必须作为横切关注点导入的部分与可以独立管理的部分分开。 OpenTelemetry 客户端也被设计成一个可扩展的框架。为了实现这些目标，每个信号都包含四种类型的包：API、SDK、语义约定和 Contrib。

### API

API packages consist of the cross-cutting public interfaces used for instrumentation. Any portion of an OpenTelemetry client which is imported into third-party libraries and application code is considered part of the API.

### SDK

The SDK is the implementation of the API provided by the OpenTelemetry project. Within an application, the SDK is installed and managed by the [application owner](glossary.md#application-owner). Note that the SDK includes additional public interfaces which are not considered part of the API package, as they are not cross-cutting concerns. These public interfaces are defined as [constructors](glossary.md#constructors) and [plugin interfaces](glossary.md#sdk-plugins). Application owners use the SDK constructors; [plugin authors](glossary.md#plugin-author) use the SDK plugin interfaces. [Instrumentation authors](glossary.md#instrumentation-author) MUST NOT directly reference any SDK package of any kind, only the API.

### 语义约定

**语义约定**定义了描述应用程序使用的常见概念、协议和操作的键和值。

- [资源约定](resource/semantic_conventions/README.md)
- [跨度约定](trace/semantic_conventions/README.md)
- [度量约定](metrics/semantic_conventions/README.md)

收集器和客户端库都应该将语义约定键和枚举值自动生成为常量（或语言惯用等效项）。在语义约定稳定之前，生成的值不应分布在稳定的包中。 [YAML](../semantic_conventions/README.md)文件必须用作生成的真实来源。每种语言实现都应该为[代码生成器](https://github.com/open-telemetry/build-tools/tree/main/semantic-conventions#code-generator)提供特定于语言的支持。

### 贡献包

The OpenTelemetry project maintains integrations with popular OSS projects which have been identified as important for observing modern web services. Example API integrations include instrumentation for web frameworks, database clients, and message queues. Example SDK integrations include plugins for exporting telemetry to popular analysis tools and telemetry storage systems.

Some plugins, such as OTLP Exporters and TraceContext Propagators, are required by the OpenTelemetry specification. These required plugins are included as part of the SDK.

可选的且独立于 SDK 的插件和工具包称为**Contrib**包。 **API Contrib**是指完全依赖于 API 的包； **SDK Contrib**是指也依赖于 SDK 的包。

Contrib 一词特指由 OpenTelemetry 项目维护的插件和工具的集合；它不指代托管在其他地方的第三方插件。

### 版本控制和稳定性

OpenTelemetry 重视稳定性和向后兼容性。有关详细信息，请参阅[版本控制和稳定性指南](./versioning-and-stability.md)。

## 跟踪信号

分布式跟踪是一组事件，由单个逻辑操作触发，跨应用程序的各个组件进行整合。分布式跟踪包含跨进程、网络和安全边界的事件。当有人按下按钮以在网站上启动操作时，可能会启动分布式跟踪 - 在此示例中，跟踪将表示在处理由按下此按钮启动的请求链的下游服务之间进行的调用。

### 痕迹

**OpenTelemetry**中的跟踪由它们的**Spans**隐式定义。特别是，可以将**Trace**视为**Spans**的有向无环图 (DAG)，其中**Spans**之间的边被定义为父/子关系。

例如，以下是由 6 个**Span**组成的示例**Trace** ：

```
Causal relationships between Spans in a single Trace

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `child` of Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F]
```

有时使用时间轴更容易可视化**跟踪**，如下图所示：

```
Temporal relationships between Spans in a single Trace

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··········································]
      [Span D······································]
    [Span C····················································]
         [Span E·······]        [Span F··]
```

### 跨度

跨度表示事务中的操作。每个**Span**都封装了以下状态：

- 操作名称
- 开始和结束时间戳
- [**属性**](./common/README.md#attribute)：键值对列表。
- 一组零个或多个**事件**，每个事件本身都是一个元组（时间戳、名称、 [**属性**](./common/README.md#attribute)）。名称必须是字符串。
- 父母的**跨度**标识符。
- [**链接**](#links-between-spans)到零个或多个因果相关**Span** （通过那些相关**Span 的 SpanContext** **）** 。
- 引用 Span 所需的**SpanContext**信息。见下文。

### 跨度上下文

表示在**Trace**中标识**Span**的所有信息，并且必须传播到子 Span 并跨越进程边界。 **SpanContext**包含跟踪标识符和从父 Span 传播到子**Span**的选项。

- **TraceId**是跟踪的标识符。它是世界上独一无二的，几乎有足够的概率被制成 16 个随机生成的字节。 TraceId 用于将所有进程中特定跟踪的所有跨度分组在一起。
- **SpanId**是跨度的标识符。通过将其制成 8 个随机生成的字节，它具有几乎足够的概率是全球唯一的。当传递给子 Span 时，此标识符将成为子**Span**的父 Span id。
- **TraceFlags**表示跟踪的选项。它表示为 1 个字节（位图）。
    - 采样位 - 表示是否对跟踪进行采样的位（掩码`0x1` ）。
- **Tracestate**在键值对列表中携带跟踪系统特定的上下文。 **Tracestate**允许不同的供应商传播额外的信息并与他们的旧 Id 格式进行互操作。有关更多详细信息，请参阅[此](https://w3c.github.io/trace-context/#tracestate-field)。

### 跨度之间的链接

一个**Span**可以链接到零个或多个其他因果相关的**Span（由 SpanContext****定义**）。**链接**可以指向单个**Trace**内或跨不同**Trace**的**Spans** 。**链接**可用于表示批处理操作，其中**Span**由多个启动**Span**启动，每个 Span 代表批处理中正在处理的单个传入项目。

使用**链接**的另一个示例是声明原始跟踪和后续跟踪之间的关系。当**Trace**进入服务的受信任边界并且服务策略需要生成新的 Trace 而不是信任传入的 Trace 上下文时，可以使用此功能。新的链接跟踪也可能表示由许多快速传入请求之一启动的长时间运行的异步数据处理操作。

当使用 scatter/gather（也称为 fork/join）模式时，根操作会启动多个下游处理操作，并且所有这些操作都会聚合回单个**Span**中。最后一个**Span**链接到它聚合的许多操作。它们都是来自同一个 Trace 的**Span** 。并且类似于**Span**的 Parent 字段。但是，建议不要在这种情况下设置**Span**的 parent，因为从语义上讲，父字段表示单个父场景，在许多情况下，父**Span**完全包含子**Span** 。在分散/聚集和批处理场景中情况并非如此。

## Metric Signal

OpenTelemetry 允许使用预定义的聚合和一[组属性](./common/README.md#attribute)来记录原始测量或度量。

使用 OpenTelemetry API 记录原始测量值允许推迟最终用户决定应该对该度量应用什么聚合算法以及定义属性（维度）。它将在 gRPC 等客户端库中用于记录原始测量值“server_latency”或“received_bytes”。因此，最终用户将决定应该从这些原始测量中收集哪种类型的聚合值。它可能是简单的平均或精细的直方图计算。

使用 OpenTelemetry API 记录具有预定义聚合的指标同样重要。它允许收集诸如 cpu 和内存使用量之类的值，或诸如“队列长度”之类的简单指标。

### 记录原始测量值

用于记录原始测量的主要类是`Measure`和`Measurement` 。可以使用 OpenTelemetry API 记录附加上下文旁边的`Measurement`列表。因此，用户可以定义聚合这些`Measurement`并使用旁边传递的上下文来定义结果度量的附加属性。

#### Measure

`Measure` describes the type of the individual values recorded by a library. It defines a contract between the library exposing the measurements and an application that will aggregate those individual measurements into a `Metric`. `Measure` is identified by name, description and a unit of values.

#### Measurement

`Measurement`描述了为`Measure`收集的单个值。 `Measurement`是 API 表面中的一个空接口。该接口在 SDK 中定义。

### Recording metrics with predefined aggregation

所有类型的预聚合指标的基类称为`Metric` 。它定义了基本的度量属性，例如名称和属性。从`Metric`继承的类定义了它们的聚合类型以及单个测量值或点的结构。 API 定义了以下类型的预聚合指标：

- 用于报告瞬时测量的计数器指标。计数器值可以上升或保持不变，但永远不会下降。计数器值不能为负数。有两种类型的计数器度量值 - `double`和`long` 。
- Gauge metric 报告数值的瞬时测量值。仪表可以上下移动。仪表值可以是负数。有两种类型的量规度量值 - `double`和`long` 。

API 允许构造所选类型的`Metric` 。 SDK 定义了查询要导出的`Metric`的当前值的方式。

Every type of a `Metric` has it's API to record values to be aggregated. API supports both - push and pull model of setting the `Metric` value.

### Metrics 数据模型和 SDK

Metrics 数据模型[在此处指定](metrics/data-model.md)并基于[metrics.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto) 。该数据模型定义了三个语义：API 使用的 Event 模型、SDK 和 OTLP 使用的动态数据模型，以及指示出口商应如何解释动态模型的 TimeSeries 模型。

不同的导出器具有不同的功能（例如支持哪些数据类型）和不同的约束（例如属性键中允许哪些字符）。度量标准旨在成为可能的超集，而不是随处支持的最低公分母。所有导出器都通过 OpenTelemetry SDK 中定义的 Metric Producer 接口使用来自 Metrics Data Model 的数据。

正因为如此，Metrics 对数据施加了最小的限制（例如，哪些字符被允许在键中），并且处理 Metrics 的代码应该避免对 Metrics 数据进行验证和清理。相反，将数据传递给后端，依靠后端执行验证，并从后端传回任何错误。

有关详细信息，请参阅[度量数据模型规范](metrics/data-model.md)。

## 日志信号

### 数据模型

[日志数据模型](logs/data-model.md)定义了 OpenTelemetry 如何理解日志和事件。

## 行李信号

除了跟踪传播之外，OpenTelemetry 还提供了一种简单的机制来传播名称/值对，称为`Baggage` 。 `Baggage`旨在使用同一事务中先前服务提供的属性来索引一项服务中的可观察性事件。这有助于在这些事件之间建立因果关系。

虽然`Baggage`可用于制作其他横切关注点的原型，但该机制主要旨在传达 OpenTelemetry 可观察性系统的价值。

这些值可以从`Baggage`中使用，并用作指标的附加属性，或日志和跟踪的附加上下文。一些例子：

- Web 服务可以从包含发送请求的服务的上下文中受益
- SaaS 提供商可以包含有关负责该请求的 API 用户或令牌的上下文
- 确定特定浏览器版本与图像处理服务中的故障相关联

为了与 OpenTracing 向后兼容，使用 OpenTracing 网桥时，Baggage 作为`Baggage`传播。具有不同标准的新关注点应该考虑创建一个新的横切关注点来覆盖它们的用例；它们可能受益于 W3C 编码格式，但使用新的 HTTP 标头在整个分布式跟踪中传送数据。

## 资源

`Resource`捕获有关为其记录遥测的实体的信息。例如，Kubernetes 容器公开的指标可以链接到指定集群、命名空间、pod 和容器名称的资源。

`Resource`可以捕获实体标识的整个层次结构。它可以描述云中的主机和特定的容器或进程中运行的应用程序。

请注意，某些进程标识信息可以通过 OpenTelemetry SDK 或特定导出器自动与遥测相关联。有关示例，请参阅 OpenTelemetry [原型](https://github.com/open-telemetry/opentelemetry-proto/blob/a46c815aa5e85a52deb6cb35b8bc182fb3ca86a0/src/opentelemetry/proto/agent/common/v1/common.proto#L28-L96)。

## 上下文传播

所有 OpenTelemetry 横切关注点（例如跟踪和度量）共享一个底层`Context`机制，用于在分布式事务的整个生命周期内存储状态和访问数据。

查看[上下文](context/README.md)

## 传播者

OpenTelemetry 使用`Propagators`序列化和反序列化横切关注点值，例如`Span` （通常只有`SpanContext`部分）和`Baggage` 。不同的`Propagator`类型定义了特定传输施加的限制并绑定到数据类型。

Propagators API 当前定义了一种`Propagator`类型：

- `TextMapPropagator`将值注入载体并从载体中提取值作为文本。

## Collector

OpenTelemetry 收集器是一组组件，可以从 OpenTelemetry 或其他监控/跟踪库（Jaeger、Prometheus 等）检测的进程收集跟踪、指标和最终的其他遥测数据（例如日志），进行聚合和智能采样，以及将跟踪和指标导出到一个或多个监控/跟踪后端。收集器将允许丰富和转换收集的遥测数据（例如添加其他属性或清理个人信息）。

OpenTelemetry 收集器有两种主要的操作模式：代理（与应用程序一起在本地运行的守护程序）和收集器（独立运行的服务）。

在 OpenTelemetry 服务[长期愿景](https://github.com/open-telemetry/opentelemetry-collector/blob/master/docs/vision.md)中了解更多信息。

## 仪器库

见[仪器库](glossary.md#instrumentation-library)

该项目的灵感是通过让每个库和应用程序直接调用 OpenTelemetry API 来使它们开箱即用地可观察。然而，许多库不会有这样的集成，因此需要一个单独的库来注入这样的调用，使用包装接口、订阅特定于库的回调或将现有遥测数据转换为 OpenTelemetry 模型等机制。

为另一个库启用 OpenTelemetry 可观察性的库称为[Instrumentation Library](glossary.md#instrumentation-library) 。

检测库的命名应遵循检测库的任何命名约定（例如，Web 框架的“中间件”）。

如果没有确定的名称，建议在包前面加上“opentelemetry-instrumentation”，然后是检测的库名称本身。示例包括：

- opentelemetry-instrumentation-flask (Python)
- @opentelemetry/instrumentation-grpc (Javascript)

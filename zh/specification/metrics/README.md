<!---
linkTitle: Metrics
--->

# 开放遥测指标

<summary>目录</summary>

<!-- toc -->

- [概述](#overview)
    - [设计目标](#design-goals)
    - [概念](#concepts)
        - [API](#api)
        - [SDK](#sdk)
        - [编程模型](#programming-model)
- [规格](#specifications)
- [参考](#references)

<!-- tocstop -->




## 概述

### 设计目标

鉴于当今存在许多完善的指标解决方案，了解 OpenTelemetry 指标工作的目标很重要：

- **能够将指标连接到其他信号**。例如，可以通过示例关联度量和跟踪，可以通过[Baggage](../baggage/api.md)和[Context](../context/README.md)丰富度量属性。此外，[资源](../resource/sdk.md)可以以一致的方式应用于[日志](../overview.md#log-signal)/[指标](../overview.md#metric-signal)/[跟踪](../overview.md#tracing-signal)。

- **为[OpenCensus](https://opencensus.io/)客户提供迁移到 OpenTelemetry 的途径**。这是 OpenTelemetry 的最初目标——融合 OpenCensus 和 OpenTracing。我们将专注于提供语义和能力，而不是对 API 进行 1-1 映射。

- **使用现有的度量仪表协议和标准**。最低目标是为[Prometheus](https://prometheus.io/)和[StatsD](https://github.com/statsd/statsd)提供全面支持——用户应该能够使用 OpenTelemetry 客户端和[Collector](../overview.md#collector)来收集和导出指标，并能够实现与其原生客户端相同的功能。

### 概念

#### API

**OpenTelemetry Metrics API** （以下简称“API”）有两个用途：

- 有效地同时捕获原始测量值。
- 将检测与[SDK](#sdk)分离，允许在应用程序中指定/包含 SDK。

如果应用程序中没有明确包含/启用[SDK](#sdk) ，则不会收集遥测数据。有关更多信息，请参阅整体[OpenTelemetry API](../overview.md#api)概念和[API 和最小实现](../library-guidelines.md#api-and-minimal-implementation)。

#### SDK

**OpenTelemetry Metrics SDK** （以下简称“SDK”）实现 API，提供配置、聚合、处理器和导出器等功能和可扩展性。

OpenTelemetry 需要[将 API 与 SDK 分离](../library-guidelines.md#requirements)，以便可以在运行时配置不同的 SDK。有关更多信息，请参阅整体[OpenTelemetry SDK](../overview.md#sdk)概念。

#### 编程模型

```text
+------------------+
| MeterProvider    |                 +-----------------+             +--------------+
|   Meter A        | Measurements... |                 | Metrics...  |              |
|     Instrument X +-----------------> In-memory state +-------------> MetricReader |
|     Instrument Y |                 |                 |             |              |
|   Meter B        |                 +-----------------+             +--------------+
|     Instrument Z |
|     ...          |                 +-----------------+             +--------------+
|     ...          | Measurements... |                 | Metrics...  |              |
|     ...          +-----------------> In-memory state +-------------> MetricReader |
|     ...          |                 |                 |             |              |
|     ...          |                 +-----------------+             +--------------+
+------------------+
```

## 规格

- [指标 API](./api.md)
- [指标 SDK](./sdk.md)
- [度量数据模型和协议](./data-model.md)
- [语义约定](./semantic_conventions/README.md)

## 参考

- Metrics API/SDK 原型设计方案 ( [OTEP 146](https://github.com/open-telemetry/oteps/blob/main/text/metrics/0146-metrics-prototype-scenarios.md) )

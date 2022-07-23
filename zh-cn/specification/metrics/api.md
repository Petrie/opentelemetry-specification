<!--- Hugo front matter used to generate the website version of this page:
linkTitle: API
--->

# 指标 API

**状态**：[稳定](../document-status.md)

<summary>目录</summary>

<!-- toc -->

- [概述](#overview)
- [MeterProvider](#meterprovider)
    - [MeterProvider 操作](#meterprovider-operations)
        - [获取仪表](#get-a-meter)
- [仪表](#meter)
    - [仪表操作](#meter-operations)
- [仪表项](#instrument)
    - [一般特征](#general-characteristics)
        - [仪器类型冲突检测](#instrument-type-conflict-detection)
        - [仪表项命名空间](#instrument-namespace)
        - [仪表项命名规则](#instrument-naming-rule)
        - [仪表项单元](#instrument-unit)
        - [仪表项说明](#instrument-description)
        - [同步和异步仪表项](#synchronous-and-asynchronous-instruments)
        - [同步仪表项 API](#synchronous-instrument-api)
        - [异步仪表项仪器 API](#asynchronous-instrument-api)
    - [计数器](#counter)
        - [计数器创建](#counter-creation)
        - [计数器操作](#counter-operations)
            - [添加](#add)
    - [异步计数器](#asynchronous-counter)
        - [异步计数器创建](#asynchronous-counter-creation)
        - [异步计数器操作](#asynchronous-counter-operations)
    - [直方图](#histogram)
        - [直方图创建](#histogram-creation)
        - [直方图操作](#histogram-operations)
            - [记录](#record)
    - [异步Gauge](#asynchronous-gauge)
        - [异步Guage创建](#asynchronous-gauge-creation)
        - [异步Guage操作](#asynchronous-gauge-operations)
    - [上下计数器](#updowncounter)
        - [上下计数器创建](#updowncounter-creation)
        - [上下计数器操作](#updowncounter-operations)
            - [添加](#add-1)
    - [异步上下计数器](#asynchronous-updowncounter)
        - [异步上下计数器创建](#asynchronous-updowncounter-creation)
        - [异步上下计数器操作](#asynchronous-updowncounter-operations)
- [指标值](#measurement)
    - [多仪表项回调](#multiple-instrument-callbacks)
- [兼容性要求](#compatibility-requirements)
- [并发要求](#concurrency-requirements)

<!-- tocstop -->




## 概述

Metrics API 由以下主要组件组成：

- [MeterProvider](#meterprovider)是 API 的入口点。它提供对`Meters`的访问。
- [Meter](#meter)是负责创建`指标`的类。
- [指标](#instrument)负责报告[指标值](#measurement)。

以下是使用度量 API 检测的流程内的对象层次结构示例：

```text
+-- MeterProvider(default)
    |
    +-- Meter(name='io.opentelemetry.runtime', version='1.0.0')
    |   |
    |   +-- Instrument<Asynchronous Gauge, int>(name='cpython.gc', attributes=['generation'], unit='kB')
    |   |
    |   +-- instruments...
    |
    +-- Meter(name='io.opentelemetry.contrib.mongodb.client', version='2.3.0')
        |
        +-- Instrument<Counter, int>(name='client.exception', attributes=['type'], unit='1')
        |
        +-- Instrument<Histogram, double>(name='client.duration', attributes=['net.peer.host', 'net.peer.port'], unit='ms')
        |
        +-- instruments...

+-- MeterProvider(custom)
    |
    +-- Meter(name='bank.payment', version='23.3.5')
        |
        +-- instruments...
```

## MeterProvider

`Meter`可以通过`MeterProvider`访问。

在 API 的实现中， `MeterProvider`是保存所有配置的有状态对象。

通常，`MeterProvider`从中心位置访问。因此，API 应该提供一种设置/注册和访问全局默认`MeterProvider`的方法。

尽管有任何全局`MeterProvider` ，但某些应用程序可能希望或必须使用多个`MeterProvider`实例，例如为每个实例具有不同的配置，或者因为它更容易使用依赖注入框架。因此， `MeterProvider`的实现应该允许创建任意数量的`MeterProvider`实例。

### MeterProvider 操作

`MeterProvider`必须提供以下功能：

- 获取`Meter`

#### 获取仪表

此 API 必须接受以下参数：

- `name` （必需）：此名称应唯一标识[检测范围](../glossary.md#instrumentation-scope)，例如[检测库](../glossary.md#instrumentation-library)（例如`io.opentelemetry.contrib.mongodb` ）、包、模块或类名称。如果应用程序或库具有内置的 OpenTelemetry 检测，则检测[库](../glossary.md#instrumented-library)和检测[库](../glossary.md#instrumentation-library)都可以引用同一个库。在这种情况下， `name`表示该库或应用程序中的模块名称或组件名称。如果指定了无效的名称（null 或空字符串），工作的 Meter 实现必须作为后备返回，而不是返回 null 或抛出异常，它的`name`属性应该保留原始的无效值，并报告指定的消息值无效应该被记录。实现 OpenTelemetry API 的库*也*可以忽略此名称并为所有调用返回默认实例，如果它不支持“命名”功能（例如，甚至与可观察性无关的实现）。如果应用程序所有者将 SDK 配置为禁止此库生成的遥测数据，MeterProvider 也可以在此处返回无操作仪表。
- `version` （可选）：如果范围有版本（例如库版本），则指定检测范围的版本。示例值： `1.0.0` 。
- [从 1.4.0 开始] `schema_url` （可选）：指定应记录在发出的遥测数据中的架构 URL。
- [从 1.13.0 开始] `attributes` （可选）：指定与发出的遥测相关联的检测范围属性。

仪表由所有这些参数标识。

当使用不同的参数值重复调用时，实现必须返回不同的`Meter`实例。请注意，始终返回一个新的`Meter`实例是一个有效的实现。此规则的唯一例外是 no-op `Meter` ：无论参数值如何，实现都可以返回相同的实例。

当使用相同的（名称，版本，schema_url，属性）参数时，是否或在什么条件下从该函数返回相同或不同的`Meter`实例是未指定的。

应用于 Meters 的术语*相同*描述了所有标识字段都相同的情况。应用于 Meters 的术语*distinct*描述了至少一个标识字段具有不同值的情况。

实现不得要求用户重复获取具有相同身份的`Meter`以获取配置更改。这可以通过允许使用过时的配置或通过确保新配置也适用于先前返回的`Meter`来实现。

注意：例如，这可以通过在`MeterProvider`中存储任何可变配置并让`Meter`实现对象具有对从中获取它们的`MeterProvider`的引用来实现。如果必须按仪表存储配置（例如禁用某个仪表），例如，仪表可以在`MeterProvider`的地图中使用其身份进行查找，或者`MeterProvider`可以维护所有返回的`Meter`的注册表s 并在配置发生变化时主动更新其配置。

将 Schema URL 与`Meter`关联的效果必须是使用`Meter`发出的遥测将与 Schema URL 关联，前提是发出的数据格式能够表示这种关联。

## 仪表

仪表负责创建[指标](#instrument) 。

注意： `Meter`不应对负责配置信息。这应该是`MeterProvider`的责任。

### 仪表操作

`Meter`必须提供创建新[Instruments](#instrument)的功能：

- [创建一个新的计数器](#counter-creation)
- [创建一个新的异步计数器](#asynchronous-counter-creation)
- [创建一个新的直方图](#histogram-creation)
- [创建一个新的异步计量器](#asynchronous-gauge-creation)
- [创建一个新的上下计数器](#updowncounter-creation)
- [创建一个新的异步 上下计数器](#asynchronous-updowncounter-creation)

有关仪表项创建的更多信息，另请参阅下面的相应部分。

## 指标项（Instrument）

仪器用于报告[指标值](#measurement)。每个指标项将具有以下字段：

- 仪表项`名称`
- 仪表项的`类型` ——无论是[计数器](#counter)还是其他种类之一，是同步的还是异步的
- 可选的计量`unit`
- 可选`description`

Instruments 在创建期间与 Meter 关联。仪器由所有这些字段标识。

语言级别的特征，例如整数和浮点数之间的区别应该被认为是识别性的。

### 指标项的特点

#### 指标类型冲突检测

当为仪表 创建多个相同的 `name`的 仪表项 时，表示为*重复的仪表项注册* ，实现必须在每种情况下创建一个有效的 Instrument。在这里，“有效”是指一种可以正常工作并且可以导出数据的指标项，尽管可能会[在数据模型中产生语义错误](data-model.md#opentelemetry-protocol-data-model-producer-recommendations)。

由于重复的仪器注册，无论在任何情况下将返回相同或不同的仪器实例是未定义的。术语*相同*的含义表示所有字段都相同的情况。术语*不同*的含义表示至少一个字段值不同的情况。

当多个不同的仪表项以相同的`name`注册到相同的 Meters 时，实现应该向用户发出警告，通知他们重复注册冲突。当同一个 MeterProvider 为给定的指标项`name`和 Meter 标识写入多个`Metric`时，该警告有助于避免[OpenTelemetry Metrics 数据模型](data-model.md#opentelemetry-protocol-data-model-producer-recommendations)中描述的语义错误状态。

#### 指标项命名空间

为了检测[重复的仪表项注册冲突](#instrument-type-conflict-detection)，必须将不同的仪表视为单独的命名空间。

#### 指标项命名规则

仪器名称必须符合以下语法（使用[Augmented Backus-Naur 形式](https://tools.ietf.org/html/rfc5234)描述）：

```abnf
instrument-name = ALPHA 0*62 ("_" / "." / "-" / ALPHA / DIGIT)

ALPHA = %x41-5A / %x61-7A; A-Z / a-z
DIGIT = %x30-39 ; 0-9
```

- 它们不是 null 或空字符串。
- 它们是不区分大小写的 ASCII 字符串。
- 第一个字符必须是字母字符。
- 后续字符必须属于字母数字字符“_”、“.”和“-”。
- 它们的最大长度为 63 个字符。

#### 指标项单位

`unit`是仪器作者提供的可选字符串。它应该被视为来自 API 和 SDK 的不透明字符串（例如，SDK 不希望验证测量单位，或执行单位转换）。

- 如果未提供`unit`或`unit`为空，API 和 SDK 必须确保其行为与空`unit`字符串相同。
- 它必须区分大小写（例如`kb`和`kB`是不同的单位），ASCII 字符串。
- 它的最大长度为 63 个字符。选择数字 63 是为了在性能至关重要时允许将单元字符串（包括某些语言运行时上的`\0`终止符）作为固定大小的数组/结构进行存储和比较。

#### 指标项描述

`description`是仪器作者提供的可选自由格式文本。它必须被视为来自 API 和 SDK 的不透明字符串。

- 如果未提供`description`或`description`为空，API 和 SDK 必须确保行为与空`description`字符串相同。
- 它必须支持[BMP (Unicode Plane 0)](https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane) ，它基本上只是 UTF-8 的前三个字节（或`utf8mb3` ）。 [OpenTelemetry API](../overview.md#api)作者可以决定是否要支持更多的 Unicode[平面](https://en.wikipedia.org/wiki/Plane_(Unicode))。
- 它必须支持至少 1023 个字符。 [OpenTelemetry API](../overview.md#api)作者可以决定是否要支持更多。

仪器根据它们是同步的还是异步的进行分类：

#### 同步和异步指标项

- 同步指标项（例如[Counter](#counter) ）旨在与应用程序/业务处理逻辑先后调用。例如，HTTP 客户端可以使用 Counter 来记录它接收到的字节数。同步指标项记录的[测量](#measurement)值可以与[Context](../context/README.md)相关联。

- 异步指标项（例如[Asynchronous Gauge](#asynchronous-gauge) ）为用户提供了一种注册回调函数的方法，回调函数将仅在需要时调用（参见 SDK[集合](sdk.md#collect)以供参考）。例如，一个嵌入式软件可以使用异步指标项每 15 秒从传感器收集一次温度，这意味着回调函数只会每 15 秒调用一次。异步指标项记录的[测量](#measurement)值不能与[Context](../context/README.md)相关联。

请注意，术语*同步*和*异步*与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)无关。

#### <a>同步仪表项 API</a>

构建同步工具的 API 必须接受以下参数：

- 仪器的`name` ，遵循[仪器命名规则](#instrument-naming-rule)。
- 遵循[仪器单位规则](#instrument-unit)的可选计量`unit` 。
- 一个可选的`description` ，遵循[仪器描述规则](#instrument-description)。

#### <a>异步仪表项仪器 API</a>

异步仪器具有相关的`callback`函数，负责报告[Measurement](#measurement) 。只有在观察仪表时才会调用回调函数。未指定回调执行的顺序。

构建异步工具的 API 必须接受以下参数：

- 仪器的`name` ，遵循[仪器命名规则](#instrument-naming-rule)。
- 遵循[仪器单位规则](#instrument-unit)的可选计量`unit` 。
- 一个可选的`description` ，遵循[仪器描述规则](#instrument-description)。
- 零个或多个`callback`函数，负责报告已创建仪器的[测量](#measurement)值。

API 必须通过将零个或多个`callback`函数永久注册到新创建的工具来支持异步工具的创建。

回调是每次通过 OpenTelemetry API 注册`callback`函数时创建的概念上的实体。

API 应该支持在异步工具创建后注册`callback`函数。

如果 API 支持在异步检测创建后注册`callback`函数，用户必须能够在注册后通过某种方式撤消特定回调的注册。

每个当前注册的与一组仪器关联的回调必须在收集期间读取该指标项集合的数据之前准确评估一次。

回调函数必须为最终用户记录如下：

- 回调函数应该是可重入安全的。 SDK 期望独立评估每个 MetricReader 的回调。
- 回调函数不应该花费无限的时间。
- 回调函数不应该对所有注册的回调进行重复的读取数据（多个具有相同`attributes`的`Measurement` ）。

回调违反任何这些建议时的结果行为未在 API 级别明确指定。

[OpenTelemetry API](../overview.md#api)作者可以决定从回调函数中捕获指标值的惯用方法是什么。这里有些例子：

- 返回单个`Measurement`值的列表（或元组、生成器、枚举器等）。
- 将*Observable Result*作为回调的形式参数传递，其中`result.Observe()`捕获单个`Measurement`值。

在创建指标项目时注册的回调必须适用于正在构建的单个工具。

创建指标项后注册的回调可能与多个指标项相关联。

多指标项回调的惯用 API 必须区分与每个观察到的`指标值`值关联的仪器。

多仪器回调必须在注册时与来自同一`Meter`实例的一组声明的异步仪器相关联。 Instruments 以声明方式与 Callbacks 关联的这一要求允许 SDK 仅执行评估配置[View](sdk.md#view)正在使用的工具所必需的回调。

API 必须将来自单个回调的观察视为逻辑上发生在单个瞬间，这样当记录时，来自单个回调的观察必须以相同的时间戳报告。

API 应该提供一些将`state`传递给回调的方法。 [OpenTelemetry API](../overview.md#api)作者可以决定什么是惯用方法（例如，它可以是回调函数的附加参数，或者被 lambda 闭包捕获，或其他）。

### 计数器

`Counter`是一种支持非负增量的[同步工具](#synchronous-instrument-api)。

`Counter`的示例用法：

- 计算接收到的字节数
- 计算完成的请求数
- 计算创建的帐户数量
- 计算运行的检查点数
- 计算 HTTP 5xx 错误的数量

#### 计数器创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建`Counter` 。这可以被称为`CreateCounter` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64Counter` 、 `CreateDoubleCounter` 、 `CreateCounter<UInt64>` 、 `CreateCounter<double>` 。

参见[同步仪器的一般要求](#synchronous-instrument-api)。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

exception_counter = meter.create_counter(name="exceptions", description="number of exceptions caught", value_type=int)
```

```csharp
// C#

var counterExceptions = meter.CreateCounter<UInt64>("exceptions", description="number of exceptions caught");

readonly struct PowerConsumption
{
    [HighCardinality]
    string customer;
};

var counterPowerUsed = meter.CreateCounter<double, PowerConsumption>("power_consumption", unit="kWh");
```

#### 计数器操作

##### 添加

将 Counter 增加一个固定量。

此 API 不应返回值（如果某些编程语言或系统需要，它可能会返回一个假的值，例如`null` 、 `undefined` ）。

必需参数：

- 可选[属性](../common/README.md#attribute)。
- 增量数量，必须是非负数值。

[OpenTelemetry API](../overview.md#api)作者可以决定允许灵活的[属性](../common/README.md#attribute)作为参数传入。如果在[计数器创建](#counter-creation)期间提供了属性名称和类型， [OpenTelemetry API](../overview.md#api)作者可以允许使用更有效的方式传入属性值（例如，在调用堆栈上分配的强类型结构，元组）。 API 必须允许调用者在调用时提供灵活的属性，而不必在仪器创建期间注册所有可能的属性名称。以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

exception_counter.add(1, {"exception_type": "IOError", "handled_by_user": True})
exception_counter.add(1, exception_type="IOError", handled_by_user=True)
```

```csharp
// C#

counterExceptions.Add(1, ("exception_type", "FileLoadException"), ("handled_by_user", true));

counterPowerUsed.Add(13.5, new PowerConsumption { customer = "Tom" });
counterPowerUsed.Add(200, new PowerConsumption { customer = "Jerry" }, ("is_green_energy", true));
```

### 异步计数器

异步计数器是一种[异步指标](#asynchronous-instrument-api)，它在观察指标时报告[单调](https://wikipedia.org/wiki/Monotonic_function)递增的值。

异步计数器的示例用法：

- [CPU时间](https://wikipedia.org/wiki/CPU_time)，可以为每个线程、每个进程或整个系统报告。例如“在用户模式下运行的进程 A 的 CPU 时间，以秒为单位”。
- 每个进程的[页面错误](https://wikipedia.org/wiki/Page_fault)数。

#### 异步计数器创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建异步计数器。这可以被称为`CreateObservableCounter` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64ObservableCounter` 、 `CreateDoubleObservableCounter` 、 `CreateObservableCounter<UInt64>` 、 `CreateObservableCounter<double>` 。

强烈建议实现使用名称`ObservableCounter` （或任何语言惯用变体，例如`observable_counter` ），除非有充分的理由不这样做。请注意，名称与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)和[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)无关。

请参阅[异步指标的一般要求](#asynchronous-instrument-api)。

注意：与接受增量值的[Counter.Add()](#add)不同，回调函数报告计数器的绝对值。计数器改变的报告速率，决定了连续指标值之间的差异。

[OpenTelemetry API](../overview.md#api)作者可以决定什么是惯用的方法。这里有些例子：

- 返回`Measurement`的列表（或元组、生成器、枚举器等）。
- 使用 observable result 参数来允许报告单个`Measurement` 。

建议用户代码不要在单个回调中提供多个具有相同`attributes`的`Measurement` 。如果发生这种情况， [OpenTelemetry SDK](../overview.md#sdk)作者可以决定如何在[SDK](./README.md#sdk)中处理它。例如，在回调调用期间，如果报告了两个测量`value=1, attributes={pid:4, bitness:64}`和`value=2, attributes={pid:4, bitness:64}` ， [OpenTelemetry SDK](../overview.md#sdk)作者可以决定只需让它们通过（以便下游消费者可以处理重复）、删除整个数据，选择最后一个，或其他。 API 必须将来自单个回调的观察视为逻辑上发生在单个瞬间，这样当记录时，来自单个回调的观察必须以相同的时间戳报告。

API 应该提供一些将`state`传递给回调的方法。 [OpenTelemetry API](../overview.md#api)作者可以决定什么是惯用方法（例如，它可以是回调函数的附加参数，或者被 lambda 闭包捕获，或其他）。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

def pf_callback():
    # Note: in the real world these would be retrieved from the operating system
    return (
        (8,        ("pid", 0),   ("bitness", 64)),
        (37741921, ("pid", 4),   ("bitness", 64)),
        (10465,    ("pid", 880), ("bitness", 32)),
    )

meter.create_observable_counter(name="PF", description="process page faults", pf_callback)
```

```python
# Python

def pf_callback(result):
    # Note: in the real world these would be retrieved from the operating system
    result.Observe(8,        ("pid", 0),   ("bitness", 64))
    result.Observe(37741921, ("pid", 4),   ("bitness", 64))
    result.Observe(10465,    ("pid", 880), ("bitness", 32))

meter.create_observable_counter(name="PF", description="process page faults", pf_callback)
```

```csharp
// C#

// A simple scenario where only one value is reported

interface IAtomicClock
{
    UInt64 GetCaesiumOscillates();
}

IAtomicClock clock = AtomicClock.Connect();

meter.CreateObservableCounter<UInt64>("caesium_oscillates", () => clock.GetCaesiumOscillates());
```

#### 异步计数器操作

异步计数器使用惯用接口通过`callback`报告测量值，该回调在[异步计数器创建](#asynchronous-counter-creation)期间注册。

对于异步指标项创建后注册的回调函数，API 需要支持注销机制。例如，从`register_callback`返回的对象可以直接支持`unregister()`方法。

```python
# Python
class Device:
    """A device with one counter"""

    def __init__(self, meter, x):
        self.x = x
        counter = meter.create_observable_counter(name="usage", description="count of items used")
        self.cb = counter.register_callback(self.counter_callback)

    def counter_callback(self, result):
        result.Observe(self.read_counter(), {'x', self.x})

    def read_counter(self):
        return 100  # ...

    def stop(self):
        self.cb.unregister()
```

### 直方图

`Histogram`是一种[同步指标项目](#synchronous-instrument-api)，可用于报告可能具有统计意义的任意值。它适用于统计数据，例如直方图、聚合数据和百分位数。

`Histogram`的示例用法：

- 请求持续时间
- 响应负载的大小

#### 直方图创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建`Histogram` 。这可能被称为`CreateHistogram` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64Histogram` 、 `CreateDoubleHistogram` 、 `CreateHistogram<UInt64>` 、 `CreateHistogram<double>` 。

参见[同步仪器的一般要求](#synchronous-instrument-api)。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

http_server_duration = meter.create_histogram(
    name="http.server.duration",
    description="measures the duration of the inbound HTTP request",
    unit="milliseconds",
    value_type=float)
```

```csharp
// C#

var httpServerDuration = meter.CreateHistogram<double>(
    "http.server.duration",
    description: "measures the duration of the inbound HTTP request",
    unit: "milliseconds"
    );
```

#### 直方图操作

##### 记录

使用指定数量更新统计信息。

此 API 不应返回值（如果某些编程语言或系统需要，它可能会返回一个虚拟值，例如`null` 、 `undefined` ）。

参数：

- `Measurement`的数量，必须是非负数值。
- 可选[属性](../common/README.md#attribute)。

[OpenTelemetry API](../overview.md#api)作者可以决定允许灵活的[属性](../common/README.md#attribute)作为单独的参数传入。 [OpenTelemetry API](../overview.md#api)作者可以允许使用更有效的方式传入属性值（例如，在调用堆栈上分配的强类型结构、元组）。以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

http_server_duration.Record(50, {"http.method": "POST", "http.scheme": "https"})
http_server_duration.Record(100, http_method="GET", http_scheme="http"})
```

```csharp
// C#

httpServerDuration.Record(50, ("http.method", "POST"), ("http.scheme", "https"));
httpServerDuration.Record(100, new HttpRequestAttributes { method = "GET", scheme = "http" });
```

### 异步计量器

异步计量器是一种[异步指标](#asynchronous-instrument-api)，它在观察指标时报告非相加值（例如室温 - 报告多个房间的温度值并将它们相加是没有意义的）。

注意：如果这些值是相加的（例如进程堆大小 - 报告多个进程的堆大小并将它们相加是有意义的，因此我们得到总堆使用量），请使用[异步计数器](#asynchronous-counter)或[异步 UpDownCounter](#asynchronous-updowncounter) 。

异步仪表的示例用法：

- 当前室温
- CPU风扇转速

#### 异步计量器创建

除了使用[`Meter`](#meter)之外，不得有任何 API 用于创建异步 Gauge。这可能被称为`CreateObservableGauge` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64ObservableGauge` 、 `CreateDoubleObservableGauge` 、 `CreateObservableGauge<UInt64>` 、 `CreateObservableGauge<double>` 。

强烈建议实现使用名称`ObservableGauge` （或任何语言惯用变体，例如`observable_gauge` ），除非有充分的理由不这样做。请注意，名称与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)和[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)无关。

请参阅[异步仪器的一般要求](#asynchronous-instrument-api)。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

def cpu_frequency_callback():
    # Note: in the real world these would be retrieved from the operating system
    return (
        (3.38, ("cpu", 0), ("core", 0)),
        (3.51, ("cpu", 0), ("core", 1)),
        (0.57, ("cpu", 1), ("core", 0)),
        (0.56, ("cpu", 1), ("core", 1)),
    )

meter.create_observable_gauge(
    name="cpu.frequency",
    description="the real-time CPU clock speed",
    callback=cpu_frequency_callback,
    unit="GHz",
    value_type=float)
```

```python
# Python

def cpu_frequency_callback(result):
    # Note: in the real world these would be retrieved from the operating system
    result.Observe(3.38, ("cpu", 0), ("core", 0))
    result.Observe(3.51, ("cpu", 0), ("core", 1))
    result.Observe(0.57, ("cpu", 1), ("core", 0))
    result.Observe(0.56, ("cpu", 1), ("core", 1))

meter.create_observable_gauge(
    name="cpu.frequency",
    description="the real-time CPU clock speed",
    callback=cpu_frequency_callback,
    unit="GHz",
    value_type=float)
```

```csharp
// C#

// A simple scenario where only one value is reported

meter.CreateObservableGauge<double>("temperature", () => sensor.GetTemperature());
```

#### 异步计量器操作

异步计量器使用惯用接口通过`callback`报告测量值，回调是在[异步计量器创建](#asynchronous-gauge-creation)期间注册的。

对于异步工具创建后注册的回调函数，API 需要支持注销机制。例如，从`register_callback`返回的对象可以直接支持`unregister()`方法。

```python
# Python
class Device:
    """A device with one gauge"""

    def __init__(self, meter, x):
        self.x = x
        gauge = meter.create_observable_gauge(name="pressure", description="force/area")
        self.cb = gauge.register_callback(self.gauge_callback)

    def gauge_callback(self, result):
        result.Observe(self.read_gauge(), {'x', self.x})

    def read_gauge(self):
        return 100  # ...

    def stop(self):
        self.cb.unregister()
```

### 上下计数器

`UpDownCounter`是一个支持递增和递减的[同步指标](#synchronous-instrument-api) 。

注意：如果值[单调](https://wikipedia.org/wiki/Monotonic_function)递增，请改用[Counter](#counter) 。

`UpDownCounter`的示例用法：

- 活跃请求数
- 队列中的项目数

`UpDownCounter`适用于未预先计算绝对值或获取“当前值”需要额外工作的情况。如果预先计算的值已经可用或获取“当前值”的快照很简单，请改用[异步 UpDownCounter](#asynchronous-updowncounter) 。

UpDownCounter 支持递增地计算**集合的大小**，例如在添加和删除时通过“颜色”和“材料”属性报告并发包中的项目数。

颜色 | 材料 | 数数
--- | --- | ---
红色的 | 铝 | 1
红色的 | 钢 | 2
蓝色的 | 铝 | 0
蓝色的 | 钢 | 5
黄色 | 铝 | 0
黄色 | 钢 | 3

```python
# Python

items_counter = meter.create_up_down_counter(
    name="store.inventory",
    description="the number of the items available")

def restock_item(color, material):
    inventory.add_item(color=color, material=material)
    items_counter.add(1, {"color": color, "material": material})
    return true

def sell_item(color, material):
    succeeded = inventory.take_item(color=color, material=material)
    if succeeded:
        items_counter.add(-1, {"color": color, "material": material})
    return succeeded
```

#### UpDownCounter 创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建`UpDownCounter` 。这可以称为`CreateUpDownCounter` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateInt64UpDownCounter` 、 `CreateDoubleUpDownCounter` 、 `CreateUpDownCounter<Int64>` 、 `CreateUpDownCounter<double>` 。

参见[同步仪器的一般要求](#synchronous-instrument-api)。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

customers_in_store = meter.create_up_down_counter(
    name="grocery.customers",
    description="measures the current customers in the grocery store",
    value_type=int)
```

```csharp
// C#

var customersInStore = meter.CreateUpDownCounter<int>(
    "grocery.customers",
    description: "measures the current customers in the grocery store",
    );
```

#### UpDownCounter 操作

##### 添加

以固定数量递增或递减 UpDownCounter。

此 API 不应返回值（如果某些编程语言或系统需要，它可能会返回一个虚拟值，例如`null` 、 `undefined` ）。

参数：

- 要添加的数量可以是正数、负数或零。
- 可选[属性](../common/README.md#attribute)。

[OpenTelemetry API](../overview.md#api)作者可以决定允许灵活的[属性](../common/README.md#attribute)作为单独的参数传入。 [OpenTelemetry API](../overview.md#api)作者可以允许使用更有效的方式传入属性值（例如，在调用堆栈上分配的强类型结构、元组）。以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python
customers_in_store.add(1, {"account.type": "commercial"})
customers_in_store.add(-1, account_type="residential")
```

```csharp
// C#
customersInStore.Add(1, ("account.type", "commercial"));
customersInStore.Add(-1, new Account { Type = "residential" });
```

### 异步 UpDownCounter

异步 UpDownCounter 是一种[异步指标](#asynchronous-instrument-api)，它在观察指标时报告附加值（例如进程堆大小 - 报告来自多个进程的堆大小并将它们相加是有意义的，因此我们得到总堆使用量） .

注意：如果值[单调](https://wikipedia.org/wiki/Monotonic_function)递增，请改用[异步计数器](#asynchronous-counter)；如果该值是非相加的，请改用[Asynchronous Gauge](#asynchronous-gauge) 。

异步 UpDownCounter 的示例用法：

- 进程堆大小
- 无锁循环缓冲区中的近似项数

#### 异步 UpDownCounter 创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建异步 UpDownCounter。这可能被称为`CreateObservableUpDownCounter` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64ObservableUpDownCounter` 、 `CreateDoubleObservableUpDownCounter` 、 `CreateObservableUpDownCounter<UInt64>` 、 `CreateObservableUpDownCounter<double>` 。

强烈建议实现使用名称`ObservableUpDownCounter` （或任何语言惯用变体，例如`observable_updowncounter` ），除非有充分的理由不这样做。请注意，名称与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)和[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)无关。

请参阅[异步仪器的一般要求](#asynchronous-instrument-api)。

注意：与采用增量值的[UpDownCounter.Add()](#add-1)不同，回调函数报告异步 UpDownCounter 的绝对值。为了确定异步 UpDownCounter 正在改变的报告速率，使用了连续测量之间的差异。

以下是[OpenTelemetry API](../overview.md#api)作者可能会考虑的一些示例：

```python
# Python

def ws_callback():
    # Note: in the real world these would be retrieved from the operating system
    return (
        (8,      ("pid", 0),   ("bitness", 64)),
        (20,     ("pid", 4),   ("bitness", 64)),
        (126032, ("pid", 880), ("bitness", 32)),
    )

meter.create_observable_updowncounter(
    name="process.workingset",
    description="process working set",
    callback=ws_callback,
    unit="kB",
    value_type=int)
```

```python
# Python

def ws_callback(result):
    # Note: in the real world these would be retrieved from the operating system
    result.Observe(8,      ("pid", 0),   ("bitness", 64))
    result.Observe(20,     ("pid", 4),   ("bitness", 64))
    result.Observe(126032, ("pid", 880), ("bitness", 32))

meter.create_observable_updowncounter(
    name="process.workingset",
    description="process working set",
    callback=ws_callback,
    unit="kB",
    value_type=int)
```

```csharp
// C#

// A simple scenario where only one value is reported

meter.CreateObservableUpDownCounter<UInt64>("memory.physical.free", () => WMI.Query("FreePhysicalMemory"));
```

#### 异步 UpDownCounter 操作

异步 UpDownCounter 使用惯用接口通过`callback`报告测量值，回调在[异步 Updowncounter 创建](#asynchronous-updowncounter-creation)期间注册。

对于异步工具创建后注册的回调函数，API 需要支持注销机制。例如，从`register_callback`返回的对象可以直接支持`unregister()`方法。

```python
# Python
class Device:
    """A device with one updowncounter"""

    def __init__(self, meter, x):
        self.x = x
        updowncounter = meter.create_observable_updowncounter(name="queue_size", description="items in process")
        self.cb = updowncounter.register_callback(self.updowncounter_callback)

    def updowncounter_callback(self, result):
        result.Observe(self.read_updowncounter(), {'x', self.x})

    def read_updowncounter(self):
        return 100  # ...

    def stop(self):
        self.cb.unregister()
```

## <a>指标值</a>

`Measurement`表示通过指标 API 报告给 SDK 的数据点。 API与SDK的交互请参考[Metrics Programming Model](./README.md#programming-model) 。

`Measurement`封装：

- 一个值
- [`Attributes`](../common/README.md#attribute)

### 多指标项回调

[Metrics API 可以支持一个接口，允许从一个注册的回调中使用多个指标项](#asynchronous-instrument-api)。注册新回调的 API 应该接受：

- `callback`函数
- `callback`函数中使用的 Instruments 列表（或元组等）。

建议 API 作者对`callback`函数使用以下形式之一：

- `callback`函数返回的列表（或元组等）包含`(Instrument, Measurement)`对。
- Observable Result 参数接收额外的`(Instrument, Measurement)`对

当通过昂贵的过程（例如读取`/proc`文件或探测垃圾收集子系统）获得多个测量值时，此接口通常是报告多个测量值的更高效的方式。

例如，

```Python
# Python
class Device:
    """A device with two instruments"""

    def __init__(self, meter, property):
        self.property = property
        self.usage = meter.create_observable_counter(name="usage", description="count of items used")
        self.pressure = meter.create_observable_gauge(name="pressure", description="force per unit area")

        # Note the two associated instruments are passed to the callback.
        meter.register_callback([self.usage, self.pressure], self.observe)

    def observe(self, result):
        usage, pressure = expensive_system_call()
        result.observe(self.usage, usage, {'property', self.property})
        result.observe(self.pressure, pressure, {'property', self.property})
```

## 兼容性要求

所有指标组件都应该允许将新的 API 添加到现有组件中，而不会引入重大更改。

如果可能，所有指标 API 都应该允许将可选参数添加到现有 API 而不会引入重大更改。

## 并发要求

对于支持并发执行的语言，Metrics API 提供特定的保证和安全性。

**MeterProvider** - 所有方法都可以安全地同时调用。

**Meter** - 所有方法都可以安全地同时调用。

**Instrument** - 任何 Instrument 的所有方法都可以安全地同时调用。

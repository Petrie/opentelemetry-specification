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
- [Instrument](#instrument)
    - [一般特征](#general-characteristics)
        - [仪器类型冲突检测](#instrument-type-conflict-detection)
        - [Instrument namespace](#instrument-namespace)
        - [Instrument naming rule](#instrument-naming-rule)
        - [Instrument unit](#instrument-unit)
        - [Instrument description](#instrument-description)
        - [Synchronous and Asynchronous instruments](#synchronous-and-asynchronous-instruments)
        - [Synchronous Instrument API](#synchronous-instrument-api)
        - [Asynchronous Instrument API](#asynchronous-instrument-api)
    - [Counter](#counter)
        - [计数器创建](#counter-creation)
        - [Counter operations](#counter-operations)
            - [添加](#add)
    - [异步计数器](#asynchronous-counter)
        - [异步计数器创建](#asynchronous-counter-creation)
        - [异步计数器操作](#asynchronous-counter-operations)
    - [直方图](#histogram)
        - [直方图创建](#histogram-creation)
        - [直方图操作](#histogram-operations)
            - [记录](#record)
    - [Asynchronous Gauge](#asynchronous-gauge)
        - [Asynchronous Gauge creation](#asynchronous-gauge-creation)
        - [Asynchronous Gauge operations](#asynchronous-gauge-operations)
    - [上下计数器](#updowncounter)
        - [UpDownCounter creation](#updowncounter-creation)
        - [UpDownCounter operations](#updowncounter-operations)
            - [添加](#add-1)
    - [Asynchronous UpDownCounter](#asynchronous-updowncounter)
        - [Asynchronous UpDownCounter creation](#asynchronous-updowncounter-creation)
        - [Asynchronous UpDownCounter operations](#asynchronous-updowncounter-operations)
- [Measurement](#measurement)
    - [Multiple-instrument callbacks](#multiple-instrument-callbacks)
- [兼容性要求](#compatibility-requirements)
- [并发要求](#concurrency-requirements)

<!-- tocstop -->




## 概述

Metrics API 由以下主要组件组成：

- [MeterProvider](#meterprovider)是 API 的入口点。它提供对`Meters`的访问。
- [Meter](#meter) is the class responsible for creating `Instruments`.
- [Instrument](#instrument) is responsible for reporting [Measurements](#measurement).

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

In implementations of the API, the `MeterProvider` is expected to be the stateful object that holds any configuration.

Normally, the `MeterProvider` is expected to be accessed from a central place. Thus, the API SHOULD provide a way to set/register and access a global default `MeterProvider`.

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

The meter is responsible for creating [Instruments](#instrument).

Note: `Meter` SHOULD NOT be responsible for the configuration. This should be the responsibility of the `MeterProvider` instead.

### 仪表操作

`Meter`必须提供创建新[Instruments](#instrument)的功能：

- [创建一个新的计数器](#counter-creation)
- [创建一个新的异步计数器](#asynchronous-counter-creation)
- [创建一个新的直方图](#histogram-creation)
- [Create a new Asynchronous Gauge](#asynchronous-gauge-creation)
- [Create a new UpDownCounter](#updowncounter-creation)
- [Create a new Asynchronous UpDownCounter](#asynchronous-updowncounter-creation)

Also see the respective sections below for more information on instrument creation.

## Instrument

Instruments are used to report [Measurements](#measurement). Each Instrument will have the following fields:

- The `name` of the Instrument
- The `kind` of the Instrument - whether it is a [Counter](#counter) or one of the other kinds, whether it is synchronous or asynchronous
- 可选的计量`unit`
- 可选`description`

Instruments 在创建期间与 Meter 关联。仪器由所有这些字段标识。

语言级别的特征，例如整数和浮点数之间的区别应该被认为是识别性的。

### General characteristics

#### Instrument type conflict detection

When more than one Instrument of the same `name` is created for identical Meters, denoted *duplicate instrument registration*, the implementation MUST create a valid Instrument in every case.  Here, "valid" means an instrument that is functional and can be expected to export data, despite potentially creating a [semantic error in the data model](data-model.md#opentelemetry-protocol-data-model-producer-recommendations).

It is unspecified whether or under which conditions the same or different Instrument instance will be returned as a result of duplicate instrument registration.  The term *identical* applied to Instruments describes instances where all identifying fields are equal.  The term *distinct* applied to Instruments describes instances where at least one field value is different.

When more than one distinct Instrument is registered with the same `name` for identical Meters, the implementation SHOULD emit a warning to the user informing them of duplicate registration conflict(s). The warning helps to avoid the semantic error state described in the [OpenTelemetry Metrics data model](data-model.md#opentelemetry-protocol-data-model-producer-recommendations) when more than one `Metric` is written for a given instrument `name` and Meter identity by the same MeterProvider.

#### Instrument namespace

Distinct Meters MUST be treated as separate namespaces for the purposes of detecting [duplicate instrument registration conflicts](#instrument-type-conflict-detection).

#### Instrument naming rule

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

#### Instrument unit

`unit`是仪器作者提供的可选字符串。它应该被视为来自 API 和 SDK 的不透明字符串（例如，SDK 不希望验证测量单位，或执行单位转换）。

- 如果未提供`unit`或`unit`为空，API 和 SDK 必须确保其行为与空`unit`字符串相同。
- 它必须区分大小写（例如`kb`和`kB`是不同的单位），ASCII 字符串。
- 它的最大长度为 63 个字符。选择数字 63 是为了在性能至关重要时允许将单元字符串（包括某些语言运行时上的`\0`终止符）作为固定大小的数组/结构进行存储和比较。

#### Instrument description

`description`是仪器作者提供的可选自由格式文本。它必须被视为来自 API 和 SDK 的不透明字符串。

- 如果未提供`description`或`description`为空，API 和 SDK 必须确保行为与空`description`字符串相同。
- 它必须支持[BMP (Unicode Plane 0)](https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane) ，它基本上只是 UTF-8 的前三个字节（或`utf8mb3` ）。 [OpenTelemetry API](../overview.md#api)作者可以决定是否要支持更多的 Unicode[平面](https://en.wikipedia.org/wiki/Plane_(Unicode))。
- 它必须支持至少 1023 个字符。 [OpenTelemetry API](../overview.md#api)作者可以决定是否要支持更多。

仪器根据它们是同步的还是异步的进行分类：

#### Synchronous and Asynchronous instruments

- Synchronous instruments (e.g. [Counter](#counter)) are meant to be invoked inline with application/business processing logic. For example, an HTTP client could use a Counter to record the number of bytes it has received. [Measurements](#measurement) recorded by synchronous instruments can be associated with the [Context](../context/README.md).

- Asynchronous instruments (e.g. [Asynchronous Gauge](#asynchronous-gauge)) give the user a way to register callback function, and the callback function will be invoked only on demand (see SDK [collection](sdk.md#collect) for reference). For example, a piece of embedded software could use an asynchronous gauge to collect the temperature from a sensor every 15 seconds, which means the callback function will only be invoked every 15 seconds. [Measurements](#measurement) recorded by asynchronous instruments cannot be associated with the [Context](../context/README.md).

请注意，术语*同步*和*异步*与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)无关。

#### Synchronous Instrument API

构建同步工具的 API 必须接受以下参数：

- 仪器的`name` ，遵循[仪器命名规则](#instrument-naming-rule)。
- 遵循[仪器单位规则](#instrument-unit)的可选计量`unit` 。
- 一个可选的`description` ，遵循[仪器描述规则](#instrument-description)。

#### Asynchronous Instrument API

异步仪器具有相关的`callback`函数，负责报告[Measurement](#measurement) 。只有在观察仪表时才会调用回调函数。未指定回调执行的顺序。

构建异步工具的 API 必须接受以下参数：

- 仪器的`name` ，遵循[仪器命名规则](#instrument-naming-rule)。
- 遵循[仪器单位规则](#instrument-unit)的可选计量`unit` 。
- 一个可选的`description` ，遵循[仪器描述规则](#instrument-description)。
- 零个或多个`callback`函数，负责报告已创建仪器的[测量](#measurement)值。

API 必须通过将零个或多个`callback`函数永久注册到新创建的工具来支持异步工具的创建。

A Callback is the conceptual entity created each time a `callback` function is registered through an OpenTelemetry API.

API 应该支持在异步工具创建后注册`callback`函数。

如果 API 支持在异步检测创建后注册`callback`函数，用户必须能够在注册后通过某种方式撤消特定回调的注册。

Every currently registered Callback associated with a set of instruments MUST be evaluated exactly once during collection prior to reading data for that instrument set.

回调函数必须为最终用户记录如下：

- 回调函数应该是可重入安全的。 SDK 期望独立评估每个 MetricReader 的回调。
- 回调函数不应该花费无限的时间。
- Callback functions SHOULD NOT make duplicate observations (more than one `Measurement` with the same `attributes`) across all registered callbacks.

回调违反任何这些建议时的结果行为未在 API 级别明确指定。

[OpenTelemetry API](../overview.md#api) authors MAY decide what is the idiomatic approach for capturing measurements from callback functions. Here are some examples:

- 返回单个`Measurement`值的列表（或元组、生成器、枚举器等）。
- 将*Observable Result*作为回调的形式参数传递，其中`result.Observe()`捕获单个`Measurement`值。

Callbacks registered at the time of instrument creation MUST apply to the single instruments which is under construction.

Callbacks registered after the time of instrument creation MAY be associated with multiple instruments.

Idiomatic APIs for multiple-instrument Callbacks MUST distinguish the instrument associated with each observed `Measurement` value.

多仪器回调必须在注册时与来自同一`Meter`实例的一组声明的异步仪器相关联。 Instruments 以声明方式与 Callbacks 关联的这一要求允许 SDK 仅执行评估配置[View](sdk.md#view)正在使用的工具所必需的回调。

API 必须将来自单个回调的观察视为逻辑上发生在单个瞬间，这样当记录时，来自单个回调的观察必须以相同的时间戳报告。

API 应该提供一些将`state`传递给回调的方法。 [OpenTelemetry API](../overview.md#api)作者可以决定什么是惯用方法（例如，它可以是回调函数的附加参数，或者被 lambda 闭包捕获，或其他）。

### Counter

`Counter`是一种支持非负增量的[同步工具](#synchronous-instrument-api)。

`Counter`的示例用法：

- 计算接收到的字节数
- 计算完成的请求数
- 计算创建的帐户数量
- 计算运行的检查点数
- 计算 HTTP 5xx 错误的数量

#### 计数器创建

There MUST NOT be any API for creating a `Counter` other than with a [`Meter`](#meter). This MAY be called `CreateCounter`. If strong type is desired, [OpenTelemetry API](../overview.md#api) authors MAY decide the language idiomatic name(s), for example `CreateUInt64Counter`, `CreateDoubleCounter`, `CreateCounter<UInt64>`, `CreateCounter<double>`.

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

#### Counter operations

##### 添加

将 Counter 增加一个固定量。

This API SHOULD NOT return a value (it MAY return a dummy value if required by certain programming languages or systems, for example `null`, `undefined`).

必需参数：

- 可选[属性](../common/README.md#attribute)。
- The increment amount, which MUST be a non-negative numeric value.

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

Asynchronous Counter is an [asynchronous Instrument](#asynchronous-instrument-api) which reports [monotonically](https://wikipedia.org/wiki/Monotonic_function) increasing value(s) when the instrument is being observed.

异步计数器的示例用法：

- [CPU时间](https://wikipedia.org/wiki/CPU_time)，可以为每个线程、每个进程或整个系统报告。例如“在用户模式下运行的进程 A 的 CPU 时间，以秒为单位”。
- 每个进程的[页面错误](https://wikipedia.org/wiki/Page_fault)数。

#### 异步计数器创建

There MUST NOT be any API for creating an Asynchronous Counter other than with a [`Meter`](#meter). This MAY be called `CreateObservableCounter`. If strong type is desired, [OpenTelemetry API](../overview.md#api) authors MAY decide the language idiomatic name(s), for example `CreateUInt64ObservableCounter`, `CreateDoubleObservableCounter`, `CreateObservableCounter<UInt64>`, `CreateObservableCounter<double>`.

强烈建议实现使用名称`ObservableCounter` （或任何语言惯用变体，例如`observable_counter` ），除非有充分的理由不这样做。请注意，名称与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)和[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)无关。

See the [general requirements for asynchronous instruments](#asynchronous-instrument-api).

Note: Unlike [Counter.Add()](#add) which takes the increment/delta value, the callback function reports the absolute value of the counter. To determine the reported rate the counter is changing, the difference between successive measurements is used.

[OpenTelemetry API](../overview.md#api)作者可以决定什么是惯用的方法。这里有些例子：

- 返回`Measurement`的列表（或元组、生成器、枚举器等）。
- 使用 observable result 参数来允许报告单个`Measurement` 。

User code is recommended not to provide more than one `Measurement` with the same `attributes` in a single callback. If it happens, [OpenTelemetry SDK](../overview.md#sdk) authors MAY decide how to handle it in the [SDK](./README.md#sdk). For example, during the callback invocation if two measurements `value=1, attributes={pid:4, bitness:64}` and `value=2, attributes={pid:4, bitness:64}` are reported, [OpenTelemetry SDK](../overview.md#sdk) authors MAY decide to simply let them pass through (so the downstream consumer can handle duplication), drop the entire data, pick the last one, or something else. The API MUST treat observations from a single callback as logically taking place at a single instant, such that when recorded, observations from a single callback MUST be reported with identical timestamps.

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

For callback functions registered after an asynchronous instrument is created, the API is required to support a mechanism for unregistration.  For example, the object returned from `register_callback` can support an `unregister()` method directly.

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

`Histogram` is a [synchronous Instrument](#synchronous-instrument-api) which can be used to report arbitrary values that are likely to be statistically meaningful. It is intended for statistics such as histograms, summaries, and percentile.

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

### Asynchronous Gauge

Asynchronous Gauge is an [asynchronous Instrument](#asynchronous-instrument-api) which reports non-additive value(s) (e.g. the room temperature - it makes no sense to report the temperature value from multiple rooms and sum them up) when the instrument is being observed.

注意：如果这些值是相加的（例如进程堆大小 - 报告多个进程的堆大小并将它们相加是有意义的，因此我们得到总堆使用量），请使用[异步计数器](#asynchronous-counter)或[异步 UpDownCounter](#asynchronous-updowncounter) 。

异步仪表的示例用法：

- 当前室温
- CPU风扇转速

#### Asynchronous Gauge creation

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

#### Asynchronous Gauge operations

Asynchronous Gauge uses an idiomatic interface for reporting measurements through a `callback`, which is registered during [Asynchronous Gauge creation](#asynchronous-gauge-creation).

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

`UpDownCounter` is a [synchronous Instrument](#synchronous-instrument-api) which supports increments and decrements.

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

Asynchronous UpDownCounter is an [asynchronous Instrument](#asynchronous-instrument-api) which reports additive value(s) (e.g. the process heap size - it makes sense to report the heap size from multiple processes and sum them up, so we get the total heap usage) when the instrument is being observed.

注意：如果值[单调](https://wikipedia.org/wiki/Monotonic_function)递增，请改用[异步计数器](#asynchronous-counter)；如果该值是非相加的，请改用[Asynchronous Gauge](#asynchronous-gauge) 。

异步 UpDownCounter 的示例用法：

- 进程堆大小
- 无锁循环缓冲区中的近似项数

#### 异步 UpDownCounter 创建

除了[`Meter`](#meter)之外，不得有任何 API 用于创建异步 UpDownCounter。这可能被称为`CreateObservableUpDownCounter` 。如果需要强类型， [OpenTelemetry API](../overview.md#api)作者可以决定语言惯用名称，例如`CreateUInt64ObservableUpDownCounter` 、 `CreateDoubleObservableUpDownCounter` 、 `CreateObservableUpDownCounter<UInt64>` 、 `CreateObservableUpDownCounter<double>` 。

强烈建议实现使用名称`ObservableUpDownCounter` （或任何语言惯用变体，例如`observable_updowncounter` ），除非有充分的理由不这样做。请注意，名称与[异步模式](https://en.wikipedia.org/wiki/Asynchronous_method_invocation)和[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)无关。

请参阅[异步仪器的一般要求](#asynchronous-instrument-api)。

Note: Unlike [UpDownCounter.Add()](#add-1) which takes the increment/delta value, the callback function reports the absolute value of the Asynchronous UpDownCounter. To determine the reported rate the Asynchronous UpDownCounter is changing, the difference between successive measurements is used.

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

## Measurement

`Measurement`表示通过指标 API 报告给 SDK 的数据点。 API与SDK的交互请参考[Metrics Programming Model](./README.md#programming-model) 。

`Measurement`封装：

- 一个值
- [`Attributes`](../common/README.md#attribute)

### Multiple-instrument callbacks

[The Metrics API MAY support an interface allowing the use of multiple instruments from a single registered Callback](#asynchronous-instrument-api).  The API to register a new Callback SHOULD accept:

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

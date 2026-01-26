# 第 5 章 指标监控与告警系统

---

在本章中，我们将探讨一个可扩展的指标监控与告警系统的设计。一个设计良好的监控与告警系统在提供基础设施健康状况的清晰可见性方面发挥着关键作用，从而确保高可用性和可靠性。

图 1 显示了市场上一些最流行的指标监控与告警服务。在本章中，我们将设计一个类似的服务，可供大型公司内部使用。

![Figure5.1.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.1.png)

图 1 流行的指标监控与告警服务

## 第 1 步 - 了解问题并确定设计范围

指标监控与告警系统对不同的公司可能意味着许多不同的事情，因此，首先与面试官明确确切的需求至关重要。例如，如果面试官心里只想的是基础设施指标，你肯定不希望设计一个侧重于日志（如 Web 服务器错误或访问日志）的系统。

在深入细节之前，让我们首先充分了解问题并建立设计范围。

**候选人**：我们要为谁构建这个系统？是为像 Facebook 或 Google 这样的大型公司构建内部系统，还是设计一个像 Datadog [1]、Splunk [2] 等那样的 SaaS 服务？  
**面试官**：这是一个很好的问题。我们仅构建供内部使用。

**候选人**：我们想要收集哪些指标？  
**面试官**：我们想要收集运维系统指标。这些可以是操作系统的底层使用数据，例如 CPU 负载、内存使用情况和磁盘空间消耗。它们也可以是高层概念，例如服务的每秒请求数或 Web 服务器池的运行服务器数量。业务指标不在本次设计的范围内。

**候选人**：我们用这个系统监控的基础设施规模有多大？  
**面试官**：1 亿日活跃用户，1,000 个服务器池，每个池 100 台机器。

**候选人**：我们需要保留数据多久？  
**面试官**：假设我们想要 1 年的保留期。

**候选人**：我们可以为了长期存储而降低指标数据的分辨率（resolution）吗？  
**面试官**：这是一个很好的问题。我们希望能够将新接收的数据保留 7 天。7 天后，你可以将其汇总（roll up）为 1 分钟的分辨率并保留 30 天。30 天后，你可以进一步将其汇总为 1 小时的分辨率。

**候选人**：支持哪些告警渠道？  
**面试官**：电子邮件、电话、PagerDuty [3] 或 webhooks（HTTP 端点）。

**候选人**：我们需要收集日志吗，比如错误日志或访问日志？  
**面试官**：不需要。

**候选人**：我们需要支持分布式系统追踪（distributed system tracing）吗？  
**面试官**：不需要。

### 高层需求与假设
现在你已经完成了从面试官那里收集需求，并有了明确的设计范围。需求如下：

*   被监控的基础设施是超大规模的。
    *   1 亿日活跃用户
    *   假设我们有 1,000 个服务器池，每个池 100 台机器，每台机器 100 个指标 => ~1000 万个指标
    *   1 年数据保留期
    *   数据保留策略：原始形式保留 7 天，1 分钟分辨率保留 30 天，1 小时分辨率保留 1 年
*   可以监控各种指标，包括但不限于：
    *   CPU 使用率
    *   请求计数
    *   内存使用情况
    *   消息队列中的消息计数

### 非功能性需求

*   可扩展性。系统应该能够扩展以适应不断增长的指标和告警量。
*   低延迟。系统需要为仪表板和告警提供低查询延迟。
*   可靠性。系统应该是高度可靠的，以避免错过关键告警。
*   灵活性。技术在不断变化，因此流水线（pipeline）应该足够灵活，以便在未来轻松集成新技术。

哪些需求超出了范围？

*   日志监控。Elasticsearch, Logstash, Kibana (ELK) 堆栈在收集和监控日志 [4] 方面非常流行。
*   分布式系统追踪 [5] [6]。分布式追踪是指一种跟踪服务请求流经分布式系统时的追踪解决方案。它在请求从一个服务转向另一个服务时收集数据。

## 第 2 步 - 提出高层设计并获得认可

在本节中，我们将讨论构建系统的一些基础知识、数据模型和高层设计。

### 基础知识
一个指标监控与告警系统通常包含五个组件，如图 2 所示。


1. 数据采集 (Data collection)
2. 数据传输 (Data transmission)
3. 数据存储 (Data storage)
4. 告警 (Alerting)
5. 可视化 (Visualization)

![Figure5.2.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.2.png)
图 2 系统的五个组件

1. 数据采集：从不同来源收集指标数据。
2. 数据传输：将数据从来源传输到指标监控系统。
3. 数据存储：整理并存储接收到的数据。
4. 告警：分析接收到的数据，检测异常并生成告警。系统必须能够将告警发送到不同的通信渠道。
5. 可视化：以图形、图表等形式展示数据。当数据以可视化方式呈现时，工程师能更好地识别模式、趋势或问题，因此我们需要可视化功能。

### 数据模型
指标数据通常被记录为时间序列（time series），它包含一组值及其关联的时间戳。序列本身可以由其名称唯一标识，并且可以选择由一组标签（labels）来标识。

让我们看两个例子。

例子 1：20:00 时生产服务器实例 i631 的 CPU 负载是多少？

![Figure5.3.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.3.png)

图 3 CPU 负载

图 3 中突出显示的数据点可以用表 1 表示。

| metric_name | cpu.load |
| :--- | :--- |
| labels | host:i631,env:prod |
| timestamp | 1613707265 |
| value | 0.29 |

表 1 用表格表示的数据点

在这个例子中，时间序列由指标名称、标签（host:i631,env:prod）以及特定时间点的单个值表示。

例子 2：过去 10 分钟内 us-west 地区所有 Web 服务器的平均 CPU 负载是多少？从概念上讲，我们会从存储中提取如下内容，其中指标名称是 “CPU.load”，地区标签是 “us-west”：

CPU.load host=webserver01,region=us-west 1613707265 50  
CPU.load host=webserver01,region=us-west 1613707265 62  
CPU.load host=webserver02,region=us-west 1613707265 43  
CPU.load host=webserver02,region=us-west 1613707265 53  
...  
CPU.load host=webserver01,region=us-west 1613707265 76  
CPU.load host=webserver01,region=us-west 1613707265 83  

平均 CPU 负载可以通过对每行末尾的值取平均来计算。上述例子中的行格式被称为行协议（line protocol）。它是市场上许多监控软件常用的输入格式。Prometheus [7] 和 OpenTSDB [8] 就是两个典型的例子。

每个时间序列由以下内容组成 [9]：

| 名称 | 类型 |
| :--- | :--- |
| 指标名称 (A metric name) | String |
| 一组标签 (A set of tags/labels) | <key:value> 键值对列表 |
| 一组值及其时间戳 (An array of values and their timestamps) | <value, timestamp> 对数组 |

表 2 时间序列

### 数据访问模式

![Figure5.4.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.4.png)

图 4 数据访问模式

在图 4 中，y 轴上的每个标签代表一个时间序列（由名称和标签唯一标识），而 x 轴代表时间。

写入负载很重。如你所见，在任何时刻都可能有许多时间序列数据点被写入。正如我们在“高层需求”部分提到的，每天大约有 1000 万个运维指标被写入，且许多指标是以高频率收集的，因此流量无疑是高频写入的（write-heavy）。

与此同时，读取负载是突发性的。可视化和告警服务都会向数据库发送查询，根据图表和告警的访问模式，读取量可能会有峰值。

换句话说，系统处于持续的重度写入负载之下，而读取负载则是突发性的。

### 数据存储系统
数据存储系统是设计的核心。不建议为此工作构建自己的存储系统或使用通用存储系统（如 MySQL）。

理论上，通用数据库可以支持时间序列数据，但需要专家级的调优才能在我们这种规模下运行。具体来说，关系型数据库并未针对时间序列数据通常进行的各种操作进行优化。例如，在滚动时间窗口内计算移动平均值（moving average）需要复杂的 SQL，且难以阅读（深度探究章节中有一个这方面的例子）。此外，为了支持数据的标记/标签化，我们需要为每个标签添加索引。而且，通用关系型数据库在持续的重度写入负载下表现不佳。在我们这种规模下，我们需要投入大量精力来调优数据库，即便如此，它的表现也可能不尽如人意。

那么 NoSQL 呢？理论上，市场上的一些 NoSQL 数据库可以有效地处理时间序列数据。例如，Cassandra 和 Bigtable [11] 都可以用于时间序列数据。然而，这将需要对每个 NoSQL 的内部工作原理有深入的了解，以设计出能够有效存储和查询时间序列数据的可扩展架构。由于现成的工业级时间序列数据库（TSDB）已经非常成熟，使用通用 NoSQL 数据库并不具有吸引力。

目前有许多针对时间序列数据进行优化的存储系统。这种优化使我们能够使用远少于通用数据库的服务器来处理相同规模的数据。其中许多数据库还拥有专门为分析时间序列数据而设计的自定义查询接口，这些接口比 SQL 易用得多。有些甚至提供了管理数据保留和数据聚合的功能。以下是时间序列数据库的一些示例。

OpenTSDB 是一个分布式时间序列数据库，但由于它基于 Hadoop 和 HBase，运行 Hadoop/HBase 集群会增加复杂性。Twitter 使用的是 MetricsDB [12]，亚马逊（Amazon）提供 Timestream 时间序列数据库 [13]。根据 DB-engines [14] 的数据，最流行的两个时间序列数据库是 InfluxDB [15] 和 Prometheus，它们旨在存储大量时间序列数据并快速执行实时分析。两者都主要依赖内存缓存和磁盘存储。而且它们在耐久性和性能方面都表现得相当出色。如图 5 所示，一个拥有 8 核和 32GB RAM 的 InfluxDB 每秒可以处理超过 250,000 次写入。

![Figure5.5.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.5.png)

图 5 InfluxDB 基准测试

由于时间序列数据库是专用数据库，除非你在简历中明确提到过，否则在面试中你不需要了解其内部细节。就面试而言，重要的是了解指标数据在本质上是时间序列，我们可以选择像 InfluxDB 这样的时间序列数据库来存储它们。

一个强大的时间序列数据库的另一个特点是能够根据标签（在某些数据库中也称为 tags）对大量时间序列数据进行高效的聚合和分析。例如，InfluxDB 为标签构建索引以方便通过标签快速查找时间序列 [15]。它提供了关于如何使用标签的清晰最佳实践指南，且不会使数据库过载。关键是要确保每个标签都是低基数（low cardinality）的（即拥有一组较小的可能值）。这一特性对于可视化至关重要，而使用通用数据库来实现这一点将需要大量的工作。

## 高层设计
高层设计图如图 6 所示。

![Figure5.6.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.6.png)

图 6 高层设计

*   指标源（Metrics source）。这可以是应用服务器、SQL 数据库、消息队列等。
*   指标收集器（Metrics collector）。它收集指标数据并将其写入时间序列数据库。
*   时间序列数据库（Time-series database）。这将指标数据存储为时间序列。它通常提供自定义查询接口，用于分析和汇总大量时间序列数据。它维护标签索引，以方便通过标签快速查找时间序列数据。
*   查询服务（Query service）。查询服务使从时间序列数据库中查询和检索数据变得容易。如果我们选择一个好的时间序列数据库，这应该是一个非常薄的封装层。它也可以完全被时间序列数据库自身的查询接口所取代。
*   告警系统（Alerting system）。这会向各种告警接收端发送告警通知。
*   可视化系统（Visualization system）。这以各种图形/图表的形式展示指标。

## 第 3 步 - 深度探究
在系统设计面试中，候选人通常需要深入探究几个关键组件或流程。在本节中，我们将详细研究以下主题：

*   指标采集
*   扩展指标传输流水线
*   查询服务
*   存储层
*   告警系统
*   可视化系统

### 指标采集
对于计数器或 CPU 使用率之类的指标采集，偶尔丢失数据并不是什么大问题。客户端采用“发后即忘（fire and forget）”的方式是可以接受的。现在让我们看一下指标采集流程。系统的这一部分位于虚线框内（图 7）。

![Figure5.7.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.7.png)

图 7 指标采集流程

## 拉取模型 vs 推送模型 (Pull vs push models)
采集指标数据有两种方式：拉取（pull）或推送（push）。关于哪种方式更好一直存在常规性的争论，且没有标准答案。让我们仔细研究一下。

### 拉取模型 (Pull model)
图 8 显示了通过 HTTP 进行拉取模型的数据采集。我们有专门的指标收集器，定期从运行中的应用程序中拉取指标值。

![Figure5.8.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.8.png)

图 8 拉取模型

在这种方法中，指标收集器需要知道要从中拉取数据的所有服务端点（service endpoints）的完整列表。一种初级的方法是在“指标收集器”服务器上使用一个文件来保存每个服务端点的 DNS/IP 信息。虽然这个想法很简单，但在大规模环境下很难维护，因为服务器会被频繁地添加或移除，而我们要确保指标收集器不会错过从任何新服务器采集指标。好消息是，我们有一种可靠、可扩展且可维护的解决方案，即通过 **服务发现（Service Discovery）** 来实现，该功能由 etcd [16]、ZooKeeper [17] 等提供；其中，服务注册其可用性，指标收集器可以在服务端点列表发生变化时由服务发现组件通知。

服务发现包含有关何时以及何处采集指标的配置规则，如图 9 所示。

![Figure5.9.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.9.png)

图 9 服务发现

图 10 详细解释了拉取模型。

![Figure5.10.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.10.png)

图 10 拉取模型细节

指标收集器从服务发现中获取服务端点的配置元数据。元数据包括拉取间隔、IP 地址、超时和重试参数等。  
指标收集器通过预定义的 HTTP 端点（例如 `/metrics`）拉取指标数据。为了暴露该端点，通常需要在服务中添加一个客户端库。在图 10 中，该服务是 Web 服务器。  
（可选）指标收集器可以在服务发现中注册变更事件通知，以便在服务端点发生变化时接收更新。或者，指标收集器可以定期轮询端点变更。

在我们的规模下，单个指标收集器将无法处理成千上万台服务器。我们必须使用一个指标收集器池来处理需求。当存在多个收集器时，一个常见的问题是多个实例可能会尝试从同一个资源拉取数据并产生重复数据。实例之间必须存在某种协调方案（coordination scheme）来避免这种情况。

一种潜在的方法是在一致性哈希环（consistent hash ring）中为每个收集器指定一个范围，然后将每个被监控的服务器通过其唯一的名称映射到哈希环上。这确保了一台指标源服务器仅由一个收集器处理。让我们看一个例子。

如图 11 所示，有四个收集器和六台指标源服务器。每个收集器负责从一组不同的服务器收集指标。收集器 2 负责从服务器 1 和服务器 5 收集指标。

![Figure5.11.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.11.png)

图 11 一致性哈希

### 推送模型 (Push model)
如图 12 所示，在推送模型中，各种指标源（如 Web 服务器、数据库服务器等）直接将指标发送给指标收集器。

![Figure5.12.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.12.png)

图 12 推送模型

在推送模型中，通常在每台被监控的服务器上安装一个**采集代理 (collection agent)**。采集代理是一个长期运行的软件，它从服务器上运行的服务中收集指标，并定期将这些指标推送到指标收集器。采集代理还可以在将指标发送给指标收集器之前，在本地对指标进行聚合（特别是简单的计数器）。

聚合是减少发送到指标收集器的数据量的有效方法。如果推送流量很高，且指标收集器因错误拒绝了推送，代理可以在本地保留一个小型的数据缓冲区（可能通过将其存储在本地磁盘上），并在稍后重试。然而，如果服务器处于经常轮换的自动扩缩容组（auto-scaling group）中，那么当指标收集器滞后时，本地保留数据（即使是暂时的）可能会导致数据丢失。

为了防止指标收集器在推送模型中滞后，指标收集器应该处于一个前面挂有负载均衡器的自动扩缩容集群中（图 13）。该集群应根据指标收集器服务器的 CPU 负载进行横向扩缩容。

![Figure5.13.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.13.png)

图 13 负载均衡器

### 拉取还是推送？ (Pull or push?)
那么，哪一个对我们来说是更好的选择呢？就像生活中的许多事情一样，没有明确的答案。双方都有广泛采用的实际用例。

*   拉取架构的示例包括 Prometheus。
*   推送架构的示例包括 Amazon CloudWatch [18] 和 Graphite [19]。

了解每种方法的优缺点比在面试中挑选出一个胜者更重要。表 3 比较了推送和拉取架构的优缺点 [20] [21] [22] [23]。

| | 拉取 (Pull) | 推送 (Push) |
| :--- | :--- | :--- |
| **易于调试** | 应用服务器上的 `/metrics` 端点可用于随时查看指标。你甚至可以在笔记本电脑上执行此操作。**拉取模式胜出。** | |
| **健康检查** | 如果应用服务器对拉取没有响应，你可以快速确定该应用服务器是否已宕机。**拉取模式胜出。** | 如果指标收集器没有收到指标，问题可能是由网络问题引起的。 |
| **短寿命任务** | | 一些批处理作业（batch jobs）可能是短寿命的，持续时间不足以被拉取。**推送模式胜出。** 这可以通过为拉取模型引入推送网关（push gateways）来解决 [24]。 |
| **防火墙或复杂的网络设置** | 拉取指标的服务器需要所有指标端点都是可达的。这在多数据中心设置中可能会有问题。它可能需要更复杂的网络基础设施。 | 如果指标收集器配置了负载均衡器和自动扩缩容组，则可以从任何地方接收数据。**推送模式胜出。** |
| **性能** | 拉取方法通常使用 TCP。 | 推送方法通常使用 UDP。这意味着推送方法提供了更低延迟的指标传输。这里对应的观点是，建立 TCP 连接的开销与发送指标负载相比是很小的。 |
| **数据真实性** | 要从中收集指标的应用服务器是在配置文件中预先定义的。从这些服务器收集的指标保证是真实的。 | 任何类型的客户端都可以将指标推送到指标收集器。这可以通过将接受指标的服务器列入白名单，或通过要求身份验证来解决。 |

表 3 拉取 vs 推送

如上所述，拉取还是推送是一个常规的辩论话题，没有明确的答案。一个大型组织可能需要同时支持两者，特别是考虑到如今无服务器（serverless）[25] 的流行。在某些情况下，可能根本无法安装用于推送数据的代理。

## 扩展指标传输流水线 (Scale the metrics transmission pipeline)

![Figure5.14.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.14.png)

图 14 指标传输流水线

让我们放大观察指标收集器和时间序列数据库。无论你使用推送还是拉取模型，指标收集器都是一个服务器集群，该集群接收海量数据。无论是推送还是拉取，指标收集器集群都配置为自动扩缩容，以确保有足够数量的收集器实例来处理需求。

然而，如果时间序列数据库不可用，则存在数据丢失的风险。为了缓解这个问题，我们引入了一个队列组件，如图 15 所示。

![Figure5.15.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.15.png)

图 15 添加队列

在此设计中，指标收集器将指标数据发送到像 Kafka 这样的队列系统。然后，消费者或流处理服务（如 Apache Storm、Flink 和 Spark）处理数据并将其推送到时间序列数据库。这种方法有几个优点：
*   Kafka 被用作高度可靠且可扩展的分布式消息平台。
*   它将数据采集服务和数据处理服务解耦。
*   通过将数据保留在 Kafka 中，可以在数据库不可用时轻松防止数据丢失。

### 通过 Kafka 进行扩展 (Scale through Kafka)
我们可以利用 Kafka 内置的分区机制（partition mechanism）通过几种方式来扩展我们的系统。
*   根据吞吐量需求配置分区数量。
*   根据指标名称对指标数据进行分区，以便消费者可以按指标名称聚合数据。

![Figure5.16.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.16.png)

图 16 Kafka 分区

*   利用标签/标记（tags/labels）对指标数据进行进一步分区。
*   对指标进行分类并确定优先级，以便优先处理重要的指标。

### Kafka 的替代方案 (Alternative to Kafka)
维护一个生产规模的 Kafka 系统绝非易事。你可能会收到面试官关于此点的质疑。目前已有大规模的监控摄取系统在不使用中间队列的情况下运行。Facebook 的 Gorilla [26] 内存时间序列数据库就是一个典型的例子；它被设计为在发生局部网络故障时仍能保持写入的高可用性。可以认为，这样的设计与拥有像 Kafka 这样的中间队列一样可靠。

### 聚合可以在哪里发生 (Where aggregations can happen)
指标可以在不同的地方进行聚合：在采集代理（客户端）、摄取流水线（写入存储之前）以及查询端（写入存储之后）。让我们仔细研究一下它们。

**采集代理（Collection agent）**。安装在客户端的采集代理仅支持简单的聚合逻辑。例如，在将计数器发送到指标收集器之前，每分钟对其进行一次聚合。

**摄取流水线（Ingestion pipeline）**。要在将数据写入存储之前聚合数据，我们通常需要诸如 Flink 之类的流处理引擎。由于仅将计算结果写入数据库，写入量将显著减少。然而，处理延迟到达的事件（late-arriving events）可能是一个挑战，另一个缺点是我们失去了数据精度和一些灵活性，因为我们不再存储原始数据。

**查询端（Query side）**。原始数据可以在查询时在给定的时间段内进行聚合。使用这种方法不会丢失数据，但查询速度可能会较慢，因为查询结果是在查询时计算的，并且需要针对整个数据集运行。

## 查询服务 (Query service)
查询服务由一个查询服务器集群组成，它们访问时间序列数据库并处理来自可视化系统或告警系统的请求。拥有专门的一组查询服务器可以将时间序列数据库与客户端（可视化和告警系统）解耦。这使我们能够根据需要随时灵活地更改时间序列数据库或可视化及告警系统。

### 缓存层 (Cache layer)
为了减轻时间序列数据库的负载并使查询服务更具性能，添加了缓存服务器来存储查询结果，如图 17 所示。

![Figure5.17.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.17.png)

图 17 缓存层

### 不使用查询服务的理由 (The case against query service)
实际上并没有迫切的需求来引入我们自己的抽象（查询服务），因为大多数工业级规模的可视化和告警系统都有强大的插件来与市场上知名的时间序列数据库对接。而且如果选择了合适的时间序列数据库，通常也不需要额外添加我们自己的缓存。

### 时间序列数据库查询语言 (Time-series database query language)
大多数流行的指标监控系统（如 Prometheus 和 InfluxDB）不使用 SQL，而是拥有自己的查询语言。一个主要原因是 SQL 很难用于查询时间序列数据。例如，正如这里 [27] 所提到的，在 SQL 中计算指数移动平均（exponential moving average）可能如下所示：

```sql
select id,
    temp,
    avg (temp) over ( partition by group_nr order by time_read) as rolling_avg
from (
    select id,
    time,
    time_read,
    interval_group,
    id - row_number() over (partition by interval_group order by time_read) as group_nr
    from (
        select time_read,
        "epoch ":: timestamp + "900 seconds ":: interval * (
        extract ( epoch from time_read ):: int4 / 900) as interval_group,
        temp,
        from readings,
    ) t1
) t2
order by time_read;
 ```
而在 Flux（一种针对时间序列分析优化的语言，用于 InfluxDB）中，它看起来像这样。如你所见，它要容易理解得多。

```flux
from(db:"telegraf")
|> range(start:-1h)
|> filter(fn: (r) => r._measurement == "foo")
|> exponentialMovingAverage(size:-10s)
```
## 存储层 (Storage layer)
现在让我们深入探讨存储层。

### 仔细选择时间序列数据库 (Choose a time-series database carefully)
根据 Facebook 发表的一篇研究论文 [26]，运维数据存储中至少 85% 的查询是针对过去 26 小时内收集的数据。如果我们使用的时间序列数据库能够利用这一特性，将对系统的整体性能产生重大影响。如果你对存储引擎的设计感兴趣，请参考 InfluxDB 存储引擎的设计文档 [28]。

### 空间优化 (Space optimization)
正如在高层需求中所解释的，需要存储的指标数据量是巨大的。这里有几种解决策略。

#### 数据编码和压缩 (Data encoding and compression)
数据编码和压缩可以显著减小数据的大小。这些功能通常内置在优秀的时间序列数据库中。下面是一个简单的例子。

**双重差分编码 (Double-delta Encoding)**

![Figure5.18.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.18.png)

图 18 数据编码

如上图所示，1610087371 和 1610087381 仅相差 10 秒，这仅需 4 位（bits）即可表示，而无需使用 32 位的完整时间戳。因此，与其存储绝对值，不如存储数值的差值（delta）以及一个基准值，例如：*1610087371, 10, 10, 9, 11*。

#### 降采样 (Downsampling)
降采样是将高分辨率数据转换为低分辨率数据以减少整体磁盘占用的过程。由于我们的数据保留期为 1 年，我们可以对旧数据进行降采样。例如，我们可以让工程师和数据科学家为不同的指标定义规则。下面是一个例子：

*   保留期：7 天，不采样
*   保留期：30 天，降采样至 1 分钟分辨率
*   保留期：1 年，降采样至 1 小时分辨率

让我们看另一个具体的例子。它将 10 秒分辨率的数据汇总（aggregates）为 30 秒分辨率的数据。

| metric | timestamp | hostname | metric_value |
| :--- | :--- | :--- | :--- |
| cpu | 2021-10-24T19:00:00Z | host-a | 10 |
| cpu | 2021-10-24T19:00:10Z | host-a | 16 |
| cpu | 2021-10-24T19:00:20Z | host-a | 20 |
| cpu | 2021-10-24T19:00:30Z | host-a | 30 |
| cpu | 2021-10-24T19:00:40Z | host-a | 20 |
| cpu | 2021-10-24T19:00:50Z | host-a | 30 |

表 4 10 秒分辨率数据

将 10 秒分辨率数据汇总为 30 秒分辨率数据：

| metric | timestamp | hostname | Metric_value (avg) |
| :--- | :--- | :--- | :--- |
| cpu | 2021-10-24T19:00:00Z | host-a | 19 |
| cpu | 2021-10-24T19:00:30Z | host-a | 25 |

表 5 30 秒分辨率数据

### 冷存储 (Cold storage)
冷存储是对不常用且处于非活跃状态的数据进行的存储。冷存储的财务成本要低得多。

简而言之，我们可能应该使用第三方可视化和告警系统，而不是自己构建。

## 告警系统 (Alert system)
出于面试的目的，让我们来看看图 19 所示的告警系统。

![Figure5.19.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.19.png)

图 19 告警系统

告警流程如下：
1. 将配置文件加载到缓存服务器。规则以磁盘上的配置文件形式定义。YAML [29] 是定义规则的常用格式。下面是一个告警规则的示例：

```yaml
- name: instance_down
  rules:

  # 为任何无法访问超过 5 分钟的实例 (instance) 发送告警 (Alert for)。
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

2. 告警管理器从缓存中获取告警配置。
3. 基于配置规则，告警管理器以预定义的时间间隔调用查询服务。如果数值违反了阈值，则创建一个告警事件。告警管理器负责以下工作：

*   对告警进行过滤、合并和去重。
*   示例：合并在一个实例 (instance1) 内触发的告警。

![Figure5.20.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.20.png)

图 20 合并告警

*   访问控制：将某些告警管理操作的访问限制在授权个人范围内。
*   重试：通过检查告警状态，确保通知至少发送一次。

4. 告警存储是一个键值数据库，例如 Cassandra，它保存所有告警的状态（非活跃、待定、正在触发、已解决）。它确保通知至少发送一次。
5. 符合条件的告警被插入 Kafka。
6. 告警消费者从 Kafka 拉取告警事件。
7. 告警消费者处理来自 Kafka 的告警事件，并向不同的渠道发送通知，如电子邮件、短信、PagerDuty 或 HTTP 端点。

### 告警系统 - 自建还是购买 (Alert system - build vs buy)
市面上有很多现成的工业级告警系统，而且大多数都提供了与流行时间序列数据库的紧密集成。这些告警系统中的许多都能与现有的通知渠道（如电子邮件和 PagerDuty）很好地集成。在现实世界中，很难证明自建告警系统是合理的。在面试场景中，尤其是对于高级职位，要准备好为你的决定辩解。

## 可视化系统 (Visualization system)
可视化构建在数据层之上。指标可以显示在不同时间跨度的指标仪表板上，告警也可以显示在告警仪表板上。图 21 显示了一个仪表板，它显示了一些指标，如当前服务器请求数、内存/CPU 利用率、页面加载时间、流量和登录信息 [30]。

![Figure5.21.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.21.png)

图 21 Grafana 界面

高质量的可视化系统很难构建。使用现成系统的理由非常充分。例如，Grafana 可以是用于此目的的一个非常好的系统。它与许多你可以购买到的流行时间序列数据库集成得非常好。

## 第 4 步 - 总结

在本章中，我们介绍了一个指标监控与告警系统的设计。在高层级上，我们讨论了数据采集、时间序列数据库、告警和可视化。然后，我们深入探讨了一些最重要的技术/组件：

*   用于采集指标数据的拉取与推送模型。
*   利用 Kafka 扩展系统。
*   选择合适的时间序列数据库。
*   使用降采样来减小数据规模。
*   告警与可视化系统的自建 vs 购买选项。

我们经过了几次迭代来完善设计，最终设计如图 22 所示：

![Figure5.22.png](..%2Fimages%2Fv2%2Fchapter05%2FFigure5.22.png)

图 22 最终设计

祝贺你学到这里！现在奖励自己一下。做得好！

## 参考资料

[1] Datadog: https://www.datadoghq.com/  
[2] Splunk: https://www.splunk.com/  
[3] PagerDuty: https://www.pagerduty.com/  
[4] Elastic stack: https://www.elastic.co/elastic-stack  
[5] Dapper，大规模分布式系统追踪基础设施：https://research.google/pubs/pub36356/  
[6] 使用 Zipkin 进行分布式系统追踪：https://blog.twitter.com/engineering/en_us/a/2012/distributed-systems-tracing-with-zipkin.html  
[7] Prometheus: https://prometheus.io/docs/introduction/overview/  
[8] OpenTSDB - 一个分布式、可扩展的监控系统：http://opentsdb.net/  
[9] 数据模型：https://prometheus.io/docs/concepts/data_model/  
[10] MySQL: https://www.mysql.com/  
[11] 时间序列数据的架构设计 | Cloud Bigtable 文档：https://cloud.google.com/bigtable/docs/schema-design-time-series  
[12] MetricsDB，Twitter 的时间序列数据库：https://blog.twitter.com/engineering/en_us/topics/infrastructure/2019/metricsdb.html  
[13] Amazon Timestream: https://aws.amazon.com/timestream/  
[14] DB-Engines 时间序列数据库排名：https://db-engines.com/en/ranking/time+series+dbms  
[15] InfluxDB: https://www.influxdata.com/  
[16] etcd: https://etcd.io/  
[17] 使用 ZooKeeper 进行服务发现：https://cloud.spring.io/spring-cloud-zookeeper/1.2.x/multi/multi_spring-cloud-zookeeper-discovery.html  
[18] Amazon CloudWatch: https://aws.amazon.com/cloudwatch/  
[19] Graphite: https://graphiteapp.org/  
[20] 推送 vs 拉取：http://bit.ly/3aIEPxE  
[21] 拉取模式无法扩展吗 —— 或者它真的可以？：https://prometheus.io/blog/2016/07/23/pull-does-not-scale-or-does-it/  
[22] 监控架构：https://developer.lightbend.com/guides/monitoring-at-scale/monitoring-architecture/architecture.html  
[23] 监控系统中的推送 vs 拉取：https://giedrius.blog/2019/05/11/push-vs-pull-in-monitoring-systems/  
[24] Pushgateway: https://github.com/prometheus/pushgateway  
[25] 使用无服务器架构构建应用程序：https://aws.amazon.com/lambda/serverless-architectures-learn-more/.  
[26] Gorilla：一种快速、可扩展、内存型时间序列数据库：http://www.vldb.org/pvldb/vol8/p1816-teller.pdf  
[27] 为什么我们要构建 Flux，一种新的数据脚本和查询语言：https://www.influxdata.com/blog/why-were-building-flux-a-new-data-scripting-and-query-language/  
[28] InfluxDB 存储引擎：https://docs.influxdata.com/influxdb/v2.0/reference/internals/storage-engine/  
[29] YAML: https://en.wikipedia.org/wiki/YAML  
[30] Grafana 演示：https://play.grafana.org/  
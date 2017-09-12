---
title: 高效管理 Elasticsearch 中基于时间的索引
date: 2017-06-26
url: es-managing-time-based-indices-efficiently
---

翻译自：[And the big one said "Rollover" — Managing Elasticsearch time-based indices efficiently](https://www.elastic.co/blog/managing-time-based-indices-efficiently)

用 Elasticsearch 来索引诸如日志事件等基于时间的数据的人可能已经习惯了“每日一索引”模式：使用以天为粒度的索引名字来存放当天的日志数据，一天过去后再建一个新索引。新索引的属性可以由[索引模板](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-templates.html)来提前控制。

这种模式很容易理解并且易于实现，但是它粉饰了索引管理的一些复杂的地方：

- 为了达到较高的写入速度，活跃索引分片需要分布在*尽可能多的*节点上。
- 为了提高搜索速度和降低资源消耗，分片数量需要*尽可能地少*，但是也不能有过大的单个分片进而不便操作
- 一天一个索引确实易于清理陈旧数据，但是一天到底需要多少个分片呢？
- 每天的写入压力是一成不变的吗？还是一天分片过多，而下一天分片不够用呢？

在这篇文章中我将介绍新的”滚动模式“和用来实现它的 API 们，这个模式可以更加简单且高效地管理基于时间的索引。

### 滚动模式

滚动模式工作流程如下：

- 有一个用于写入的索引别名，其指向活跃索引
- 另外一个用于读取（搜索）的索引别名，指向不活跃索引
- 活跃索引具有和热节点数量一样多的分片，可以充分发挥昂贵硬件的索引写入能力
- 当活跃索引太满或者太老的时候，它就会[滚动](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-rollover-index.html)：新建一个索引并且索引别名自动从老索引切换到新索引
- 移动老索引到冷节点上并且[缩小](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-shrink-index.html)为一个分片，之后可以[强制合并](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-forcemerge.html)和压缩。

### 入门

假设我们有一个具有 10 个*热*节点和一个*冷*节点池的集群。理想情况下我们的活跃索引（接收所有写入的索引）应该在每个热节点上均匀分布一个分片，以此来尽可能地在多个机器上分散写入压力。

我们让每个主分片都有一个复制分片来允许一个节点失效而不丢失数据。这意味着我们的活跃索引应该有 5 个主分片，加起来一共 10 个分片（每个节点一个）。我们也可以用 10 个主分片（包含冗余一共 20 个分片），这样每个节点两个分片。

首先，为活跃索引创建一个[索引模版](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-templates.html)：

```json
PUT _template/active-logs
{
  "template": "active-logs-*",
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "routing.allocation.include.box_type": "hot",
    "routing.allocation.total_shards_per_node": 2
  },
  "aliases": {
    "active-logs": {}, 
    "search-logs": {}
  }
}
```

由这个模板创建的索引会被分配到标记为 `box_type:hot` 的节点上，而 `total_shards_per_node` 配置会保证将分片均匀分布在*热*节点中。我把其设置为 `2` 而不是 `1`，这样当一个节点失效时也可以继续分配分片。

我们将会用 `active-logs` 别名来写入当前的活跃索引，用 `search-logs` 别名来查询所有的日志索引。

下面是非活跃索引的模板：

```json
PUT _template/inactive-logs
{
  "template": "inactive-logs-*", 
  "settings": { 
    "number_of_shards": 1, 
    "number_of_replicas": 0,
    "routing.allocation.include.box_type": "cold",
    "codec": "best_compression"
  }
}
```

归档的索引应该被分配到*冷*节点上并且使用 `deflate` 压缩来节约磁盘空间。我会在之后解释为什么把 `replicas` 设置为 `0`。

现在可以创建第一个活跃索引了：

```
PUT active-logs-1
```

Rollover API 会将名字中的 `-1` 识别为一个计数器。

### 索引日志事件

当创建 `active-logs-1` 索引时，我们也创建了 `active-logs` 别名。在此之后，我们应该仅使用别名来写入，文档会被发送到当前的活动索引：

```
POST active-logs/log/_bulk
{ "create": {}} { "text": "Some log message", "@timestamp": "2016-07-01T01:00:00Z" }
{ "create": {}} { "text": "Some log message", "@timestamp": "2016-07-02T01:00:00Z" }
{ "create": {}} { "text": "Some log message", "@timestamp": "2016-07-03T01:00:00Z" }
{ "create": {}} { "text": "Some log message", "@timestamp": "2016-07-04T01:00:00Z" }
{ "create": {}} { "text": "Some log message", "@timestamp": "2016-07-05T01:00:00Z" }
```

### 滚动索引

在某个时间点，活跃索引变得过大或者过老，这时你想用一个新的空索引来替换它。[Rollover API](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-rollover-index.html) 允许你指定触发滚动操作的具体大小或者时间限制。

多大才是过大？一如以往，看情况。这取决于你的硬件性能，你的搜索操作的类型，你想要达到的性能效果和你能接受的分片恢复时间等等。可以从例如 1 亿或者 10 亿这种数字开始，依据搜索性能、数据保留时间和可用磁盘空间来上下调整。

一个分片能包含的文档数有一个硬限制：2147483519。如果你打算把活跃索引缩小到一个分片，那么活跃索引中的文档数不能超过 21 亿。如果活跃索引中的文档一个分片放不下，你可以将活跃索引缩小到多个分片，只要目标分片数是原来分片数的因子，例如 6 到 3 或者 6 到 2。

基于时间来滚动索引很方便，因为可以按照小时、天或者星期来整理索引。但其实按照索引中的文档数来滚动索引更加高效。按照数量来滚动的优点之一就是所有的分片会具有大致相同的大小，这样做负载均衡更加方便。

可以用定时任务来定期调用 rollover API 去检查是否到达了 `max_docs` 或者 `max_age` 限制。当超过某个限制时，索引就会被滚动。因为我们在例子中只索引了 5 个文档，我们将 `max_docs` 值设置为 `5`，并且（为了完整性）将 `max_age` 设置为一周：

```json
POST active-logs/_rollover
{
  "conditions": {
    "max_age": "7d",
    "max_docs": 5
  }
}
```

这个请求告诉 Elasticsearch 去滚动 `active-logs` 别名指向的索引，如果这个索引至少在七天之前创建或者至少包含 5 个文档。应答如下：

```json
{
  "old_index": "active-logs-1",
  "new_index": "active-logs-2",
  "rolled_over": true,
  "dry_run": false,
  "conditions": {
    "[max_docs: 5]": true,
    "[max_age: 7d]": false
  }
}
```

因为满足了 `max_docs: 5` 条件，`active-logs-1` 索引被滚动到 `active-logs-2` 索引。这意味着一个叫做 `active-logs-2` 的索引被创建（基于 `active-logs` 模板），并且 `active-logs` 别名从 `active-logs-1` 切换到 `active-logs-2`。

顺带一提，如果你想覆写索引模板中的某些值（例如 `settings` 或者 `mappings`），只需要把它们放在 `_rollover` 的请求体中就可以（和[创建索引 API](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-create-index.html) 一样）。

#### 为什么不支持 `max_size` 限制？

既然想尽可能地让分片大小相似，为什么不在 `max_docs` 之外在加上支持 `max_size` 限制呢？答案是分片的大小并不是一个可靠的测量标准，因为正在进行中的合并会产生大量的临时分片大小增长，而当合并结束后这些增长会消失掉。五个主分片，每个都在合并到一个 5GB 分片的过程中，那么此时索引大小会临时增多 25GB！而对于文档数量来说，它的增长则是可以预测的。

### 缩小索引

此时 `acitve-logs-1` 不再用于写入，我们可以把它移到冷节点上并且把它缩小到一个分片，这个新索引叫做 `inactive-logs-1`。在缩小之前，我们必须：

- 设置索引为只读
- 将所有分片移动到同一个节点上。可以任意选择目标节点，比如选择具有最大剩余空间的*冷*节点

用以下命令来做这些事情：

```json
PUT active-logs-1/_settings
{
  "index.blocks.write": true,
  "index.routing.allocation.require._name": "some_node_name"
}
```

`allocation` 配置保证了每个分片的至少一个拷贝会被移动到 `some_node_name` 节点上。这并不会移动所有分片——因为复制分片不能和主分片分配在同一个节点上——但它会保证至少一个主分片或者复制分片会被移动。

当索引完成迁移后（用[集群健康 API](https://www.elastic.co/guide/en/elasticsearch/reference/master/cluster-health.html) 来检查），使用一下请求来缩小索引：

```
POST active-logs-1/_shrink/inactive-logs-1
```

如果你的文件系统支持硬链接，那么缩小会瞬间完成。如果你的文件系统不支持硬链接，那你就得等待所有的分段文件从一个索引拷贝到另一个索引……

你可以用 [恢复状态查询 API](https://www.elastic.co/guide/en/elasticsearch/reference/master/cat-recovery.html) 或者集群健康 API 来监控缩小过程：

```
GET _cluster/health/inactive-logs-1?wait_for_status=yellow
```

当缩小完成后，你就可以从 `search-logs` 别名中删除老索引并加入新索引：

```json
POST _aliases
{
  "actions": [
    {
      "remove": {
        "index": "active-logs-1",
        "alias": "search-logs"
      } 
    },
    {
      "add": {
        "index": "inactive-logs-1",
        "alias": "search-logs"
      }
    }
  ]
}
```

### 节约空间

我们的索引已经缩小到单个分片，但它依旧包含和之前相同数量的段文件，并且 `best_compression` 设置并没有生效，因为没有任何写入操作。我们可以用[强制合并](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-forcemerge.html)来将单分片索引优化为单分段索引，如下：

```
POST inactive-logs-1/_forcemerge?max_num_segments=1
```

这个请求会创建一个新的分段来替换之前的多个分段。并且因为 Elasticsearch 必须要写入新分段，`best_compression` 设置就会起作用，新分段会用 `deflate` 压缩写入。

在主分片和复制分片上分别运行强制合并是没有意义的，这就是为什么我们的非活跃索引模板中 `number_of_replicas` 设置被为 `0`。现在当强制合并结束后，我们可以打开复制分片以获得冗余：

```json
PUT inactive-logs-1/_settings
{ "number_of_replicas": 1 }
```

当复制分片被分配之后（用 `?wait_for_status=green` API 查询），我们就可以确定拥有了一个冗余，此时便可以安全地删掉 `active-logs-1` 索引：

```
DELETE active-logs-1
```

### 删除旧索引

在使用老的每日一索引模式时，决定删除哪些索引十分方便。而在使用滚动模式时，似乎并不好确定索引包含了什么时间段的数据。

幸运的是，[字段统计 API](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-field-stats.html) 可以轻松确定这些。我们只需要具有找出超过我们阈值的最大 `@timestamp` 字段的索引列表就可以了：

```json
GET search-logs/_field_stats?level=indices
{
  "fields": ["@timestamp"],
  "index_constraints": {
    "@timestamp": {
      "max_value": {
        "lt": "2016/07/03",
        "format": "yyyy/MM/dd"
      }
    }
  }
}
```

这个请求返回的索引都可以删除。

### 未来的改进

通过[滚动](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-rollover-index.html)、[缩小](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-shrink-index.html)、[强制合并](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-forcemerge.html)和[字段统计](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-field-stats.html) API，我们向你提供了高效管理基于时间的索引的基础工具。

当然，这里有许多步骤可以被自动化来让生活更美好。这些步骤并在 Elasticsearch 中并不是很容易内置，因为我们需要在发生意料之外的情况时通知别人。这是在 Elasticsearch 之上构建的工具或应用程序的职责。

期待可以在 [Curator index management tool](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html) 和 [X-Pack](https://www.elastic.co/guide/en/x-pack/current/index.html) 中看到相应的工作流和 UI。
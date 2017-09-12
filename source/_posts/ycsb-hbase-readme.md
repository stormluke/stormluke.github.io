---
title: 使用 YCSB 压力测试 HBase
date: 2017-05-23
url: ycsb-hbase-readme
---

[YCSB](https://github.com/brianfrankcooper/YCSB)（Yahoo! Cloud Serving Benchmark）是雅虎开源的用于测试新式数据库（主要为 NoSQL）性能的框架，使用 Java 实现，可以测试 HBase、Cassandra、Infinispan、MongoDB 等等。

YCSB 包括两个部分：

- YCSB 客户端，一个可以扩展的 workload 生成器
- Core workloads，预先配置好的 workloads

### 安装 YCSB

下载 [YCSB 最新发行版](https://github.com/brianfrankcooper/YCSB/releases/latest)

```sh
curl -O --location https://github.com/brianfrankcooper/YCSB/releases/download/0.12.0/ycsb-0.12.0.tar.gz
tar xfvz ycsb-0.12.0.tar.gz
cd ycsb-0.12.0
```

其中 `workloads` 文件夹中为预置的 Core workload，`*-binding` 文件夹中为预置的数据库接口层。

### 运行一个压力测试

运行一个压力测试需要 6 步：

- 配置需要测试的数据库
- 选择合适的数据库接口层
- 选择合适的 workload
- 选择合适的运行时参数
- 装载数据（loading phase）
- 运行测试（transaction phase）

这些步骤适合于在单个客户端上运行（小型和中型集群，10 个左右的服务节点），更大的集群需要多个客户端同时运行测试。

#### 第一步：配置需要测试的数据库

YCSB 不负责表的创建，需要在数据库中手动创建用于测试的表。例如在 HBase 中，需要手动创建一个表和一个列族。

为了让测试数据均匀分布在不同节点中，创建 HBase 表时需要设置预先分片策略（参考 [HBASE-4163](https://issues.apache.org/jira/browse/HBASE-4163)）。

```
hbase(main):001:0> n_splits = 20 # HBase recommends (10 * number of regionservers)
hbase(main):002:0> create 'usertable', 'cf', {SPLITS => (1..n_splits).map {|i| "user#{1000+i*(9999-1000)/n_splits}"}}
```

这里 `n_splits` 推荐设置为 `RegionServer` 数量 * 10。

#### 第二步：选择合适的数据库接口层

YCSB 预置了一些常用数据库的接口层，对于 HBase 测试，需要选择 `com.yahoo.ycsb.db.HBaseClient` 作为接口。

另外 YCSB 需要知道如何连接到 HBase 的 Zookeeper 服务，最简单的方法是将目标测试服务器上的 HBase 配置文件 `$HBASE_HOME/conf/hbase-site.xml` 复制到 `$YCSB/hbase12/conf` 中（低版本 HBase 需要复制到对应的 `hbase10` 等文件夹中）。

YCSB 也提供了一个 `com.yahoo.ycsb.BasicDB` 数据库接口层，这个接口层仅仅打印收到的数据库操作请求，可以用来调试。

可以用 `bin/yscb shell <interface>` 来测试数据库接口层配置是否正确，例如使用 `BasicDB` 层：

```sh
$ ./bin/ycsb shell basic
> help
Commands:
  read key [field1 field2 ...] - Read a record
  scan key recordcount [field1 field2 ...] - Scan starting at key
  insert key name1=value1 [name2=value2 ...] - Insert a new record
  update key name1=value1 [name2=value2 ...] - Update a record
  delete key - Delete a record
  table [tablename] - Get or [set] the name of the table
  quit - Quit
```

#### 第三步：选择合适的 workload

Workload 定义了如何向数据库中加载测试数据，包括两个部分：

- Workload Java 类（`com.yahoo.ycsb.Workload` 的子类）
- 配置文件（Java Properties 格式）

YCSB 的 CoreWorkload 预置了一些标准测试数据，可以直接使用，包括 6 个不同的类型：

- Workload A：
    - 重更新，50% 读 50% 写，例如 session sotre
- Workload B：
    - 读多写少，95% 读 5% 写，例如 photo tagging
- Workload C：
    - 只读：100% 读，例如 user profile cache
- Workload D：
    - 读最近更新：这个 workload 会插入新纪录，越新的纪录读取概率越大，例如：user status updates
- Workload E：
    - 小范围查询：这个 workload 会查询小范围的纪录，而不是单个纪录，例如：threaded conversations
- Workload F：
    - 读取-修改-写入：这个 workload 会读取一个纪录，然后修改这个纪录，最后写回，例如：user database

可以根据测试需求选择合适的 workload，也可以新建一个新的 workload。

#### 第四步：选择合适的运行时参数

除了在 workload 中配置参数外，YCSB 还支持这些运行时参数：

- `-threads`：客户端线程数，默认为 1
- `-target`：每秒的目标操作数，默认为无限制（尽可能快地完成操作）。例如一个操作需要 100 ms，那么一个线程 1s 内可以完成 10 个操作，通过 `-target` 参数可以将操作放缓，控制在 10 个以下
- `-s`：每 10s 打印一次客户端状态，用于调试

#### 第五步：装载数据

Workload 包含两个阶段：装载阶段和事务阶段。在装载阶段向数据库中插入测试数据。对于 HBase 测试，可以使用下面的命令装载数据：

```sh
./bin/ycsb load hbase12 -P workloads/workloada -p table=usertable -p columnfamily=cf -p recordcount=100000000 -p operationcount=100000000 -thread 50 -s
```

各个参数的含义为：

- `hbase12`：使用 HBase 1.2.x 版本的数据库连接层
- `-P` 指定 workload 配置文件路径，使用 Workload A 类型
- `-p` 指定单个配置（会覆盖之前文件中的配置）
    - `table=`，`columnfamily=`, 指定 HBase 表名和列族
    - `recordcount=`，`operationcount=`，指定纪录数和操作数
- `-thread` 指定客户端线程数
- `-s` 打印状态

#### 第六步：运行测试

当装载完测试数据后，就可以运行 workload 测试了。对于 HBase 测试命令为：

```sh
./bin/ycsb run hbase12 -P workloads/workloada -p table=usertable -p columnfamily=cf -p recordcount=100000000 -p operationcount=100000000 -thread 50 -s
```

参数含义与装载数据时相同，区别在用 `run` 替代了 `load`。

运行结束后可以看到 YCSB 打印出了一些测试统计结果：

``` sh
[OVERALL], RunTime(ms), 11551.0
[OVERALL], Throughput(ops/sec), 8657.259111765215
[TOTAL_GCS_PS_Scavenge], Count, 12.0
[TOTAL_GC_TIME_PS_Scavenge], Time(ms), 66.0
[TOTAL_GC_TIME_%_PS_Scavenge], Time(%), 0.5713791013765043
[TOTAL_GCS_PS_MarkSweep], Count, 0.0
[TOTAL_GC_TIME_PS_MarkSweep], Time(ms), 0.0
[TOTAL_GC_TIME_%_PS_MarkSweep], Time(%), 0.0
[TOTAL_GCs], Count, 12.0
[TOTAL_GC_TIME], Time(ms), 66.0
[TOTAL_GC_TIME_%], Time(%), 0.5713791013765043
[READ], Operations, 50011.0
[READ], AverageLatency(us), 4418.49443122513
[READ], MinLatency(us), 1584.0
[READ], MaxLatency(us), 208895.0
[READ], 95thPercentileLatency(us), 8207.0
[READ], 99thPercentileLatency(us), 11463.0
[READ], Return=OK, 50011
[CLEANUP], Operations, 100.0
[CLEANUP], AverageLatency(us), 1393.0
[CLEANUP], MinLatency(us), 1.0
[CLEANUP], MaxLatency(us), 138623.0
[CLEANUP], 95thPercentileLatency(us), 18.0
[CLEANUP], 99thPercentileLatency(us), 258.0
[UPDATE], Operations, 49989.0
[UPDATE], AverageLatency(us), 6124.871431714977
[UPDATE], MinLatency(us), 1947.0
[UPDATE], MaxLatency(us), 429311.0
[UPDATE], 95thPercentileLatency(us), 10087.0
[UPDATE], 99thPercentileLatency(us), 15335.0
[UPDATE], Return=OK, 49989
```

各个参数的含义在结果解析中解释。

如果想获得直方图或者时间序列结果（参考测量参数），可以使用 `measurementtype=timeseries` 参数。客户端默认每 1s 纪录一次平均延迟，可以通过 `timeseries.granularity=2000` 参数（毫秒）来修改。此时测试报告为：

```sh
...
[READ], 0, 12064.403652090341
[READ], 1000, 6515.2218055956155
[READ], 2000, 5907.763636363637
[READ], 3000, 4967.781086610593
[READ], 4000, 4397.277823577906
[READ], 5000, 5179.516736401673
[READ], 6000, 4398.908602150537
[READ], 7000, 4493.980899323517
[READ], 8000, 4431.4615697437985
[READ], 9000, 4365.6925873560895
[READ], 10000, 5771.521574205784
[READ], 11000, 4300.872277227722
[READ], 12000, 3728.5879396984924
...
```

此时整个测试过程就结束了。

注意 YCSB 的延迟为端到端的延迟，在开始调用数据库接口层方法前开始计时，在方法返回时结束计时，也就是说延迟包括这几个部分：

- 在接口层内代码的运行时间
- 客户端到数据库服务器的网络延迟
- 数据库的执行时间

其中并不包括由 `-target` 节流参数引入的延迟。

### 主要参数

#### YCSB 参数

所有的 workload 文件都可以使用以下参数：

- `workload`：workload 类
- `db`：数据库接口层类，默认为 `com.yahoo.ycsb.BasicDB`，也可以通过命令行指定
- `exporter`：报告类，默认为 `com.yahoo.ycsb.measurements.exporter.TextMeasurementsExporter`
- `exportfile`：报告输出路径，默认 `stdout`
- `threadcount`：客户端线程数，默认 1
- `measurementtype`：测量类型，支持 `histogram` 和 `timeseries`，默认 `histogram`

#### 预置 Core workload 包参数

预置的 workload 支持以下参数：

- `fieldcount`：每个纪录中字段的个数，默认：10
- `fieldlength`：每个字段的大小，默认：100
- `readallfields`：是否读取所有字段还是一个字段，默认：true（读取所有字段）
- `readproportion`：读取操作所占比例，默认 0.95
- `updateproportion`：更新操作所占比例，默认 0.05
- `insertproportion`：插入操作所占比例，默认 0
- `scanproportion`：扫描操作所占比例，默认 0
- `readmodifywriteproportion`：读-更新-写操作所占比例：默认 0
- `requestdistribution`：请求的概率分布类型，支持 `uniform`、`zipfian`、`latest`，默认 `uniform`
    - `uniform`：均匀分布
    - `zipfian`：[齐夫分布](https://www.wikiwand.com/zh/%E9%BD%8A%E5%A4%AB%E5%AE%9A%E5%BE%8B)
    - `latest`：数据越新访问概率越高
- `maxscanlength`：扫描操作最长的范围，默认 1000
- `scanlengthdistribution`：扫描操作范围的概率分布类型（从 1 到 `scanlengthdistribution`），默认 `uniform`
- `insertorder`：插入顺序
    - `ordered`：根据 key 排序插入
    - `hashed`：根据 hash 结果排序插入
- `operationcount`：操作数
- `maxexecutiontime`：最长总体执行时间（秒）
- `table`：数据库表名，默认 `usertable`
- `recordcount`：纪录数，默认 0
- `core_workload_insertion_retry_limit`：插入失败重试次数，默认 0
- `core_workload_insertion_retry_interval`：重试间隔（秒），默认 3

#### 测量参数

- `hdrhistogram` 生成 `hdrhistogram` 格式的报告，可以使用 [HdrHistogram Plotter](http://hdrhistogram.github.io/HdrHistogram/plotFiles.html) 展示图形结果
    - `hdrhistogram.percentiles`：测量百分位分区，用逗号隔开，默认 95,99（参考[第95个百分位（95th percentile）是什么概念？](https://www.zhihu.com/question/20575291)）
    - `hdrhistogram.fileoutput=true|false`：是否输出到文件，文件地址由 `hdrhistogram.output.path` 指定
- `histogram`
    - `histogram.buckets`：直方图的分区数，默认 1000
- `timeseries`
    - `timeseries.granularity`：时间序列的粒度（毫秒），默认 1000

### 结果解析

```
[OVERALL], RunTime(ms), 16487.0
[OVERALL], Throughput(ops/sec), 6065.384848668648
```

- `[OVERALL]` 区显示测试总体情况
    - `RunTime(ms)` 运行总时间
    - `Throughput(ops/sec)` 吞吐量，每秒操作数

```
[TOTAL_GCS_PS_Scavenge], Count, 23.0
[TOTAL_GC_TIME_PS_Scavenge], Time(ms), 88.0
[TOTAL_GC_TIME_%_PS_Scavenge], Time(%), 0.5337538666828411
[TOTAL_GCS_PS_MarkSweep], Count, 0.0
[TOTAL_GC_TIME_PS_MarkSweep], Time(ms), 0.0
[TOTAL_GC_TIME_%_PS_MarkSweep], Time(%), 0.0
[TOTAL_GCs], Count, 23.0
[TOTAL_GC_TIME], Time(ms), 88.0
[TOTAL_GC_TIME_%], Time(%), 0.5337538666828411
```

- `[TOTAL_GC*]` 区显示垃圾回收情况（[JVM有关垃圾回收机制的配置](http://www.jdon.com/idea/jvm-gc.html)）
    - `[TOTAL_GCS_PS_Scavenge], Count, 23.0` Parallel Scavenge 回收次数
    - `[TOTAL_GC_TIME_PS_Scavenge], Time(ms), 88.0` Parallel Scavenge 回收时间
    - `[TOTAL_GC_TIME_%_PS_Scavenge], Time(%), 0.5337538666828411` Parallel Scavenge 回收时间百分比
    - `[TOTAL_GCS_PS_MarkSweep], Count, 0.0` PS MarkSweep 回收次数
    - `[TOTAL_GC_TIME_PS_MarkSweep], Time(ms), 0.0` PS MarkSweep 回收时间
    - `[TOTAL_GC_TIME_%_PS_MarkSweep], Time(%), 0.0` PS MarkSweep 回收时间百分比
    - `[TOTAL_GCs], Count, 23.0` 全局 GC 次数
    - `[TOTAL_GC_TIME], Time(ms), 88.0` 全局 GC 时间
    - `[TOTAL_GC_TIME_%], Time(%), 0.5337538666828411` 全局 GC 时间百分比

```
[READ], Operations, 50011.0
[READ], AverageLatency(us), 4418.49443122513
[READ], MinLatency(us), 1584.0
[READ], MaxLatency(us), 208895.0
[READ], 95thPercentileLatency(us), 8207.0
[READ], 99thPercentileLatency(us), 11463.0
[READ], Return=OK, 50011
```

- `[READ]` 区显示读取操作的统计结果
    - `Operations` 总操作数
    - `AverageLatency(us)` 平均延迟（微秒）
    - `MinLatency(us)` 最小延迟
    - `MaxLatency(us)` 最大延迟
    - `95thPercentileLatency(us)` 第 95 百分位延迟（[第95个百分位（95th percentile）是什么概念？](https://www.zhihu.com/question/20575291)）
    - `99thPercentileLatency(us)` 第 99 百分位延迟
    - `Return=OK, 50011` 结果（正确），总操作数

`[CLEANUP]`（清理操作）、`[UPDATE]`（更新操作）等等和 `[READ]` 区类似，不再介绍。

### 并行测试

客户端默认会读取并操作所有的测试记录数据，在想多个客户端并行测试时这可能有些问题。可以用 `insertstart` 和 `insertcount` 两个参数来限制客户端操作测试纪录的范围，例如对于 4 个客户端测试 100000000 条纪录可以这样配置：

第一个客户端：

```
insertstart=0
insertcount=25000000
```

第二个客户端：

```
insertstart=25000000
insertcount=25000000
```

第三个客户端：

```
insertstart=50000000
insertcount=25000000
```

第四个客户端：

```
insertstart=75000000
insertcount=25000000
```
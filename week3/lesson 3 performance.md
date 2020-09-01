##题目描述
分值： 1个有效issues，有效pr根据实际效果进行加分，比如能节省CPU，减少内存占用，减少IO次数等等。
使用上一节可以讲的 sysbench、go-ycsb 或者 go-tpc 对 TiDB 进行压力测试，然后对 TiDB 或 TiKV 的 CPU 、内存或 IO 进行 profile，寻找潜在可以优化的地方并提 enhance 类型的 issue 描述。

issue 描述应包含：

部署环境的机器配置（CPU、内存、磁盘规格型号），拓扑结构（TiDB、TiKV 各部署于哪些节点）
跑的 workload 情况
对应的 profile 看到的情况
建议如何优化？
【可选】提 PR 进行优化：按照 PR 模板提交优化 PR
输出：对 TiDB 或 TiKV 进行 profile，写文章描述分析的过程，对于可以优化的地方提 issue 描述，并将 issue 贴到文章中（或【可选】提 PR 进行优化，将 PR 贴到文章中）

##tidb issues
https://github.com/pingcap/tidb/issues/19686

##部署环境 
aliyunECS * 5 
机器配置
|instance	|vCPU|	mem（GiB）	|local(GiB)|	网络带宽能力(O/I)(Gbit/s)	|网络收发包能力(O+I)(万PPS)|IPv6|连接数(万)|多队列|弹性网卡(包括一块主网卡)|单块弹性网卡的私有IP	|云盘IOPS(万)	|云盘带宽(Gbit/s)|云盘类型|
|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|
|ecs.c6e.xlarge|	4|	8.0	|无	|突发最高10.0|	100|	support|	max25|	4|	4|	15|	4.0|	1.5|ESSD云盘PL0 40GiB (2280 IOPS)|

##拓扑结构
**DB+DB+(KV+PD)+(KV+PD)+(DeployNode+Monitor)**
```
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

monitored:
  node_exporter_port: 9100
  blackbox_exporter_port: 9115

server_configs:
  tidb:
    log.slow-threshold: 300
    binlog.enable: false
    binlog.ignore-error: false
  tikv:
    server.grpc-concurrency: 4
    raftstore.apply-pool-size: 2
    raftstore.store-pool-size: 2
    rocksdb.max-sub-compactions: 1
    storage.block-cache.capacity: "4GB"
    readpool.unified.max-thread-count: 12
    readpool.storage.use-unified-pool: false
    readpool.coprocessor.use-unified-pool: true
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64

pd_servers:
  - host: xxx.xxx.xxx.xx8
  - host: xxx.xxx.xxx.xx0

tidb_servers:
  - host: xxx.xxx.xxx.xx9
  - host: xxx.xxx.xxx.xx7

tikv_servers:
  - host: xxx.xxx.xxx.xx0
  - host: xxx.xxx.xxx.xx8

monitoring_servers:
  - host: xxx.xxx.xxx.xx6

grafana_servers:
  - host: xxx.xx.xxx.xx6

alertmanager_servers:
  - host: xxx.xxx.xxx.xx6
```

尝试使用vTune，但是AliyunECS centos7.8 不支持开启vPMU 无法获取数据，卒。
```
cat /proc/cpuinfo|grep arch_perfmon
```

profile tidb cpu usage(1/3) 获取数据
```
curl http://182.92.88.75:10080/debug/zip?seconds=60 --output debug.zip
unzip debug.zip
tree
.
├── config
├── goroutine
├── heap
├── mutex
├── profile
└── version

0 directories, 6 files
```

profile tidb cpu usage(2/3) 图形化分析
```
go tool pprof -http:8080 debug/profile
```
profile tidb cpu usage(3/3)
在页面中可以查看调用的svg关系图和火焰图

profile tikv cpu usage 
使用tikv dashboard 或者 通过 git clone https://github.com/pingcap/tidb-inspect-tools获取 tracing_tools/perf 工具
```
sudo sh ./cpu_tikv.sh $tikv-pid
sudo sh ./dot_tikv.sh $tikv-pid
```
试用sysbench 进行 olap_read_write.lua 进行随机读写测试
问题1 在prepare阶段两个节点的kv集群只有一个负载很高

```
mysql-user=root
mysql-password=
mysql-port=4000
mysql-host=172.27.133.127
mysql-db=sbtest
time=600
threads=8
report-interval=5
verbosity=5
validate
db-driver=mysql
#percentile=99
```
db=8 records=1000000
prepare
![IMAGE](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/5F211CDD464100348A33EBD25C2E7E44.jpg)
run阶段 
![IMAGE](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/09B745DF57C1829CFE068B7FA1DB9AC0.jpg)
![IMAGE](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/045668831F7A242F1797EB83D47424AB.jpg)
cpu使用和memory和qps也都集中在kv-node-1上

```
mysql-user=root
mysql-password=
mysql-port=4000
mysql-host=172.27.133.127
mysql-db=sbtest
time=120
threads=128
report-interval=5
verbosity=5
validate
db-driver=mysql
percentile=99
```
db=32 records=1000000 
![IMAGE](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/CD04CA0797C0FFA2B479E63F8F43221C.jpg)

```
SQL statistics:
    queries performed:
        read:                            7896756
        write:                           0
        other:                           0
        total:                           7896756
    transactions:                        7896756 (13161.19 per sec.)
    queries:                             7896756 (13161.19 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0017s
    total number of events:              7896756

Latency (ms):
         min:                                    0.20
         avg:                                    0.30
         max:                                  260.67
         95th percentile:                        0.37
         sum:                              2397419.63

Threads fairness:
    events (avg/stddev):           1974189.0000/4457.03
    execution time (avg/stddev):   599.3549/0.00
```
pd config 中和region调度相关的应该也没有问题
```
    "enable-debug-metrics": "false",
    "enable-location-replacement": "true",
    "enable-make-up-replica": "true",
    "enable-one-way-merge": "false",
    "enable-remove-down-replica": "true",
    "enable-remove-extra-replica": "true",
    "enable-replace-offline-replica": "true",
    "high-space-ratio": 0.7,
    "hot-region-cache-hits-threshold": 3,
    "hot-region-schedule-limit": 4,
    "leader-schedule-limit": 4,
    "leader-schedule-policy": "count",
    "low-space-ratio": 0.8,
    "max-merge-region-keys": 200000,
    "max-merge-region-size": 20,
    "max-pending-peer-count": 16,
    "max-snapshot-count": 3,
    "max-store-down-time": "30m0s",
    "merge-schedule-limit": 8,
    "patrol-region-interval": "100ms",
    "region-schedule-limit": 2048,
    "replica-schedule-limit": 64,
    "scheduler-max-waiting-operator": 5,
    "split-merge-interval": "1h0m0s",
    "store-limit-mode": "manual",
    "tolerant-size-ratio": 0
```


IO
```
iostat -x 1 -m 
```
|关注参数|关注问题|
|-|-|
|await|平均每次设备I/O操作的等待时间(毫秒)|
|r_await|读操作等待时间|
|w_await|写操作等待时间|
|iosize|连续性|
|io|连续性|
|io|读写瞬间流量|
|iops|每秒的读写次数|

iostat从disk角度出发
iotop 从进程角度出发
```
iotop -o
```
iosnoop 磁盘延迟毛刺
```
iosnoop -ts
```

Memory
```
go tool pprof -http=:8080 debug/heap
```
```
go tool ppof -alloc_space -http=:8080 debug/heap
```
Mutex
```
go tool pprof -contentions -http=:8080 debug/mutex
```

Profile TIKV Memory
```
cd tracing_tools/perf
sudo sh mem.sh $tikv-pid
```

debug.zip和dashboard导出获得的火焰图和调用图
[profiling_6_19_tikv_172_27_133_128_20160164743229.svg](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/D4B209A5ABE4D23FAD23A7131387BD78.svg)
[profiling_6_20_tikv_172_27_133_130_20160294708344.svg](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/30A528A7F1AA4F31D49AD91A6C39A570.svg)
[profiling_7_22_tidb_172_27_133_129_4000044554540.svg](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/DE17C85BF2FEF1025759DE1291170E90.svg)
[profiling_7_21_tidb_172_27_133_127_4000333331611.svg](https://github.com/SailerNote/High-Performance-TiDB-StudyPlan/blob/master/week2/resources/29D7729DD734FB3B9D396385D4020FBF.svg)


尝试进行扩容多个tikv进行测试，不过aliyun线路出现了问题，目前不能添加ECS。




## 基本步骤  
### 获取源码   
```
# make dir
mkdir ti-src 
mkdir ti-bin
cd ti-src
# git clone
git clone -b v4.0.4 https://github.com/pingcap/tidb.git
git clone -b v4.0.4 https://github.com/pingcap/pd.git
git clone -b v4.0.4 https://github.com/tikv/tikv.git
```
### 修改TIDB代码  
```go
// file: /kv/txn.go
	for i := uint(0); i < maxRetryCnt; i++ {
		logutil.BgLogger().Info("hello transaction")  // here
		txn, err = store.Begin()

```
### 本地编译  
```
# change dir name 
//change dir name tidb, pd, tikv
# cd tidb pd tikv
cd tidb
make
//...
cd pd
make
//...
cd tikv
make
//...
```
### 使用Tiup构建集群  
```
cd ti-bin
cp ~/ti-src/tidb/bin/tidb-server .
cp ~/ti-src/tikv/bin/tikv-server .
cp ~/ti-src/pd/bin/pd-server . 
tiup playground --db.binpath ~/ti-bin/tidb-server  --pd.binpath ~/ti-bin/pd-server --kv.binpath ~/ti-bin/tikv-server --db 1 --pd 1 --kv 3
```



以下记录按照timeline进行，记录完整homework的过程和遇到的问题。

**Q 交代一下开发环境和编译的目标版本**	  
_Thu Aug 13 02:09:01 CST 2020_	  
OS macOS Catalina v10.15.5	  
Goland 编译TiDB PD	  
Clion 用来编译TiKV	  
DataGrip 作为sql client	  
GolangSDK version v1.13.11	  
tidb release-v4.0.4	  
tikv release-v4.0	  
pd   release v4.0	  

**Q 直接寻找可重现的文档**  
_hu Aug 13 02:12:17 CST 2020_  	
网上可以找到可以tidb编译的相关资料  
TiDB - 如何在国内编译 	  
https://my.oschina.net/tzj/blog/2875585  
PingCap社区文章 q=编译  
https://docs.pingcap.com/zh/search/?lang=zh&type=tidb&version=v4.0&q=%E7%BC%96%E8%AF%91   
如何在没有代理的情况下编译 tidb server  
https://www.cnblogs.com/lijingshanxi/p/10890232.html  

**Q 尝试下载代码**  
_Thu Aug 13 02:12:50 CST 2020_  
```
$ du -sh tidb-4.0.4
  33M	tidb-4.0.4
```
代码大小25M   

**Q goland plugins**  
_Thu Aug 13 02:24:49 CST 2020_	  
GoYacc 用来格式化yacc文件  
IdeaVim 官方Vim  
Makefile support  
WakaTime 统计编程时间  

**如何编译**  
_Thu Aug 13 02:28:04 CST 2020_  
x因为不知道goland的ToolsChain  
这里去看看github的ReadME和wiki有没有什么说法  
答案是没有   
那就直接makefile里面的default试试  
毕竟  
```
	@echo Build TiDB Server successfully!
```
```
/usr/bin/make -f /Users/conor/go/src/tidb-4.0.4/Makefile default

CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-12 06:44:55" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
```
makefilelog termianl是茫茫多的日志拉下来很多的依赖. 
应该是和我不用golang开发有关系. 
因为这里面有一些. 
```
go: finding google.golang.org/api v0.7.0
```
所以需要代理不然有些依赖可能获取不到。   

**Q 在这个地方卡住了**
_Thu Aug 13 02:52:32 CST 2020_  
```
go: github.com/pingcap/tidb@v1.1.0-beta.0.20200715100003-b4da443a3c4c: git fetch -f origin refs/heads/*:refs/heads/* refs/tags/*:refs/tags/* in /Users/conor/go/pkg/mod/cache/vcs/023ec28de881fe16123b69e600d28e8671b1fa6b70863a0d65ca08ea4bcc7d6d: exit status 128:
	error: RPC failed; curl 18 transfer closed with outstanding read data remaining
	fatal: the remote end hung up unexpectedly
	fatal: early EOF
	fatal: index-pack failed
go: error loading module requirements
make: *** [server] Error 1
```
keywors:  
```
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
```
"git clone时RPC failed; curl 18 transfer closed with outstanding read data remaining"
https://www.cnblogs.com/zjfjava/p/10392150.html   
```
$ git config http.postBuffer 524288000
fatal: not in a git directory
```
这里比较尴尬，我用的源码不是从指定的commit上clone下来的，  
那无git的源码就不能直接make了吗。  

**Q 删除源码使用git的**  
_Thu Aug 13 03:00:18 CST 2020_  
https://github.com/pingcap/tidb/tree/release-4.0.4
```
git clone https://github.com/pingcap/tidb.git
git checkout release-4.0.4
```
然而还是卡在  
```
go: finding github.com/pingcap/tidb v1.1.0-beta.0.20200715100003-b4da443a3c4c
```
直接make  
```
make
```
原来goland在不配置的情况下不走代理。甚至不走termianl的代理。  
在Pycharm/Perference/搜索Proxy  
配饰manual_paoxy  

**Q 编译pd**    
_Thu Aug 13 03:21:17 CST 2020_  
```
git clone https://github.com/pingcap/pd.git
cd pd
git checkout release-4.0
make
```
成功

**Q 编译tikv**   
_Thu Aug 13 03:26:19 CST 2020_  
tikv是rust的 要上clion  
```
info: component 'rustfmt' for target 'x86_64-apple-darwin' is up to date
```
在macos下不知道能不能正常编译

**Q warning**  
_Thu Aug 13 03:43:09 CST 2020_  
```rust
warning: variable does not need to be mutable
   --> src/server/gc_worker/applied_lock_collector.rs:138:13
    |
138 |         let mut send_fp = || {
    |             ----^^^^^^^
    |             |
    |             help: remove this `mut`
    |
    = note: `#[warn(unused_mut)]` on by default


```
编译的时候看到个warning，当然我不觉得这玩意有机会提出pr
```
        #[allow(unused_mut)]
        let mut send_fp = || {
```
**Q 编译成功**    
_Thu Aug 13 03:53:46 CST 2020_  
不过我用的 all 参数这里在进行测试

**Q pd是做什么的**    
_Thu Aug 13 03:54:25 CST 2020_  
>PD是Placement Driver的缩写。它用于管理和调度TiKV集群。 PD通过嵌入etcd支持分布和容错。

**Q 测试不通过**  
_Thu Aug 13 04:15:36 CST 2020_    
有很多测试 没有通过。不过目标是“启动事物事务log写一段”
先尝试搭建集群

**Q 如何run起来服务，如何构建集群**  
_Thu Aug 13 04:23:18 CST 2020_    

**Q 怎么算自己搭建**      
_Thu Aug 13 11:34:44 CST 2020_  
可以使用官方的几个工具嘛。
然后发现了这个
https://docs.pingcap.com/zh/tidb/dev/tiup-playground
```
Flags:
      --db int                   设置集群中的 TiDB 数量（默认为1）
      --db.binpath string        指定 TiDB 二进制文件的位置（开发调试用，可忽略）
      --db.config string         指定 TiDB 的配置文件（开发调试用，可忽略）
      --db.host host             指定 TiDB 的监听地址
      --drainer int              设置集群中 Drainer 数据
      --drainer.binpath string   指定 Drainer 二进制文件的位置（开发调试用，可忽略）
      --drainer.config string    指定 Drainer 的配置文件
  -h, --help                     打印帮助信息
      --host string              设置每个组件的监听地址（默认为 127.0.0.1），如果要提供给别的电脑访问，可设置为 0.0.0.0
      --kv int                   设置集群中的 TiKV 数量（默认为1）
      --kv.binpath string        指定 TiKV 二进制文件的位置（开发调试用，可忽略）
        --kv.config string         指定 TiKV 的配置文件（开发调试用，可忽略）
        --monitor                  是否启动监控
      --pd int                   设置集群中的 PD 数量（默认为1）
      --pd.binpath string        指定 PD 二进制文件的位置（开发调试用，可忽略）
      --pd.config string         指定 PD 的配置文件（开发调试用，可忽略）
        --pump int                 指定集群中 Pump 的数量（非 0 的时候 TiDB 会开启 TiDB Binlog）
        --pump.binpath string      指定 Pump 二进制文件的位置（开发调试用，可忽略）
      --pump.config string       指定 Pump 的配置文件（开发调试用，可忽略）
    --tiflash int              设置集群中 TiFlash 数量（默认为0）
      --tiflash.binpath string   指定 TiFlash 的二进制文件位置（开发调试用，可忽略）
      --tiflash.config string    指定 TiFlash 的配置文件（开发调试用，可忽略）
```
注意其中的
```
      --db.binpath string        指定 TiDB 二进制文件的位置（开发调试用，可忽略）
      --pd.binpath string        指定 PD 二进制文件的位置（开发调试用，可忽略）
    --kv.binpath string        指定 TiKV 二进制文件的位置（开发调试用，可忽略）
```

**Q 安装tiup**  
_Thu Aug 13 12:51:16 CST 2020_  
官方github  
https://github.com/pingcap/tiup
```
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | shs
```
安装后会有常见的环境变量的问题，在~/.profile ~/.zshrc里面配置一下，
source或者重启一个termianl

**Q 已经配置了路径但是执行命令的时候 应该是没有找到二进制文件的意思**  
_Thu Aug 13 13:01:05 CST 2020_  
```
The component `tidb` is not installed; downloading from repository.
```
手动测试三组服务，可以启动  
```
./pd-server
./tidb-server
./tikv-server
```
>默认启动 playground 时，各个组件都是使用官方镜像组件包中的二进制文件启动的，如果本地编译了一个临时的二进制文件想要放入集群中测试，可以使用 --{comp}.binpath 这个参数替换，例如执行以下命令替换 TiDB 的二进制文件：
>```
>tiup playground --db.binpath /xx/tidb-server  

指定启动二进制文件目录，指定集群内节点数量  
```
tiup playground --db.binpath /Users/conor/go/src/cluster/tidb-server  --pd.binpath /Users/conor/go/src/cluster/pd-server --kv.binpath /Users/conor/go/src/cluster/tikv-server --db 1 --pd 1 --kv 3
Starting component `playground`: /Users/conor/.tiup/components/playground/v1.0.9/tiup-playground --db.binpath /Users/conor/go/src/cluster/tidb-server --pd.binpath /Users/conor/go/src/cluster/pd-server --kv.binpath /Users/conor/go/src/cluster/tikv-server --db 1 --pd 1 --kv 3
Use the latest stable version: v4.0.4

    Specify version manually:   tiup playground <version>
    The stable version:         tiup playground v4.0.0
    The nightly version:        tiup playground nightly

Playground Bootstrapping...
Start pd instance...
Start tikv instance...
Start tikv instance...
Start tikv instance...
Start tidb instance...
......
Waiting for tikv 127.0.0.1:20160 ready
Waiting for tikv 127.0.0.1:20161 ready
Waiting for tikv 127.0.0.1:20162 ready
Start tiflash instance...
The component `tiflash` is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/tiflash-v4.0.4-darwin-amd64.tar.gz 55.09 MiB / 55.09 MiB 100.00% 52.39 MiB p/s
Waiting for tiflash 127.0.0.1:3930 ready ....
CLUSTER START SUCCESSFULLY, Enjoy it ^-^
To connect TiDB: mysql --host 127.0.0.1 --port 4000 -u root
To view the dashboard: http://127.0.0.1:2379/dashboard
To view the Prometheus: http://127.0.0.1:53006
To view the Grafana: http://127.0.0.1:3000
```
这里其他的组件用了官方的安装
不过目的应该是能够对上面3个工具进行编译开发，相比这里应该没有问题。
其实官方这里推荐的也是试用 Cluster Platground 和 Ansible是三种部署方式

**Q这里默认的dashboard用户密码是什么**  
_Thu Aug 13 13:18:49 CST 2020_  
emmm 直接signin就可以了    
ps dashboard的账号是root密码空 Grafana的账号admin密码admin    
左侧cluster info 可以查看集群节点信息，  
看Deployment Directory应该是没有问题  
TiFlash挂了 先继续有问题再来启动，再不行就也编译一个本地的  

**尝试链接**  
_Thu Aug 13 13:22:54 CST 2020_  
mysql --host 127.0.0.1 --port 4000 -u root
这里用的是JB家的DataGrip

**Q 尝试查询 创建表 创建事务**  
_Thu Aug 13 13:29:48 CST 2020_  
```
[2020-08-13 13:34:22] Connected
> create database tidemo
[2020-08-13 13:34:22] completed in 79 ms
> use tidemo
[2020-08-13 13:35:00] completed in 8 ms

[2020-08-13 13:37:07] 'id' INT UNSIGNED AUTO_INCREMENT,
[2020-08-13 13:37:07] 'title' VARCHAR(100) NOT NULL,
[2020-08-13 13:37:07] 'author' VARCHAR(40) NOT NULL,
[2020-08-13 13:37:07] 'date' DATE,
[2020-08-13 13:37:07] PRIMARY KEY ( 'id' )
[2020-08-13 13:37:07] )ENGINE=InnoDB DEFAULT CHARSET=utf8"
tidemo> CREATE TABLE IF NOT EXISTS tbl(
           id INT UNSIGNED AUTO_INCREMENT,
           title VARCHAR(100) NOT NULL,
           author VARCHAR(40) NOT NULL,
           date DATE,
           PRIMARY KEY ( id )
        )ENGINE=InnoDB DEFAULT CHARSET=utf8
[2020-08-13 13:37:37] completed in 133 ms
```




**Q 事务的起点在哪里**  
_Sun Aug 16 10:59:10 CST 2020_  
全局搜索transaction并且限定*.go文件  
可以找到方法  
```
func RunInNewTxn(store Storage, retryable bool, f func(txn Transaction) error) error {
```
然后发现有`Transaction`应该是个interface追过去看实现
```
// Transaction defines the interface for operations inside a Transaction.
// This is not thread safe.
type Transaction interface {
	MemBuffer
	// Commit commits the transaction operations to KV store.
	Commit(context.Context) error
	// Rollback undoes the transaction operations to KV store.
	Rollback() error
	// String implements fmt.Stringer interface.
	String() string
	// LockKeys tries to lock the entries with the keys in KV store.
	LockKeys(ctx context.Context, lockCtx *LockCtx, keys ...Key) error
	// SetOption sets an option with a value, when val is nil, uses the default
	// value of this option.
	SetOption(opt Option, val interface{})
	// DelOption deletes an option.
	DelOption(opt Option)
	// IsReadOnly checks if the transaction has only performed read operations.
	IsReadOnly() bool
	// StartTS returns the transaction start timestamp.
	StartTS() uint64
	// Valid returns if the transaction is valid.
	// A transaction become invalid after commit or rollback.
	Valid() bool
	// GetMemBuffer return the MemBuffer binding to this transaction.
	GetMemBuffer() MemBuffer
	// GetSnapshot returns the Snapshot binding to this transaction.
	GetSnapshot() Snapshot
	// SetVars sets variables to the transaction.
  	SetVars(vars *Variables)
	// GetVars gets variables from the transaction.
	GetVars() *Variables
	// BatchGet gets kv from the memory buffer of statement and transaction, and the kv storage.
	// Do not use len(value) == 0 or value == nil to represent non-exist.
	// If a key doesn't exist, there shouldn't be any corresponding entry in the result map.
	BatchGet(ctx context.Context, keys []Key) (map[string][]byte, error)
  	IsPessimistic() bool
  }
  ```
  里面有常见的`SetVars`,`commit`,`rollback` ，那么begin哪里去了。
  
  **Q 事物的Begin方法在哪里。**  
  _Sun Aug 16 11:02:59 CST 2020_  
  全局搜索Begin() MatchCase *.go  
  ```
  // Storage defines the interface for storage.
  // Isolation should be at least SI(SNAPSHOT ISOLATION)
  type Storage interface {
  	// Begin transaction
  	Begin() (Transaction, error)
	// BeginWithStartTS begins transaction with startTS.
  	BeginWithStartTS(startTS uint64) (Transaction, error)
  	// GetSnapshot gets a snapshot that is able to read any data which data is <= ver.
  	// if ver is MaxVersion or > current max committed version, we will use current version for this snapshot.
  	GetSnapshot(ver Version) (Snapshot, error)
  	// GetClient gets a client instance.
  	GetClient() Client
  	// Close store
  	Close() error
  	// UUID return a unique ID which represents a Storage.
	UUID() string
  	// CurrentVersion returns current max committed version.
  	CurrentVersion() (Version, error)
	// GetOracle gets a timestamp oracle client.
  	GetOracle() oracle.Oracle
  	// SupportDeleteRange gets the storage support delete range or not.
  	SupportDeleteRange() (supported bool)
  	// Name gets the name of the storage engine
  	Name() string
  	// Describe returns of brief introduction of the storage
  	Describe() string
	// ShowStatus returns the specified status of the storage
	ShowStatus(ctx context.Context, key string) (interface{}, error)
}
```
Ok, 这个叫Stage中有一个可以返回Transaction的Begin(), 这里居然是拆开写的  
原因我也不知道。  


**Q 加入需要输出的“hello transaction”**  
_Sun Aug 16 11:10:48 CST 2020_  

```
#file kv/txn.go 		
txn, err = store.Begin()
```
```
[2020/08/16 15:37:22.304 +08:00] [INFO] [txn.go:37] ["hello transaction"]
[2020/08/16 15:37:22.304 +08:00] [INFO] [txn.go:37] ["hello transaction"]
...
```


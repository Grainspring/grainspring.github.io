---
layout: post
title:  "架构杂谈:TiKV弹性设计及实现"
date:   2022-03-28 00:08:08
categories: Rust LRFRC TiKV async
excerpt: 学习Rust
---

* content
{:toc}

TiKV作为首屈一指的用Rust语言实现的分布式KV存储引擎，获得了业界极大的关注和商业上的成功。网上也有不少TiKV架构设计及源码解析的文章，这篇文章主要探讨TiKV如何运用Rust语言的抽象及并发来实现TiKV的弹性，以供后续深入运用Rust语言和TiKV备忘参考。

![TiKV架构](/imgs/tikv_overview.png "TiKV架构")

每一个TiDB节点实例作为TiKV Client来与TiKV节点实例进行交互；

每一个TiKV节点实例包含一个存储实例Store，可包括多个Raft Regions，与PD集群节点进行交互；

PD集群作为整个TiDB/TiKV数据服务的控制中心，以上帝视角来调度、监控TiKV各个节点实例的状态，动态弹性触发调整数据分区存储、查询、副本迁移等；

Raft Group Regions中分区Region是TiKV达成分布共识，弹性分片的核心；

一个Raft Group Region对应一个Raft共识分组，由多个分布在不同节点中的Region组成，它只负责指定Region内的K/V存储及查询；

一个Raft Group Region中的Region谁是Leader、Follower由PD根据系统状态动态来决定；

---
#### 二、何为TiKV弹性设计？
这里讲的TiKV弹性主要是指TiKV的动态伸缩能力，它可以在PD调度策略安排下，动态支持不同分区的数据写入、读取、迁移，以达到数据安全、读写流量均衡、高性能的目的。

分布式存储系统作为一个复杂的系统，涉及到网络、存储、安全基础设施，其动态伸缩的弹性设计其实是整个系统的核心，直接决定系统的优劣。

TiKV弹性涉及的主要方面包括：
##### 1、TiKV节点实例启动、退出的弹性支持
每一个TiKV节点首次启动时，须与PD交互注册分配到属于自己的唯一ID，并记录到本地存储中；

![TiKV](/imgs/tikv_tidb_pd.png "TiKV")

启动成功后，不间断向PD汇报当前节点的状态，PD根据收集到的所有TiKV节点状态及整个系统的状态和配置后，通知TiKV节点加入或退出或Split或Merge某些分区Region；

---
##### 2、TiKV节点中多个不同分区Region的弹性支持
###### 2.1.多个Raft共识分组支持

TiKV独创性的实现了一个TiKV节点可支持多个Raft共识分组；
![TiKV Raft Groups](/imgs/tikv_raft_group.png "TiKV Raft Groups")

这样极大增加了系统实现的复杂性，其目的在于可将写入和读取流量均衡的分配到不同TiKV节点，增加系统的弹性，这也成为TiKV架构设计及实现的突出亮点；

这样设计的原因，跟Raft共识算法保证强一致性的实现逻辑及性能需求相关，因为Raft分布式共识算法本身往往离不开CAP理论的约束；

在越来越多的大量高并发的数据读取、写入需求下，分区分片处理数据成为一种必然的选择，于是TiKV选择一个节点支持多个Raft共识分组，一个共识分组的读取、写入流量由该共识分组的Leader节点来处理，其他Follower角色节点往往只负责达成共识、存储数据副本、时刻准备成为Leader；

---
###### 2.2.动态参与不同Region的数据存储
一个Region对应Key值在指定范围[a, b)的数据存储，一个Region A可分割成两个Region A1 [a, c)和Region A2 [c, b)，两个Region可合并成一个新的Region；

从TiKV节点的角度来看，它可以不同角色参与到不同Region的数据存储，动态支持加入、退出、分割、合并Region，以保证数据及流量的动态调整；

从PD的角度来看，由它来为不同Region的分配ID标识、记录状态包括其Range、触发不同TiKV节点以不同角色加入、退出、分割、合并某个Region；

TiKV Client则依据PD提供的Region状态，根据自身业务以Region为主自主决定与不同TiKV节点进行交互；

---
##### 3.不同查询算子的弹性支持

由于TiKV节点离数据最近，TiKV可高性能低流量的方式支持不同的查询子算法，这样TiDB可动态分配不同算子给不同TiKV节点，以无状态聚合者的模式来实现SQL查询需求，增加整个系统查询需求的弹性支撑；

---
##### 4.原生KV和事务型KV操作支持

TiKV同时支持原生KV操作和基于MVCC算法的事务型KV操作，以支持多种使用场景的KV操作需求；

原生KV操作交互流程如下：
![TiKV RawKV](/imgs/tikv_rawkv.jpeg "TiKV RawKV")


事务型KV操作交互流程如下：
![TiKV TxnKV](/imgs/tikv_txnkv.jpeg "TiKV TxnKV")


---
#### 三、TiKV分层实现

TiKV的实现可分为API接口层Server/RaftClient、事务支撑层Storage、共识/算法层RaftBatchSystem/RaftGroup Regions、数据存储层RaftEngine/KvEngine，如下图所示：

![TiKV架构实现](/imgs/tikv_arch_impl.png "TiKV架构实现")

从IO及数据流转角度看，API接口层和数据存储层，涉及大量Socket IO和File IO，需要异步读写支持；

事务支撑层、共识/算法层，涉及大量并发并行的计算任务，需要异步并发任务池支撑；

---
#### 四、TiKV中应用Rust语言抽象
Rust语言作为新一代系统编程语言，提供强大的内存安全、高性能的框架支持；

一个好的系统架构设计往往会通过抽象分层来解藕聚合架构的各个部分，从而让整个系统有机的整合在一起；

##### 4.1.Trait接口定义及类型实现

在TiKV内部实现中大量运用先定义Trait接口，再在不同泛型对象结构实现不同Trait接口，比如：

ServerTransport实现Transport Trait用来向不同TiKV实例发送RaftMsg；

RaftRouter实现RaftStoreRouter和LocalReadRouter Trait分别用来发送StoreMsg/RaftMsg改写请求和读请求对应的Message；

RaftKv实现Engine Trait用来实现高层次数据的异步改写、获取快照能力；

BatchSystem中抽象出Fsm、FsmScheduler、PollHandler Trait，

其中ControlFsm、StoreFsm、PeerFsm、ApplyFsm实现Fsm Trait来表示它们可接收相应类型的Message并进行处理；

RaftPoller和ApplyPoller实现PollHandler Trait来批量处理StoreMsg、RaftMsg和ApplyTask，以便不同region的写操作一次性写入RocksDB；

NormalScheduler实现FsmScheduler Trait来表示收到指定Fsm相关Message后，触发PollHandler实现者来处理指定Fsm收到的Msg；

---
###### 4.1.1.KvEngine和RaftEngine引擎Trait定义示例：
```
/// A TiKV key-value store
pub trait KvEngine:
    Peekable
    + SyncMutable
    + Iterable
    + WriteBatchExt
    + DBOptionsExt
    + CFNamesExt
    + CFOptionsExt
    + ImportExt
    + SstExt
    + TablePropertiesExt
    + CompactExt
    + RangePropertiesExt
    + MvccPropertiesExt
    + TtlPropertiesExt
    + PerfContextExt
    + MiscExt
    + Send
    + Sync
    + Clone
    + Debug
    + Unpin
    + 'static
{
    /// A consistent read-only snapshot of the database
    type Snapshot: Snapshot;
    /// Create a snapshot
    fn snapshot(&self) -> Self::Snapshot;
    /// Syncs any writes to disk
    fn sync(&self) -> Result<()>;
    /// Flush metrics to prometheus
    /// `instance` is the label of the metric to flush.
    fn flush_metrics(&self, _instance: &str) {}
    /// Reset internal statistics
    fn reset_statistics(&self) {}
    /// Cast to a concrete engine type
    /// This only exists as a temporary hack during refactor.
    /// It cannot be used forever.
    fn bad_downcast<T: 'static>(&self) -> &T;
}
```

KvEngine Trait基于多个Super Trait来描述一个基础KvEngine需要提供的能力，其Snapshot作为关联类型来支持引擎数据的读取；

```
pub trait RaftEngine: Clone + Sync + Send + 'static {
    type LogBatch: RaftLogBatch;
    fn log_batch(&self, capacity: usize) -> Self::LogBatch;
    /// Synchronize the Raft engine.
    fn sync(&self) -> Result<()>;
    fn get_raft_state(&self, raft_group_id: u64)
        -> Result<Option<RaftLocalState>>;
    fn get_entry(&self, raft_group_id: u64, index: u64)
        -> Result<Option<Entry>>;
    /// Append some log entries and return written bytes.
    fn append(&self, raft_group_id: u64, entries: Vec<Entry>)
        -> Result<usize>;
    fn put_raft_state(&self, raft_group_id: u64
        , state: &RaftLocalState) -> Result<()>;
    // 省略部分其他方法
}

pub trait RaftLogBatch: Send {
    fn append(&mut self, raft_group_id: u64
        , entries: Vec<Entry>) -> Result<()>;
    /// Remove Raft logs in [`from`, `to`) 
    /// which will be overwritten later.
    fn cut_logs(&mut self, raft_group_id: u64
        , from: u64, to: u64);
    fn put_raft_state(&mut self, raft_group_id: u64
        , state: &RaftLocalState) -> Result<()>;
    fn is_empty(&self) -> bool;
}
```
RaftEngine trait用来描述不同raft分组存储raft log的引擎，其LogBatch作为关联类型来支持RaftEngine；

```
#[derive(Clone, Debug)]
pub struct Engines<K, R> {
    pub kv: K,
    pub raft: R,
}

impl<K: KvEngine, R: RaftEngine> Engines<K, R> {
    pub fn new(kv_engine: K, raft_engine: R) -> Self {
        Engines {
            kv: kv_engine,
            raft: raft_engine,
        }
    }
    // 省略部分其他方法
}
```

由于每一次来自Client端的写入都需要先以Log的方式写入RaftEngine，待可apply后才写入KvEngine，所以TiKV抽象出一个包含kv和raft的Engines<K, R>结构来支持数据的读取与写入；

---
###### 4.1.2.KvEngine Trait实现
```
//基于RocksDB来提供RocksEngine以实现KvEngine trait；
pub struct RocksEngine {
    db: Arc<DB>,
    shared_block_cache: bool,
}

impl KvEngine for RocksEngine {
    type Snapshot = RocksSnapshot;
    fn snapshot(&self) -> RocksSnapshot {
        RocksSnapshot::new(self.db.clone())
    }
    // 省略部分其他方法
}
```

---
###### 4.1.3.RaftEngine Trait实现
TiKV支持两种方式来存储RaftLog，其中一个基于RocksDB的RocksEngine来实现，另一个由额外的raft-engine提供的RaftLogEngine来实现；

由启动TiKV时根据config.raft_engine.enable配置项来决定使用哪种实现；

RaftLogEngine实现RaftEngine Trait:
```
use raft_engine::{EntryExt, Error as RaftEngineError,
    LogBatch, RaftLogEngine as RawRaftEngine};

pub struct RaftLogEngine(RawRaftEngine<Entry, EntryExtTyped>);

impl RaftEngine for RaftLogEngine {
    type LogBatch = RaftLogBatch;

    fn log_batch(&self, _capacity: usize) -> Self::LogBatch {
        RaftLogBatch::default()
    }
    // 省略部分其他方法
}
```

RocksEngine实现RaftEngine Trait:
```
use crate::{RocksEngine, RocksWriteBatch};

use engine_traits::{Error, RaftEngine, RaftLogBatch, Result};

impl RaftEngine for RocksEngine {
    type LogBatch = RocksWriteBatch;

    fn log_batch(&self, capacity: usize) -> Self::LogBatch {
        RocksWriteBatch::with_capacity(self.as_inner().clone()
          , capacity)
    }

    fn sync(&self) -> Result<()> {
        self.sync_wal()
    }
    // 省略部分其他方法
}
```

---
##### 4.2.使用ThreadPool调度执行不同Task
自定义实现了一个ThreadPool/FuturePool来并行调度执行Task或Future，充分利用多核并发的能力，并与Rust语言提供的异步Future/Channel完美整合；
```
/// Scheduler provides interface to schedule task
/// to underlying workers.
pub struct Scheduler<T: Display + Send> {
    counter: Arc<AtomicUsize>,
    sender: UnboundedSender<Msg<T>>,
    pending_capacity: usize,
    metrics_pending_task_count: IntGauge,
}

enum Msg<T: Display + Send> {
    Task(T),
    Timeout,
}

pub trait Runnable: Send {
    type Task: Display + Send + 'static;
    /// Runs a task.
    fn run(&mut self, _: Self::Task) {
        unimplemented!()
    }
    fn on_tick(&mut self) {}
    fn shutdown(&mut self) {}
}

/// A worker that can schedule time consuming tasks.
#[derive(Clone)]
pub struct Worker {
    pool: Arc<Mutex<Option<ThreadPool<
      yatp::task::future::TaskCell>>>>,
    remote: Remote<yatp::task::future::TaskCell>,
    pending_capacity: usize,
    counter: Arc<AtomicUsize>,
    stop: Arc<AtomicBool>,
    thread_count: usize,
}

pub struct Builder<S: Into<String>> {
    name: S,
    thread_count: usize,
    pending_capacity: usize,
}

impl<S: Into<String>> Builder<S> {
    pub fn thread_count(mut self, thread_count: usize)->Self{
        self.thread_count = thread_count;
        self
    }
    pub fn create(self) -> Worker {
        let pool = YatpPoolBuilder::new(
           DefaultTicker::default())
           .name_prefix(self.name)
           .thread_count(self.thread_count, self.thread_count)
           .build_single_level_pool();
        let remote = pool.remote().clone();
        let pool = Arc::new(Mutex::new(Some(pool)));
        Worker {
            remote,
            stop: Arc::new(AtomicBool::new(false)),
            pool,
            counter: Arc::new(AtomicUsize::new(0)),
            pending_capacity: self.pending_capacity,
            thread_count: self.thread_count,
        }
    }
}

impl Worker {
    pub fn new<S: Into<String>>(name: S) -> Worker {
        Builder::new(name).create()
    }
    pub fn start<R: Runnable + 'static, S: Into<String>>(
        &self,
        name: S,
        runner: R,
    ) -> Scheduler<R::Task> {
        let (tx, rx) = unbounded();
        self.start_impl(runner, rx
            , metrics_pending_task_count.clone());
        Scheduler::new(
            tx,
            self.counter.clone(),
            self.pending_capacity,
            metrics_pending_task_count,
        )
    }
    fn start_impl<R: Runnable + 'static>(
        &self,
        runner: R,
        mut receiver: UnboundedReceiver<Msg<R::Task>>,
        metrics_pending_task_count: IntGauge,
    ) {
        let counter = self.counter.clone();
        self.remote.spawn(async move {
            let mut handle = RunnableWrapper {inner: runner};
            while let Some(msg) = receiver.next().await {
                match msg {
                    Msg::Task(task) => {
                      handle.inner.run(task);
                      counter.fetch_sub(1, Ordering::SeqCst);
                      metrics_pending_task_count.dec();
                    }
                    Msg::Timeout => (),
                }
            }
        });
    }
    // 省略部分其他方法
}

impl<T: Display + Send> Scheduler<T> {
    /// Schedules a task to run.
    ///
    /// If the worker is stopped or number pending tasks
    /// exceeds capacity, an error will return.
    pub fn schedule(&self, task: T) -> Result<()
        , ScheduleError<T>> {
        debug!("scheduling task {}", task);
        if self.counter.load(Ordering::Acquire) >= 
            self.pending_capacity {
            return Err(ScheduleError::Full(task));
        }
        self.counter.fetch_add(1, Ordering::SeqCst);
        self.metrics_pending_task_count.inc();
        if let Err(e) = self.sender.unbounded_send(
            Msg::Task(task)) {
            if let Msg::Task(t) = e.into_inner() {
                self.counter.fetch_sub(1, Ordering::SeqCst);
                self.metrics_pending_task_count.dec();
                return Err(ScheduleError::Stopped(t));
            }
        }
        Ok(())
    }
    // 省略部分其他方法
}
```
基于上面Worker、Scheduler自定义不同的CleanupRunner、CleanupTask、CompactRunner、CompactTask、PdRunner、PdTask、RegionRunner、RegionTask、SplitCheckRunner、SplitCheckTask等则可实现不同后台任务的并发执行；

---
#### 五、总结
上面简要介绍了TiKV弹性设计涉及到的主要内容，并从其实现代码中初步理解如何使用Trait抽象和实现来达到弹性分层抽象，如何使用ThreadPool、Channel、Worker、Task、Future来实现子任务弹性并发执行；

由于整个TiKV实现涉及到Raft算法、MVCC、BatchSystem、Region处理等，其具体实现比较复杂，有兴趣可自行阅读相关代码或文档，期望该文能对理解TiKV及其实现有所帮助。

---
参考
* [<font color="blue">Deep Dive TiKV</font>](https://tikv.org/deep-dive/key-value-engine/introduction/)


---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")


## Subspace 代码简析
基于 [subspace/subspace@7d5945](https://github.com/subspace/subspace/tree/7d594532e2ab8ac8edaa0cace7a75d004d93bd24) 的代码对 [subspace 项目](https://subspace.network/) 进行一个简单的分析。

### bin

#### node
node 是进行同步、交互等的节点。我们以 bin 文件 [crates/subspace-node/src/bin/subspace-node.rs](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-node/src/bin/subspace-node.rs) 中的 [节点构造、启动](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-node/src/bin/subspace-node.rs#L432-L727) 代码块为基础展开。

使用了 [substrate](https://substrate.io/) 作为链开发框架，提供存储、网络等底层功能。

主节点 FullNode 的主要组件：
- 网络
  - NetworkService (substrate)
  - SyncingService (substrate)

- 交互
  - SubspaceNotificationStream\<NewSlotNotification>
  - SubspaceNotificationStream\<RewardSigningNotification>
  - SubspaceNotificationStream\<BlockImportingNotification\<Block>>
  - SubspaceNotificationStream\<ArchivedSegmentNotification>

- 共识
  - `sc_proof_of_time::start_slot_worker`

运行在 FullNode 运行时中的主要任务 (task)：
- subspace-networking
  - node-runner

- sync-target-follower

- subspace-archiver

- sync-from-dsn
  - observer
  - worker

- node_metrics

- offchain-worker
  - offchain-workers-runner

- pot
  - pot-source
  - pot-gossip

- block-authoring
  - subspace-proposer

- node-metrics-server
  - node-metrics-server


#### farmer
farmer 功能性的描述可以参考：[subspace-farmer#architecture](https://github.com/subspace/subspace/tree/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-farmer#architecture)。

其基本逻辑为：
- [plotting](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-farmer/src/single_disk_farm/plotting.rs) 子模块接受链的历史数据并进行封装
- [farming](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-farmer/src/single_disk_farm/farming.rs) 子模块接收链挑战，并尝试在本地已封装的数据块中找出满足挑战条件的数据块，即出块

##### farming
[farming](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-farmer/src/single_disk_farm/farming.rs):

在 farming 循环中，持续接受
```
/// Information about new slot that just arrived
#[derive(Debug, Copy, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct SlotInfo {
    /// Slot number
    pub slot_number: SlotNumber,
    /// Global slot challenge
    pub global_challenge: Blake3Hash,
    /// Acceptable solution range for block authoring
    pub solution_range: SolutionRange,
    /// Acceptable solution range for voting
    pub voting_solution_range: SolutionRange,
}
```

并响应如下：


##### plotting
[plotting](https://github.com/subspace/subspace/blob/7d594532e2ab8ac8edaa0cace7a75d004d93bd24/crates/subspace-farmer/src/single_disk_farm/plotting.rs):


##### PlotAudit::audit



#### node-farmer 关系
farmer 依赖于 node，且设置自己的 `reward address`，基本可以视为两者采用了一种分布式的协作方式。

### networking
网络部分还是基于 [libp2p](https://libp2p.io/) 的 rust 语言实现 [rust-libp2p](https://github.com/libp2p/rust-libp2p)，这意味着具备用 libp2p 的其他语言实现替代、辅助节点。

### archiving

### consensus

#### proof of space

#### proof of time

### 其他
- pos table: proof of space table
- timekeeper
- domain node
- kzg

## subspace 拓展

### farming

#### auditing
- audit_plot_sync 并行化，截止

### plotting

### pooling

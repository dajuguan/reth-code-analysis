# Reth 当前代码中的 sparse-trie 更新、缓存、partial proof 与 Storage V2

> 分析基线：本机 `okx-reth`，commit `81573d2c7d806d5df9ff2d512dd000c2bf542b1b`（2026-08-01）。
>
> 这是 OKX fork 当前 HEAD 的实现分析，其中包含 `xl/reth-v2.3.0` 以及该分支后续改动；它比《Releasing Reth 2.0》文章里的概念描述更具体，也有一些演进，例如 execution 期间流式生成 hashed state、arena sparse trie、Proof V2 和 RocksDB history index。
>
> 本文重点只跟踪 account trie。storage trie 是嵌套在 account leaf 下的同一种机制；必要处会指出二者耦合，但不展开 storage 的编码与并行调度。

背景文章：<https://www.paradigm.xyz/writing/releasing-reth-2-0>

## 1. 先给结论

当前实现不是“把完整 MPT 装进内存，然后每个 block 重算一次”，而是下面这套流水线：

```text
EVM execution
  │  每批 EvmState / 预先 hash 的 HashedPostState
  ▼
trie-hashing task
  │  keccak(address)，合并本 block 的最终改动
  ▼
SparseTrieCacheTask ── 尝试修改 sparse trie
  │                         │
  │ 命中已 reveal 路径       │ 遇到 blinded node
  │ 直接更新                 ▼
  │                   ProofV2Target(key, min_len)
  │                         │
  │                   ProofWorker 从持久化 trie + hashed leaves
  │                         │
  └──────── reveal proof ◄──┘
             │
             └─ 对尚未完成的更新重试，直到可计算 root

root + TrieUpdates + HashedPostState
  │
  ├─ 与 block header.state_root 比较
  ├─ 放入 ExecutedBlock（排序工作延后到后台）
  ├─ 批量持久化 HashedAccounts 与 AccountsTrie
  └─ 将 sparse trie 锚定到新 root，commit 后按热度 prune，供下一块复用
```

几个最重要的区分：

1. `HashedAccounts` 是**当前 account leaf 数据**：`keccak(address) -> Account`。
2. `AccountsTrie` 不是另一份 account state，而是**持久化的中间 trie 元数据/缓存**：`nibble path -> BranchNodeCompact`。它让 proof/root 计算可以跳过未变化子树。
3. 内存 sparse trie 只 reveal 当前更新需要的局部路径；其余子树保持 blinded RLP reference/hash。
4. sparse-trie cache 的 pruning 只是把冷的已展开节点重新折叠为 blinded node，不是删除 canonical current state。
5. historical account 不靠旧版 `PlainAccountState`：plain address 仍用于 history index 和 changeset；最新值从 `HashedAccounts` 读取，旧值从“下一次修改”的 account changeset 读取。

因此，文章中的 “only store hashed state on MDBX, dropping the plain state tables” 并不表示地址从此完全消失。准确说法是：**不再同时保存一份以 plain address 为 key 的最新状态表；历史辅助数据仍保留 plain address，因为 RPC/EVM 查询天然从 address 出发。**

### 1.1 一个更形象的模型：面向 Merkle 状态计算的微内核

更完整的对应关系是：

```text
                       ┌─────────────────────────┐
EVM state updates ────▶│ SparseTrieCacheTask::run│
prefetch requests ────▶│    微内核调度器          │
proof completions ────▶│                         │
                       └───────────┬─────────────┘
                                   │
               ┌───────────────────┼───────────────────┐
               ▼                   ▼                   ▼
        apply updates       dispatch faults      calculate subtries
               │                   │                   │
               ▼                   ▼                   │
         Sparse Trie         Proof Workers             │
          虚拟内存              I/O Workers             │
               │                   │                   │
               │ blind fault       │ partial proof     │
               └───────────────────┴───────────────────┘
                                   │
                              reveal + retry
                                   │
                                   ▼
                         root_with_updates()
                                   │
                                   ▼
                    state_root == header.state_root
                         确定性 commit barrier
```

它们可以这样对应：

| OS / runtime 概念 | Reth 中的机制 |
|---|---|
| dependency-aware scheduler | `SparseTrieCacheTask::run` 事件循环 |
| virtual memory | 只展开当前所需路径的 sparse trie |
| page fault | leaf update 遇到 blinded node |
| demand paging | 根据 `(key, min_len)` 获取 partial proof |
| I/O workers | account/storage proof worker pools |
| page cache | 跨 block 复用的 `PreservedSparseTrie` |
| address-space version | cache 的 `parent_state_root` anchor |
| cache replacement | 按热度保留路径、将冷路径重新折叠的 pruning |
| dirty pages / write-back set | `TrieUpdates` 中 updated/removed intermediate nodes |
| speculative prefetch | prewarm 发送 `LeafUpdate::Touched`，只 reveal、不修改 leaf |
| work quantum / load balancing | `chunk_size` 和 idle-worker-aware proof 分片 |
| deterministic commit barrier | computed root 必须等于 header `state_root` |

这个调度器调度的不是传统 CPU 时间片，而是三类就绪事件：

1. 新的 state update 或 prefetch request 到达；
2. proof I/O 完成，可以 reveal 并重试被阻塞的更新；
3. 当前没有更紧急工作，可以预计算 subtrie hashes。

account update 会沿依赖关系逐步推进：

```text
Received / Prefetched
  -> Hashed（预先 hashed 的消息可跳过）
  -> PendingApply
  -> BlockedOnProof
  -> ProofInFlight
  -> Revealed
  -> Applied
  -> RootComputed
  -> Validated
  -> Committed
```

完整实现里 account update 还依赖 storage root，而 storage root 又依赖 storage-path proofs，所以它本质上是 dependency-aware scheduling，而不只是普通消息循环。

高度概括就是：

> `SparseTrieCacheTask::run` 是 dependency-aware 微内核调度器；它协调作为虚拟内存的 sparse trie、作为按需分页机制的 partial proof、作为跨 block page cache 的 preserved trie，以及作为 I/O workers 的 proof worker pool，最后以 state-root 校验作为确定性的 commit barrier。

## 2. 关键对象和存储布局

### 2.1 三种状态不要混为一谈

| 层 | 当前实现中的类型/表 | key | value | 用途 |
|---|---|---|---|---|
| 执行态 | `EvmState` / `BundleState` | `Address` | account 变化 | EVM 易于按原始地址访问 |
| 当前 hashed state | `HashedPostState`、MDBX `HashedAccounts` | `keccak256(address): B256` | `Option<Account>` / `Account` | MPT leaf 顺序、proof 和 root 计算 |
| 持久化 trie 元数据 | MDBX `AccountsTrie` | nibble path | `BranchNodeCompact` | 缓存子树结构和 hash，避免扫描所有 leaves |
| 内存工作 trie | `SparseStateTrie<ConfigurableSparseTrie>` | hashed-key nibble path | revealed node 或 blinded reference | 对本 block 局部更新并求根 |
| 历史定位 | RocksDB `AccountsHistory` | `ShardedKey<Address>` | 有序 block-number list | 找目标高度之后第一次修改 |
| 历史值 | static-file account changesets | block + `Address` | 修改前 `Option<Account>` | 还原历史值和 unwind |

表声明可从 `crates/storage/db-api/src/tables/mod.rs` 中的 `PlainAccountState`、`HashedAccounts`、`AccountsTrie` 查看。

### 2.2 为什么 `HashedAccounts` 仍然容易定位

给定普通地址 `address`，读取最新账户只需：

```text
hashed_address = keccak256(address)
MDBX.get(HashedAccounts, hashed_address)
```

`DatabaseProvider::basic_account` 在启用 hashed state 时就是这样做的：
`crates/storage/provider/src/providers/database/provider.rs:1456`。

这里没有“哈希表无法寻址”的问题。`HashedAccounts` 的名字表示 key 已被 Keccak，并不是指一个只能遍历的无序容器。MDBX 仍是有序 B-tree，32-byte key 可以 exact seek；同时它的排序正好就是 Ethereum account trie 的 leaf 顺序。

旧 `PlainAccountState[address]` 的优势只是省一次 Keccak。Storage V2 选择承担这次很小的 CPU 成本，换掉一整份重复的最新状态和写放大。

## 3. 一块交易执行时的端到端流程

### 3.1 先把 `run` 还原为六元组

不要先逐行阅读 `select_biased!`。`SparseTrieCacheTask::run` 可以先还原成：

```text
Scheduler = (State, Events, Transitions, Readiness, Policy, Commit)
```

| 维度 | 当前代码中的含义 |
|---|---|
| `State` | sparse trie、new/pending account/storage updates、pending/fetched proof targets、finish 标记、最终 hashed state |
| `Events` | execution/prewarm 消息与 proof-worker completion |
| `Transitions` | hash、尝试 apply、因 blind 阻塞、dispatch proof、reveal、retry、promote account |
| `Readiness` | 路径已 reveal；storage root 已完成；proof target 尚未请求；所有输入和依赖已 drain |
| `Policy` | update channel 优先、proof result coalescing、target chunking、idle 时预计算 subtrie hash |
| `Commit` | `root_with_updates` 成功，随后 validator 检查 computed root 等于 header root |

#### 唯一状态 owner

`SparseTrieCacheTask` 是 working sparse trie 和所有调度集合的唯一 mutable owner。execution、hashing task 和 proof workers 不直接修改 trie，只通过 channel 发送 command/completion：

```text
producers/workers                 single owner

StateUpdate ───────────────┐
PrefetchProofs ────────────┼──> SparseTrieCacheTask
ProofResult ───────────────┘          │
                                     └── 唯一可以 reveal/apply/root
```

这种“一个 owner 串行合并状态，多个 worker 并行做重活”的结构减少了锁和并发写冲突。并行发生在 proof/hash 计算层；canonical working state 的推进仍是确定性的。

#### 事件字母表

主循环实际接收两路事件：

```text
updates channel:
    PrefetchProofs
    StateUpdate / HashedStateUpdate
    FinishedStateUpdates

proof-result channel:
    ProofResult(Result<DecodedMultiProofV2, ...>)
```

其中 `FinishedStateUpdates` 只表示“以后不会再来新的 state update”，并不表示可以立即求根。已经 pending 的 update、已派发的 proof 和 reveal 后的 retry 仍必须 drain。

#### 等待集合和转换

代码中的多个 map/set 可以按调度状态理解，而不是按数据结构逐个记忆：

```text
new_*_updates
    │ process_new_updates
    ▼
*_updates ── blind fault ──> pending_targets
    │                            │ dispatch
    │ apply success              ▼
    ▼                       proof in flight
  removed                        │ completion
                                 ▼
                          reveal + retry remaining updates
```

`pending_account_updates` 还有一层业务依赖：真实 account leaf 必须等 storage root 就绪后才能编码，所以由 `promote_pending_account_updates` 将它提升为可 apply 的 account leaf update。account-only 模型可以固定 empty storage root，从而去掉这层依赖。

#### Readiness 与调度策略

主循环区分“能否执行”和“先执行谁”：

- **Readiness/correctness**：blind path 未 reveal 时绝不能假装 leaf 不存在；account leaf 未拿到 storage root 时不能编码；proof 必须对应 parent-state provider；
- **Policy/performance**：`select_biased!` 优先接收 update；一次取到 proof completion 后尽量 coalesce 其他已完成 proof；targets 足够多时按 `chunk_size` 派发；完全空闲时计算 upper-subtrie hashes。

`chunk_size` 同时影响派发时机和 proof task 粒度，但不影响最终 root。当前默认值是 5；只有 targets 很多或有多个 idle workers 时才真正切 chunk，超过 300 targets 则强制切分，避免一个 worker 被超大 proof 长期占用。实现位于 `payload_processor/multiproof.rs`。

#### Drain 与 commit barrier

事件循环可以退出的核心条件是：

```text
finished_state_updates
&& account_updates.is_empty()
&& all storage_updates are empty
```

结合循环中对 pending targets/proof results 的派发和消费，这表达的是“输入已关闭且所有影响 root 的工作已完成”，而不是 channel 暂时为空。退出后才调用 `root_with_updates()`。

随后还有两层 barrier：

1. sparse task 产生 `state_root + TrieUpdates`；
2. validator 验证 `computed_state_root == header.state_root`，失败则清 cache/走 fallback，绝不 publish 为 canonical result。

因此读这段代码时，可以把每次循环理解为：

```text
observe event
  -> mutate scheduler state
  -> promote newly-ready work
  -> dispatch within resource limits
  -> consume completions
  -> test drain condition
  -> root/validate/commit
```

### 3.2 创建 state-root pipeline

payload processor 在执行区块前创建：

- execution/update channel；
- `ProofTaskCtx` 和 `ProofWorkerHandle`；
- sparse-trie task；
- state-root、hashed-state 结果 channel。

入口在：

- `crates/engine/tree/src/tree/payload_processor/mod.rs:432`
- `crates/engine/tree/src/tree/payload_processor/mod.rs:518`
- `crates/engine/tree/src/tree/payload_processor/mod.rs:748`

初始 sparse trie 通常是 blind root。当前默认实现为：

```text
ConfigurableSparseTrie::Arena(ArenaParallelSparseTrie::default())
```

不是 HashMap 版本。`ConfigurableSparseTrie` 的默认选择见
`crates/trie/sparse/src/traits.rs:374`。

如果上一块留下的 `PreservedSparseTrie` 恰好锚定在本块的 `parent_state_root`，则直接复用；否则调用 `clear`，保留已经分配的 arena 容量，但不复用不匹配 root 的节点内容。该约束在
`crates/engine/tree/src/tree/payload_processor/preserved_sparse_trie.rs`。

这条 root anchor 是 cache 正确性的核心：缓存不是“看到同一个 address 就能用”，而是“只有父状态根完全一致，里面已 reveal 的路径才仍代表当前 canonical state”。

### 3.3 execution 流式发送 state update

`StateRootMessage` 当前支持：

- `PrefetchProofs`：只预取路径，不表示 canonical state 改动；
- `StateUpdate(EvmState)`：执行态的 plain-address 更新；
- `HashedStateUpdate(HashedPostState)`：调用方已完成 hash；
- `BlockAccessList`；
- `FinishedStateUpdates`。

定义在 `crates/trie/parallel/src/state_root_task.rs:14`。

普通执行路径通过 state hook 持续发送 `StateUpdate`，而不是等整块结束再一次性转换。`StateHookSender` drop 时发送 `FinishedStateUpdates`，见
`crates/trie/parallel/src/state_root_task.rs:95` 和 `:169`。

独立的 `trie-hashing` task 接收这些消息，把 touched account 的地址做 Keccak，形成 `HashedPostState`，再交给 sparse task。转换规则在
`crates/trie/parallel/src/state_root_task.rs:176`：

- account 更新：`accounts[keccak(address)] = Some(account)`；
- selfdestruct：写入 `None`；
- storage slot 同理 hash（本文略）。

并行 BAL execution 路径也可以直接发送已经 hash 的更新，避免重复转换。相关代码在
`crates/engine/tree/src/tree/payload_processor/prewarm.rs:635`。

`SparseTrieCacheTask` 同时把所有更新合并到 `final_hashed_state`，最终把本块最后生效的 hashed delta 发回 validator。因此当前 HEAD 已不局限于文章时期“执行完再对整个 post-state 做一次 `hash_state_slow`”的模型。validator 仍保留 fallback：如果该结果 channel 不可用，就根据 execution outcome 重新计算 hashed post-state。

### 3.4 speculative prewarm 只揭示路径

prewarm task 可以在交易正式执行前推测其访问集，生成 multiproof targets，并发送 `PrefetchProofs`：
`crates/engine/tree/src/tree/payload_processor/prewarm.rs:150`。

这些目标进入 sparse task 后被标记为 `LeafUpdate::Touched`：

- 它会沿路径访问并触发缺失 proof；
- 但不会改变 leaf value；
- 推测错了最多浪费一次预取，不会污染 canonical state。

这使 DB proof I/O 能与 EVM execution 重叠。真正状态更新到来时，如果路径已经 reveal，就直接命中 cache。

### 3.5 第一次尝试更新 sparse trie

`SparseTrieCacheTask::run` 是主事件循环，见
`crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:261`。它使用 biased select 优先消化状态更新，同时接收 proof 结果。

对于 account-only 模型，逻辑可简化为：

```text
on HashedStateUpdate(account_key, new_account):
    account_updates[key] = Touched
    pending_account_updates[key] = new_account

when account leaf value can be finalized:
    pending_account_updates[key]
      -> Changed(rlp(TrieAccount))
      -> update_leaves(account_updates)
```

真实 Ethereum account leaf 包含 `storage_root`，所以完整代码必须先算完对应 storage trie root，再把 account 编码为 `TrieAccount`。该 promotion 在
`crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:740`。纯 account 模拟中可以把 `storage_root` 固定为 empty root，因而直接编码。

`SparseTrie::update_leaves` 的契约很关键：

- 成功应用的 update 会从输入 map 删除；
- 路径被 blinded node 阻挡的 update 保留；
- callback 返回缺失目标 `(full_hashed_key, min_len)`；
- reveal proof 后，用同一个剩余 map 重试。

接口说明在 `crates/trie/sparse/src/traits.rs:281`。

Arena 实现在 `crates/trie/sparse/src/arena/mod.rs:2790` 附近先按 nibble path 排序更新。当 seek 撞到 blinded node 时，它计算 `min_len = 当前逻辑 branch path 深度 + 1` 并请求 proof。删除操作还有一个特殊情况：删除 leaf 可能让 branch 退化，必须知道 blinded sibling 是 leaf、extension 还是 branch 才能正确压缩，此时也会补 sibling proof，而不是猜测结构。

因此 partial proof 不是只发生一次，而是一个有限的 reveal/retry 循环：

```text
pending updates
  -> update known paths
  -> collect missing (key, min_len)
  -> fetch partial proofs
  -> reveal nodes
  -> retry only remaining updates
  -> ...
  -> all updates applied
```

## 4. Partial Proof V2 到底“partial”在哪里

### 4.1 `key + min_len`

Proof V2 的目标为：

```text
ProofV2Target {
    key_nibbles: 64 nibbles of keccak(address),
    min_len: minimum returned node-path depth,
}
```

定义在 `crates/trie/common/src/target_v2.rs:7`。

普通完整 proof 会返回 root 到 leaf 路径上的所有节点；但 sparse trie 往往已持有上半段。假设已知到深度 17，在深度 18 遇到 blinded child，则只需要足以把该 child 展开的后缀，不必再次传回 root 到 17 的祖先链。

`min_len` 的精确定义是“保留的 proof node path 至少有多深”。proof 构造仍可能需要在 `min_len - 1` 处构造父 branch，才能形成要 reveal 的子树边界；它不是简单地让数据库游标从某个 byte offset 开始读。

Proof calculator 的筛选逻辑在：

- `crates/trie/trie/src/proof_v2/target.rs`
- `crates/trie/trie/src/proof_v2/mod.rs:197`

### 4.2 proof worker 从哪取数据

Proof worker 使用独立的只读 provider/transaction，并组合两类 cursor：

```text
AccountsTrie cursor       -> 已持久化的 BranchNodeCompact / 子树 hash
HashedAccounts cursor     -> account leaf values，按 hashed key 排序
```

入口在 `crates/trie/parallel/src/proof_task.rs:837`。`ProofCalculator` 可以利用 `AccountsTrie` 中已有的子树 hash 跳过无关子树，只在目标路径需要时消费 `HashedAccounts` leaves。目标会排序、分 chunk，并由 proof worker pool 并行处理；结果以 `DecodedMultiProofV2` 发回 sparse task，相关主路径在同文件 `:961` 和 `:1004`。

所以并不是“只保留 hashed state 后，每次 proof 都扫描整张 HashedAccounts”。真正避免性能退化的是：

1. `HashedAccounts` 已按 trie key 排序；
2. `AccountsTrie` 持久化中间节点和子树 hash；
3. proof 只针对本次缺失路径；
4. `min_len` 避免重复返回 cache 已知的前缀；
5. proof workers 使用独立读事务并行处理；
6. speculative prewarm 把 I/O 提前到 execution 期间。

### 4.3 reveal 后如何重试

proof result 到达后，`SparseStateTrie::reveal_decoded_multiproof_v2` 将其中节点嵌入 sparse trie，见
`crates/trie/sparse/src/state.rs:295`。

随后 `process_account_leaf_updates` 再次调用 `update_leaves`。已经完成的项此前已从 map 删除，只有被 blind path 阻挡的项参与下一轮。`SparseTrieCacheTask` 还记录每个 target 已请求过的最小 `min_len`；只有新请求比旧请求更深/覆盖更多缺失信息时才重新派发，从而压制重复 proof。

对应代码在：

- `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:520`
- `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:529`
- `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:632`
- `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:813`

一个需要明确的安全边界：当前 `reveal_decoded_multiproof_v2` 的注释说明它**不会做完整的独立 proof 验证**。这里的 proof 来自本节点针对同一个 parent-state provider/overlay 的内部 worker，而不是不可信网络对端。结构错误会导致计算失败或错误 root；最终共识检查仍是 computed root 必须等于 block header 的 `state_root`。不要把该内部接口直接当成网络 witness verifier。

## 5. Sparse trie cache 如何工作和 pruning

### 5.1 缓存的内容

Arena sparse trie 用 slotmap/arena 保存节点。高层和低层 subtrie 在固定深度附近拆分，低层 subtries 可并行更新与求 hash。未 reveal 的 child 保存为 blinded RLP reference；对大节点它等价于 hash reference，但 MPT 小节点可能 inline，所以概念上不应一律写成 `B256`。

实现概览在 `crates/trie/sparse/src/arena/mod.rs:562`。

缓存包含：

- 已 reveal 的 branch/extension/leaf；
- 仍 blinded 的子树 reference；
- 本块更新产生但尚待持久化的 intermediate-node updates；
- account/storage 的 LFU 热度记录。

### 5.2 block 成功后的顺序

state root 成功后，复用路径不是直接 prune，而是：

```text
1. commit_updates(TrieUpdates)
2. prune cold revealed nodes
3. shrink arena if needed
4. PreservedSparseTrie::Anchored { trie, state_root: new_root }
```

`commit_updates` 先把刚计算出的持久化中间节点变化应用回内存 trie，使 cache 与新 root 对齐；然后才允许把冷路径重新折叠。代码在：

- `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs:219`
- `crates/trie/sparse/src/state.rs:728`
- `crates/trie/sparse/src/state.rs:784`

LFU tracker 会衰减频率，只保留最热的 account/storage 路径。`prune(retained_leaves)` 把其他已展开节点重新编码/哈希为 blinded subtree。这样释放的是**内存中的 proof 展开结果**，而不是 MDBX 的 `HashedAccounts` 或 canonical trie 数据。

这也回答“hash table 不好 pruning”这一疑问：

- 当前状态表不能按 block 历史 prune，它本来就只保存 latest value；账户被删除时 exact-delete 即可；
- 可 prune 的历史是 changesets + history indices，它们有 block number；
- 可 prune 的 sparse cache 是内存局部 trie，它按 LFU/path 折叠；
- 这三者是完全不同的 pruning 问题。

### 5.3 cache 失效

以下情形不会继续信任旧 cache 内容：

- 下一块 parent root 与 cache anchor 不同；
- state-root task 失败或被取消；
- 发生竞态，任务结果没有成为获胜结果。

实现会 clear trie，必要时只保留 arena allocation 以减少重新分配。发送结果前还有 lock/ownership 协调，避免新任务在旧任务完成 cache 更新前抢先复用。见
`crates/engine/tree/src/tree/payload_processor/mod.rs:748`。

## 6. root、TrieUpdates 与持久化

### 6.1 求根并产生增量

所有 leaves 更新完成后，`SparseStateTrie::root_with_updates` 自底向上计算 dirty subtries，并同时取出 intermediate-node 变化：

```text
StateRootComputeOutcome {
    state_root,
    trie_updates: TrieUpdates
}
```

`SparseTrieUpdates` 的核心是：

- `updated_nodes: path -> BranchNodeCompact`；
- `removed_nodes: Set<path>`；
- `wiped` 标记。

定义见 `crates/trie/sparse/src/traits.rs:312`；state trie 汇总见
`crates/trie/sparse/src/state.rs:415`。

如果 account trie 整体仍是 blind、且本块没有状态改动，可以直接沿用 `parent_state_root`，无需为“空更新”展开 root。

### 6.2 validator 的 correctness gate 和 fallback

validator 等待 sparse task 结果，并把计算 root 与 header root 比较：
`crates/engine/tree/src/tree/payload_validator.rs:680` 附近。

如果 sparse task 报错、超时或给出错误 root，配置允许时会退回串行 state-root 计算。若 streaming hashed-state 的结果也不再可信，则从 execution outcome 重算 `HashedPostState`，并重新运行 hashed-state validation hook。最后无论走哪条算法，只有 root 等于 header 才接受区块。

### 6.3 为什么还有 deferred trie data

验证 root 时需要无序/工作态的 `HashedPostState` 与 `TrieUpdates`，但持久化希望两者已排序以提高批量游标写入效率。当前实现把排序从 validation critical path 移到后台：

```text
DeferredTrieData::pending(hashed_state, trie_updates)
    -> background compute_and_publish()
    -> ComputedTrieData {
         hashed_state: Arc<HashedPostStateSorted>,
         trie_updates: Arc<TrieUpdatesSorted>
       }
    -> ExecutedBlock
```

后台任务还基于 forward `TrieUpdates` 计算 trie changesets，并写入 `ChangesetCache`，供 reorg/unwind 使用。代码在：

- `crates/engine/tree/src/tree/payload_validator.rs:1810`
- `crates/chain-state/src/deferred_trie.rs`
- `crates/chain-state/src/in_memory.rs:748`

第一次消费 `ExecutedBlock::trie_data()` 时，如果后台尚未 publish，调用方等待；之后 clone 共享缓存结果。

### 6.4 MDBX 最终写什么

`save_blocks` 对一批 oldest-to-newest blocks 做合并，但 `merge_batch` 以 newest-to-oldest 输入，保证同一 key 的最终值胜出：
`crates/storage/provider/src/providers/database/provider.rs:719`。

持久化分两组：

1. `write_hashed_state`
   - `Some(account)`：upsert `HashedAccounts[hashed_address]`；
   - `None`：exact seek 后删除；
   - 见 `crates/storage/provider/src/providers/database/provider.rs:2693`。
2. `write_trie_updates_sorted`
   - 将 account intermediate nodes 写入 `AccountsTrie[path] = BranchNodeCompact`；
   - removed path 则删除；
   - root 的空 path 不持久化；
   - 见 `crates/storage/provider/src/providers/database/provider.rs:3084`。

启用 Storage V2 / `use_hashed_state()` 时，`write_state_changes` 明确跳过 `PlainAccountState` 和 `PlainStorageState`，但仍写 bytecode：
`crates/storage/provider/src/providers/database/provider.rs:2628`。

注意：一块 accepted 但尚未落 MDBX 时，后续块和查询还可能通过 in-memory overlay 看到它。overlay 合并 `ExecutedBlock` 中的 sorted hashed state 与 trie updates；这就是 deferred trie data 仍被挂到 executed block 上的原因之一。

## 7. Historical account 在 Storage V2 中如何查询

### 7.1 “下一次修改”的模型

`HistoricalStateProviderRef` 的内部语义是“目标 block 开始时的状态”，即不包含该 block 内变化：
`crates/storage/provider/src/providers/state/historical.rs:112`。

公开查询“block N 执行后的状态”时，provider 传入 `N + 1`：
`crates/storage/provider/src/providers/database/provider.rs:309`。

查询 `address` 在目标时刻的值时：

```text
AccountsHistory[address]
    -> 找目标时刻之后第一次修改 address 的 block M

if 找到 M:
    AccountChangeSet[M, address] 保存的是 M 修改前的值
    这个值就是目标历史值

if 后面再也没修改:
    目标值等于 latest state
    -> HashedAccounts[keccak(address)]

if 目标早于该账户首次出现且 history 未被 prune:
    -> None
```

`HistoryInfo` 把结果表达为：

- `NotYetWritten`；
- `InChangeset(block)`；
- `InPlainState`；
- `MaybeInPlainState`。

后两个名字是 legacy terminology。在 Storage V2 分支中，“plain state lookup” 实际会 hash 地址后读取 `HashedAccounts`，并不要求 `PlainAccountState` 存在。定义与 account 查询分别在：

- `crates/storage/provider/src/providers/state/historical.rs:56`
- `crates/storage/provider/src/providers/state/historical.rs:184`
- `crates/storage/provider/src/providers/state/historical.rs:349`

### 7.2 history index 为什么仍用 plain address

RocksDB `AccountsHistory` 使用 `ShardedKey<Address>`，value 是压缩的 block-number list。写入阶段从每块 execution outcome/reverts 收集发生变化的 plain addresses，然后 append shard：
`crates/storage/provider/src/providers/rocksdb/provider.rs:1396`。

这是合理的分工：

- trie/root 热路径按 `keccak(address)` 排序；
- 用户和 EVM 查询从 `Address` 开始；
- historical index 只做 address -> changed blocks，不参与 state root；
- changeset 必须保存 plain address，才能既支持历史查询又支持 unwind 后重新 hash。

因此 Reth 2.0 去掉的是 duplicated **latest plain state**，不是去掉所有 plain-keyed metadata。

RocksDB snapshot 查询还接收 MDBX visible tip，只使用不高于该 tip 的 history entries，避免 RocksDB 与 MDBX 分阶段提交/快照时读到“未来索引”。相关逻辑在
`crates/storage/provider/src/providers/rocksdb/provider.rs:1531`。

### 7.3 historical pruning 的真实对象

Account-history pruning 同步推进：

1. static-file 中的 account changesets；
2. RocksDB 中对应 address 的 history shards；
3. MDBX 中记录的 prune checkpoint/最低可查询高度。

Storage V2 路径会流式扫描指定 block range 的 static-file changesets，统计每个 address 已删除到的最高 block，按 address 排序后批量 prune RocksDB shards；全部完成后才删除低于边界的 static-file jars。实现见
`crates/prune/prune/src/segments/user/account_history.rs:230`。

RocksDB 的 shard pruning 删除 `<= to_block` 的 block numbers，并把最后一个剩余 shard 重新设为 `u64::MAX` sentinel，见
`crates/storage/provider/src/providers/rocksdb/provider.rs:2095`。

公开 historical provider 会读取 `AccountHistory` prune checkpoint，把最低可用高度设为 `checkpoint + 1`。请求更早且无法安全回答的历史状态时返回 `StateAtBlockPruned`，而不是用 latest state 猜测。设置边界的代码在
`crates/storage/provider/src/providers/database/provider.rs:309`。

这里可以看出为何 hashed-state latest table 本身“不好按历史 pruning”并不是问题：历史 retention 已由 block-indexed changesets/index 解决；latest table 永远只留一个版本。

## 8. Reorg / unwind 如何恢复

从 block `from` 开始 unwind 时，provider 执行：

```text
account changesets[from..]
  -> 取出每个 address 的修改前值
  -> keccak(address)
  -> 反向聚合，恢复 HashedAccounts

same changesets
  -> unwind AccountsHistory indices

trie ChangesetCache / 必要时从 DB 计算 revert
  -> 恢复 AccountsTrie intermediate nodes
```

入口在 `crates/storage/provider/src/providers/database/provider.rs:842`。

`unwind_account_hashing` 会把 changeset 的 plain address 重新 Keccak，按逆序合并后 upsert/delete `HashedAccounts`：
`crates/storage/provider/src/providers/database/provider.rs:3180`。

`AccountsTrie` 的 revert 则来自 forward trie updates 计算出的 trie changesets，并通过 `ChangesetCache::get_or_compute_range` 获得。也就是说 canonical unwind 不依赖内存 sparse cache；cache 只是加速器，root 不匹配时自然失效。

## 9. 对 account-only 最小模拟的直接映射

如果要实现前一份 account-only spec，当前代码可收缩为以下七个组件：

### 9.1 `InMemoryDb`

```text
hashed_accounts: BTreeMap<B256, Account>
account_trie:    BTreeMap<Nibbles, BranchNodeCompact>
history_index:   BTreeMap<Address, Vec<BlockNumber>>
changesets:      BTreeMap<BlockNumber, Vec<(Address, Option<Account>)>>
```

使用 `BTreeMap` 比 `HashMap` 更接近 MDBX cursor 的有序遍历语义。

### 9.2 `SparseTrie`

最少支持：

- blind root；
- revealed branch/extension/leaf；
- blinded RLP/hash reference；
- `update_leaves` 返回 `(key, min_len)`；
- reveal proof；
- retry pending updates；
- root + updated/removed intermediate nodes。

### 9.3 `ProofWorker`

输入 `Vec<ProofV2Target>`，从 `account_trie + hashed_accounts` 构造仅覆盖缺失后缀的 proof。为了展示机制，可以同步执行；保留 target chunk 接口即可，不必先实现线程池。

### 9.4 `SparseTrieCache`

保存：

```text
anchor_root
sparse_trie
touch_frequency
```

只有 `parent_root == anchor_root` 才复用。成功后先 commit updates，再将冷的 revealed subtree fold 回 blind reference，最后把 anchor 改为新 root。

### 9.5 `TrieScheduler`

用单线程事件循环模拟 `SparseTrieCacheTask::run`，独占 working sparse trie，并显式保存：

```text
generation / phase
new + pending updates
pending proof targets
inflight proof work
completed proof results
finished-updates flag
```

proof worker 同步执行也没关系，但结果必须包装成 completion event 返回 scheduler，不能直接修改 trie。这样可以测试 proof 乱序完成、stale generation、backpressure、drain 和 no-progress，而不必引入 Tokio。

### 9.6 `BlockProcessor`

作为 driver 把 block input 展开为 events，并驱动 scheduler/worker/commit：

```text
plain changes
 -> hashed post-state
 -> sparse update / partial proof / reveal / retry
 -> root verification
 -> persist HashedAccounts + AccountsTrie
 -> append changeset + history index
 -> preserve/prune sparse cache
```

### 9.7 `HistoricalProvider`

实现 `account_at(address, block_after_execution)`：内部转成 `start_of(block + 1)`，用 history index 找下一次改动；命中则读 changeset，否则读 `hashed_accounts[keccak(address)]`。再加 prune checkpoint 即可演示 archive/full/pruned 三种语义。

可以先省略：

- storage trie 和 account/storage 两阶段 promotion；
- BAL parallel execution；
- provider overlay；
- arena upper/lower subtrie 并行；
- RocksDB/MDBX/static-file 跨后端提交；
- deferred sorting 和 changeset cache；
- proof worker pool。

但不应省略：root anchor、blind/reveal/retry、`min_len`、`HashedAccounts` 与 `AccountsTrie` 的分离、plain-address history index + before-value changeset。这五项正是当前机制的骨架。

## 10. 推荐阅读顺序

若要直接顺着代码读，建议按下面顺序：

1. `crates/trie/parallel/src/state_root_task.rs`
2. `crates/engine/tree/src/tree/payload_processor/mod.rs`
3. `crates/engine/tree/src/tree/payload_processor/sparse_trie.rs`
4. `crates/trie/sparse/src/traits.rs`
5. `crates/trie/sparse/src/arena/mod.rs`
6. `crates/trie/sparse/src/state.rs`
7. `crates/trie/common/src/target_v2.rs`
8. `crates/trie/trie/src/proof_v2/mod.rs`
9. `crates/trie/parallel/src/proof_task.rs`
10. `crates/engine/tree/src/tree/payload_validator.rs`
11. `crates/chain-state/src/deferred_trie.rs`
12. `crates/storage/provider/src/providers/database/provider.rs`
13. `crates/storage/provider/src/providers/state/historical.rs`
14. `crates/storage/provider/src/providers/rocksdb/provider.rs`
15. `crates/prune/prune/src/segments/user/account_history.rs`

## 11. 一句话模型

可以把当前 Reth 的设计记成：

> `HashedAccounts` 保存 canonical latest leaves，`AccountsTrie` 保存可跳过子树的持久化结构，内存 sparse trie 只展开热路径；遇到 blind path 就按 `min_len` 补局部 proof 并重试。历史查询则完全绕开旧版 latest plain table，用 plain-address history index 找 changeset，只有“之后再未修改”时才回到 hashed latest state。

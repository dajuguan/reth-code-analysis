# Reth 交易池 ↔ 网络层:交易传播与背压机制分析

本文分析 reth 交易在 P2P 网络层的**入站**与**出站**完整链路、`TransactionsManager` 的调度模型,以及高压场景下"静默丢弃转发通知"的成因。

代码引用均为仓库根相对路径 `文件:行号`,行号以分析时的代码版本为准,阅读时以就近的函数/符号名为准。

---

## 0. 全局结构:一个 `TransactionsManager`,一条 channel 进,N 个 peer 出

- 全节点只有**一个** `TransactionsManager` 实例(不是每 peer 一个)。
- 它在构造时向 pool 注册**一个** pending-tx 监听器,拿到 `rx` 存为 `self.pending_transactions`(crates/net/network/src/transactions/mod.rs:391、411)。
- peer 连接**不会**新建监听器;per-peer 的扇出发生在收到 `rx` 数据之后,在 `TransactionsManager` 内部对 `self.peers: HashMap<PeerId, PeerMetadata>` 遍历完成。

监听器通道的**方向**要点(crates/transaction-pool/src/pool/mod.rs:274):
- `mpsc::channel(...)` 造出 `(sender, rx)`;
- `sender` 留在 pool 内部的 `Vec`(pool 是**生产者**),`rx` 返回给调用方(**消费者**,如 `TransactionsManager`、RPC 订阅等);
- pool 端 `Vec` 里的元素数 = 调用过 `pending_transactions_listener()` 的**子系统**数,而非 peer 数。

监听器缓冲大小由配置 `pending_tx_listener_buffer_size` 决定(默认 2048,常量见 crates/transaction-pool/src/pool/mod.rs:133),CLI 通过 `--txpool.max-pending-txns` 设置(crates/node/core/src/args/txpool.rs:382)。**注意该名字有误导性:它不是"池内 pending 交易上限",而是"通知通道的缓冲深度"。**

---

## 1. 出站(outbound)链路及限流机制

### 1.1 链路

```
pool 新 pending 交易
  → 遍历 pending_transaction_listener 里的每个 sender,try_send(hash)
  → TransactionsManager.pending_transactions (rx)  [唯一]
  → poll_recv_many 批量取                          (crates/net/network/src/transactions/mod.rs:1583)
  → on_new_pending_transactions(hashes)            (mod.rs:878)
  → propagate_all(hashes)                          (mod.rs:1125)
  → propagate_transactions:遍历 self.peers 扇出    (mod.rs:1033)
```

### 1.2 生产端与限流:`try_send` + 满则静默丢弃

pool 侧发送用 `try_send`(crates/transaction-pool/src/pool/listener.rs:285 `send_all`):
- 成功:正常入队;
- `TrySendError::Full`:**静默丢弃该哈希通知**,只打一条 `debug!("failed to send pending tx; channel full")`,并 `return true`(视为存活,连"清理死 listener"都不触发);
- `TrySendError::Closed`:`return false` → 上层 `retain(|l| !l.sender.is_closed())` 清理。

三处会往通道灌数据:
- `on_new_pending_transaction`(mod.rs:771):新交易进入 pending 态;
- `notify_on_new_state`(mod.rs:859):区块 canonical 更新后 promote 的交易;
- `notify_on_transaction_updates`(mod.rs:917):交易状态更新。

**限流本质:这是"丢弃型"背压,绝不阻塞 pool 主流程。** buffer 越小,高压时越易丢通知;buffer 越大,内存越高但丢得越少。它限制的是"通知积压上限",不是池容量。

### 1.3 per-peer 决策:发全量还是发 hash

`propagate_transactions`(mod.rs:1033):
- 一部分 peer 发完整交易体(`send_transactions`),数量由 `propagation_mode.full_peer_count(peers.len())` 决定;其余 peer 只发 hash 公告(`send_transactions_hashes`)(mod.rs:1053)。
- 每个 peer 用 `seen_transactions` 去重(mod.rs:1068),发出后把 hash 记入该 peer 的 `seen_transactions`。
- 单条 `NewPooledTransactionHashes` 消息按软上限 4096 truncate(mod.rs:1086)。

---

## 2. 入站(inbound)机制

### 2.1 两个数据源汇聚到 `import_transactions`

```
peer session → NetworkManager → (unbounded mpsc) → TransactionsManager.transaction_events
  → poll (mod.rs:1628) → on_network_tx_event (mod.rs:1308)
      ├─ IncomingTransactions(全量广播)
      │     → import_transactions(source=Broadcast)          (mod.rs:1327)
      └─ IncomingPooledTransactionHashes(只有 hash 的公告)
            → on_new_pooled_transaction_hashes
            → TransactionFetcher 发 GetPooledTransactions 拉全量
            → 拉回后 on_fetch_event
            → import_transactions(source=Response)           (mod.rs:1518)
```

入站网络事件通道 `transaction_events`(`from_network`)是 **unbounded** 的,所以背压不在这条 channel 上。

### 2.2 `import_transactions` 的过滤漏斗与入站限流(mod.rs:1351)

1. 同步期短路:`is_initially_syncing()` / `tx_gossip_disabled()` 直接返回(mod.rs:1358);
2. **入站背压闸门**:`has_capacity_for_pending_pool_imports()`,in-flight import 达上限 `DEFAULT_MAX_COUNT_PENDING_POOL_IMPORTS` 则 return(mod.rs:1366)——**这是入站方向真正的限流闸门**;
3. 按剩余容量 `remaining_pool_import_capacity()` truncate,防超大 payload(mod.rs:1375);
4. 廉价内存过滤:去掉 `transactions_by_peers` 已追踪、`bad_imports` 已知坏的(mod.rs:1410);
5. pool 去重:`self.pool.retain_unknown()`(mod.rs:1430,抢 pool 锁);
6. **并行 ecrecover**:`into_par_iter().try_into_recovered()` 恢复签名者(mod.rs:1441,rayon 并行但 `.collect()` 阻塞等待);
7. 异步批量入池:构造 future 调 `pool.add_external_transactions(new_txs)`,push 进 `self.pool_imports`(`FuturesUnordered`),并给 `pending_pool_imports` 计数 +N(mod.rs:1465-1491)。

### 2.3 落点:`add_external_transactions` → `add_transactions`

- `add_external_transactions` = `add_transactions(TransactionOrigin::External, ...)`(crates/transaction-pool/src/traits.rs:139)。网络来的交易一律 `External`(区别于 RPC 的 `Local`/`Private`)。
- `lib.rs` 的 `add_transactions`(crates/transaction-pool/src/lib.rs:502):先 `validate_transactions(...).await`(验证在校验任务/`additional_validation_tasks` 上跑,此处 `.await` 让出),再 `pool.add_transactions(origin, validated)`(crates/transaction-pool/src/pool/mod.rs:655,同步插入,内部触发 `on_new_pending_transaction`)。

### 2.4 结果回收 + in-flight 计数释放

- `pool_imports` 在 poll 循环被驱动(mod.rs:1670)。future 完成时先把 `pending_pool_imports` 计数 -N(mod.rs:1486)——**这就是 2.2 步 2 那个闸门的释放**。
- 结果交给 `on_batch_import_result`(mod.rs:590):成功 → `on_good_import`;失败 → `on_bad_import`(记入 `bad_imports` LRU,可能扣 peer reputation)。

### 2.5 入站/出站对称关系

| | 出站(propagate) | 入站(import) |
|---|---|---|
| 触发 | pool 新 pending 交易 | peer 发来 `Transactions`/公告 |
| 通道 | pool→TxManager mpsc,`pending_tx_listener_buffer_size` | NetworkManager→TxManager **unbounded** mpsc |
| 背压机制 | channel 满 `try_send` 丢弃 | `has_capacity_for_pending_pool_imports` 闸门 + truncate |
| 落点 | 扇出到 `self.peers` | `add_external_transactions` → `add_transactions` → pool |

---

## 3. `poll_recv_many` 的 tokio 批量窗口调度节奏

出站合批**不是定时器、也不是固定阈值**,而是**按事件循环 tick 机会性合批**。

### 3.1 一次 poll 尽量排干(mod.rs:1583)

```rust
let mut new_txs = Vec::new();
let maybe_more_pending_txns = match this.pending_transactions.poll_recv_many(
    cx, &mut new_txs,
    SOFT_LIMIT_COUNT_HASHES_IN_NEW_POOLED_TRANSACTIONS_BROADCAST_MESSAGE,  // = 4096
) { ... };
if !new_txs.is_empty() { this.on_new_pending_transactions(new_txs); }
```

- 低流量:通道里往往仅 1 条,取到即返回 → 近乎实时逐条;
- 高流量:两次 poll 之间积压 N 条,一次排空合成一批 → 每 peer 一条消息;流量越高合批越强。

### 3.2 `poll_recv_many` 的三态语义(tokio `mpsc::Receiver`)

| 返回 | 含义 |
|---|---|
| `Poll::Ready(n)`,`n>=1` | 取到 n 条,已 append 到 buffer |
| `Poll::Ready(0)` | channel **已关闭且空**(所有 sender drop),不是"暂时没有" |
| `Poll::Pending` | 开着但当前空,**已注册 waker**,来数据时唤醒本 task |

关键约定:**返回 `Ready` 不注册 waker,只有 `Pending` 才注册。**

### 3.3 为何写成"两次调用"(mod.rs:1589-1601)

```rust
Poll::Ready(count) => {
    if count == 4096 {
        true            // 填满 → 可能还有,下一轮再排
    } else {
        // count < 4096:很可能已空。但第一次是 Ready,没注册 waker!
        let limit = 4096 - new_txs.len();
        this.pending_transactions.poll_recv_many(cx, &mut new_txs, limit).is_ready()
        // 第二次大概率 Pending,借此把 waker 装上;顺便捞走零头
    }
}
Poll::Pending => false,  // 一开始就空 → 已注册 waker
```

第二次调用的核心目的是**注册 waker**(第一次 `Ready` 没注册),否则新交易到了没人唤醒 task,传播会卡住。

### 3.4 4096 的含义

`SOFT_LIMIT_COUNT_HASHES_IN_NEW_POOLED_TRANSACTIONS_BROADCAST_MESSAGE = 4096`(crates/net/network/src/transactions/constants.rs:9):
- eth 协议 `NewPooledTransactionHashes` 单条消息 hash 软上限;
- 同时作为 `poll_recv_many` 单次收取上限;
- 积压超 4096 时返回值 == 4096,`maybe_more = true`,下一轮 poll 立即再排(拆多批发,不丢不等)。

---

## 4. 高压下静默丢弃转发通知的机制

**现象:一个节点一次性收到海量 tx 时,可能不把部分交易主动转发给其他节点。** 交易本身仍进本地 pool(入站有独立容量闸门),丢的只是"通知传播 task 去广播它"。后果是**主动传播退化/漏发**,不是交易丢失(gossip 网络通常靠其他节点补齐;但对抢首达/低延迟场景是实打实的延迟来源)。

### 4.1 poll 调度模型:单 task、单线程、顺序执行、无外层循环

`TransactionsManager::poll`(mod.rs:1554)自上而下跑一遍所有 stream,每条 stream 有**独立预算**(不是共享池),末尾若还有活就 `wake_by_ref()` + 返回 `Pending`(mod.rs:1702-1712),让 tokio 重排一次 poll(中间会让别的 task 跑)。

预算常量(crates/net/network/src/budget.rs):
- `DEFAULT_BUDGET_TRY_DRAIN_STREAM = 10`
- `..._NETWORK_TRANSACTION_EVENTS = 10`(入站消息条数)
- `..._PENDING_POOL_IMPORTS = 40`(import 结果批数)

预算宏 `poll_nested_stream_with_budget!`(budget.rs:40):循环取,每取一条 `budget -= 1`,取满即 `break true`(强制停手,保公平)。

**单次 poll 内的关键执行顺序:**
```
① mod.rs:1583  排空 pending_transactions(消费 CONSUME)
② mod.rs:1628  poll transaction_events(预算 10 消息)→ import_transactions
                 ├─ retain_unknown 抢 pool 锁      (mod.rs:1430,同步阻塞)
                 ├─ into_par_iter ecrecover        (mod.rs:1441,同步阻塞)
                 └─ push future 到 pool_imports    (mod.rs:1491)
③ mod.rs:1670  驱动 pool_imports(预算 40 批)→ future 完成
                 → pool.add_transactions          (pool/mod.rs:655,抢 pool 写锁)
                 → on_new_pending_transaction      (pool/mod.rs:771)
                 → try_send 灌进 channel           (listener.rs:287,生产 PRODUCE)
```

**消费 ① 在生产 ③ 之前。** ③ 产出的通知要等**下一次 poll 的 ①** 才排走。

### 4.2 两个叠加成因

**机制 A(同 task 内,produce-after-consume,最直接):**
- ①(消费,1583)在 ③(生产,1670)之前;
- ③ 单次 poll 可完成多达 40 批 import(预算 `..._PENDING_POOL_IMPORTS = 40`),每批 `pool.add_transactions` 同步插入并对每笔新 pending `try_send` 一次;
- 此刻**没有任何东西消费 channel**(① 已过),这些通知要等下次 ①;
- 若这一波产出的新 pending 笔数 > 2048,`try_send` **当场**命中 `Full` 丢弃(listener.rs:290)。**单次 poll 内就溢出,与下次 poll 多快无关。**

**机制 B(跨 task,ecrecover/锁撑大"两次①之间"的墙钟时间):**
- "两次 ① 之间的间隔" = 本次 poll 里 ① 之后的剩余全部(②的 ecrecover 1441 + 各处 pool 锁 1430/pool.mod.rs:655 + ③ + fetcher + commands)+ 返回 Pending 后调度间隙 + 下次 poll 走到 ① 之前;
- ②的 ecrecover 与 pool 锁都排在 ① **之后**且同步阻塞,直接拉长这个间隔;
- 这段时间里,**别的 task/线程的生产者**照灌同一条 channel:RPC 提交(`add_transaction`,RPC task)、本地交易、区块 canonical 更新时 `notify_on_new_state`(pool/mod.rs:859)对 promoted pending 的通知;
- 间隔越长,积压越多,越易在下次 ① 前把 2048 填满。

### 4.3 可观测性盲区

- 丢弃点**无任何 metric**,只有 debug 日志(listener.rs:291);线上通常 info 级别,**完全看不见**。
- 出站 metric `propagated_transactions`(mod.rs:1116)只统计"发出去的",不反映"丢掉的"。
- 间接信号:`TxManagerPollDurations` 里 `acc_pending_imports` / `acc_network_events` 等 poll 耗时飙高是前兆。

### 4.4 对策(三个层次)

1. **治标**:调大 `pending_tx_listener_buffer_size`(`--txpool.max-pending-txns`)——只推迟溢出,治不了"poll 被拖住"的根因。
2. **补盲区**:在 `send_all` 的 `Full` 分支加 counter(如 `reth_txpool_pending_listener_dropped`),把静默丢弃变可观测。
3. **治本**:把 ecrecover 移出 poll 线程(改动较大),缩短单次 poll 的同步耗时,从根上减小机制 A/B 的窗口。


## 复现

### 5.0 目标与假设

用 devnet 复现"高压下 rpc 节点因 `pending_tx_listener_buffer_size` 溢出而静默丢弃转发通知,导致 sequencer 漏收部分交易"。

假设链路(单向,无旁路):

```
adventure(海量非gasless ERC20) → rpc2 → [EL P2P] → rpc(max-pending-txns=20) → [EL P2P] → seq(出块)
                                  只连rpc            唯一 choke point            唯一入口
```

rpc 的 pending→TxManager channel 只有 20,高压下 `try_send` 溢出丢弃 → rpc 不再把这些交易转发给 seq → **seq 漏收 → 这些交易永不落块**。

### 5.1 前提条件(缺一不可)

1. **交易必须是非 gasless**(正常 gas price)。gasless(0-price)交易在 XLayer op-reth 上会经 `--rollup.allow-gasless` 直接转发到 sequencer,**绕开** pending-tx-listener channel,使实验失效。adventure 的 `gasPriceGwei: 1` 满足。
2. **无 sequencer 直转旁路**:确认 rpc 节点未配 `--rollup.sequencer-http` 等 RPC 直转 sequencer 的路径(当前 devnet 未配)。否则交易走 RPC 直转,绕开 P2P。
3. **rpc2 只连 rpc**(不连 seq):否则 rpc2 会直发 seq,seq 从直连拿到全部,实验无法证明漏收。
4. **rpc→seq 通路正常**:rpc 的 trusted 含 seq、`--tx-propagation-policy=trusted` 下会转发给 seq——这正是被测的 choke point。
5. **rpc 上真设 `--txpool.max-pending-txns=20`**(是 pending 那条,不是 `--txpool.max-new-txns`,后者是另一条 full-tx listener 的 buffer,与传播无关)。

### 5.2 拓扑说明(xlayer-toolkit devnet)

- 三个 EL 节点:`op-reth-seq`(sequencer,脚本 `devnet/entrypoint/reth-seq.sh`,propagation=`all`)、`op-reth-rpc`、`op-reth-rpc2`(后两者共用 `devnet/entrypoint/reth-rpc.sh`,propagation=`trusted`)。
- `TRUSTED_PEERS`(`devnet/.env`)默认 = {seq, rpc};所有节点 `--disable-discovery`,故 **`--trusted-peers` 列表完全决定连接对象**。
- ingress 策略未显式配置 → 默认 `All`(接收任何 peer 的交易);只有 egress(传播)被设为 `trusted`。

### 5.3 devnet 配置改动

把 peer/传播 flag 从共享脚本挪到 compose 的 per-service `command:`,以获得 per-node 拓扑控制(避免脚本已有 `--trusted-peers` 与 command 追加撞成 clap 重复参数)。

**① `devnet/entrypoint/reth-rpc.sh`:删除 peer 配置块**(原 `if [ "${RETH_NO_PEERS...}" ] ... --max-outbound-peers ... --tx-propagation-policy=trusted ...` 整段),其余(如 `--disable-discovery`)保留。

**② `devnet/docker-compose.yml` 的 `op-reth-rpc` 服务加 `command:`**
```yaml
    command:
      - --max-outbound-peers=10
      - --max-inbound-peers=10
      - "--trusted-peers=${TRUSTED_PEERS}"
      - --tx-propagation-policy=trusted
      - --txpool.max-pending-txns=20
```

**③ `op-reth-rpc2` 服务加 `command:`(只连 rpc)**,并移除已失效的 `RETH_NO_PEERS` env:
```yaml
    command:
      - --max-outbound-peers=10
      - --max-inbound-peers=10
      - "--trusted-peers=enode://<op-reth-rpc 的 pubkey>@op-reth-rpc:30303"
      - --tx-propagation-policy=trusted
```
(rpc 的 enode 取自 `devnet/.env` 的 `TRUSTED_PEERS` 中 `@op-reth-rpc:30303` 那条;rpc2 的 P2P key 需固定,否则重启后自身 enode 变化。)

- `RUST_LOG=info,txpool=debug` 保持在 `op-reth-rpc`(观察节点)。
- 改后需 `docker compose up -d --force-recreate op-reth-rpc op-reth-rpc2`。

### 5.4 adventure 配置(`tools/adventure/testdata/config.json`)

- `rpc` 指向 **op-reth-rpc2**(devnet 映射端口 `127.0.0.1:8128` → rpc2:8545)。
- `gasPriceGwei: 1`(非 gasless)、`accounts: 5000`、`concurrency: 20`、`targetTPS: 1000`——多账户高并发突发,足以在 rpc 的单次 poll 内产出 >20 pending。
- 用 `make erc20` 发送(见 `tools/adventure/README.md`)。

### 5.5 观察与判定

日志(观察节点 op-reth-rpc):
```
docker logs -f op-reth-rpc 2>&1 | awk '{print} /failed to send pending tx; channel full/{exit}'
docker logs -f op-reth-rpc 2>&1 | grep --line-buffered 'failed to send pending'
```

## 几个有意思的问题
1. NetworkManager 和 TransactionsManager 都是被 tokio 当作一个 Future（一个 task）跑的。它们的 poll 里要把好几个 channel/stream 尽量抽干（drain），每抽出一条就处理它、可能又产生新的内部事件。

在高负载下（大量消息/事件），数据到达速度可能快过处理速度——处理一个事件的同时又来了更多。如果无脑 while let Poll::Ready(Some) = ... 一直抽，这个 loop 永远不会返回 Pending，就变成 busy loop：

饿死同一个 executor 上等待执行的其他 task
tokio 的协作式调度（cooperative scheduling）失效，表现为"poll 像卡住了"
参考的就是注释里贴的 tokio 抢占博客 和 cooperative scheduling 文档。

2. poll_recv_many的批量调度窗口
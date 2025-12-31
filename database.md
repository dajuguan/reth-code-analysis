## Difference with Geth
Reth's memory overylay is only 2 layers (should_persist condition), while Geth's default memory layer is 128. So, for mid-reorg (>2 & <128) Reth has to replay the change sets in DB.

## Database initialization
```rust
// bin/reth/src/main.rs
Cli::<EthereumChainSpecParser, RessArgs>::parse().run
    // crates/ethereum/cli/src/interface.rs
    - run => self.with_runner() => app.run()
        // crates/ethereum/cli/src/app.rs
        - self.run_with_components() => run_commands_with => Command::Node => command.execute()
            // crates/cli/commands/src/node.rs
            - execute() => init_db(db_path.clone())
                // crates/storage/db/src/mdbx.rs
                - init_db() => init_db_for() => create_db => DatabaseEnv::open
                    // crates/storage/db/src/implementation/mdbx/mod.rs
                    - open() -> DatabaseEnv (impl Database + DatabaseMetrics)
```

## Database provider
```rust
// crates/ethereum/cli/src/interface.rs                
- builder.node
    // crates/node/builder/src/builder/mod.rs
    - launch_with_debug_capabilities => builder.launch_with
        - launch_with => launcher.launch_node
        // crates/node/builder/src/launch/engine.rs
        - launch_node 
            => with_provider_factory 
                // crates/node/builder/src/launch/common.rs
                - self.create_provider_factory => ProviderFactory::new(db, static_files)
                => with_blockchain_db => BlockchainProvider::new
                // crates/storage/provider/src/providers/blockchain_provider.rs
                - new => Self::with_latest => BlockchainProvider {database, canonical_in_memory_state};
```

## DB persistence and in-memory db pruning
```rust
    // crates/engine/tree/src/tree/mod.rs 
    - advance_persistence 
        - (remove in-memory overlays after persist_blocks) on_new_persisted_block => self.remove_before => self.state.tree_state.remove_until
        - self.should_persist > DEFAULT_PERSISTENCE_THRESHOLD (2) 
            - (async save change sets with channel) persist_blocks => self.persistence.save_blocks 
                // crates/engine/tree/src/persistence.rs
                - self.on_save_blocks => provider_rw.save_blocks

```

## Why dupsort table on PlainStorage?

From MDBX pespective, SubKey is just the prefix of the (sorted, duplicate) values.
从libmdbx这种db层面只有kv映射，所以实际上对于PlainStorage这种语义上的addr=>(slot key=> slot value)映射关系，在底层会被转换为 addr=> multiple(slotKey+slotValue)的物理存储，也就是说slotKey只是DB存储value的prefix。所以只能先删掉相同的prefix再插入具体的value才是正确的更新storage value的语义
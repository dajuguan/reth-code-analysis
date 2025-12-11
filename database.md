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
## Execution core process
```rust
    // bin/reth/src/main.rs
    - builder.node(EthereumNode::default()).launch_with_debug_capabilities
    // crates/node/builder/src/builder/mod.rs
    - builder.launch_with()
    // crates/node/builder/src/builder/states.rs
    - launcher.launch_node
    // crates/node/builder/src/launch/engine.rs
    - launch_node => self.launch_node() => EngineService::new()
    // crates/engine/service/src/service.rs
    - new
    // crates/engine/tree/src/tree/mod.rs
    - spawn_new => task.run() => run => on_engine_message => BeaconEngineMessage::NewPayload 
    // crates/engine/tree/src/tree/mod.rs
    - on_new_payload=> try_insert_payload => insert_payload =>insert_block_or_payload => validator.validate_payload
            // crates/engine/tree/src/tree/payload_validator.rs
            - validate_payload => validate_block_with_state => self.state_provider_builder 
                => StateProviderBuilder {provider, historical, overlay: blocks} 
                => provider_builder.build() => state_provider= MemoryOverlayStateProvider::new(provider, overlay) 
                => state_provider = CachedStateProvider::new_with_caches(state_provider,  handle.caches():prewardingCache)
                => self.execute_block(&state_provider, env, &input, &mut handle)
```

## Pre-warming
```rust
// crates/engine/tree/src/tree/payload_validator.rs#L298
- validate_block_with_state
    - self.spawn_payload_processor
        - self.payload_processor.spawn_cache_exclusive
        // crates/engine/tree/src/tree/payload_processor/mod.rs#L275
        - spawn_cache_exclusive => self.spawn_caching_with => prewarm_task.run
```
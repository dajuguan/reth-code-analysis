# Engine API

## Core logic
```
Beacon Node
   ↓ (Engine API RPC)
RPC Server
   ↓
Engine API Handler
   ↓
ConsensusEngineHandle
   ↓
consensus_engine_tx
   ↓
consensus_engine_rx
   ↓
EngineService event loop
```

## engine_api RPC server:
```rust
// crates/rpc/rpc-engine-api/src/engine_api.rs:878
- new_payload_v1 => new_payload_v1_metered => new_payload_v1(self, payload)
    => self.inner.beacon_consensus.new_payload(payload)
        -   let (tx, rx) = oneshot::channel();
            let _ = self.to_engine.send(BeaconEngineMessage::NewPayload { payload, tx });
```

## engine API handler 
```rust
// crates/node/builder/src/launch/engine.rs:64
- launch_node
    - let (consensus_engine_tx, consensus_engine_rx) = unbounded_channel();
    - let beacon_engine_handle = ConsensusEngineHandle::new(consensus_engine_tx.clone())  // sender for engine_api to send msgs. e.g. the above new_payload
    - consensus_engine_stream = UnboundedReceiverStream::from(consensus_engine_rx) // wrap engine_api receiver to stream
    - EngineService::new(...,incoming_requests:consensus_engine_rx,...)
        // crates/engine/service/src/service.rs:72
        - EngineApiTreeHandler::<N::Primitives, _, _, _, _>::spawn_new
            -  task.run() => self.run => self.try_recv_engine_message() => on_engine_message => BeaconEngineMessage::NewPayload 
        - to_tree_tx, from_tree // to_tree_tx: sender channel for sending engine API requests to EngineApiTreeHandler
        - EngineApiRequestHandler::new(to_tree_tx, from_tree) // processes engine API requests by delegating to an execution task.
        - EngineHandler::new(engine_handler, downloader, incoming_requests)
        /// It is responsible for handling the following:
        /// - Delegating incoming requests to the [`EngineRequestHandler`].
        /// - Advancing the [`EngineRequestHandler`] by polling it and emitting events.
        /// - Downloading blocks on demand from the network if requested by the [`EngineApiRequestHandler`].

```

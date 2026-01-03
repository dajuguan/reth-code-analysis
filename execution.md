## Execution core process
```rust
    // bin/reth/src/main.rs
    - builder.node(EthereumNode::default()).launch_with_debug_capabilities
    // crates/node/builder/src/builder/mod.rs
    - builder.launch_with()
    // crates/node/builder/src/builder/states.rs
    - launcher.launch_node
    // crates/node/builder/src/launch/engine.rs:209
    - launch_node => self.launch_node() => EngineService::new()
    // crates/engine/service/src/service.rs
    - new
    // crates/engine/tree/src/tree/mod.rs
    - spawn_new => task.run() => self.run => self.try_recv_engine_message() => on_engine_message => BeaconEngineMessage::NewPayload 
    // crates/engine/tree/src/tree/mod.rs
    - on_new_payload=> try_insert_payload => insert_payload =>insert_block_or_payload => validator.validate_payload
            // crates/engine/tree/src/tree/payload_validator.rs
            - validate_payload => validate_block_with_state => self.state_provider_builder 
                => StateProviderBuilder {provider, historical, overlay: blocks} 
                => provider_builder.build() => state_provider= MemoryOverlayStateProvider::new(provider, overlay) 
                => state_provider = CachedStateProvider::new_with_caches(state_provider,  handle.caches():prewardingCache)
                => self.execute_block(&state_provider, env, &input, &mut handle)
```

## ChainOrchestrator: the central scheduler that coordinates Backfill (historical sync) and the EngineHandler (live sync).
```rust
// crates/engine/service/src/service.rs:72, create the ChainOrchestrator
EngineSevice::new() {
    let (to_tree_tx, from_tree) = EngineApiTreeHandler::<N::Primitives, _, _, _, _>::spawn_new(
        blockchain_db,
        consensus,
        payload_validator,
        persistence_handle,
        payload_builder,
        canonical_in_memory_state,
        tree_config,
        engine_kind,
        evm_config,
    );

    // to_tree_tx: used to send engine API request to engine
    // from_tree: used to send handled engine api status from the engine to ChainOrchestrator (by polling it)
    let engine_handler = EngineApiRequestHandler::new(to_tree_tx, from_tree);
    let handler = EngineHandler::new(engine_handler, downloader, incoming_requests);

    let backfill_sync = PipelineSync::new(pipeline, pipeline_task_spawner);

    Self { orchestrator: ChainOrchestrator::new(handler, backfill_sync) }
}
// crates/engine/service/src/service.rs:119
ChainOrchestrator::new(handler, backfill_sync)
    fn poll_next_event(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<ChainEvent<T::Event>> {
        let this = self.get_mut();

        // This loop polls the components
        //
        // 1. Polls the backfill sync to completion, if active.
        // 2. Advances the chain by polling the handler.
        'outer: loop {
            // try to poll the backfill sync to completion, if active
            match this.backfill_sync.poll(cx) {
                Poll::Ready(backfill_sync_event) => match backfill_sync_event {
                    BackfillEvent::Started(_) => {
                        // notify handler that backfill sync started, inject into the engine api handler
                        this.handler.on_event(FromOrchestrator::BackfillSyncStarted);
                        return Poll::Ready(ChainEvent::BackfillSyncStarted);
                    }
                    BackfillEvent::Finished(res) => {
                        return match res {
                            Ok(ctrl) => {
                                tracing::debug!(?ctrl, "backfill sync finished");
                                // notify handler that backfill sync finished
                                this.handler.on_event(FromOrchestrator::BackfillSyncFinished(ctrl));
                                Poll::Ready(ChainEvent::BackfillSyncFinished)
                            }
                            Err(err) => {
                                tracing::error!( %err, "backfill sync failed");
                                Poll::Ready(ChainEvent::FatalError)
                            }
                        }
                    }
                    BackfillEvent::TaskDropped(err) => {
                        tracing::error!( %err, "backfill sync task dropped");
                        return Poll::Ready(ChainEvent::FatalError);
                    }
                },
                Poll::Pending => {}
            }

            // poll the handler for the next event
            match this.handler.poll(cx) {
                Poll::Ready(handler_event) => {
                    match handler_event {
                        HandlerEvent::BackfillAction(action) => {
                            // forward action to backfill_sync
                            this.backfill_sync.on_action(action);
                        }
                        HandlerEvent::Event(ev) => {
                            // bubble up the event
                            return Poll::Ready(ChainEvent::Handler(ev));
                        }
                        HandlerEvent::FatalError => {
                            error!(target: "engine::tree", "Fatal error");
                            return Poll::Ready(ChainEvent::FatalError)
                        }
                    }
                }
                Poll::Pending => {
                    // no more events to process
                    break 'outer
                }
            }
        }

        Poll::Pending
    }
    // crates/engine/tree/src/backfill.rs:187
    - backfill_sync: poll
    // crates/engine/tree/src/engine.rs:82
    - handler(engine api):
         fn poll(&mut self, cx: &mut Context<'_>) -> Poll<HandlerEvent<Self::Event>> {
            loop {
                // drain the handler first
                while let Poll::Ready(ev) = self.handler.poll(cx) { // handler is EngineRequestHandler
                    match ev {
                        RequestHandlerEvent::HandlerEvent(ev) => {
                            return match ev {
                                HandlerEvent::BackfillAction(target) => {
                                    // bubble up backfill sync request
                                    self.downloader.on_action(DownloadAction::Clear);
                                    Poll::Ready(HandlerEvent::BackfillAction(target))
                                }
                                HandlerEvent::Event(ev) => {
                                    // bubble up the event
                                    Poll::Ready(HandlerEvent::Event(ev))
                                }
                                HandlerEvent::FatalError => Poll::Ready(HandlerEvent::FatalError),
                            }
                        }
                        RequestHandlerEvent::Download(req) => {
                            // delegate download request to the downloader
                            self.downloader.on_action(DownloadAction::Download(req));
                        }
                    }
                }

                // pop the next incoming request (engine API)
                if let Poll::Ready(Some(req)) = self.incoming_requests.poll_next_unpin(cx) {
                    // and delegate the request to the handler
                    self.handler.on_event(FromEngine::Request(req.into()));
                    // skip downloading in this iteration to allow the handler to process the request
                    continue
                }

                // advance the downloader
                if let Poll::Ready(outcome) = self.downloader.poll(cx) {
                    if let DownloadOutcome::Blocks(blocks) = outcome {
                        // delegate the downloaded blocks to the handler
                        self.handler.on_event(FromEngine::DownloadedBlocks(blocks));
                    }
                    continue
                }

                return Poll::Pending
            }
        }
    // crates/engine/tree/src/engine.rs:203
    - engineRequestHandler
        fn poll(&mut self, cx: &mut Context<'_>) -> Poll<RequestHandlerEvent<Self::Event>> {
            let Some(ev) = ready!(self.from_tree.poll_recv(cx)) else {
                return Poll::Ready(RequestHandlerEvent::HandlerEvent(HandlerEvent::FatalError))
            };

            let ev = match ev {
                EngineApiEvent::BeaconConsensus(ev) => {
                    RequestHandlerEvent::HandlerEvent(HandlerEvent::Event(ev))
                }
                EngineApiEvent::BackfillAction(action) => {
                    RequestHandlerEvent::HandlerEvent(HandlerEvent::BackfillAction(action))
                }
                EngineApiEvent::Download(action) => RequestHandlerEvent::Download(action),
            };
            Poll::Ready(ev)
        }
    // crates/engine/tree/src/tree/mod.rs:1347
    - engineApiTreeHandler
        fn on_engine_message(
            &mut self,
            msg: FromEngine<EngineApiRequest<T, N>, N::Block>,
        ) -> Result<(), InsertBlockFatalError> {
            match msg {
                FromEngine::Event(event) => match event {
                    FromOrchestrator::BackfillSyncStarted => {
                        debug!(target: "engine::tree", "received backfill sync started event");
                        self.backfill_sync_state = BackfillSyncState::Active;
                    }
                    FromOrchestrator::BackfillSyncFinished(ctrl) => {
                        self.on_backfill_sync_finished(ctrl)?;
                    }
                },
                FromEngine::Request(request) => {...}
            }
            ...
        }
            - on_backfill_sync_finished => emit_event => self.outgoing.send(event)  // ongoing <=> from_tree channel
```


## Execution engine handled event
- ChainEvent::Handler:  is the only trusted signal in reth that the chain state has been deterministically advanced. It propagates finalized chain state changes to the network layer, RPC interfaces, subscription systems, and external observers (such as indexers).
- BackfillSync events, in contrast, represent internal synchronization progress within the node and are primarily used to reflect the node’s sync status as perceived by peers.
```
loop {
    tokio::select! {
        payload = built_payloads.select_next_some() => {
            if let Some(executed_block) = payload.executed_block() {
                debug!(target: "reth::cli", block=?executed_block.recovered_block().num_hash(),  "inserting built payload");
                engine_service.orchestrator_mut().handler_mut().handler_mut().on_event(EngineApiRequest::InsertExecutedBlock(executed_block).into());
            }
        }
        event = engine_service.next() => {
            let Some(event) = event else { break };
            debug!(target: "reth::cli", "Event: {event}");
            match event {
                ChainEvent::BackfillSyncFinished => {
                    if terminate_after_backfill {
                        debug!(target: "reth::cli", "Terminating after initial backfill");
                        break
                    }
                    if startup_sync_state_idle {
                        network_handle.update_sync_state(SyncState::Idle);
                    }
                }
                ChainEvent::BackfillSyncStarted => {
                    network_handle.update_sync_state(SyncState::Syncing);
                }
                ChainEvent::FatalError => {
                    error!(target: "reth::cli", "Fatal error in consensus engine");
                    res = Err(eyre::eyre!("Fatal error in consensus engine"));
                    break
                }
                ChainEvent::Handler(ev) => {
                    if let Some(head) = ev.canonical_header() {
                        // Once we're progressing via live sync, we can consider the node is not syncing anymore
                        network_handle.update_sync_state(SyncState::Idle);
                        let head_block = Head {
                            number: head.number(),
                            hash: head.hash(),
                            difficulty: head.difficulty(),
                            timestamp: head.timestamp(),
                            total_difficulty: chainspec.final_paris_total_difficulty().filter(|_| chainspec.is_paris_active_at_block(head.number())).unwrap_or_default(),
                        };
                        network_handle.update_status(head_block);

                        let updated = BlockRangeUpdate {
                            earliest: provider.earliest_block_number().unwrap_or_default(),
                            latest:head.number(),
                            latest_hash:head.hash()
                        };
                        network_handle.update_block_range(updated);
                    }
                    event_sender.notify(ev);
                }
            }
        }
    }
}
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
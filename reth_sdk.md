# What does modularity mean in reth?
One-sentence takeaway: Reth uses Rust’s type system to fully decouple control-flow stability from semantic variability. Sounds abstract? To answer this intuitively, I built a small example (~300 LOC) showing modularity as a first-class design goal, not an afterthought. 

Consensus, EVM, executor, etc. (similating Reth-like replaceable modules) can be swapped freely, while launch_node (simulating the core control flow) remains unchanged.

This mirrors how Reth works:
complex orchestration logic is stable, while chain-specific semantics vary via types.
Code:
https://github.com/dajuguan/Rust_learn/blob/master/design-pattern/src/compiletime_plugin.rs

As a result, swapping components becomes a structural capability rather than a high-risk refactor.
Reth-SDK is effectively an implicit compile-time plugin system.

Official examples (custom EVM, custom node, Base Reth) show how to use this modularity.
The deeper lesson is how to design it with modulariy in mind: generics + traits all the way down. Kudos to the reth team!


For contrast, consider VS Code’s plugin system (**runtime modularity**):
The core control flow is fixed and non-replaceable. Extensions inject behavior at runtime via register, events, or hooks—but cannot recompose the control flow itself. 
Code: https://github.com/dajuguan/Rust_learn/blob/master/design-pattern/src/runtime_plugin.rs

Reth’s modularity is compile-time modularity: The control flow itself is parameterized by types. By swapping trait implementations, the entire execution path is reassembled at compile time.


```
NodeBuilder::new()
  .with_consensus(BscConsensus)
  .with_engine(BscEngine)
  .build()
```
- 没有“插件目录”
- 没有“加载”
- 只有类型组合


# References
- [reth doc](https://reth.rs/sdk) 
- [bera-chain:Building Modular Execution Clients on Reth](https://zorz.substack.com/p/what-does-modularity-mean-in-reth)
- [How to Build Custom RPC Methods with Reth](https://www.quicknode.com/guides/infrastructure/build-custom-rpc-methods-with-reth)

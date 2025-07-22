# dancing-with-firedancer: 第四章 - 运行时核心 (`flamenco` & `discoh` 模块)

## 引言

我们已经跟踪了一笔交易从网络进入，经过验证、去重、打包，并最终被 PoH tile “盖上时间戳”的全过程。现在，我们来到了交易生命周期中最关键的一站：**执行**。交易的执行是区块链的核心功能，它意味着状态的改变、智能合约的运行以及价值的转移。

在 Firedancer (Frankendancer) 中，这个核心任务由 `bank` tile 负责。然而，`bank` tile 本身并不直接执行交易，而是作为一个高效的 C/C++ “工头”，通过一个定义良好的 ABI (应用二进制接口)，将工作外包给 Solana Labs 原生的、身经百战的 Rust 运行时。本章，我们将深入 `flamenco` 和 `discoh` 模块，剖析 `bank` tile 是如何实现这种巧妙的混合式架构的。

## `bank` Tile: C 与 Rust 的桥梁

`bank` tile 的主要职责是协调和翻译，它接收 `pack` tile 打包好的微区块，将其转换为 Rust 运行时能够理解的格式，调用外部函数执行，处理结果，并将回执和状态更新分发给下游。

### 核心逻辑

1.  **接收微区块**: `bank` tile 通过 `mcache` 从 `pack` tile 接收微区块。`before_frag` 回调会根据 `pack` tile 附加的信号，确保只有被指定的 `bank` tile 实例才会处理该微区块，从而实现并行处理。

2.  **与 Rust 运行时交互**: 这是 `bank` tile 的核心。它通过一系列 `extern` C 函数调用一个外部的 Rust 共享库来完成交易执行。

3.  **处理结果与分发**: `bank` tile 接收 Rust 运行时的执行结果（成功、失败、消耗的 CU 等），更新内部状态，并将这些信息分发给 `poh` tile (用于混入 PoH 链) 和 `pack` tile (用于 CU 返还)。

### 关键源码 (`src/discoh/bank/fd_bank_tile.c`)

`handle_microblock` 函数展示了与 Rust ABI 交互的完整流程。

```c
// 声明一组外部函数，这些是与 Rust 运行时交互的 ABI 接口
extern void * fd_ext_bank_load_and_execute_txns( ... );
extern void   fd_ext_bank_commit_txns( ... );

/**
 * @brief 处理单个微区块的核心函数。
 */
static inline void
handle_microblock( fd_bank_ctx_t *     ctx,
                   // ...
                 ) {
  // ...
  // 1. 准备阶段：将 Firedancer 格式的交易翻译成 Rust ABI 格式
  for( ulong i=0UL; i<txn_cnt; i++ ) {
    // 调用 ABI 函数进行翻译和初始化
    int result = fd_bank_abi_txn_init( abi_txn, ..., ctx->_bank, ..., txn->payload, ... );
    // ...
  }

  // 2. 执行阶段：调用外部函数，将整个微区块交给 Rust 运行时执行
  void * load_and_execute_output = fd_ext_bank_load_and_execute_txns( 
      ctx->_bank,           // Rust Bank 对象指针
      ctx->txn_abi_mem,     // 翻译好的交易数组
      // ... (输出参数，用于接收结果)
  );

  // 3. 结果处理：遍历 Rust 返回的结果，更新 Firedancer 侧的交易状态
  for( ulong i=0UL; i<txn_cnt; i++ ) {
    // ...
    // 根据返回的错误码和 CU 消耗，更新交易的 flags 和 bank_cu 字段
    txn->bank_cu.rebated_cus = ...;
    txn->bank_cu.actual_consumed_cus = ...;
    txn->flags |= FD_TXN_P_FLAGS_EXECUTE_SUCCESS;
    // ...
  }

  // 4. 提交阶段：调用外部函数，同步地提交交易结果，使其状态变更正式生效
  fd_ext_bank_commit_txns( ctx->_bank, ctx->txn_abi_mem, sanitized_txn_cnt, load_and_execute_output, pre_balance_info );

  // 5. 发送忙闲信号：更新共享序列号，通知 pack tile 自己已空闲
  fd_fseq_update( ctx->busy_fseq, seq );

  // 6. 准备 CU 回执：将所有交易的 CU 返还信息累加到 rebater 中
  fd_pack_rebate_sum_add_txn( ctx->rebater, ... );

  // 7. 发送结果给 PoH：计算交易哈希，并将带有执行结果的微区块发布给 PoH tile
  hash_transactions( ... );
  fd_stem_publish( stem, 0UL, bank_sig, ctx->out_chunk, new_sz, ... );
  
  // 8. 发送 CU 回执：将累积的 CU 回执信息发布给 pack tile
  while( 0UL!=(written_sz=fd_pack_rebate_sum_report( ... )) ) {
    fd_stem_publish( stem, 1UL, slot, ctx->rebate_chunk, written_sz, ... );
  }
}
```

## 总结

`bank` tile 是 Firedancer (Frankendancer) 混合式架构的精髓所在。它通过一个定义良好的 ABI，巧妙地复用了经过大量审计和生产环境验证的 Solana Labs Rust 运行时，同时将这个运行时无缝地集成到了 Firedancer 高性能的、流水线式的多进程架构中。

这种设计使得 Firedancer 可以在保证核心执行逻辑与主网完全兼容的前提下，专注于优化网络、调度、共识等外围系统，从而实现性能的大幅超越。理解了 `bank` tile，我们也就理解了 Frankendancer 能够快速落地并取得成功的关键原因。

在下一章中，我们将把目光投向 P2P 网络，看看节点之间是如何通过 `gossip` 协议发现彼此并同步信息的。

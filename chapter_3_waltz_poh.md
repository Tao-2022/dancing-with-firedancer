# dancing-with-firedancer: 第三章 - 共识核心 (`waltz` & `discoh` 模块)

## 引言

在上一章中，我们跟踪了交易从网络接口到被打包成微区块的完整流水线。然而，一个微区块仅仅是交易的有序集合，它自身并不包含任何时间或顺序的证明。为了让整个分布式网络对交易的发生顺序达成一致，Solana 引入了其核心创新之一：历史证明 (Proof of History, PoH)。

本章，我们将深入 Firedancer 的“节拍器”——`waltz` 模块，并聚焦于其在 `discoh` 中的具体实现 `poh` tile。我们将探索 PoH 是如何生成的，以及它如何为交易“盖上”不可篡改的时间戳，从而为共识奠定基础。

## PoH: 并非时钟，而是时间序列的证明

在分析代码前，理解 PoH 的本质至关重要。`src/discoh/poh/fd_poh_tile.c` 文件开头的注释对此有精彩的阐述：

*   **PoH 不是时钟**: 它不与现实世界的时间严格同步。网络的“时间”是由领导者在 400ms 的 slot 内打包交易来驱动的。
*   **PoH 的核心作用**: 它的主要作用是**证明时间的流逝**，尤其是在出现空块（skipped slot）时。一个领导者如果声称跳过了 N 个 slot，它必须出示在这段时间内连续不断进行哈希运算的证明，否则网络不会接受它的区块。这也被称为 **Proof of Skipping**。
*   **混入交易**: 将交易（的哈希）混入 PoH 链，为交易在全球范围内提供了一个统一的、可验证的发生顺序。一旦一个交易被混入，就无法在不改变后续所有 PoH 哈希的情况下篡改它或它的顺序。

## `poh` Tile: 共识的节拍器

`poh` tile 的核心职责就是维护一个不断向前滚动的哈希链，并在作为领导者时，将 `bank` tile 执行完毕的微区块混入链中。

### 核心逻辑

1.  **持续哈希 (`after_credit`)**: `poh` tile 的主循环大部分时间都在做一件事：`fd_sha256_hash( ctx->hash, 32UL, ctx->hash );`。它不断地对自身的 `hash` 状态进行哈希，无论当前是否是领导者，从而创造一个连续的时间流。

2.  **响应领导者状态**: `poh` tile 通过外部（Agave 的 Replay Stage）调用 `fd_ext_poh_begin_leader` 和 `fd_ext_poh_reset` 来与网络的整体状态保持同步。当轮到自己成为领导者时，它会收到一个新的 `bank` 对象，并开始准备打包。

3.  **处理并混入微区块 (`after_frag`)**: 这是将交易锚定在历史中的关键步骤。

**关键源码 (`src/discoh/poh/fd_poh_tile.c`)**:

```c
/**
 * @brief 在接收到来自 bank tile 的已执行微区块后，由 fd_stem 框架调用。
 * @param ctx poh tile 的上下文。
 * ... (其他参数)
 *
 * 该函数负责将微区块的哈希“混入”PoH 链，并转发给 shred tile。
 */
static inline void
after_frag( fd_poh_ctx_t *      ctx,
            // ...
            fd_stem_context_t * stem ) {
  // ... (省略了对输入来源的检查和数据的拷贝)

  // 从微区块的 trailer 中获取其交易的 Merkle 树根
  uchar const * mixin_hash = ctx->_microblock_trailer->hash;

  // **PoH 的核心混入操作**
  uchar two_hashes[ 64 ];
  fd_memcpy( two_hashes,      ctx->hash,    32UL );
  fd_memcpy( two_hashes+32UL, mixin_hash, 32UL );
  fd_sha256_hash( two_hashes, 64UL, ctx->hash ); // 将微区块哈希混入 PoH 链

  // 前进 hashcnt
  ctx->hashcnt++;
  if( FD_UNLIKELY( ctx->hashcnt==ctx->hashcnt_per_slot ) ) {
    ctx->slot++;
    ctx->hashcnt=0UL;
  }

  // 将这个带有 PoH 证明的微区块发布给 shred tile
  publish_microblock( ctx, stem, slot, hashcnt_delta, txn_cnt );

  // ...
}

/**
 * @brief 在每个循环迭代的后期由 fd_stem 框架调用。这是 PoH tile 的“节拍器”。
 */
static inline void
after_credit( fd_poh_ctx_t *      ctx,
              fd_stem_context_t * stem,
              // ...
            ) {
  // ...
  // **核心的 PoH 循环**
  while( ctx->hashcnt<target_hashcnt ) {
    fd_sha256_hash( ctx->hash, 32UL, ctx->hash );
    ctx->hashcnt++;
  }
  // ...
  // 如果是领导者，并且到达了一个 tick 边界
  if( FD_UNLIKELY( is_leader && !(ctx->hashcnt%ctx->hashcnt_per_tick) ) ) {
    // 通知 bank 对象
    fd_ext_poh_register_tick( ctx->current_leader_bank, ctx->hash );
    // 将这个 tick 发布给 shred tile
    publish_tick( ctx, stem, ctx->hash, 0 );
  }
  // ...
}
```

## 总结

`poh` tile 通过一个看似简单却至关重要的哈希循环，为整个 Solana 网络提供了统一的时间度量衡。它将 `bank` tile 的执行结果（微区块）与这个时间度量衡绑定，生成了不可篡改的历史记录。这个记录随后被 `shred` tile 切片并广播，构成了其他验证者进行投票和状态同步的基础。

理解了 PoH，我们才能真正理解 Solana 为何能实现如此高的速度和吞吐量。在下一章中，我们将深入交易执行的核心——`bank` tile，看看它是如何与 PoH 协同工作的。

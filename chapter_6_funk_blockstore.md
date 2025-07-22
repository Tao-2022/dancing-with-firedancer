# dancing-with-firedancer: 第六章 - 持久化存储 (`funk` 模块)

## 引言

我们已经跟踪了交易数据在 Firedancer 中从接收、处理、执行到共识的全过程。然而，区块链的核心价值在于其**持久性**和**不可篡改性**。所有已确认的区块数据都必须被安全、高效地存储到磁盘上，形成不可磨灭的账本。这个关键任务在 Firedancer 中由 `blockstore` (块存储) 完成，它由 `funk` 模块提供底层的键值存储能力。

本章是 Firedancer 核心流程分析的最后一站。我们将深入 `funk` 和 `flamenco/runtime` 模块，探索 Firedancer 是如何设计其账本存储，并实现高效、并发、持久的数据写入和读取的。

## `blockstore`: 不仅仅是数据库

`blockstore` 不仅仅是一个简单的数据库接口，它是一个复杂的系统，负责：

*   **缓存 Shreds**: 从网络接收到的 Shreds 会被临时缓存在内存中，直到一个完整的区块可以被重建。
*   **状态管理**: 为每个 slot 维护一个 `fd_block_info_t` 状态机，跟踪其 Shreds 的接收进度、处理状态（已完成、已处理、已确认等）。
*   **索引**: 在内存中为 slot、shred 和交易签名建立高效的哈希索引，以便快速查询。
*   **持久化**: 将最终确认的区块数据归档到磁盘上的文件中。
*   **垃圾回收**: 通过“剪枝”和“归档”机制，定期清理内存中不再需要的旧数据。

### 核心数据结构与 API

`blockstore` 的核心数据结构和 API 定义在 `src/flamenco/runtime/fd_blockstore.h` 中。

**关键数据结构 (`fd_blockstore.h`)**:

```c
/**
 * @brief 单个 slot (区块) 的元数据。
 * 这是 blockstore 中最核心的数据结构之一，用于跟踪一个 slot 的所有状态。
 */
struct fd_block_info {
  ulong slot; // slot 号
  ulong parent_slot; // 父 slot 号
  // ... (子 slot 列表)
  fd_hash_t block_hash;   // 区块哈希 (PoH 哈希)
  fd_hash_t bank_hash;    // 银行哈希 (状态哈希)
  uchar     flags;        // 状态标志位 (e.g., RECEIVING, COMPLETED, PROCESSED)
  // ... (Shred 接收窗口的关键索引)
};

/**
 * @brief blockstore 的本地句柄。
 */
struct fd_blockstore {
  fd_blockstore_shmem_t * shmem; // 指向共享内存区域
  fd_buf_shred_pool_t shred_pool[1]; // shred 内存池
  fd_buf_shred_map_t  shred_map[1];  // shred 哈希表
  fd_block_map_t      block_map[1];  // slot 元数据哈希表
};
```

### 核心写入与归档逻辑

`blockstore` 的实现位于 `src/flamenco/runtime/fd_blockstore.c`。

**关键源码 (`fd_blockstore.c`)**:

```c
/**
 * @brief 将一个 shred 插入 blockstore。
 * 这是由 shred tile 调用的核心写入函数。
 */
void
fd_blockstore_shred_insert( fd_blockstore_t * blockstore, fd_shred_t const * shred ) {
  // 1. 检查 slot 是否过时
  // 2. 检查 shred 是否已存在 (去重)
  // 3. 从内存池获取一个空闲的 fd_buf_shred_t 对象
  fd_buf_shred_t * ele = fd_buf_shred_pool_acquire( ... );
  // 4. 拷贝 shred 数据并将其插入 shred_map 哈希表
  memcpy( &ele->buf, shred, fd_shred_sz( shred ) );
  fd_buf_shred_map_insert( blockstore->shred_map, ele, ... );

  // 5. 更新该 slot 的元数据 (fd_block_info_t)
  // a. 如果是该 slot 的第一个 shred，则创建新的 block_info 条目
  if( FD_UNLIKELY( !fd_blockstore_block_info_test( blockstore, slot ) ) ) {
    // ... (调用 fd_block_map_prepare 创建新条目并初始化)
  }
  // b. 更新索引和状态标志 (received_idx, slot_complete_idx 等)
  // c. 更新父子关系链
}

/**
 * @brief 发布（归档和剪枝）所有低于水印 (wmk) 的 slot。
 */
void
fd_blockstore_publish( fd_blockstore_t * blockstore,
                       int fd,
                       ulong wmk ) {
  // 1. 使用广度优先搜索 (BFS) 从旧的水印开始遍历区块树
  while( !fd_slot_deque_empty( q ) ) {
    ulong slot = fd_slot_deque_pop_head( q );
    // ...
    // 2. 将所有不属于新水印所在分叉的子 slot 从内存中移除 (剪枝)
    // 3. 如果 block 是最终确认的，则归档到磁盘 (当前代码中被注释)
    /* if( fd_uchar_extract_bit( block_info->flags, FD_BLOCK_FLAG_FINALIZED ) ) {
      ... (写入磁盘文件 fd)
    } */
    // 4. 从内存中移除这个 slot 的所有信息
    fd_blockstore_slot_remove( blockstore, slot );
  }

  // 5. 更新共享内存中的水印值
  blockstore->shmem->wmk = wmk;
}
```

## 总结

`funk` 模块和 `blockstore` 的实现，为 Firedancer 提供了一个高性能、支持并发的、基于共享内存的持久化账本存储。它通过内存优先、延迟归档、并发安全等设计，确保了在不牺牲性能的前提下，实现数据的安全持久化。

至此，我们已经完成了对 Firedancer 从网络 I/O、交易处理、共识到最终存储的全链路分析。我们不仅理解了每个核心模块的职责，更重要的是，我们看到了这些模块是如何通过高效的 IPC 机制和精巧的并发模型，协同工作，共同构成一个高性能的分布式系统。

Firedancer 的学习之旅远未结束，但我们已经为其宏伟的架构绘制了一幅清晰的地图。希望这个系列能为您后续更深入的探索打下坚实的基础。

# dancing-with-firedancer: 第二章 - 交易处理流水线 (`disco` 模块)

## 引言

在第一章中，我们了解了 Firedancer 是如何通过 `fdctl` 启动并管理其多进程架构的。现在，我们将深入这个架构的核心——`disco` 模块。`disco` 模块是 Firedancer 的“消化系统”，它构建了一条高效的、流水线式的处理链，负责将原始的网络数据包，一步步处理成可以被执行和广播的、干净的交易数据。

本章将顺着数据流的方向，逐一剖析构成 `disco` 流水线的五个核心 Tile，揭示 Firedancer 高性能交易处理的秘密。

## 整体流程图 (修正版)

```mermaid
graph TD
    subgraph 网络接口 (NIC)
        A[网络数据包]
    end

    subgraph "1. Net Tile"
        B(AF_XDP 零拷贝接收)
        C{解析 L3/L4 头}
        D[发布到 mcache]
        B --> C --> D;
    end

    subgraph "2. Verify Tile"
        E[从 mcache 消费]
        F{负载均衡}
        G[解析交易]
        H{签名验证 & 去重};
        I[发布到下游 dcache]
        J[丢弃]
        E --> F --> G --> H;
        H -- 成功 --> I;
        H -- 失败 --> J;
    end

    subgraph "3. Dedup Tile"
        K[从多个 Verify Tile 消费]
        L{全局去重 (tcache)}
        M[发布到下游 dcache]
        N[丢弃]
        K --> L;
        L -- 唯一 --> M;
        L -- 重复 --> N;
    end

    subgraph "4. Pack Tile"
        O[从 Dedup Tile 消费]
        P(存入内部交易池)
        Q[PoH Tile 信号: 成为领导者]
        R{调度决策}
        S[打包成微区块]
        T[发布给 Bank 和 PoH Tiles]
        O --> P --> R;
        Q --> R;
        R --> S --> T;
    end

    subgraph "5. PoH Tile (中间步骤)"
        U(接收微区块)
        V{混合PoH哈希/盖时间戳};
        W[转发给 Shred Tile];
        U --> V --> W;
    end

    subgraph "6. Shred Tile"
        X[从 PoH Tile 消费]
        Y{累积成批次}
        Z[切片 & 纠删码编码]
        AA{签名}
        BB[计算 Turbine 目的地]
        CC[通过 mcache 发送给 Net Tile]
        X --> Y --> Z --> AA --> BB;
    end
    
    %% Connections between Tiles
    A --> B;
    D --> E;
    I --> K;
    M --> O;
    T --> U;
    W --> X;
    CC --> A;
```

## 各阶段详解与关键源码

### 1. Net Tile: 网络IO的入口

**职责**: 使用 Linux 的 `AF_XDP` 技术，从网卡零拷贝地接收原始网络包，并根据 UDP 目标端口号，将数据包快速分发到下游不同的处理流水线。

**关键源码 (`src/disco/net/xdp/fd_xdp_tile.c`)**:
`net_rx_packet` 函数是核心的接收和分发逻辑。

```c
// 当一个新数据包从 XDP socket 接收到时被调用。
static void
net_rx_packet( fd_net_ctx_t * ctx,
               ulong          umem_off,
               ulong          sz,
               uint *         freed_chunk ) {

  // 1. 解析 L3/L4 头部，获取 IP 地址和端口号
  uint ip_srcaddr    = ...;
  ushort udp_dstport = ...;

  // 2. 根据目标端口号，选择下游的 out context (mcache)
  fd_net_out_ctx_t * out;
  if(      FD_UNLIKELY( udp_dstport==ctx->shred_listen_port ) ) {
    proto = DST_PROTO_SHRED;
    out = ctx->shred_out;
  } else if( FD_UNLIKELY( udp_dstport==ctx->quic_transaction_listen_port ) ) {
    proto = DST_PROTO_TPU_QUIC;
    out = ctx->quic_out;
  } // ... 其他端口的判断

  // 3. 创建一个信号 (signature)，包含了路由所需的信息
  ulong sig = fd_disco_netmux_sig( ip_srcaddr, udp_srcport, 0U, proto, ... );

  // 4. 将新数据包的元数据发布到 mcache
  fd_mcache_publish( out->mcache, out->depth, out->seq, sig, chunk, sz, ... );

  // ...
}
```

### 2. Verify Tile: 签名验证与初步去重

**职责**: 从 `net` tile 消费交易数据，通过轮询方式实现多个 `verify` tile 实例间的负载均衡。然后对交易进行解析、签名验证，并使用本地的 `tcache` (Transaction Cache) 进行初步去重。

**关键源码 (`src/disco/verify/fd_verify_tile.c`)**:
`after_frag` 函数是其核心处理逻辑。

```c
// 在数据拷贝完成后，由 fd_stem 框架调用。
static inline void
after_frag( fd_verify_ctx_t *   ctx,
            // ...
            fd_stem_context_t * stem ) {

  // 1. 解析交易，将原始字节流转换为结构化数据
  txnm->txn_t_sz = (ushort)fd_txn_parse( fd_txn_m_payload( txnm ), txnm->payload_sz, txnt, NULL );
  if( FD_UNLIKELY( !txnm->txn_t_sz ) ) {
    ctx->metrics.parse_fail_cnt++;
    return; // 解析失败，丢弃
  }

  // 2. 验证交易 (包括去重和签名验证)
  int res = fd_txn_verify( ctx, fd_txn_m_payload( txnm ), txnm->payload_sz, txnt, !is_bundle, &_txn_sig );
  if( FD_UNLIKELY( res!=FD_TXN_VERIFY_SUCCESS ) ) {
    if( FD_LIKELY( res==FD_TXN_VERIFY_DEDUP ) ) ctx->metrics.dedup_fail_cnt++; // 是重复交易
    else                                        ctx->metrics.verify_fail_cnt++; // 签名验证失败
    return; // 验证失败，丢弃
  }

  // 3. 验证成功，将处理好的交易发布到下游 tile
  fd_stem_publish( stem, 0UL, 0UL, ctx->out_chunk, realized_sz, 0UL, tsorig, tspub );
  // ...
}
```

### 3. Dedup Tile: 全局交易去重

**职责**: 聚合来自所有 `verify` tile 和 `gossip` tile 的数据流，使用一个全局共享的 `tcache` 进行最终的、彻底的去重，确保下游的 `pack` tile 不会处理重复的交易。

**关键源码 (`src/disco/dedup/fd_dedup_tile.c`)**:
`after_frag` 函数同样是核心，但其重点在于 `FD_TCACHE_INSERT`。

```c
// 在数据拷贝完成后，由 fd_stem 框架调用。
static inline void
after_frag( fd_dedup_ctx_t *    ctx,
            // ...
            fd_stem_context_t * stem ) {
  // ...
  int is_dup = 0;
  // 1. 对普通交易，计算签名哈希
  ulong ha_dedup_tag = fd_hash( ctx->hashmap_seed, fd_txn_m_payload( txnm )+txn->signature_off, 64UL );

  // 2. 使用 tcache 检查并插入。如果已存在，is_dup 会被设为 1。
  FD_TCACHE_INSERT( is_dup, *ctx->tcache_sync, ctx->tcache_ring, ctx->tcache_depth, ctx->tcache_map, ctx->tcache_map_cnt, ha_dedup_tag );

  if( FD_LIKELY( is_dup ) ) {
    // 如果是重复交易，则更新指标并丢弃
    ctx->metrics.dedup_fail_cnt++;
  } else {
    // 3. 如果不是重复交易，则发布到下游
    fd_stem_publish( stem, 0UL, 0, ctx->out_chunk, realized_sz, 0UL, tsorig, tspub );
    // ...
  }
}
```

### 4. Pack Tile: 智能区块打包

**职责**: 这是交易处理的决策中心。它从 `dedup` tile 接收干净的交易，并根据领导者状态、银行（bank）忙闲、交易费用、CU 限制等多种因素，智能地将交易打包成微区块，**发送给 `bank` tile 执行和 `poh` tile 进行历史回溯**。

**关键源码 (`src/disco/pack/fd_pack_tile.c`)**:
`after_credit` 函数是其“大脑”，负责调度决策。

```c
// 在每个循环迭代的后期由 fd_stem 框架调用。
static inline void
after_credit( fd_pack_ctx_t *     ctx,
              fd_stem_context_t * stem,
              // ...
            ) {
  // ...
  // 1. 检查领导者状态和银行忙闲状态
  if( FD_UNLIKELY( ctx->leader_slot==ULONG_MAX ) ) return; // 不是领导者，返回

  // 2. 决策是否打包：检查是否有足够的交易，或者是否已等待足够长的时间
  if( FD_UNLIKELY( (ulong)(now-ctx->last_successful_insert) <
        ctx->wait_duration_ticks[ ... ] ) ) {
    return; // 时机未到，继续等待
  }

  // 3. 尝试调度下一个微区块
  if( FD_LIKELY( ctx->bank_idle_bitset ) ) { // 必须有空闲的 bank
    int i = fd_ulong_find_lsb( ctx->bank_idle_bitset ); // 选择一个空闲的 bank

    // 调用核心调度函数！
    ulong schedule_cnt = fd_pack_schedule_next_microblock( ctx->pack, CUS_PER_MICROBLOCK, VOTE_FRACTION, (ulong)i, flags, microblock_dst );

    if( FD_LIKELY( schedule_cnt ) ) {
      // 4. 调度成功，将打包好的微区块发布给下游 bank 和 poh tiles
      ulong sig = fd_disco_poh_sig( ctx->leader_slot, POH_PKT_TYPE_MICROBLOCK, (ulong)i );
      fd_stem_publish( stem, 0UL, sig, chunk, msg_sz, ... );
      // ...
    }
  }
  // ...
}
```

### 5. Shred Tile: 切片、编码与广播预备

**职责**: **从 `poh` tile 接收带有历史证明的微区块**，然后将其转换成 Solana 网络中物理传输的 Shreds。它负责切片、生成纠删码、签名，并计算出每个 Shred 在 Turbine 网络中的传播路径，最后交给 `net` tile 发送。

**关键源码 (`src/disco/shred/fd_shred_tile.c`)**:
`during_frag` 负责从 `poh` tile 累积批次并触发切片。

```c
// 在接收到一个新的数据分片时由 fd_stem 框架调用。
static void
during_frag( fd_shred_ctx_t * ctx, ... ) {
  // 检查输入是否来自 PoH tile
  if( FD_UNLIKELY( ctx->in_kind[ in_idx ]==IN_KIND_POH ) ) {
    // --- 作为领导者的逻辑 ---
    // 1. 累积从 PoH tile 来的微区块到 pending_batch
    // ...
    // 2. 检查是否达到触发切片的条件
    if( FD_LIKELY( process_current_batch )) {
      if( FD_UNLIKELY( SHOULD_PROCESS_THESE_SHREDS ) ) {
        // 3. 初始化 shredder
        fd_shredder_init_batch( ... );
        // 4. 循环生成 FEC sets
        while( pend_sz > 0UL ) {
          // 调用 shredder 核心函数，生成一个 FEC set
          FD_TEST( fd_shredder_next_fec_set( ctx->shredder, out, ... ) );
          // 记录下已生成的 FEC set 索引，准备在 after_frag 中发送
          ctx->send_fec_set_idx[ ctx->send_fec_set_cnt++ ] = ...;
        }
        fd_shredder_fini_batch( ctx->shredder );
      }
      // ...
    }
  }
}
```
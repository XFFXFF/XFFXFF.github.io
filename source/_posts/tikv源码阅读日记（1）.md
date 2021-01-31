---
title: tikv源码阅读日记（1）
date: 2021-01-31 15:09:39
tags:
---
## [Pull Request #70](https://github.com/tikv/tikv/pull/70)

```rust
pub trait Storage {
    fn initial_state(&self) -> Result<RaftState>;
    fn entries(&self, low: u64, high: u64, max_size: u64) -> Result<Vec<Entry>>;
    fn term(&self, idx: u64) -> Result<u64>;
    fn first_index(&self) -> Result<u64>;
    fn last_index(&self) -> Result<u64>;
    fn snapshot(&self) -> Result<Snapshot>;
}
```

Storage这个trait有一些查询功能，也就是从raft log中查询，但没有向raft log写入的接口。

https://pingcap.com/blog-cn/tikv-source-code-reading-2/#storage-trait 这个连接有关于这个trait的一些注释。

```rust
pub struct RegionStorage {
    engine: Arc<DB>,

    meta: Arc<RwLock<RegionMeta>>,
}

impl RegionStorage {
    pub fn new(engine: Arc<DB>, meta: Arc<RwLock<RegionMeta>>) -> RegionStorage {
        RegionStorage {
            engine: engine,
            meta: meta,
        }
    }
}
```

这个RegionStorage会实现Storage这个trait，查询entries，term都是从engine中查，所以说这个engine存储了raft log。

## [Pull Request #92](https://github.com/tikv/tikv/pull/92)

```rust
pub struct Peer {
    engine: Arc<DB>,
    store_id: u64,
    peer_id: u64,
    region_id: u64,
    pub raft_group: RawNode<RaftStorage>,
    pub storage: Arc<RwLock<PeerStorage>>,
}
```

```rust
pub struct PeerStorage {
    engine: Arc<DB>,

    pub region: metapb::Region,
    pub last_index: u64,
    pub applied_index: u64,
    pub truncated_state: RaftTruncatedState,
}
```

```rust
pub struct RaftStorage {
    store: Arc<RwLock<PeerStorage>>,
}
```

Peer struct中engine，raft_group, storage公用一个storage engine。RaftStorage实现了Storage trait， PeerStorage没有实现Storage trait，PeerStorage有一些其他的接口，不明白有什么用。

```rust
// Append the given entries to the raft log using previous last index or self.last_index.
// Return the new last index for later update. After we commit in engine, we can set last_index
// to the return one.
pub fn append<T: Mutator>(&self,
    w: &T,
    prev_last_index: u64,
    entries: &[Entry])
-> Result<u64>

// Apply the peer with given snapshot.
pub fn apply_snapshot<T: Mutator>(&mut self,
    w: &T,
    snap: &Snapshot)
-> Result<ApplySnapResult> 
```

Storage对于Raft来说到底有啥用？

> Raft 算法中的日志复制部分抽象了一个可以不断追加写入新日志的持久化数组，这一数组在 raft-rs 中即对应 Storage
>
> https://pingcap.com/blog-cn/tikv-source-code-reading-2/#storage-trait

还是不太明白。
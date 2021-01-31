---
title: tikv源码阅读日记（2）
date: 2021-01-31 16:26:43
tags:
---
## [Pull Request #126](https://github.com/tikv/tikv/pull/126)

现在raftserver/store/终于有一些tests了，顺着test，更容易理清各模块之间的关系。

一个store就是一个节点，上面会运行一个raft服务（raft节点？），Store(raftserver/store/store.rs)其实是一个mio::Handler，用于相应在event loop中注册的各个事件。（这里用的mio版本是0.5，现在的版本中已经没有mio::Handler和EventLoop了，接口的变化还是挺大的）。

```rust
impl<T: Transport> mio::Handler for Store<T> {
    type Timeout = Msg;
    type Message = Msg;

    fn notify(&mut self, event_loop: &mut EventLoop<Self>, msg: Msg) {
        match msg {
            Msg::RaftMessage(data) => ...
            Msg::RaftCommand{request, callback} => ...
            Msg::Quit => ...
            _ => panic!("invalid notify msg type {:?}", msg),
        }
    }

    fn timeout(&mut self, event_loop: &mut EventLoop<Self>, timeout: Msg) {
        match timeout {
            Msg::RaftBaseTick => self.handle_raft_base_tick(event_loop),
            _ => panic!("invalid timeout msg type {:?}", timeout),
        }
    }

    fn tick(&mut self, event_loop: &mut EventLoop<Self>) ...
}
```

Store在启动时就会注册timeout事件用于每隔一定的时间调用raft.tick，对超时进行处理。比如对Follower而言，如果它tick的时候发现很久没有收到heartbeats了，就会变成candidate，发起选举。

```rust
pub fn run(&mut self) -> Result<()> {
    try!(self.prepare());

    let mut event_loop = self.event_loop.take().unwrap();
    self.register_raft_base_tick(&mut event_loop);
    try!(event_loop.run(self));
    Ok(())
}
```

注意到Store的notify函数中会对RaftMessage和RaftCommand事件进行响应，tests/raftserver/test_single_store.rs和tests/raftserver/test_multi_store.rs现在还都只对RaftCommand事件进行了测试。我的理解是RaftCommand都是外部应用程序向集群发送的Request，比如put，get等。

RaftCommandRequest定义在src/proto/raft_cmdpb.proto中，request的类型有get, seek, put和delete。  

把RaftCommandRequest封装成一个RaftCommand，通过sender（event loop channel)发送，这样Store的notify函数就会被调用，并且处理RaftCommand事件。需要注意的是这里只有raft group中的leader所在的store才会收到RaftCommand，接着会调用raft.propse，这就是raft的相关逻辑了，可以结合[TiKV 源码解析系列文章（二）raft-rs proposal 示例情景分析](https://pingcap.com/blog-cn/tikv-source-code-reading-2/)理解。
## Source code

Starknet papyrus-node server

* *P2p network*
* *Monitoring server.*
* *Json RPC*
* *p2p sync server*

### sequencing: create_block && create proposal && vote 

papyrus/crates/sequencing/papyrus_consensus/lib.rs

构建新的共识L2区块，经过recevier 验证之后，同步给其他的节点（一个中心化的服务，也不知道有什么同步的:) ）

验证区块则是验证区块中的消息是否和接收到的一致，以及区块的hash，确认无误后触发statemachine event 

### P2p sync server 

这个服务主要是用来响应查询的需求，包含：

```rust
pub struct P2PSyncServer<
    HeaderQueryReceiver,
    StateDiffQueryReceiver,
    TransactionQueryReceiver,
    ClassQueryReceiver,
    EventQueryReceiver,
> {
    storage_reader: StorageReader,
    header_queries_receiver: HeaderQueryReceiver, // block header
    state_diff_queries_receiver: StateDiffQueryReceiver, // state
    transaction_queries_receiver: TransactionQueryReceiver, // tx
    class_queries_receiver: ClassQueryReceiver, //class maybe dapp?
    event_queries_receiver: EventQueryReceiver, //event
}
```

### p2p network

注册用来接受各种信息的channle

```rust
fn run_network(config: Option<NetworkConfig>) -> anyhow::Result<NetworkRunReturn> {
  ...  register netwokr_manager server and recv && send channels 
  
  Ok((
        network_manager.run().boxed(),
        Some(p2p_sync_channels),
        Some((
            header_server_channel,
            state_diff_server_channel,
            transaction_server_channel,
            class_server_channel,
            event_server_channel,
        )),
        consensus_channels,
        local_peer_id,
    ))
}
```

### Monitoring server

监控网关服务

```rust
async fn run_server(&self) -> std::result::Result<(), hyper::Error> {
        let server_address = SocketAddr::from_str(&self.config.server_address)
            .expect("Configuration value for monitor server address should be valid");
        let app = app(
            self.config.starknet_url.clone(),
            self.storage_reader.clone(),
            self.version,
            self.full_general_config_presentation.clone(),
            self.public_general_config_presentation.clone(),
            self.config.present_full_config_secret.clone(),
            self.prometheus_handle.clone(),
            self.own_peer_id.clone(),
        );
        debug!("Starting monitoring gateway.");
        axum::Server::bind(&server_address).serve(app.into_make_service()).await
    }
```


<div class="date-published">07 Feb 2025</div>

# Building Leader Election in Distributed Systems with etcd

## Introduction

In distributed systems, leader election is a critical process that ensures coordination and consistency across multiple nodes. One way to implement leader election is through the use of a reliable key-value store, such as etcd. In this article, I'll explore how to implement a simple leader election implementation in Rust using an etcd server.
Briefly, the 'leader election' problem in a distributed system addresses the coordination challenge among multiple nodes. The goal is to elect a leader node that makes decisions on behalf of all nodes in a cluster, avoiding conflicts, race conditions, and inconsistencies.
Etcd is a distributed key-value store providing a reliable way to store data across a cluster of nodes (machines), ensuring consistency and availability. Besides the basic CRUD operations on key-value pairs, etcd provdes other crucial components such lease and distributed shared lock. In simple terms, a lease is a mechanism used to manage the lifetime of a key-value pair in the etcd key-value store. You can create a lease with a specific Time-To-Live (TTL) and then associate it with a key. When the lease’s TTL expires, the key is automatically deleted. Similarly, distributed lock is used to coordinate access to a shared resource in a distributed system, ensuring that only one node can hold the lock at a time.
These features, when combined, form the core of a leader election service. Version 3 of etcd introduces a leader election provides along with the corresponding methods.

## Implementation

The solution is based on a lease acquisition and renewal mechanism.
When a node starts, it spawns a separate task that creates a lease and then calls the campaign method with the lease ID as a parameter.
If multiple nodes are competing to be elected as the leader, only one node is selected, while the others are blocked by the campaign method. The blocked nodes remain in this state until the current leader either relinquishes leadership or fails, e.g., due to network partitioning at which point the lease expires. Once the lease expires, the blocked nodes are awakened and will retry to acquire leadership.
Once a node acquires leadership, it maintains its status by periodically sending keep-alive requests to the etcd server to renew the lease. This ensures that the lease doesn't expire while the other nodes remain blocked and wait for the next leader election phase.

```rust
async fn participate_in_election(args: &Args) -> Result<(), Error> {
    let mut client = connect(args).await?;

    loop {
        let resp = client.lease_grant(TTL, None).await?;
        let lease_id = resp.id();

        info!("Starting a new campaign.");
        let resp = client
            .campaign(ELECTION_NAME, args.node.clone(), lease_id)
            .await?;
        let leader_key = resp
            .leader()
            .ok_or(Error::ElectError("Failed to retrieve the leader.".into()))?;
        info!("🥳 I am the leader ({})", args.node);

        if let Ok((mut keeper, _)) = client.lease_keep_alive(lease_id).await {
            loop {
                info!("⏰ Keeping alive the lease {}...", leader_key.key_str()?);
                keeper.keep_alive().await?;
                time::sleep(Duration::from_secs(7)).await;
            }
        } else {
            error!("Failed to keep lease alive. Re-campaigning.");
        }
    }
}
```
At the same time, nodes observe election proclamations in order, as made by the elected leaders, to stay aware of the current leader.

```rust
async fn observe_election(args: &Args, state: Arc<RwLock<State>>) -> Result<(), Error> {
    let mut client = connect(args).await?;

    let mut msg = client.observe(ELECTION_NAME).await?;
    loop {
        if let Some(resp) = msg.message().await? {
            let kv = resp
                .kv()
                .ok_or(Error::WatchError("Unable to retrieve key/value".into()))?;
            let key = kv.key_str()?;
            let val = kv.value_str()?;

            let mut st = state.write().await;
            (*st).is_leader = val == args.node;
            info!(
                "🟢 Current leader is {val} with key {key}, node.is_leader={}",
                (*st).is_leader
            );
        }
    }
}
```

You can find the full implementation [here](https://github.com/fade2black/node)

To launch the entire process for three nodes, you can start each node in a separate console window as follows

```bash
% node --node node1 --host 127.0.0.1 --port 50686
```

```bash
% node --node node2 --host 127.0.0.1 --port 50686
```

```bash
% node --node node3 --host 127.0.0.1 --port 50686
```

where the `host` and `port` refer to the URL and port number of the etcd server used to interact with the etcd cluster.


## Conclusion
eader election is a critical component in distributed systems, ensuring high availability and fault tolerance. The implementation discussed here serves as a starting point, but to build a production-ready leader election system, it’s important to consider additional nuances specific to your use case. For example, factors like network partitioning, node failure handling, and consistency guarantees should be carefully addressed.
For a deeper dive into best practices, AWS provides insights and best-practice [recommendations](https://aws.amazon.com/builders-library/leader-election-in-distributed-systems/#:~:text=Leader%20election%20is%20the%20simple,all%20requests%20in%20the%20system.).
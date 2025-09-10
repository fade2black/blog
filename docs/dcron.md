<div class="meta-data">16 feb 20205 </div>

# Implementing Distributed Cron Jobs with etcd

## Introduction

In this article I’ll walk you through a project I built using Rust that acts as a distributed cron scheduler. This project allows users to add, list, and remove cron jobs via a set of RESTful endpoints. The system continuously monitors the current time, ensuring that scheduled tasks are executed at the right moment. The advantage of being distributed is the increased availability of the service. If one node goes down, other nodes can continue processing requests and monitoring jobs.

## The Inner Workings

``` mermaid
flowchart
    n1["etcd"] <--> n2["Monitor"] & n3["http server"]
    n4[user] --> n3["http server"]
    n1@{ shape: cyl}
```

The distributed cron starts by spawning a thread monitoring current time and http server for processing RESTful requests.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    tracing_subscriber::fmt::init();

    let result = tokio::try_join!(daemon::monitor(), run_http_server());

    if let Err(err) = result {
        error!("{err}");
    }

    info!("Terminating...");

    Ok(())
}
```

For processing endpoint requests, I’ve used the Tokio Axum server. Cron jobs are stored in an etcd server as key-value pairs. When a new `add job` request is received, the application parses the submitted cron expression and command to execute. It then generates a unique key string and creates a JSON object with three fields to store in etcd: `pattern`, `next`, and `command`.

- `Pattern`: holds the cron expression.
- `Next`: stores the next scheduled execution time (the upcoming time when the command should run).
- `Command`: holds the command that will be executed.

This structure ensures that all cron jobs are stored consistently, and the use of the etcd server enables mutual access and synchronisation across multiple nodes.

```rust
async fn store_cron_job(json_str: &str) -> Result<(), Box<dyn Error>> {
    let mut client = Client::new().await?;

    let lock_key = client.lock().await?;
    let key = generate_unique_key("cron");
    client.store_cron_job(&key, json_str).await?;
    client.unlock(&lock_key).await?;

    Ok(())
}
```

where `json_str` looks, for example, like

```json
{
  "pattern": "* * * * *",
  "next": "2025-02-16 14:14:00 +01:00",
  "command": "echo hello, world"
}
```

When the server starts, it spawns a separate thread that periodically checks the current time. This thread locks access to the cron jobs, reads all the jobs from etcd, and processes them one by one. For each job, it checks the "next" occurrence time. If the scheduled time is less than the current time, the server spawns a separate child process to run the job’s command, then updates the job's "next" occurrence time accordingly. After processing, the server sleeps for 3 seconds before repeating the process. 

```rust
pub async fn monitor() -> Result<(), Box<dyn Error>> {
    let mut client = Client::new().await?;

    loop {
        let lock_key = client.lock().await?;
        let kvs = client.get_cron_jobs().await?;

        for kv in kvs {
            process(&mut client, &kv.0, &kv.1).await?;
        }
        client.unlock(&lock_key).await?;

        time::sleep(time::Duration::from_secs(SLEEP_TIME)).await;
    }
}
```

The role of etcd in this application is to provide exclusive access to the cron jobs by offering a distributed locking mechanism. This is crucial because only one node should run a particular job at any given time. Both the monitoring thread and the HTTP actions that manipulate cron jobs stored in etcd lock a mutex before entering the critical section. Once the task is completed, the mutex is unlocked. For example,

```rust
pub async fn delete(key: &str) -> Result<(), Box<dyn Error>> {
    let mut client = Client::new().await?;

    let lock_key = client.lock().await?;
    client.delete_cron_job(key).await?;
    client.unlock(&lock_key).await?;

    Ok(())
}
```

## Usage

The easiest way to run the app is to build it and execute it locally. Assuming etcd is running locally, for example start three separate processes to emulate three different nodes. Be sure to set a unique port for each Axum server process and specify the `ETCD_URL` environment variable to match the etcd server URL.

```bash
export ETCD_URL=http:://1.2.3.4:1234
./dcron
```

Another option is to create a Kubernetes deployment with three replicas of the server, along with a service that exposes a single IP address for interacting with the dcron server.

## Conclusion

While this simple project is by no means a replacement for established frameworks like [Kubernetes' CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/), it served as an exploration of Rust's asynchronous features and the functionality of the etcd server. By building this distributed cron scheduler, I gained valuable hands-on experience with these tools and saw how they can be leveraged in real-world applications.
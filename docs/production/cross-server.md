# Cross-Server Deployment

SnerdMQ natively supports distributed execution across multiple servers (e.g., a cluster of EC2 instances, Droplets, or Kubernetes pods) while maintaining a unified, sharded queue.

Because SnerdMQ relies entirely on OS-level file locking and atomic file operations rather than network protocols, **you must use a shared, POSIX-compliant distributed filesystem.**

## Filesystem Compatibility

The SnerdMQ membership layer and shard lock guarantees depend on two features:
1. True POSIX `flock(2)` support across the network.
2. Atomic `rename(2)` for crash-safe metadata writes.

### Supported Filesystems
- ✅ **AWS EFS** (Elastic File System - uses NFSv4)
- ✅ **GlusterFS**
- ✅ **Lustre**
- ✅ **GPFS**

### Unsupported (Do Not Use)
- ❌ **NFS v3**: Uses advisory NLM locking which is notoriously flaky in split-brain scenarios and can lead to silent lock overlap (meaning two daemons could process the same queue shard).
- ❌ **SMB/CIFS (Windows shares)**: Linux clients mounting SMB shares often emulate `flock` poorly and inconsistently.
- ❌ **FUSE-mounted Object Storage (e.g., S3FS, Goofys)**: S3 is not a POSIX filesystem. It does not support true locking or atomic directory renames. Pointing SnerdMQ at an S3FS mount will instantly corrupt the queue.

## Clock Skew Tolerance (NTP)

Cross-server deployments rely on the `membership.json` lease protocol to handle crashes. 
- The owning daemon writes a `lease_expiry` timestamp.
- A challenger daemon takes over the shard if `challenger_current_time > lease_expiry + skew_margin`.

Because the challenger and the owner are on different servers, **their system clocks must be synchronized.**

### Requirements
1. **You MUST run an NTP daemon** (like `chrony` or `systemd-timesyncd`) on all servers in the cluster.
2. The drift between any two servers must be less than the configured `SNERD_CLOCK_SKEW_MARGIN`.

By default, the skew margin is **5 seconds**. 
If Server A's clock runs 6 seconds faster than Server B, Server A might incorrectly conclude that Server B's lease has expired, causing a split-brain where both daemons try to claim the shard (prevented gracefully by the secondary `flock`, but it forces Server B into a `Paused` state).

If you are running in an environment with high clock drift, you can widen the margin:

```bash
export SNERD_CLOCK_SKEW_MARGIN=15
```
*Note: A higher margin increases the time it takes for the cluster to detect and recover from a daemon crash.*

## At-Least-Once Delivery & Idempotency

When you scale across multiple servers, the probability of a node crash (kernel panic, hardware failure, spot instance termination) increases.

When a server crashes ungracefully, any jobs that were actively executing on its shards are abandoned mid-flight. After the lease expires, another server will claim the lapsed shard and **re-execute those abandoned jobs**.

!!! danger "Strict Idempotency Mandate"
    Because crash-takeovers are an expected event in a cross-server deployment, SnerdMQ provides an **at-least-once** delivery guarantee. 
    **Your job handlers MUST be idempotent.** If a job is executed twice, it must safely no-op or overwrite the previous attempt without corrupting your data or double-charging a customer.

## Deployment Example (Docker Compose / EFS)

If you are running multiple containers across different EC2 instances, you can mount the shared EFS volume to the same path in every container.

```yaml
services:
  worker:
    image: my-worker-app:latest
    environment:
      # Each instance claims up to 2 shards from the shared queue
      - SNERD_MAX_SHARDS=2 
    volumes:
      # Mount the EFS volume containing the .snerdata directory
      - /mnt/efs/snerd-queue:/app/.snerdata
```

With this setup:
1. Run `snerdmq add-shards 10 /mnt/efs/snerd-queue` once to provision the shards.
2. Spin up 5 EC2 instances. 
3. The 5 instances will automatically negotiate via the EFS mount, each claiming 2 shards.
4. If an instance is terminated, the remaining 4 instances will detect the lapsed leases and take over the orphaned shards.

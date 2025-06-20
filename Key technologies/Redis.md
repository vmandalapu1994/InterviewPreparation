# Redis: Key Concepts & Operations

---

## Table of Contents
1. [Does Redis Distribute Keys Using Consistent Hashing?](#does-redis-distribute-keys-using-consistent-hashing)
2. [How Redis Distributes Keys](#how-redis-distributes-keys)
3. [Consistent Hashing vs Hash Slots](#consistent-hashing-vs-hash-slots)
4. [Can Redis Use Consistent Hashing?](#can-redis-use-consistent-hashing)
5. [Redis Cluster Key Distribution Example](#redis-cluster-key-distribution-example)
6. [Consistent Hashing vs Redis Hash Slots](#consistent-hashing-vs-redis-hash-slots)
7. [Code Example: Hash Slot Calculation](#code-example-hash-slot-calculation)
8. [Summary Table: Hash Slots vs Consistent Hashing](#summary-table-hash-slots-vs-consistent-hashing)
9. [Redis Replication](#redis-replication)
10. [Redis Modes](#redis-modes)
11. [Configuring Sentinel Mode](#configuring-sentinel-mode)
12. [Redis Cluster: Node Addition & Failure](#redis-cluster-node-addition--failure)
13. [Lua Scripts and Atomicity in Redis Cluster](#lua-scripts-and-atomicity-in-redis-cluster)
14. [Colocating Keys Using Hash Tags](#colocating-keys-using-hash-tags)
15. [Lua Scripts: Keys in Different Slots but Same Node?](#lua-scripts-keys-in-different-slots-but-same-node)

---

## Does Redis Distribute Keys Using Consistent Hashing?

**Q:** Does Redis distribute the keys using consistent hashing?

**A:** No, Redis does not use consistent hashing by default. Instead, Redis uses a slot-based partitioning mechanism for distributing keys across a cluster.

---

## How Redis Distributes Keys

Redis Cluster uses **hash slot partitioning** (not consistent hashing):

- The key space is divided into **16,384 hash slots**.
- When a key is stored, Redis computes a CRC16 hash of the key and takes the result modulo 16,384 to find the slot:

```python
hash_slot = CRC16(key) % 16384
```

- Each Redis node in the cluster is assigned a range of slots.
- A key is stored in the node responsible for the slot it hashes to.

---

## Consistent Hashing vs Hash Slots

| Feature            | Redis Hash Slots         | Consistent Hashing           |
|--------------------|-------------------------|------------------------------|
| Key Distribution   | Fixed number of slots   | Keys placed on a hash ring   |
| Node Assignment    | Slots mapped to nodes   | Nodes placed on the ring     |
| Scalability        | Manual slot reassignment| Nodes can join/leave smoothly|
| Flexibility        | Less flexible           | More dynamic for scaling     |
| Used in            | Redis Cluster           | Cassandra, DynamoDB          |

---

## Can Redis Use Consistent Hashing?

While Redis itself doesn't use consistent hashing internally, client-side sharding libraries (e.g., in Jedis or Lettuce) may implement consistent hashing in non-clustered environments to balance load across multiple Redis instances.

---

## Redis Cluster Key Distribution Example

**Example:**

- **Total Slots:** 0 to 16383 (16,384 slots)
- **3 Redis nodes:**
  - Node A: slots 0 - 5460
  - Node B: slots 5461 - 10922
  - Node C: slots 10923 - 16383

When a key like `"user:123"` comes in:
1. Redis calculates `CRC16("user:123") = 5783`
2. `5783 % 16384 = 5783`
3. Since 5783 falls in Node B's slot range, the key is stored in Node B

**Visual:**

```
|-----------|-----------|-----------|
|  Node A   |  Node B   |  Node C   |
|  0-5460   | 5461-10922|10923-16383|
             ↑
        "user:123" maps here
```

---

## Consistent Hashing vs Redis Hash Slots

**Consistent Hashing:**
- All nodes and keys are placed on a circular hash ring.
- A key is assigned to the first node clockwise from its hash position.
- When a node is added or removed, only nearby keys are remapped — helps in elastic scaling.

**Redis Hash Slots:**
- Has fixed 16,384 slots.
- When a new node joins, you must manually or programmatically reassign slots to it.
- This makes Redis predictable and fast, but less flexible for dynamic scaling.

---

## Code Example: Hash Slot Calculation

```python
import binascii

def redis_hash_slot(key: str) -> int:
    crc16 = binascii.crc_hqx(key.encode(), 0)
    return crc16 % 16384

print(redis_hash_slot("user:123"))  # E.g., prints 5783
```

---

## Summary Table: Hash Slots vs Consistent Hashing

| Aspect              | Redis Hash Slots | Consistent Hashing         |
|---------------------|------------------|----------------------------|
| Used By             | Redis Cluster    | Cassandra, DynamoDB, etc.  |
| Number of Partitions| Fixed (16,384)   | Dynamic (based on nodes)   |
| Rebalancing Effort  | Manual           | Automatic                  |
| Elastic Scaling     | Harder           | Easier                     |
| Client Control      | Less Needed      | Often needed               |

---

## Redis Replication

**Q:** Does Redis use replication by default to improve availability?

**A:** Yes, Redis supports replication, but it is not enabled by default in the standalone setup.

### How Redis Replication Works

- Redis follows a **master-replica (primary-secondary) model**:
  - One primary (master) node handles writes.
  - One or more replica (slave) nodes copy data from the master and handle read-only traffic.

#### When is replication used?
- **High Availability:** If the master fails, a replica can be promoted (automatically with Redis Sentinel or Redis Cluster).
- **Scalability:** Reads can be offloaded to replicas.

#### How to enable replication

In a Redis config file (`redis.conf`), on the replica node, add:

```conf
replicaof <master_ip> <master_port>
# Example:
replicaof 192.168.1.100 6379
```

Or at runtime:

```
127.0.0.1:6379> replicaof 192.168.1.100 6379
```

#### Notes
- Replication is asynchronous by default (replica may lag slightly).
- Redis Cluster mode uses both replication and partitioning.
- Redis Sentinel automates failover and replica promotion.

#### Summary Table

| Feature                | Enabled by Default? | Purpose                        |
|------------------------|--------------------|--------------------------------|
| Replication            | ❌ No              | Adds read scaling & redundancy |
| Sentinel for failover  | ❌ No              | Detects failures, promotes replica |
| Redis Cluster replication | ✅ Yes (by design) | Ensures high availability      |

---

## Redis Modes

Redis supports multiple modes of operation, each catering to different scalability, availability, and use case requirements.

### 1. Standalone Mode (Default)
- ✅ Default installation
- A single Redis instance running on a machine
- No replication, no clustering

**Use Case:**
- Development
- Simple caching or session storage

### 2. Master-Replica Mode (Replication)
- One master (primary) node handles all writes
- One or more replica (secondary) nodes replicate data from the master

**Features:**
- Asynchronous replication
- Read-scaling possible via replicas
- Manual or Sentinel-based failover

**Use Case:**
- High availability
- Read-heavy applications

### 3. Redis Sentinel Mode
- A monitoring system that provides:
  - Automatic failover
  - Monitoring of Redis instances
  - Notification and reconfiguration during failures

**Features:**
- Promotes a replica to master if the master fails
- Works with Master-Replica setup

**Use Case:**
- Production systems needing high availability without clustering

### 4. Redis Cluster Mode
- Supports automatic sharding and horizontal scaling
- Data is split across multiple masters
- Each master can have replicas

**Features:**
- No need for manual partitioning
- Scales writes and reads
- Built-in failover support
- Uses hash slot partitioning (16,384 slots)

**Use Case:**
- Large datasets
- High throughput systems
- Geo-distributed apps

### 5. Redis with Persistence Modes
(Not a separate deployment mode but a data durability feature)
- RDB (Snapshotting) – Saves snapshots at intervals
- AOF (Append-Only File) – Logs every write operation
- Hybrid – Uses both for balance of durability and performance

#### Summary Table

| Mode              | Description                    | Use Case                        |
|-------------------|-------------------------------|----------------------------------|
| Standalone        | Single Redis instance          | Dev, simple apps                 |
| Master-Replica    | One master, multiple replicas  | Read scaling, high availability  |
| Sentinel          | Monitors & manages failover    | Auto failover for master-replica |
| Cluster           | Sharded data across nodes      | High-scale distributed systems   |
| Persistence Modes | RDB, AOF, both                | Data durability and recovery     |

---

## Configuring Sentinel Mode

**Q:** How to configure Sentinel mode? It is on top of master-replica deployment mode, right?

**A:** Yes, Redis Sentinel is a monitoring and failover system built on top of the master-replica setup. It monitors Redis instances and automatically promotes a replica to master if the master fails.

### Example 3-Node Setup
- 1 Master
- 1 Replica
- 1 Sentinel (for production, 3+ sentinels are recommended)

#### Folder Structure
```
redis-sentinel-demo/
├── redis-master.conf
├── redis-replica.conf
├── sentinel.conf
```

#### Step-by-Step Setup

**1. Master Configuration (`redis-master.conf`)**
```conf
port 6379
dir ./
```

**2. Replica Configuration (`redis-replica.conf`)**
```conf
port 6380
dir ./
replicaof 127.0.0.1 6379
```
Replace IP with the actual master IP in real setups.

**3. Sentinel Configuration (`sentinel.conf`)**
```conf
port 26379
dir ./
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```
- `mymaster` – name of the master
- `127.0.0.1 6379` – IP and port of master
- `2` – quorum (how many sentinels must agree master is down)

#### Start Redis Instances
```sh
# Terminal 1 - start master
redis-server redis-master.conf

# Terminal 2 - start replica
redis-server redis-replica.conf

# Terminal 3 - start sentinel
redis-sentinel sentinel.conf
```

Sentinel will start monitoring the master. If the master goes down, the sentinel will promote the replica to master.

#### Test Failover
1. Kill the master:
   ```sh
   ps aux | grep redis
   kill <master-pid>
   ```
2. Sentinel will detect failure and promote the replica.
3. Once master is back, it becomes a replica of the new master.

#### Tips
- For production, always use 3 or 5 sentinels to ensure quorum.
- Sentinels also act as configuration providers. Clients can use them to discover current master.
- You can query sentinel status:
  ```sh
  redis-cli -p 26379
  > SENTINEL get-master-addr-by-name mymaster
  ```

#### Example `sentinel.conf` File
```conf
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

#### Explanation of Each Setting
- `port 26379`: Default port on which the Sentinel process listens.
- `sentinel monitor <master-name> <ip> <port> <quorum>`: Tells Sentinel which master to monitor and quorum size.
- `sentinel down-after-milliseconds <master-name> <time-in-ms>`: Time after which master is considered down.
- `sentinel failover-timeout <master-name> <timeout-ms>`: Max time to wait for failover or re-election.
- `sentinel parallel-syncs <master-name> <N>`: Number of replicas syncing in parallel.

**Bonus Settings:**
- Logging: `logfile "sentinel.log"`
- Persistence: Sentinel updates its own state in a file called `sentinel-state-*.conf` (do not edit manually).

#### Summary Table

| Setting                        | Description                           |
|--------------------------------|---------------------------------------|
| port 26379                     | Port on which Sentinel runs           |
| sentinel monitor               | Tells Sentinel which master to monitor|
| sentinel down-after-milliseconds| Time after which master is considered down |
| sentinel failover-timeout       | Max time to wait for failover or re-election |
| sentinel parallel-syncs         | Number of replicas syncing in parallel|

#### What Happens During Failover?
1. Sentinel loses contact with the master.
2. Quorum (2 or more Sentinels) agrees master is down.
3. One Sentinel is elected leader.
4. It promotes a replica to master.
5. Other replicas are reconfigured to point to new master.
6. Clients using Sentinel discover new master automatically.

---

## Redis Cluster: Node Addition & Failure

**Q:** Since Redis uses fixed slot mapping, new node additions require manual data transfer and mapping, right? Also, in case one node goes down then all that data is lost, right?

**A:**

### Fixed Slot Mapping
- Redis Cluster partitions data across nodes using 16,384 hash slots.
- Every key is hashed (using CRC16) and mapped to a slot.
- Each node is responsible for a subset of those slots.

### Node Addition Requires Manual Slot Migration
- When you add a new node:
  - You must manually reassign slots using `redis-cli` (or use a tool like `redis-trib.rb` or `redis-cli --cluster`).
  - You must migrate keys from existing nodes to the new node using `CLUSTER SETSLOT`, `MIGRATE`, etc.

**Example:**
```sh
# Add 4096 slots to the new node
redis-cli --cluster add-node <new-node-ip>:<port> <existing-node-ip>:<port>
redis-cli --cluster reshard <existing-node-ip>:<port>
```
There is no automatic rebalance — it's manual (unless you're using something like Redis Enterprise).

### Node Failure
- **Without replicas:**
  - All keys in that node's hash slots are inaccessible.
  - Cluster goes into fail state if the down node had slots assigned.
- **With replicas:**
  - Redis Cluster can promote a replica automatically.
  - This provides high availability (though not strict consistency).

**Example node-slot mapping with replica:**
- Node A (master) → slots 0–5460
- Node B (replica of A)

If Node A dies, Node B is promoted to master — and cluster continues working.

#### Best Practice Summary

| Concern           | Risk                  | Mitigation                        |
|-------------------|----------------------|-----------------------------------|
| New node addition | Manual and error-prone| Automate with tooling              |
| Node failure      | Data loss if no replica| Always have at least 1 replica per node |
| Rebalancing       | Not automatic         | Use scripts or automation          |
| Operational overhead | High in open source | Use managed Redis (e.g., AWS ElastiCache, Redis Enterprise) |

---

## Lua Scripts and Atomicity in Redis Cluster

**Q:** As per my understanding, Lua scripts are run atomically. Suppose it contains 2 key updates which are different nodes, in this case Redis doesn't allow any other operations to run in these 2 nodes as long as Lua script is running? If yes, it is a bottleneck for performance, right?

**A:**

Excellent question — you're touching on an important nuance in Redis Cluster behavior with Lua scripts.

---

### Atomicity of Lua Scripts

- In **standalone Redis**, atomicity means:
  - The entire script runs as a single operation, without interruption.

- In **Redis Cluster**, there are key limitations and caveats:

#### 🚫 Redis Cluster doesn't allow cross-slot Lua script execution
- If a Lua script accesses keys that are mapped to different hash slots (i.e., different nodes):
  - Redis refuses to run it.
  - It returns an error like:

```
(error) CROSSSLOT Keys in request don't hash to the same slot
```

#### 🚦 Why?
- Redis Cluster cannot ensure atomicity across nodes. Lua script execution is atomic only on a single node.
- Redis avoids this entirely by disallowing scripts that touch multiple hash slots.

#### ✅ Solution — Use hash tags to force co-location
- You can force multiple keys to be stored in the same hash slot using curly braces in the key:

```
SET user:{123}:name "Alice"
SET user:{123}:email "alice@example.com"
```

- Both keys hash to slot for `{123}` → ✅ can be accessed atomically in Lua.

#### ❓ Are other commands blocked on the node during Lua execution?
- Yes, on a single node:
  - While a Lua script is running, Redis queues other incoming commands.
  - Other clients can't interrupt it.
- 🔁 But only for that node. Other nodes continue serving requests unaffected.

#### 🚨 Performance Bottleneck?
- If your script is long-running, it can block that node and cause latency.
- Redis recommends keeping Lua scripts short and efficient.

#### Summary Table

| Aspect                | Redis Cluster Behavior                                 |
|-----------------------|-------------------------------------------------------|
| Atomic Lua execution  | ✅ Only if all keys are in the same slot (node)        |
| Cross-slot access     | ❌ Disallowed for scripts                              |
| Blocking behavior     | 🔒 Blocks that node only, not the cluster              |
| Performance concern   | 🚨 Yes, long scripts may hurt latency for that node    |
| Best practice         | ✅ Use key tags to co-locate data, keep scripts short  |

---

## Colocating Keys Using Hash Tags

**Q:** Please explain colocating keys using hash tags.

**A:**

Let's walk through co-locating keys in Redis Cluster using hash tags — a technique crucial when you want to:
- Use Lua scripts, transactions, or pipelining across multiple keys
- Ensure those keys live on the same Redis node

### 🔧 Why Co-location Is Needed in Redis Cluster
- Redis Cluster distributes keys across 16,384 hash slots using CRC16. Each node manages a subset of slots.
- If two keys are hashed to different slots, they end up on different nodes — and you can't access them together atomically.

### 🧩 What Are Hash Tags?
- A hash tag is the part of the key enclosed in `{curly_braces}`.
- Only the string inside the braces is used to compute the hash slot.

**Example:**
```
SET order:{123}:status "shipped"
SET order:{123}:amount "99.00"
SET order:{123}:items "2"
```
All of these keys hash to the same slot — the one based on `{123}` — so they are stored on the same node.

This lets you:
- Use Lua scripts
- Run multi-key operations
- Improve access locality

### ❌ What if You Don't Use Hash Tags?
```
SET order:123:status "shipped"
SET order:123:amount "99.00"
SET order:123:items "2"
```
- Each key's full name is hashed.
- They may end up on different nodes — which breaks atomicity for Lua scripts or multi-key operations.

### ✅ Best Practices for Hash Tags

| Practice         | Description                                 |
|------------------|---------------------------------------------|
| `{user_id}` as tag | Use a stable identifier (user, order, session) |
| Tag early        | Design your schema with tagging in mind      |
| Group related keys | Keep logically related keys (like a user profile) together |

### 🧪 Test It with redis-cli
```
CLUSTER KEYSLOT order:{123}:status
CLUSTER KEYSLOT order:{123}:amount
```
They'll both return the same slot number.

### 🚫 Don't Abuse It
- Too many keys with the same tag can lead to hotspots — one node doing all the work. Balance carefully.

#### 🔁 Summary Table

| Feature                | With Hash Tag | Without Hash Tag |
|------------------------|:-------------:|:---------------:|
| Keys colocated         | ✅ Yes        | ❌ No            |
| Lua scripts across keys| ✅ Allowed    | ❌ Fails with CROSSSLOT |
| Multi-key ops          | ✅ Possible   | ❌ Not allowed   |
| Slot calculated on     | Contents inside `{}` | Entire key |

---

## Lua Scripts: Keys in Different Slots but Same Node?

**Q:** What about keys hashing to different slots but on the same node? Will Redis allow Lua script like that?

**A:**

Great follow-up! The short answer is:

**No, Redis Cluster does NOT allow this.**

---

### 🔥 TL;DR
- Even if keys hash to different slots but end up on the same node, Redis Cluster will **NOT** allow a Lua script to operate on them together.

#### ❓ Why?
- Redis Cluster enforces that all keys involved in a Lua script or multi-key operation must be in the same hash slot — not just the same node.
- This is a design choice, not a technical limitation. Redis does not check node locality, it checks hash slot consistency to ensure correctness and future-proofing in distributed deployments.

#### ❌ Example – Not Allowed
```sh
-- These keys hash to different slots
eval "return redis.call('GET', KEYS[1]) .. redis.call('GET', KEYS[2])" 2 user:100 name:200
```
Even if both keys are accidentally stored on the same physical node, Redis Cluster will throw:
```
CROSSSLOT Keys in request don't hash to the same slot
```

#### ✅ How to Make It Work?
- Use hash tags to force both keys to hash to the same slot:

```
SET user:{100}:name "Alice"
SET user:{100}:email "alice@example.com"
```
And in Lua:
```
EVAL "return redis.call('GET', KEYS[1]) .. redis.call('GET', KEYS[2])" 2 user:{100}:name user:{100}:email
```
✔️ This will work because both keys share the same hash tag and thus same slot.

#### 🔐 Why This Restriction?
- Redis Cluster doesn't track node-to-key mappings in scripts at runtime.
- Even if today both keys are on the same node, resharding or failover may place them on different nodes in the future.
- Enforcing the hash slot constraint ensures predictability and robustness.

#### ✅ Summary Table

| Case                                 | Lua Script Allowed? | Why                        |
|---------------------------------------|:-------------------:|----------------------------|
| Keys in same slot (via hash tags)     | ✅ Yes              | Designed for atomic ops    |
| Keys in different slots, same node    | ❌ No               | Violates slot consistency  |
| Keys in different slots, diff. nodes  | ❌ No               | Same reason                |

---

*Let me know if you'd like to simulate this using Redis CLI or Lua script in Java using RedisTemplate.*